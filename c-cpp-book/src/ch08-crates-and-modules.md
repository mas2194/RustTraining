# Rustのクレートとモジュール

> **学習目標:** モジュールとクレートを使用したRustのコード構造化方法（デフォルトで非公開となる可視性、`pub` 修飾子、ワークスペース、`crates.io` エコシステムなど）を学びます。これらはC/C++のヘッダーファイル、`#include`、CMakeの依存関係管理を置き換えるものです。

- モジュールは、クレート内のコードを構造化する基本的な単位です
    - 各ソースファイル（.rs）はそれ自体が1つのモジュールであり、`mod` キーワードを使用してネストされたモジュールを作成できます
    - （サブ）モジュール内のすべての型はデフォルトで**非公開（private）**であり、明示的に `pub`（public）と指定しない限り、同一クレート内であっても外部から参照できません。`pub` のスコープは `pub(crate)` などでさらに制限できます
    - 型が公開されていても、`use` キーワードを使用してインポートしない限り、別のモジュールのスコープ内で自動的に可視になるわけではありません。子サブモジュールは `use super::` を使用して親スコープの型を参照できます
    - ソースファイル（.rs）は、`main.rs`（バイナリ）または `lib.rs`（ライブラリ）で明示的に宣言されない限り、クレートに自動的には含まれ**ません**

# 演習: モジュールと関数
- [hello world](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=522d86dbb8c4af71ff2ec081fb76aee7) を変更して別の関数を呼び出してみましょう
    - 前述のとおり、関数は `fn` キーワードで定義します。`->` は関数が戻り値を返すことを宣言し（デフォルトは戻り値なしの void / unit）、ここでは `u32`（符号なし32ビット整数）型を返します
    - 関数はモジュールごとにスコープが分かれているため、2つのモジュールに全く同じ名前の関数があっても名前の衝突は発生しません
        - モジュールスコープはすべての型に適用されます（例えば、`mod a { struct foo; }` 内の `struct foo` は、`mod b { struct foo; }`（`b::foo`）とは別個の型（`a::foo`）です）

**スターターコード** — 関数を完成させてください:
```rust
mod math {
    // TODO: pub fn add(a: u32, b: u32) -> u32 を実装
}

fn greet(name: &str) -> String {
    // TODO: "Hello, <name>! The secret number is <math::add(21,21)>" を返す
    todo!()
}

fn main() {
    println!("{}", greet("Rustacean"));
}
```

<details><summary>解答例（クリックして展開）</summary>

```rust
mod math {
    pub fn add(a: u32, b: u32) -> u32 {
        a + b
    }
}

fn greet(name: &str) -> String {
    format!("Hello, {}! The secret number is {}", name, math::add(21, 21))
}

fn main() {
    println!("{}", greet("Rustacean"));
}
// 出力: Hello, Rustacean! The secret number is 42
```

</details>


## ワークスペースとクレート（パッケージ）

- 規模の大きなRustプロジェクトでは、コンポーネントとなるクレートを整理するためにワークスペースを使用すべきです
    - ワークスペースは、ターゲットバイナリのビルドに使用されるローカルクレートの集合体にすぎません。ワークスペースルートの `Cargo.toml` には、構成要素となるパッケージ（クレート）へのポインタを指定します

```toml
[workspace]
resolver = "2"
members = ["package1", "package2"]
```

```text
workspace_root/
|-- Cargo.toml      # ワークスペース設定
|-- package1/
|   |-- Cargo.toml  # パッケージ1設定
|   `-- src/
|       `-- lib.rs  # パッケージ1ソースコード
|-- package2/
|   |-- Cargo.toml  # パッケージ2設定
|   `-- src/
|       `-- main.rs # パッケージ2ソースコード
```

---
## 演習: ワークスペースとパッケージ依存関係の使用
- シンプルなパッケージを作成し、それを `hello world` プログラムから使用してみましょう
- ワークスペースディレクトリを作成します
```bash
mkdir workspace
cd workspace
```
- `Cargo.toml` というファイルを作成し、以下を追加します。これにより空のワークスペースが作成されます
```toml
[workspace]
resolver = "2"
members = []
```
- パッケージを追加します（`cargo new --lib` は実行ファイルではなくライブラリを指定します）
```bash
cargo new hello
cargo new --lib hellolib
```

## 演習: ワークスペースとパッケージ依存関係の使用
- `hello` と `hellolib` で生成された Cargo.toml を確認してください。両方が上位レベルの `Cargo.toml` に追加されていることに注意してください
- `hellolib` に `lib.rs` が存在することは、ライブラリパッケージであることを意味します（カスタマイズオプションについては https://doc.rust-lang.org/cargo/reference/cargo-targets.html を参照）
- `hello` の `Cargo.toml` に `hellolib` への依存関係を追加します
```toml
[dependencies]
hellolib = {path = "../hellolib"}
```
- `hellolib` の `add()` を使用します
```rust
fn main() {
    println!("Hello, world! {}", hellolib::add(21, 21));
}
```

<details><summary>解答例（クリックして展開）</summary>

ワークスペースのセットアップ全体:

```bash
# ターミナルコマンド
mkdir workspace && cd workspace

# ワークスペースの Cargo.toml を作成
cat > Cargo.toml << 'EOF'
[workspace]
resolver = "2"
members = ["hello", "hellolib"]
EOF

cargo new hello
cargo new --lib hellolib
```

```toml
# hello/Cargo.toml — 依存関係を追加
[dependencies]
hellolib = {path = "../hellolib"}
```

```rust
// hellolib/src/lib.rs — cargo new --lib によりすでに add() が生成されている
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}
```

```rust,ignore
// hello/src/main.rs
fn main() {
    println!("Hello, world! {}", hellolib::add(21, 21));
}
// 出力: Hello, world! 42
```

</details>

# crates.io のコミュニティクレートの利用
- Rustには活気あるコミュニティクレートのエコシステムが存在します（https://crates.io/ を参照）
    - Rustの哲学は、標準ライブラリをコンパクトに保ち、多くの機能をコミュニティクレートに委ねることです
    - コミュニティクレートの採用に関する厳密なルールはありませんが、目安としてクレートの成熟度（バージョン番号で判断）が高く、活発にメンテナンスされているかを確認すべきです。判断に迷った場合は社内の詳しい人に相談してください
- `crates.io` に公開されているすべてのクレートにはメジャーバージョンとマイナーバージョンがあります
    - クレートは https://doc.rust-lang.org/cargo/reference/semver.html で定義されているメジャーおよびマイナーの `SemVer`（セマンティックバージョニング）ガイドラインに従うことが期待されます
    - 要約すると、同一マイナーバージョン内では互換性を壊す変更（breaking change）があってはなりません。例えば、v0.11 は v0.15 と互換性がなければなりません（ただし、v0.20 では破壊的変更が含まれる可能性があります）

# クレートの依存関係とセマンティックバージョニング（SemVer）
- クレートは、特定バージョン、特定のマイナー/メジャーバージョン、あるいは「任意（don't care）」への依存関係を定義できます。以下の例は、`rand` クレートへの依存を宣言する `Cargo.toml` のエントリを示しています
- 少なくとも `0.10.0` 以上、ただし `< 0.11.0` 未満なら可
```toml
[dependencies]
rand = { version = "0.10.0"}
```
- `0.10.0` のみ（完全一致）
```toml
[dependencies]
rand = { version = "=0.10.0"}
```
- バージョン不問。`cargo` が最新バージョンを選択
```toml
[dependencies]
rand = { version = "*"}
```
- 参考: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html
----
# 演習: rand クレートの使用
- `helloworld` の例を変更して乱数を出力してみましょう
- `cargo add rand` を実行して依存関係を追加します
- APIのリファレンスとして `https://docs.rs/rand/latest/rand/` を参照してください

**スターターコード** — `cargo add rand` を実行した後、`main.rs` に以下を追加してください:
```rust,ignore
use rand::RngExt;

fn main() {
    let mut rng = rand::rng();
    // TODO: 1..=100 の範囲のランダムな u32 を生成して出力
    // TODO: ランダムな bool を生成して出力
    // TODO: ランダムな f64 を生成して出力
}
```

<details><summary>解答例（クリックして展開）</summary>

```rust
use rand::RngExt;

fn main() {
    let mut rng = rand::rng();
    let n: u32 = rng.random_range(1..=100);
    println!("乱数 (1-100): {n}");

    // ランダムなブール値を生成
    let b: bool = rng.random();
    println!("ランダムなbool: {b}");

    // 0.0 から 1.0 の間のランダムな浮動小数点数を生成
    let f: f64 = rng.random();
    println!("ランダムな浮動小数点数: {f:.4}");
}
```

</details>

# Cargo.toml と Cargo.lock
- 前述のとおり、Cargo.lock は Cargo.toml から自動的に生成されます
    - Cargo.lock の主な目的は、再現可能なビルドを保証することです。例えば、`Cargo.toml` で `0.10.0` を指定した場合、cargo は `< 0.11.0` の任意のバージョンを選択できます
    - Cargo.lock には、ビルド時に実際に使用された rand クレートの*特定*のバージョンが記録されます
    - 再現可能なビルドを保証するため、`Cargo.lock` を git リポジトリに含めることが推奨されます

## Cargoのテスト機能
- Rustのユニットテストは（慣例として）同じソースファイル内に配置され、通常は別のモジュールとしてグループ化されます
    - テストコードが実際のバイナリに含まれることはありません。これは `cfg`（構成）機能によって実現されています。構成は、例えばプラットフォーム固有のコード（`Linux` vs `Windows`）を作成する場合などにも役立ちます
    - テストは `cargo test` で実行できます。参考: https://doc.rust-lang.org/reference/conditional-compilation.html

```rust
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}
// テスト時のみ含まれる
#[cfg(test)]
mod tests {
    use super::*; // 親スコープのすべての型を可視にする
    #[test]
    fn it_works() {
        let result = add(2, 2); // あるいは super::add(2, 2);
        assert_eq!(result, 4);
    }
}
```

# その他のCargoの機能
- `cargo` には、以下を含むいくつかの便利な機能があります:
    - `cargo clippy` は、Rustコードの優れた静的解析（lint）ツールです。一般に警告は修正すべきです（真に必要な場合に限り抑制します）
    - `cargo format` は `rustfmt` ツールを実行してソースコードをフォーマットします。このツールを使用することで、チェックインされるコードのフォーマットが標準化され、スタイルに関する議論に終止符を打つことができます
    - `cargo doc` は `///` 形式のコメントからドキュメントを生成するために使用できます。`crates.io` 上のすべてのクレートのドキュメントはこの方法で生成されています

### ビルドプロファイル: 最適化の制御

C言語では、gcc/clang に `-O0`, `-O2`, `-Os`, `-flto` などを渡します。Rustでは、`Cargo.toml` でビルドプロファイルを構成します:

```toml
# Cargo.toml — ビルドプロファイルの設定

[profile.dev]
opt-level = 0          # 最適化なし（高速コンパイル、-O0 相当）
debug = true           # 完全なデバッグシンボル（-g 相当）

[profile.release]
opt-level = 3          # 最大限の最適化（-O3 相当）
lto = "fat"            # リンク時最適化（Link-Time Optimization, -flto 相当）
strip = true           # シンボルの削除（stripコマンド相当）
codegen-units = 1      # 単一コード生成単位 — コンパイルは低速だがより優れた最適化
panic = "abort"        # 巻き戻しテーブルを生成しない（バイナリサイズの縮小）
```

| C/GCC フラグ | Cargo.toml のキー | 設定値 |
|------------|---------------|--------|
| `-O0` / `-O2` / `-O3` | `opt-level` | `0`, `1`, `2`, `3`, `"s"`, `"z"` |
| `-flto` | `lto` | `false`, `"thin"`, `"fat"` |
| `-g` / `-g` なし | `debug` | `true`, `false`, `"line-tables-only"` |
| `strip` コマンド | `strip` | `"none"`, `"debuginfo"`, `"symbols"`, `true`/`false` |
| — | `codegen-units` | `1` = 最良の最適化、最遅のコンパイル |

```bash
cargo build              # [profile.dev] を使用
cargo build --release    # [profile.release] を使用
```

### ビルドスクリプト（`build.rs`）: Cライブラリのリンク

C言語では、ライブラリのリンクやコード生成の実行に Makefile や CMake を使用します。Rustでは、クレートルートにある `build.rs` ファイルを使用します:

```rust
// build.rs — クレートのコンパイル前に実行される

fn main() {
    // システムのCライブラリをリンク（gccの -lbmc_ipmi 相当）
    println!("cargo::rustc-link-lib=bmc_ipmi");

    // ライブラリの検索場所（-L/usr/lib/bmc 相当）
    println!("cargo::rustc-link-search=/usr/lib/bmc");

    // Cヘッダーが変更された場合に再実行
    println!("cargo::rerun-if-changed=wrapper.h");
}
```

RustクレートからC言語のソースファイルを直接コンパイルすることも可能です:

```toml
# Cargo.toml
[build-dependencies]
cc = "1"  # Cコンパイラ連携
```

```rust
// build.rs
fn main() {
    cc::Build::new()
        .file("src/c_helpers/ipmi_raw.c")
        .include("/usr/include/bmc")
        .compile("ipmi_raw");   // libipmi_raw.a が生成され、自動的にリンクされる
    println!("cargo::rerun-if-changed=src/c_helpers/ipmi_raw.c");
}
```

| C / Make / CMake | Rust `build.rs` |
|-----------------|-----------------|
| `-lfoo` | `println!("cargo::rustc-link-lib=foo")` |
| `-L/path` | `println!("cargo::rustc-link-search=/path")` |
| Cソースのコンパイル | `cc::Build::new().file("foo.c").compile("foo")` |
| コード生成 | `$OUT_DIR` にファイルを出力し、`include!()` で取り込む |

### クロスコンパイル

C言語では、クロスコンパイルには個別のツールチェーン（`arm-linux-gnueabihf-gcc` など）のインストールと Make/CMake の設定が必要です。Rustの場合:

```bash
# クロスコンパイルターゲットのインストール
rustup target add aarch64-unknown-linux-gnu

# クロスコンパイルの実行
cargo build --target aarch64-unknown-linux-gnu --release
```

`.cargo/config.toml` でリンカを指定します:

```toml
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
```

| C言語のクロスコンパイル | Rustでの対応 |
|-----------------|-----------------|
| `apt install gcc-aarch64-linux-gnu` | `rustup target add aarch64-unknown-linux-gnu` + リンカのインストール |
| `CC=aarch64-linux-gnu-gcc make` | `.cargo/config.toml` の `[target.X] linker = "..."` |
| `#ifdef __aarch64__` | `#[cfg(target_arch = "aarch64")]` |
| 個別のMakefileターゲット | `cargo build --target ...` |

### フィーチャーフラグ: 条件付きコンパイル

C言語では条件付きコンパイルに `#ifdef` や `-DFOO` を使用します。Rustでは `Cargo.toml` で定義されたフィーチャーフラグを使用します:

```toml
# Cargo.toml
[features]
default = ["json"]         # デフォルトで有効
json = ["dep:serde_json"]  # オプショナルな依存関係
verbose = []               # 依存関係を持たないフラグ
gpu = ["dep:cuda-sys"]     # オプショナルなGPUサポート
```

```rust
// フィーチャーによって分岐するコード:
#[cfg(feature = "json")]
pub fn parse_config(data: &str) -> Result<Config, Error> {
    serde_json::from_str(data).map_err(Error::from)
}

#[cfg(feature = "verbose")]
macro_rules! verbose {
    ($($arg:tt)*) => { eprintln!("[VERBOSE] {}", format!($($arg)*)); }
}
#[cfg(not(feature = "verbose"))]
macro_rules! verbose {
    ($($arg:tt)*) => {}; // 何もコンパイルされない
}
```

| Cプリプロセッサ | Rustフィーチャーフラグ |
|---------------|-------------------|
| `gcc -DDEBUG` | `cargo build --features verbose` |
| `#ifdef DEBUG` | `#[cfg(feature = "verbose")]` |
| `#define MAX 100` | `const MAX: u32 = 100;` |
| `#ifdef __linux__` | `#[cfg(target_os = "linux")]` |

### 統合テスト vs ユニットテスト

ユニットテストは `#[cfg(test)]` を付けてコードと同じファイル内に配置されます。**統合テスト**は `tests/` ディレクトリに配置され、クレートの**公開APIのみ**をテストします:

```rust
// tests/smoke_test.rs — #[cfg(test)] は不要
use my_crate::parse_config;

#[test]
fn parse_valid_config() {
    let config = parse_config("test_data/valid.json").unwrap();
    assert_eq!(config.max_retries, 5);
}
```

| 観点 | ユニットテスト (`#[cfg(test)]`) | 統合テスト (`tests/`) |
|--------|----------------------------|------------------------------|
| 配置場所 | コードと同じファイル内 | 独立した `tests/` ディレクトリ |
| アクセス範囲 | 非公開アイテム＋公開アイテム | **公開APIのみ** |
| 実行コマンド | `cargo test` | `cargo test --test smoke_test` |


### テストパターンと戦略

C言語のファームウェア開発チームは、通常 CUnit、CMocka、または大量のボイラープレートを伴うカスタムフレームワークでテストを作成します。Rustの組み込みテストハーネスははるかに高機能です。このセクションでは、プロダクションコードに必要なパターンを取り上げます。

#### `#[should_panic]` — 予期されるパニックのテスト

```rust
// 特定の条件でパニックが発生することをテスト（C言語のアサート失敗に相当）
#[test]
#[should_panic(expected = "index out of bounds")]
fn test_bounds_check() {
    let v = vec![1, 2, 3];
    let _ = v[10];  // パニックするはず
}

#[test]
#[should_panic(expected = "temperature exceeds safe limit")]
fn test_thermal_shutdown() {
    fn check_temperature(celsius: f64) {
        if celsius > 105.0 {
            panic!("temperature exceeds safe limit: {celsius}°C");
        }
    }
    check_temperature(110.0);
}
```

#### `#[ignore]` — 時間のかかるテストやハードウェア依存のテスト

```rust
// 特別な条件が必要なテストにマーク（C言語の #ifdef HARDWARE_TEST 相当）
#[test]
#[ignore = "requires GPU hardware"]
fn test_gpu_ecc_scrub() {
    // このテストはGPUを搭載したマシンでのみ実行される
    // 実行方法: cargo test -- --ignored
    // 実行方法: cargo test -- --include-ignored  （すべてのテストを実行）
}
```

#### Resultを返すテスト（`unwrap` チェーンの置き換え）

```rust
// 実際の失敗原因を覆い隠してしまう多数の unwrap() 呼び出しの代わりに:
#[test]
fn test_config_parsing() -> Result<(), Box<dyn std::error::Error>> {
    let json = r#"{"hostname": "node-01", "port": 8080}"#;
    let config: ServerConfig = serde_json::from_str(json)?;  // unwrap() の代わりに ? を使用
    assert_eq!(config.hostname, "node-01");
    assert_eq!(config.port, 8080);
    Ok(())  // エラーなくここまで到達すればテスト成功
}
```

#### ビルダー関数を用いたテストフィクスチャ

C言語では `setUp()`/`tearDown()` 関数を使用します。Rustではヘルパー関数と `Drop` を使用します:

```rust
struct TestFixture {
    temp_dir: std::path::PathBuf,
    config: Config,
}

impl TestFixture {
    fn new() -> Self {
        let temp_dir = std::env::temp_dir().join(format!("test_{}", std::process::id()));
        std::fs::create_dir_all(&temp_dir).unwrap();
        let config = Config {
            log_dir: temp_dir.clone(),
            max_retries: 3,
            ..Default::default()
        };
        Self { temp_dir, config }
    }
}

impl Drop for TestFixture {
    fn drop(&mut self) {
        // 自動クリーンアップ — C言語の tearDown() 相当だが呼び出し忘れが発生しない
        let _ = std::fs::remove_dir_all(&self.temp_dir);
    }
}

#[test]
fn test_with_fixture() {
    let fixture = TestFixture::new();
    // fixture.config, fixture.temp_dir を使用...
    assert!(fixture.temp_dir.exists());
    // fixture はここで自動的にドロップされる → クリーンアップが実行される
}
```

#### ハードウェアインターフェース向けトレイトのモック

C言語では、ハードウェアのモックにはプリプロセッサのトリックや関数ポインタの差し替えが必要です。Rustでは、トレイトにより自然にモックを実現できます:

```rust
// IPMI通信用の本番トレイト
trait IpmiTransport {
    fn send_command(&self, cmd: u8, data: &[u8]) -> Result<Vec<u8>, String>;
}

// 実際の実装（本番環境で使用）
struct RealIpmi { /* BMC接続の詳細 */ }
impl IpmiTransport for RealIpmi {
    fn send_command(&self, cmd: u8, data: &[u8]) -> Result<Vec<u8>, String> {
        // 実際のBMCハードウェアと通信
        todo!("Real IPMI call")
    }
}

// モック実装（テストで使用）
struct MockIpmi {
    responses: std::collections::HashMap<u8, Vec<u8>>,
}
impl IpmiTransport for MockIpmi {
    fn send_command(&self, cmd: u8, _data: &[u8]) -> Result<Vec<u8>, String> {
        self.responses.get(&cmd)
            .cloned()
            .ok_or_else(|| format!("No mock response for cmd 0x{cmd:02x}"))
    }
}

// 実装とモックの双方で動作するジェネリックな関数
fn read_sensor_temperature(transport: &dyn IpmiTransport) -> Result<f64, String> {
    let response = transport.send_command(0x2D, &[])?;
    if response.len() < 2 {
        return Err("Response too short".into());
    }
    Ok(response[0] as f64 + (response[1] as f64 / 256.0))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_temperature_reading() {
        let mut mock = MockIpmi { responses: std::collections::HashMap::new() };
        mock.responses.insert(0x2D, vec![72, 128]); // 72.5°C

        let temp = read_sensor_temperature(&mock).unwrap();
        assert!((temp - 72.5).abs() < 0.01);
    }

    #[test]
    fn test_short_response() {
        let mock = MockIpmi { responses: std::collections::HashMap::new() };
        // レスポンスが設定されていない → エラー
        assert!(read_sensor_temperature(&mock).is_err());
    }
}
```

#### `proptest` によるプロパティベーステスト

特定の値だけをテストするのではなく、常に成立すべき**プロパティ（性質）**をテストします:

```rust
// Cargo.toml: [dev-dependencies] proptest = "1"
use proptest::prelude::*;

fn parse_sensor_id(s: &str) -> Option<u32> {
    s.strip_prefix("sensor_")?.parse().ok()
}

fn format_sensor_id(id: u32) -> String {
    format!("sensor_{id}")
}

proptest! {
    #[test]
    fn roundtrip_sensor_id(id in 0u32..10000) {
        // プロパティ: フォーマットした後にパースすると元の値に戻るはず
        let formatted = format_sensor_id(id);
        let parsed = parse_sensor_id(&formatted);
        prop_assert_eq!(parsed, Some(id));
    }

    #[test]
    fn parse_rejects_garbage(s in "[^s].*") {
        // プロパティ: 's' で始まらない文字列はパースに失敗するはず
        let result = parse_sensor_id(&s);
        prop_assert!(result.is_none());
    }
}
```

#### C言語とRustのテスト比較

| C言語でのテスト | Rustでの対応 |
|-----------|----------------|
| `CUnit`, `CMocka`, カスタムフレームワーク | 組み込みの `#[test]` + `cargo test` |
| `setUp()` / `tearDown()` | ビルダー関数 + `Drop` トレイト |
| `#ifdef TEST` によるモック関数 | トレイトベースの依存性注入（DI） |
| `assert(x == y)` | 差分を自動表示する `assert_eq!(x, y)` |
| 個別のテスト実行可能ファイル | 同一バイナリ、`#[cfg(test)]` による条件付きコンパイル |
| `valgrind --leak-check=full ./test` | `cargo test`（デフォルトでメモリ安全） + `cargo miri test` |
| コードカバレッジ: `gcov` / `lcov` | `cargo tarpaulin` または `cargo llvm-cov` |
| テストの検出: 手動登録 | 自動 — すべての `#[test]` 関数が自動検出される |
