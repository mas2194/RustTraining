# 15. クレートのアーキテクチャと API 設計 🟡

> **学ぶこと:**
> - モジュール構成の慣例と再エクスポート戦略
> - 洗練されたクレートのための公開 API 設計チェックリスト
> - 人間工学的なパラメータパターン: `impl Into`、`AsRef`、`Cow`
> - `TryFrom` とバリデーション済み型による「検証するな、パースせよ（Parse, don't validate）」
> - フィーチャーフラグ、条件付きコンパイル、ワークスペース構成

## モジュール構成の慣例

```text
my_crate/
├── Cargo.toml
├── src/
│   ├── lib.rs          # クレートルート — 再エクスポートと公開 API
│   ├── config.rs       # 機能モジュール
│   ├── parser/         # サブモジュールを持つ複雑なモジュール
│   │   ├── mod.rs      # または親レベルの parser.rs（Rust 2018+）
│   │   ├── lexer.rs
│   │   └── ast.rs
│   ├── error.rs        # エラー型
│   └── utils.rs        # 内部ヘルパー (pub(crate))
├── tests/
│   └── integration.rs  # 統合テスト
├── benches/
│   └── perf.rs         # ベンチマーク
└── examples/
    └── basic.rs        # cargo run --example basic
```

```rust
// lib.rs — 再エクスポートで公開 API を整理・キュレーションする:
mod config;
mod error;
mod parser;
mod utils;

// ユーザーが必要とするものを再エクスポート:
pub use config::Config;
pub use error::Error;
pub use parser::Parser;

// 公開型はクレートルートに配置 — 利用者は以下のように記述可能:
// use my_crate::Config;
// 以下のようには書かせない: use my_crate::config::Config;
```

**可視性修飾子**:

| 修飾子 | 公開範囲 |
|----------|-----------|
| `pub` | すべての場所（完全公開） |
| `pub(crate)` | このクレート内のみ |
| `pub(super)` | 親モジュール |
| `pub(in path)` | 指定された祖先モジュール |
| (なし) | 現在のモジュールとその子モジュール |

### 公開 API 設計チェックリスト

1. **参照を受け取り、所有値を返す** — `fn process(input: &str) -> String`
2. **パラメータには `impl Trait` を使用する** — より簡潔なシグネチャのために `fn read<R: Read>(r: R)` ではなく `fn read(r: impl Read)` を用いる
3. **`panic!` ではなく `Result` を返す** — エラー処理方法は呼び出し側に委ねる
4. **標準トレイトを実装する** — `Debug`、`Display`、`Clone`、`Default`、`From`/`Into`
5. **不正な状態を表現不可能にする** — 型状態（タイプステート）やニュータイプを活用する
6. **複雑な設定には Builder パターンを採用する** — 必須フィールドがある場合は型状態パターンを併用
7. **ユーザーに実装させたくないトレイトは封印（Seal）する** — `pub trait Sealed: private::Sealed {}`
8. **型や関数に `#[must_use]` を付与する** — 重要な `Result`、ガード、値が暗黙のうちに無視・破棄されるのを防止する。戻り値を無視することがほぼ確実にバグとなる任意の型に適用する:
   ```rust
   #[must_use = "ガードをドロップすると直ちにロックが解除されます"]
   pub struct LockGuard<'a, T> { /* ... */ }

   #[must_use]
   pub fn validate(input: &str) -> Result<ValidInput, ValidationError> { /* ... */ }
   ```

```rust
// 封印トレイト（Sealed trait）パターン — ユーザーは利用できるが実装はできない:
mod private {
    pub trait Sealed {}
}

pub trait DatabaseDriver: private::Sealed {
    fn connect(&self, url: &str) -> Connection;
}

// このクレート内の型のみが Sealed を実装可能 → 自クレートのみが DatabaseDriver を実装可能
pub struct PostgresDriver;
impl private::Sealed for PostgresDriver {}
impl DatabaseDriver for PostgresDriver {
    fn connect(&self, url: &str) -> Connection { /* ... */ }
}
```

> **`#[non_exhaustive]`** — 公開 enum や struct に付与することで、将来バリアントやフィールドが追加されても破壊的変更（breaking change）にならないようにします。下流クレートは match 文でワイルドカードアーム（`_ =>`）を使用する必要があり、構造体リテラル構文で直接型を構築することはできなくなります:
> ```rust
> #[non_exhaustive]
> pub enum DiagError {
>     Timeout,
>     HardwareFault,
>     // 将来のリリースで新しいバリアントを追加してもセマンティックバージョニング上の破壊的変更にはならない
> }
> ```

### 人間工学的なパラメータパターン — `impl Into`、`AsRef`、`Cow`

Rust における極めて効果的な API パターンの一つが、関数のパラメータにおいて**最も汎用的な型**を受け入れることです。これにより、呼び出し側は呼び出し箇所ごとに `.to_string()` や `&*s`、`.as_ref()` などを繰り返し書く必要がなくなります。これは「受け入れるものには寛容であれ（be liberal in what you accept）」という堅牢性の原則の Rust 版です。

#### `impl Into<T>` — 変換可能なあらゆるものを受け入れる

```rust
// ❌ 摩擦（Friction）: 呼び出し側が手動で変換する必要がある
fn connect(host: String, port: u16) -> Connection {
    // ...
}
connect("localhost".to_string(), 5432);  // 煩わしい .to_string()
connect(hostname.clone(), 5432);          // すでに String を持っている場合は不要な clone

// ✅ 人間工学的（Ergonomic）: String に変換可能なあらゆるものを受け入れる
fn connect(host: impl Into<String>, port: u16) -> Connection {
    let host = host.into();  // 関数内部で一度だけ変換
    // ...
}
connect("localhost", 5432);     // &str — 呼び出し側の負担ゼロ
connect(hostname, 5432);        // String — ムーブされ、クローン不要
```

これは、Rust の `From`/`Into` トレイトペアが包括的な変換を提供しているために機能します。`impl Into<T>` を受け入れることは、「`T` に変換する方法を知っている任意のものを受け取る」と宣言することを意味します。

#### `AsRef<T>` — 参照として借用する

`AsRef<T>` は `Into<T>` の借用版です。所有権を取得せず、データを*読み取る*だけでよい場合に使用します:

```rust
use std::path::Path;

// ❌ 呼び出し側に &Path への変換を強制する
fn file_exists(path: &Path) -> bool {
    path.exists()
}
file_exists(Path::new("/tmp/test.txt"));  // 扱いにくい

// ✅ &Path として振る舞えるあらゆるものを受け入れる
fn file_exists(path: impl AsRef<Path>) -> bool {
    path.as_ref().exists()
}
file_exists("/tmp/test.txt");                    // &str ✅
file_exists(String::from("/tmp/test.txt"));      // String ✅
file_exists(Path::new("/tmp/test.txt"));         // &Path ✅
file_exists(PathBuf::from("/tmp/test.txt"));     // PathBuf ✅

// 文字列ライクなパラメータでも同様のパターン:
fn log_message(msg: impl AsRef<str>) {
    println!("[LOG] {}", msg.as_ref());
}
log_message("hello");                    // &str ✅
log_message(String::from("hello"));      // String ✅
```

#### `Cow<T>` — 書き込み時クローン（Clone on Write）

`Cow<'a, T>` (Clone on Write) は、変更が必要になるまでメモリ割り当てを遅延させます。借用された `&T` または所有された `T::Owned` のいずれかを保持します。これは、ほとんどの呼び出しでデータの変更が不要な場合に最適です:

```rust
use std::borrow::Cow;

/// 診断メッセージを正規化する — 変更が必要な場合のみメモリ割り当てを行う
fn normalize_message(msg: &str) -> Cow<'_, str> {
    if msg.contains('\t') || msg.contains('\r') {
        // メモリ割り当てが必要 — 内容を変更する必要がある
        Cow::Owned(msg.replace('\t', "    ").replace('\r', ""))
    } else {
        // メモリ割り当てなし — 元のデータをそのまま借用
        Cow::Borrowed(msg)
    }
}

// ほとんどのメッセージはメモリ割り当てなしでそのまま通過:
let clean = normalize_message("All tests passed");          // Borrowed — コストゼロ
let fixed = normalize_message("Error:\tfailed\r\n");        // Owned — メモリ割り当て発生

// Cow<str> は Deref<Target=str> を実装しているため、&str と同様に扱える:
println!("{}", clean);
println!("{}", fixed.to_uppercase());
```

#### クイックリファレンス: どれを使うべきか

```text
関数内部でデータの所有権が必要か？
├── はい → impl Into<T>
│          「T に変換できるものなら何でも渡してください」
└── いいえ → 読み取るだけで十分か？
     ├── はい → impl AsRef<T> または &T
     │          「&T として借用できるものなら何でも渡してください」
     └── 場合による（たまに変更する必要があるか？）
          └── Cow<'_, T>
              「可能な限り借用し、必要な場合のみクローンする」
```

| パターン | 所有権 | メモリ割り当て | 使用場面 |
|---------|-----------|------------|-------------|
| `&str` | 借用 | なし | 単純な文字列パラメータ |
| `impl AsRef<str>` | 借用 | なし | String、&str などを許容 — 読み取り専用 |
| `impl Into<String>` | 所有 | 変換時に発生 | &str、String を許容 — 内部で保持・所有する場合 |
| `Cow<'_, str>` | どちらか一方 | 変更時のみ発生 | 通常は変更を行わない処理 |
| `&[u8]` / `impl AsRef<[u8]>` | 借用 | なし | バイト指向の API |

> **`Borrow<T>` vs `AsRef<T>`**: どちらも `&T` を提供しますが、`Borrow<T>` はさらに元の型と借用形態の間で `Eq`、`Ord`、`Hash` の挙動が**一貫している**ことを保証します。これが、`HashMap<String, V>::get()` が `AsRef` ではなく `&Q where String: Borrow<Q>` を受け入れる理由です。借用形態をルックアップキーとして使用する場合は `Borrow` を使い、一般的な「参照を渡す」パラメータには `AsRef` を使用します。

#### API における変換の組み合わせ

```rust
/// 人間工学的なパラメータを採用した、適切に設計された診断 API:
pub struct DiagRunner {
    name: String,
    config_path: PathBuf,
    results: HashMap<String, TestResult>,
}

impl DiagRunner {
    /// name には任意の文字列ライクな型を、config には任意のパスライクな型を受け入れる
    pub fn new(
        name: impl Into<String>,
        config_path: impl Into<PathBuf>,
    ) -> Self {
        DiagRunner {
            name: name.into(),
            config_path: config_path.into(),
        }
    }

    /// 読み取り専用ルックアップのために任意の AsRef<str> を受け入れる
    pub fn get_result(&self, test_name: impl AsRef<str>) -> Option<&TestResult> {
        self.results.get(test_name.as_ref())
    }
}

// これらすべてが呼び出し側の負担なしに機能する:
let runner = DiagRunner::new("GPU Diag", "/etc/diag_tool/config.json");
let runner = DiagRunner::new(format!("Diag-{}", node_id), config_path);
let runner = DiagRunner::new(name_string, path_buf);
```

***

## ケーススタディ: 公開クレート API の設計 — 改善前と改善後

文字列型に依存（Stringly-typed）していた内部 API を、人間工学的で型安全な公開 API へと発展させる実践例です。設定パーサークレートを考えてみましょう:

**改善前**（文字列型に過度に依存し、誤用しやすい）:

```rust
// ❌ すべてのパラメータが文字列 — コンパイル時検証なし
pub fn parse_config(path: &str, format: &str, strict: bool) -> Result<Config, String> {
    // どのフォーマットが有効か？ "json"? "JSON"? "Json"?
    // path はファイルパスなのか URL なのか？
    // "strict" とは何を意味しているのか？
    todo!()
}
```

**改善後**（型安全で自己文書化されている）:

```rust
use std::path::Path;

/// サポートされている設定フォーマット
#[derive(Debug, Clone, Copy)]
#[non_exhaustive]  // フォーマットを追加しても下流クレートを破壊しない
pub enum Format {
    Json,
    Toml,
    Yaml,
}

/// パースの厳密性を制御する
#[derive(Debug, Clone, Copy, Default)]
pub enum Strictness {
    /// 未知のフィールドを拒絶（ライブラリのデフォルト）
    #[default]
    Strict,
    /// 未知のフィールドを無視（将来の互換性を持たせたい設定向け）
    Lenient,
}

pub fn parse_config(
    path: &Path,          // 型で強制: ファイルシステムパスでなければならない
    format: Format,       // enum: 無効なフォーマットを渡すことは不可能
    strictness: Strictness,  // 単なる bool ではなく、名前付きの選択肢
) -> Result<Config, ConfigError> {
    todo!()
}
```

**改善点**:

| 観点 | 改善前 | 改善後 |
|--------|--------|-------|
| フォーマット検証 | 実行時の文字列比較 | コンパイル時の enum |
| パス型 | 生の `&str`（何でも渡せてしまう） | `&Path`（ファイルシステム専用） |
| 厳密性 | 意味不明な `bool` | 自己文書化された enum |
| エラー型 | `String`（不透明） | `ConfigError`（構造化） |
| 拡張性 | 破壊的変更につながる | `#[non_exhaustive]` |

> **経験則**: 文字列値に対して `match` を書いていることに気づいたら、そのパラメータを enum に置き換えることを検討してください。文脈から自明でない boolean パラメータがある場合は、2つのバリアントを持つ enum の使用を検討してください。

***

### 検証するな、パースせよ — `TryFrom` とバリデーション済み型

「検証するな、パースせよ（Parse, don't validate）」とは、**「データを検証した後に未検証の生データを引き回すのではなく、データが有効である場合にのみ存在できる型へとパースせよ」**という原則です。Rust の `TryFrom` トレイトはこのための標準的なツールです。

#### 問題点: 強制力のないバリデーション

```rust
// ❌ 検証してから使用: チェック後に不正な値が使われるのを防ぐ手段がない
fn process_port(port: u16) {
    if port == 0 || port > 65535 {
        panic!("Invalid port");           // チェックはしたものの...
    }
    start_server(port);                    // 誰かが直接 start_server(0) を呼び出したらどうなるか？
}

// ❌ 文字列への過度の依存: メールアドレスが単なる String — どんなゴミ値も通過してしまう
fn send_email(to: String, body: String) {
    // `to` は実際に有効なメールアドレスか？ 分からない。
    // 誰かが "not-an-email" を渡しても、SMTP サーバーに到達するまで検知できない。
}
```

#### 解決策: `TryFrom` で検証済みニュータイプにパースする

```rust
use std::convert::TryFrom;
use std::fmt;

/// バリデーション済みの TCP ポート番号（1〜65535）。
/// `Port` インスタンスが存在するなら、それは有効であることが保証される。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Port(u16);

impl TryFrom<u16> for Port {
    type Error = PortError;

    fn try_from(value: u16) -> Result<Self, Self::Error> {
        if value == 0 {
            Err(PortError::Zero)
        } else {
            Ok(Port(value))
        }
    }
}

impl Port {
    pub fn get(&self) -> u16 { self.0 }
}

#[derive(Debug)]
pub enum PortError {
    Zero,
    InvalidFormat,
}

impl fmt::Display for PortError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            PortError::Zero => write!(f, "ポート番号は 0 以外でなければなりません"),
            PortError::InvalidFormat => write!(f, "無効なポート形式です"),
        }
    }
}

impl std::error::Error for PortError {}

// 型システムが妥当性を強制する:
fn start_server(port: Port) {
    // バリデーション不要 — Port は TryFrom 経由でのみ構築可能であり、
    // 有効であることはすでに検証済み。
    println!("ポート {} でリスニング中", port.get());
}

// 使用例:
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let port = Port::try_from(8080)?;   // ✅ 境界部分で一度だけ検証
    start_server(port);                  // 下流のどこでも再検証は不要

    let bad = Port::try_from(0);         // ❌ Err(PortError::Zero)
    Ok(())
}
```

#### 実践例: バリデーション済み IPMI アドレス

```rust
/// バリデーション済みの IPMI スレーブアドレス（0x20〜0xFE、偶数のみ）。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct IpmiAddr(u8);

#[derive(Debug)]
pub enum IpmiAddrError {
    Odd(u8),
    OutOfRange(u8),
}

impl fmt::Display for IpmiAddrError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            IpmiAddrError::Odd(v) => write!(f, "IPMI アドレス 0x{v:02X} は偶数でなければなりません"),
            IpmiAddrError::OutOfRange(v) => {
                write!(f, "IPMI アドレス 0x{v:02X} は範囲外です (0x20..=0xFE)")
            }
        }
    }
}

impl TryFrom<u8> for IpmiAddr {
    type Error = IpmiAddrError;

    fn try_from(value: u8) -> Result<Self, Self::Error> {
        if value % 2 != 0 {
            Err(IpmiAddrError::Odd(value))
        } else if value < 0x20 || value > 0xFE {
            Err(IpmiAddrError::OutOfRange(value))
        } else {
            Ok(IpmiAddr(value))
        }
    }
}

impl IpmiAddr {
    pub fn get(&self) -> u8 { self.0 }
}

// 下流のコードで再チェックする必要は一切ない:
fn send_ipmi_command(addr: IpmiAddr, cmd: u8, data: &[u8]) -> Result<Vec<u8>, IpmiError> {
    // addr.get() は有効な偶数の IPMI アドレスであることが保証されている
    raw_ipmi_send(addr.get(), cmd, data)
}
```

#### `FromStr` による文字列のパース

テキスト（CLI 引数、設定ファイルなど）からパースされることが多い型には、`FromStr` を実装します:

```rust
use std::str::FromStr;

impl FromStr for Port {
    type Err = PortError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let n: u16 = s.parse().map_err(|_| PortError::InvalidFormat)?;
        Port::try_from(n)
    }
}

// .parse() で機能するようになる:
let port: Port = "8080".parse()?;   // 1ステップでバリデーション

// clap による CLI パースでも利用可能:
// #[derive(Parser)]
// struct Args {
//     #[arg(short, long)]
//     port: Port,   // clap は FromStr を自動的に呼び出す
// }
```

#### 複雑なバリデーションのための `TryFrom` チェーン

```rust
// この例のためのスタブ型 — 本番環境では個別のモジュールに配置され、
// それぞれの TryFrom 実装を持つことになります。
```

```rust
# struct Hostname(String);
# impl TryFrom<String> for Hostname {
#     type Error = String;
#     fn try_from(s: String) -> Result<Self, String> { Ok(Hostname(s)) }
# }
# struct Timeout(u64);
# impl TryFrom<u64> for Timeout {
#     type Error = String;
#     fn try_from(ms: u64) -> Result<Self, String> {
#         if ms == 0 { Err("timeout must be > 0".into()) } else { Ok(Timeout(ms)) }
#     }
# }
# struct RawConfig { host: String, port: u16, timeout_ms: u64 }
# #[derive(Debug)]
# enum ConfigError {
#     InvalidHost(String),
#     InvalidPort(PortError),
#     InvalidTimeout(String),
# }
# impl From<std::io::Error> for ConfigError {
#     fn from(e: std::io::Error) -> Self { ConfigError::InvalidHost(e.to_string()) }
# }
# impl From<serde_json::Error> for ConfigError {
#     fn from(e: serde_json::Error) -> Self { ConfigError::InvalidHost(e.to_string()) }
# }
/// すべてのフィールドが有効な場合にのみ存在できるバリデーション済み設定
pub struct ValidConfig {
    pub host: Hostname,
    pub port: Port,
    pub timeout_ms: Timeout,
}

impl TryFrom<RawConfig> for ValidConfig {
    type Error = ConfigError;

    fn try_from(raw: RawConfig) -> Result<Self, Self::Error> {
        Ok(ValidConfig {
            host: Hostname::try_from(raw.host)
                .map_err(ConfigError::InvalidHost)?,
            port: Port::try_from(raw.port)
                .map_err(ConfigError::InvalidPort)?,
            timeout_ms: Timeout::try_from(raw.timeout_ms)
                .map_err(ConfigError::InvalidTimeout)?,
        })
    }
}

// 境界部分で一度だけパースし、内部ではバリデーション済みの型を使用する:
fn load_config(path: &str) -> Result<ValidConfig, ConfigError> {
    let raw: RawConfig = serde_json::from_str(&std::fs::read_to_string(path)?)?;
    ValidConfig::try_from(raw)  // すべての検証がここで行われる
}
```

#### まとめ: 検証（Validate） vs パース（Parse）

| アプローチ | データチェック有無 | コンパイラによる正当性強制 | 再検証の必要性 |
|----------|:---:|:---:|:---:|
| 実行時チェック (if/assert) | ✅ | ❌ | 関数の境界ごとに必要 |
| バリデーション済みニュータイプ + `TryFrom` | ✅ | ✅ | 不要 — 型そのものが証明となる |

原則: **境界部分でパースし、内部のあらゆる場所でバリデーション済みの型を使用する。**
生の文字列、整数、バイトスライスがシステムに入力されたら、`TryFrom`/`FromStr` を介して検証済みの型にパースします。その時点以降、型システムがその正当性を保証します。

### フィーチャーフラグと条件付きコンパイル

```toml
# Cargo.toml
[features]
default = ["json"]          # デフォルトで有効
json = ["dep:serde_json"]   # JSON サポートを有効化
xml = ["dep:quick-xml"]     # XML サポートを有効化
full = ["json", "xml"]      # メタフィーチャー: すべてを有効化

[dependencies]
serde = "1"
serde_json = { version = "1", optional = true }
quick-xml = { version = "0.31", optional = true }
```

```rust
// フィーチャーに基づく条件付きコンパイル:
#[cfg(feature = "json")]
pub fn to_json<T: serde::Serialize>(value: &T) -> String {
    serde_json::to_string(value).unwrap()
}

#[cfg(feature = "xml")]
pub fn to_xml<T: serde::Serialize>(value: &T) -> String {
    quick_xml::se::to_string(value).unwrap()
}

// 必要なフィーチャーが有効化されていない場合はコンパイルエラーにする:
#[cfg(not(any(feature = "json", feature = "xml")))]
compile_error!("少なくとも 1 つのフォーマットフィーチャー (json, xml) を有効にする必要があります");
```

**ベストプラクティス**:
- `default` フィーチャーは最小限に抑える — ユーザーが必要に応じてオプトインできるようにする
- 暗黙のフィーチャーが作成されるのを避けるため、オプションの依存関係には `dep:` 構文（Rust 1.60+）を使用する
- フィーチャーを README やクレートレベルのドキュメントに明記する

### ワークスペース構成

大規模なプロジェクトでは、依存関係とビルド成果物を共有するために Cargo ワークスペースを使用します:

```toml
# ルート Cargo.toml
[workspace]
members = [
    "core",         # 共有型とトレイト
    "parser",       # パースライブラリ
    "server",       # バイナリ — メインアプリケーション
    "client",       # クライアントライブラリ
    "cli",          # CLI バイナリ
]

# 共通の依存関係バージョン:
[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
tracing = "0.1"

# 各メンバーの Cargo.toml 内で以下のように指定:
# [dependencies]
# serde = { workspace = true }
```

**メリット**:

- 単一の `Cargo.lock` — すべてのクレートが同一の依存関係バージョンを使用
- `cargo test --workspace` ですべてのテストを実行可能
- ビルドキャッシュの共有 — 1つのクレートのコンパイルが全体に恩恵をもたらす
- コンポーネント間の依存関係の境界が明確になる

### `.cargo/config.toml`: プロジェクトレベルの設定

`.cargo/config.toml` ファイル（ワークスペースのルートまたは `$HOME/.cargo/` に配置）は、`Cargo.toml` を変更することなく Cargo の動作をカスタマイズします:

```toml
# .cargo/config.toml

# このワークスペースのデフォルトターゲット
[build]
target = "x86_64-unknown-linux-gnu"

# カスタムランナー — 例: クロスコンパイルされたバイナリを QEMU 経由で実行
[target.aarch64-unknown-linux-gnu]
runner = "qemu-aarch64-static"
linker = "aarch64-linux-gnu-gcc"

# Cargo エイリアス — カスタムショートカットコマンド
[alias]
xt = "test --workspace --release"        # cargo xt = 全テストを release モードで実行
ci = "clippy --workspace -- -D warnings" # cargo ci = 警告をエラーとしてリント実行
cov = "llvm-cov --workspace"             # cargo cov = カバレッジ測定 (cargo-llvm-cov が必要)

# ビルドスクリプト用環境変数
[env]
IPMI_LIB_PATH = "/usr/lib/bmc"

# カスタムレジストリの使用（社内パッケージなど）
# [registries.internal]
# index = "https://gitlab.internal/crates/index"
```

一般的な設定パターン:

| 設定項目 | 目的 | 例 |
|---------|---------|---------|
| `[build] target` | デフォルトのコンパイルターゲット | 静的ビルド向けの `x86_64-unknown-linux-musl` |
| `[target.X] runner` | バイナリの実行方法 | クロスコンパイル向けの `"qemu-aarch64-static"` |
| `[target.X] linker` | 使用するリンカー | `"aarch64-linux-gnu-gcc"` |
| `[alias]` | カスタム `cargo` サブコマンド | `xt = "test --workspace"` |
| `[env]` | ビルド時の環境変数 | ライブラリパス、機能トグル |
| `[net] offline` | ネットワークアクセスの防止 | エアギャップ（隔離環境）ビルド向けの `true` |

### コンパイル時環境変数: `env!()` と `option_env!()`

Rust はコンパイル時に環境変数をバイナリに埋め込むことができます。バージョン文字列、ビルドメタデータ、設定情報などに便利です:

```rust
// env!() — 環境変数が存在しない場合はコンパイル時にパニックする
const VERSION: &str = env!("CARGO_PKG_VERSION"); // Cargo.toml から "0.1.0"
const PKG_NAME: &str = env!("CARGO_PKG_NAME");   // Cargo.toml からのクレート名

// option_env!() — Option<&str> を返し、存在しなくてもパニックしない
const BUILD_SHA: Option<&str> = option_env!("GIT_SHA");
const BUILD_TIME: Option<&str> = option_env!("BUILD_TIMESTAMP");

fn print_version() {
    println!("{PKG_NAME} v{VERSION}");
    if let Some(sha) = BUILD_SHA {
        println!("  commit: {sha}");
    }
    if let Some(time) = BUILD_TIME {
        println!("  built:  {time}");
    }
}
```

Cargo は有用な環境変数を多数自動設定します:

| 変数 | 値 | ユースケース |
|----------|-------|----------|
| `CARGO_PKG_VERSION` | `"1.2.3"` | バージョン報告 |
| `CARGO_PKG_NAME` | `"diag_tool"` | バイナリの識別 |
| `CARGO_PKG_AUTHORS` | `Cargo.toml` から | アバウト/ヘルプテキスト |
| `CARGO_MANIFEST_DIR` | `Cargo.toml` の絶対パス | テストデータファイルの配置場所特定 |
| `OUT_DIR` | ビルド出力ディレクトリ | `build.rs` のコード生成先 |
| `TARGET` | ターゲットトリプル | `build.rs` 内のプラットフォーム固有ロジック |

`build.rs` からカスタム環境変数を設定することもできます:
```rust
// build.rs
fn main() {
    println!("cargo::rustc-env=GIT_SHA={}", git_sha());
    println!("cargo::rustc-env=BUILD_TIMESTAMP={}", timestamp());
}
```

### `cfg_attr`: 条件付き属性

`cfg_attr` は条件が真の場合に**のみ**属性を適用します。項目全体を含める/除外する `#[cfg()]` よりも細やかな制御が可能です:

```rust
// "serde" フィーチャーが有効な場合のみ Serialize を derive する:
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
#[derive(Debug, Clone)]
pub struct DiagResult {
    pub fc: u32,
    pub passed: bool,
    pub message: String,
}
// "serde" フィーチャーなし: serde への依存は一切不要
// "serde" フィーチャーあり: DiagResult はシリアライズ可能

// テスト用の条件付き属性:
#[cfg_attr(test, derive(PartialEq))]  // テストビルド時のみ PartialEq を derive
pub struct LargeStruct { /* ... */ }

// プラットフォーム固有の関数属性:
#[cfg_attr(target_os = "linux", link_name = "ioctl")]
#[cfg_attr(target_os = "freebsd", link_name = "__ioctl")]
extern "C" fn platform_ioctl(fd: i32, request: u64) -> i32;
```

| パターン | 動作 |
|---------|-------------|
| `#[cfg(feature = "x")]` | 項目全体を含める/除外する |
| `#[cfg_attr(feature = "x", derive(Foo))]` | フィーチャー "x" が有効な場合のみ `derive(Foo)` を追加 |
| `#[cfg_attr(test, allow(unused))]` | テストビルドでのみ警告を抑制 |
| `#[cfg_attr(doc, doc = "...")]` | `cargo doc` でのみドキュメントを表示 |

### `cargo deny` と `cargo audit`: サプライチェーンセキュリティ

```bash
# セキュリティ監査ツールのインストール
cargo install cargo-deny
cargo install cargo-audit

# 依存関係内の既知の脆弱性をチェック
cargo audit

# 包括的なチェック: ライセンス、禁止クレート、アドバイザリ、ソース
cargo deny check
```

ワークスペースのルートに `deny.toml` を配置して `cargo deny` を設定します:

```toml
# deny.toml
[advisories]
vulnerability = "deny"      # 既知の脆弱性があれば失敗とする
unmaintained = "warn"        # メンテナンスされていないクレートは警告

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause"]
deny = ["GPL-3.0"]          # コピーレフトライセンスを拒否

[bans]
multiple-versions = "warn"  # 同一クレートの複数バージョンが存在する場合は警告
deny = [
    { name = "openssl" },   # openssl の代わりに rustls の使用を強制
]

[sources]
allow-git = []              # 本番環境では git 依存関係を禁止
```

| ツール | 目的 | 実行タイミング |
|------|---------|-------------|
| `cargo audit` | 依存関係の既知の CVE をチェック | CI パイプライン、リリース前 |
| `cargo deny check` | ライセンス、禁止クレート、アドバイザリ、ソースのチェック | CI パイプライン |
| `cargo deny check licenses` | ライセンス遵守の確認のみ | オープンソース化前 |
| `cargo deny check bans` | 特定のクレートの混入防止 | アーキテクチャ上の決定事項の強制 |

### ドキュメントテスト: ドキュメント内のテスト

Rust のドキュメントコメント（`///`）には、**コンパイルされてテストとして実行される**コードブロックを含めることができます:

```rust
/// 文字列から診断フォールトコードをパースする。
///
/// # 例
///
/// ```
/// use my_crate::parse_fc;
///
/// let fc = parse_fc("FC:12345").unwrap();
/// assert_eq!(fc, 12345);
/// ```
///
/// 無効な入力はエラーを返す:
///
/// ```
/// use my_crate::parse_fc;
///
/// assert!(parse_fc("not-a-fc").is_err());
/// ```
pub fn parse_fc(input: &str) -> Result<u32, ParseError> {
    input.strip_prefix("FC:")
        .ok_or(ParseError::MissingPrefix)?
        .parse()
        .map_err(ParseError::InvalidNumber)
}
```

```bash
cargo test --doc  # ドキュメントテストのみを実行
cargo test        # 単体テスト + 統合テスト + ドキュメントテストを実行
```

**モジュールレベルのドキュメント**はファイルの先頭で `//!` を使用します:

```rust
//! # 診断フレームワーク
//!
//! このクレートは、コアとなる診断実行エンジンを提供します。
//! 診断テストの実行、結果の収集、
//! および IPMI を介した BMC への報告をサポートします。
//!
//! ## クイックスタート
//!
//! ```no_run
//! use diag_framework::Framework;
//!
//! let mut fw = Framework::new("config.json")?;
//! fw.run_all_tests()?;
//! ```
```

### Criterion によるベンチマーク測定

> **詳細な解説**: 完全な `criterion` のセットアップ、API の例、`cargo bench` との比較表については、第14章（テストとベンチマークパターン）の「[criterion によるベンチマーク測定](ch14-testing-and-benchmarking-patterns.md#benchmarking-with-criterion)」を参照してください。以下はアーキテクチャ固有の利用法に関するクイックリファレンスです。

クレートの公開 API をベンチマーク測定する際は、ベンチマークコードを `benches/` に配置し、パーサー、シリアライザー、バリデーション境界などのホットパスに焦点を絞ります:

```bash
cargo bench                  # すべてのベンチマークを実行
cargo bench -- parse_config  # 特定のベンチマークを実行
# 結果は target/criterion/ に HTML レポートとして出力される
```

> **重要ポイント — アーキテクチャと API 設計**
> - 最も汎用的な型を受け入れ（`impl Into`、`impl AsRef`、`Cow`）、最も具体的な型を返す
> - 検証するな、パースせよ（Parse Don't Validate）: `TryFrom` を用いて、構造的に妥当な型を作成する
> - 公開 enum に `#[non_exhaustive]` を付与して、バリアント追加時の破壊的変更を防止する
> - `#[must_use]` で重要な値の暗黙的な破棄を検出する

> **関連情報:** 公開 API におけるエラー型の設計については「[第10章 エラー処理パターン](ch10-error-handling-patterns.md)」、クレートの公開 API のテストについては「[第14章 テストとベンチマークパターン](ch14-testing-and-benchmarking-patterns.md)」を参照してください。

---

### 演習: クレート API のリファクタリング ★★ (約30分)

以下の「文字列型に依存した（stringly-typed）」API を、`TryFrom`、ニュータイプ、Builder パターンを使用する設計にリファクタリングしてください:

```rust,ignore
// 改善前: 誤用しやすい
fn create_server(host: &str, port: &str, max_conn: &str) -> Server { ... }
```

バリデーション済み型 `Host`、`Port` (1〜65535)、および `MaxConnections` (1〜10000) を持つ `ServerConfig` を設計し、パース時に無効な値を拒絶するようにしてください。

<details>
<summary>🔑 解答例</summary>

```rust
#[derive(Debug, Clone)]
struct Host(String);

impl TryFrom<&str> for Host {
    type Error = String;
    fn try_from(s: &str) -> Result<Self, String> {
        if s.is_empty() { return Err("ホスト名を空にすることはできません".into()); }
        if s.contains(' ') { return Err("ホスト名にスペースを含めることはできません".into()); }
        Ok(Host(s.to_string()))
    }
}

#[derive(Debug, Clone, Copy)]
struct Port(u16);

impl TryFrom<u16> for Port {
    type Error = String;
    fn try_from(p: u16) -> Result<Self, String> {
        if p == 0 { return Err("ポート番号は 1 以上でなければなりません".into()); }
        Ok(Port(p))
    }
}

#[derive(Debug, Clone, Copy)]
struct MaxConnections(u32);

impl TryFrom<u32> for MaxConnections {
    type Error = String;
    fn try_from(n: u32) -> Result<Self, String> {
        if n == 0 || n > 10_000 {
            return Err(format!("max_connections は 1〜10000 である必要があります。指定された値: {n}"));
        }
        Ok(MaxConnections(n))
    }
}

#[derive(Debug)]
struct ServerConfig {
    host: Host,
    port: Port,
    max_connections: MaxConnections,
}

impl ServerConfig {
    fn new(host: Host, port: Port, max_connections: MaxConnections) -> Self {
        ServerConfig { host, port, max_connections }
    }
}

fn main() {
    let config = ServerConfig::new(
        Host::try_from("localhost").unwrap(),
        Port::try_from(8080).unwrap(),
        MaxConnections::try_from(100).unwrap(),
    );
    println!("{config:?}");

    // 無効な値はパース時に検出される:
    assert!(Host::try_from("").is_err());
    assert!(Port::try_from(0).is_err());
    assert!(MaxConnections::try_from(99999).is_err());
}
```

</details>

***
