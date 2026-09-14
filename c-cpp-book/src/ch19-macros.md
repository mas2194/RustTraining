## Rustのマクロ: プリプロセッサからメタプログラミングへ

> **学習目標:** Rustのマクロの仕組み、関数やジェネリクスではなくマクロを使用すべき場面、そしてマクロがC/C++のプリプロセッサをどのように置き換えるかを学びます。この章を終える頃には、独自の `macro_rules!` マクロを作成できるようになり、`#[derive(Debug)]` が内部で何を行っているかを理解できるようになります。

マクロは、Rustを学び始めて最初に遭遇するもの（1行目の `println!("hello")`）の1つですが、ほとんどの学習コースで解説されるのは一番最後になります。本章ではこの順序を解消し、基礎から解き明かします。

### なぜマクロが存在するのか

Rustにおけるコードの再利用の大部分は、関数とジェネリクスで処理できます。マクロは、型システムだけでは手が届かないギャップを埋める役割を果たします:

| 要求事項 | 関数 / ジェネリクス？ | マクロ？ | 理由 |
|------|-------------------|--------|-----|
| 値の計算 | ✅ `fn max<T: Ord>(a: T, b: T) -> T` | — | 型システムで処理可能 |
| 可変長引数の受け入れ | ❌ Rust には可変長引数関数がない | ✅ `println!("{} {}", a, b)` | マクロは任意の数のトークンを受け取れる |
| 定型的な `impl` ブロックの反復生成 | ❌ ジェネリクス単体では不可能 | ✅ `macro_rules!` | マクロはコンパイル時にコードを生成する |
| コンパイル時のコード実行 | ❌ `const fn` には制限がある | ✅ 手続き的マクロ | コンパイル時に完全な Rust コードを実行できる |
| コードの条件付き取り込み | ❌ | ✅ `#[cfg(...)]` | 属性マクロがコンパイルを制御する |

C/C++のバックグラウンドを持つ方なら、マクロを「*プリプロセッサの唯一の正しい代替手段*」と捉えるとよいでしょう。ただし、生のテキストではなく構文木（AST）に対して操作を行うため、健全（hygienic: 予期せぬ名前の衝突がない）であり、型も認識されます。

> **C開発者への補足:** Rustのマクロは `#define` を完全に置き換えます。テキストベースのプリプロセッサは存在しません。プリプロセッサからRustへの完全なマッピングについては [ch18](ch18-cpp-rust-semantic-deep-dives.md) を参照してください。

---

## `macro_rules!` による宣言的マクロ

宣言的マクロ（「例示によるマクロ: macros by example」とも呼ばれます）は、Rustで最も一般的なマクロの形式です。値に対する `match` と同様に、構文に対するパターンマッチングを行います。

### 基本構文

```rust
macro_rules! say_hello {
    () => {
        println!("Hello!");
    };
}

fn main() {
    say_hello!();  // 展開後: println!("Hello!");
}
```

名前の後ろに付く `!` が、マクロ呼び出しであることを読者（およびコンパイラ）に伝えています。

### 引数を伴うパターンマッチング

マクロはフラグメント指定子（fragment specifiers）を使用して、*トークンツリー*に対してマッチングを行います:

```rust
macro_rules! greet {
    // パターン 1: 引数なし
    () => {
        println!("Hello, world!");
    };
    // パターン 2: 式の引数が1つ
    ($name:expr) => {
        println!("Hello, {}!", $name);
    };
}

fn main() {
    greet!();           // "Hello, world!"
    greet!("Rust");     // "Hello, Rust!"
}
```

#### フラグメント指定子リファレンス

| 指定子 | マッチする対象 | 例 |
|-----------|---------|---------|
| `$x:expr` | 任意の式 | `42`, `a + b`, `foo()` |
| `$x:ty` | 型 | `i32`, `Vec<String>`, `&str` |
| `$x:ident` | 識別子 | `foo`, `my_var` |
| `$x:pat` | パターン | `Some(x)`, `_`, `(a, b)` |
| `$x:stmt` | 文 | `let x = 5;` |
| `$x:block` | ブロック | `{ println!("hi"); 42 }` |
| `$x:literal` | リテラル | `42`, `"hello"`, `true` |
| `$x:tt` | 単一のトークンツリー | 何でもマッチするワイルドカード |
| `$x:item` | アイテム（fn, struct, impl など） | `fn foo() {}` |

### 繰り返し — 最大の強み

C/C++のマクロではループ処理ができません。Rustのマクロではパターンを繰り返すことができます:

```rust
macro_rules! make_vec {
    // カンマ区切りの0個以上の式にマッチ
    ( $( $element:expr ),* ) => {
        {
            let mut v = Vec::new();
            $( v.push($element); )*  // マッチした各要素に対して繰り返す
            v
        }
    };
}

fn main() {
    let v = make_vec![1, 2, 3, 4, 5];
    println!("{v:?}");  // [1, 2, 3, 4, 5]
}
```

`$( ... ),*` という構文は、「このパターンに一致する0個以上の要素がカンマで区切られているものにマッチする」という意味です。展開部の `$( ... )*` は、マッチした要素ごとに本体を1回ずつ繰り返します。

> **標準ライブラリの `vec![]` も、まったく同じ方法で実装されています。** 実際のソースコードは以下のとおりです:
> ```rust
> macro_rules! vec {
>     () => { Vec::new() };
>     ($elem:expr; $n:expr) => { vec::from_elem($elem, $n) };
>     ($($x:expr),+ $(,)?) => { <[_]>::into_vec(Box::new([$($x),+])) };
> }
> ```
> 末尾の `$(,)?` により、末尾のカンマ（トレイリングカンマ）を任意で許容しています。

#### 繰り返し演算子

| 演算子 | 意味 | 例 |
|----------|---------|---------|
| `$( ... )*` | 0個以上 | `vec![]`, `vec![1]`, `vec![1, 2, 3]` |
| `$( ... )+` | 1個以上 | 1個以上の要素が必須 |
| `$( ... )?` | 0個または1個 | 任意の要素（省略可能） |

### 実用例: `hashmap!` コンストラクタ

標準ライブラリには `vec![]` がありますが、`hashmap!{}` はありません。実際に作ってみましょう:

```rust
macro_rules! hashmap {
    ( $( $key:expr => $value:expr ),* $(,)? ) => {
        {
            let mut map = std::collections::HashMap::new();
            $( map.insert($key, $value); )*
            map
        }
    };
}

fn main() {
    let scores = hashmap! {
        "Alice" => 95,
        "Bob" => 87,
        "Carol" => 92,  // $(,)? のおかげで末尾カンマも許容される
    };
    println!("{scores:?}");
}
```

### 実用例: 診断チェックマクロ

組み込みや診断コードでよく見られるパターン — 条件をチェックし、満たさない場合はエラーを返すマクロです:

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum DiagError {
    #[error("Check failed: {0}")]
    CheckFailed(String),
}

macro_rules! diag_check {
    ($cond:expr, $msg:expr) => {
        if !($cond) {
            return Err(DiagError::CheckFailed($msg.to_string()));
        }
    };
}

fn run_diagnostics(temp: f64, voltage: f64) -> Result<(), DiagError> {
    diag_check!(temp < 85.0, "GPU too hot");
    diag_check!(voltage > 0.8, "Rail voltage too low");
    diag_check!(voltage < 1.5, "Rail voltage too high");
    println!("すべてのチェックに合格");
    Ok(())
}
```

> **C/C++ との比較:**
> ```c
> // Cプリプロセッサ — 単なるテキスト置換、型安全性なし、非健全
> #define DIAG_CHECK(cond, msg) \
>     do { if (!(cond)) { log_error(msg); return -1; } } while(0)
> ```
> Rust版は適切な `Result` 型を返し、二重評価のリスクもなく、コンパイラが `$cond` が実際に `bool` 式であるかを検証します。

### 健全性（Hygiene）: なぜRustのマクロは安全なのか

C/C++のマクロのバグは、名前の衝突（シャドーイング）に起因することが多々あります:

```c
// C: 危険 — x が呼び出し側の x をシャドーイングする恐れがある
#define SQUARE(x) ((x) * (x))
int x = 5;
int result = SQUARE(x++);  // 未定義動作（UB）: x が2回インクリメントされる！
```

Rustのマクロは**健全（hygienic）**です。マクロの内部で作成された変数が外に漏れ出すことはありません:

```rust
macro_rules! make_x {
    () => {
        let x = 42;  // この x はマクロ展開のスコープ内に閉じる
    };
}

fn main() {
    let x = 10;
    make_x!();
    println!("{x}");  // 42 ではなく 10 を出力 — 健全性によって衝突が防がれる
}
```

マクロ内の `x` と呼び出し側の `x` は、たとえ同じ名前であっても、コンパイラによって異なる変数として扱われます。**これはCのプリプロセッサでは不可能です。**

---

## 標準ライブラリの一般的なマクロ

第1章からこれらを使ってきましたが、内部で実際に何をしているかをまとめます:

| マクロ | 動作 | 展開内容（簡略化） |
|-------|-------------|------------------------|
| `println!("{}", x)` | フォーマットして標準出力に出力 + 改行 | `std::io::_print(format_args!(...))` |
| `eprintln!("{}", x)` | 標準エラー出力に出力 + 改行 | 同等だが標準エラー出力へ |
| `format!("{}", x)` | フォーマットして `String` を生成 | `String` を割り当てて返す |
| `vec![1, 2, 3]` | 要素を含む `Vec` を作成 | 概ね `Vec::from([1, 2, 3])` 相当 |
| `todo!()` | 未完成のコードを示す | `panic!("not yet implemented")` |
| `unimplemented!()` | 意図的に未実装のコードを示す | `panic!("not implemented")` |
| `unreachable!()` | コンパイラが到達不能を証明できないコードを示す | `panic!("unreachable")` |
| `assert!(cond)` | 条件が false の場合にパニック | `if !cond { panic!(...) }` |
| `assert_eq!(a, b)` | 値が等しくない場合にパニック | 失敗時に両方の値を表示 |
| `dbg!(expr)` | 式と値を標準エラー出力に出力し、その値を返す | `eprintln!("[file:line] expr = {:#?}", &expr); expr` |
| `include_str!("file.txt")` | コンパイル時にファイルの内容を `&str` として埋め込む | コンパイル中にファイルを読み込む |
| `include_bytes!("data.bin")` | コンパイル時にファイルの内容を `&[u8]` として埋め込む | コンパイル中にファイルを読み込む |
| `cfg!(condition)` | コンパイル時の条件を `bool` として評価 | ターゲットに応じて `true` または `false` |
| `env!("VAR")` | コンパイル時に環境変数を読み取る | 設定されていない場合はコンパイルエラー |
| `concat!("a", "b")` | コンパイル時にリテラルを結合する | `"ab"` |

### `dbg!` — 日常的に使用するデバッグマクロ

```rust
fn factorial(n: u32) -> u32 {
    if dbg!(n <= 1) {     // 出力: [src/main.rs:2] n <= 1 = false
        dbg!(1)           // 出力: [src/main.rs:3] 1 = 1
    } else {
        dbg!(n * factorial(n - 1))  // 中間値を出力
    }
}

fn main() {
    dbg!(factorial(4));   // ファイル名:行番号付きですべての再帰呼び出しを出力
}
```

`dbg!` はラップした値をそのまま返すため、プログラムの動作を変更することなくどこにでも挿入できます。標準出力ではなく標準エラー出力に出力されるため、プログラムの正規の出力を妨げません。**コードをコミットする前に、すべての `dbg!` の呼び出しを削除してください。**

### フォーマット文字列の構文

`println!`、`format!`、`eprintln!`、`write!` はすべて同じフォーマット機構を使用しています。以下にクイックリファレンスを示します:

```rust
let name = "sensor";
let value = 3.14159;
let count = 42;

println!("{name}");                    // 変数名による指定（Rust 1.58+）
println!("{}", name);                  // 位置引数
println!("{value:.2}");                // 小数点第2位まで: "3.14"
println!("{count:>10}");               // 右寄せ、幅10: "        42"
println!("{count:0>10}");              // ゼロ埋め: "0000000042"
println!("{count:#06x}");              // プレフィックス付き16進数: "0x002a"
println!("{count:#010b}");             // プレフィックス付き2進数: "0b00101010"
println!("{value:?}");                 // Debug フォーマット
println!("{value:#?}");                // 整形表示（Pretty-print）Debug フォーマット
```

> **C開発者への補足:** これは型安全な `printf` だと考えてください。`{:.2}` が文字列ではなく浮動小数点数に適用されているかをコンパイラがチェックします。`%s` や `%d` のフォーマット不一致によるバグは起きません。
>
> **C++開発者への補足:** これにより、`std::cout << std::fixed << std::setprecision(2) << value` が可読性の高い単一のフォーマット文字列に置き換わります。

---

## 導出マクロ（Derive Macros）

本書のほぼすべての構造体で `#[derive(...)]` を目にしてきました:

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: f64,
    y: f64,
}
```

`#[derive(Debug)]` は**導出マクロ（derive macro）**であり、トレイトの実装を自動生成する特別な手続き的マクロ（procedural macro）の一種です。これが生成するコード（簡略化版）は以下のとおりです:

```rust
// Point に対して #[derive(Debug)] が生成するコード:
impl std::fmt::Debug for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
```

`#[derive(Debug)]` がなければ、すべての構造体に対してこの `impl` ブロックを手作業で書く必要があります。

### よく導出される一般的なトレイト

| Derive | 生成されるもの | 使用する場面 |
|--------|-------------------|-------------|
| `Debug` | `{:?}` フォーマット | ほぼ常に指定 — デバッグ用の出力を可能にする |
| `Clone` | `.clone()` メソッド | 値の複製が必要な場合 |
| `Copy` | 代入時の暗黙のコピー | 小型のスタック限定型（整数型、`[f64; 3]` など） |
| `PartialEq` / `Eq` | `==` および `!=` 演算子 | 等値比較が必要な場合 |
| `PartialOrd` / `Ord` | `<`, `>`, `<=`, `>=` 演算子 | 順序比較が必要な場合 |
| `Hash` | `HashMap`/`HashSet` のキー用ハッシュ化 | マップのキーとして使用される型 |
| `Default` | `Type::default()` コンストラクタ | 意味のあるゼロ値や空値を持つ型 |
| `serde::Serialize` / `Deserialize` | JSON/TOML 等のシリアライゼーション | API の境界を越えるデータ型 |

### 導出（derive）の判断ツリー

```text
derive するべきか？
  │
  ├── その型は、当該トレイトを実装している型のみで構成されているか？
  │     ├── はい → #[derive] が動作する
  │     └── いいえ → 手動で impl を書く（または導出を諦める）
  │
  └── その型の利用者がこの挙動を期待するのは妥当か？
        ├── はい → derive する（Debug, Clone, PartialEq はほぼ常に妥当）
        └── いいえ → derive しない（例: ファイルハンドルを持つ型に Copy を導出しない）
```

> **C++ との比較:** `#[derive(Clone)]` は正しいコピーコンストラクタの自動生成に相当します。`#[derive(PartialEq)]` は各フィールドを比較する `operator==` の自動生成に相当します（これは C++20 の `= default` 宇宙船演算子でようやく提供された機能です）。

---

## 属性マクロ（Attribute Macros）

属性マクロは、付加された対象のアイテムを変換します。すでにいくつかのマクロを使用してきました:

```rust
#[test]                    // 関数をテストとしてマーク
fn test_addition() {
    assert_eq!(2 + 2, 4);
}

#[cfg(target_os = "linux")] // この関数を条件付きで含める
fn linux_only() { /* ... */ }

#[derive(Debug)]            // Debug 実装を生成
struct MyType { /* ... */ }

#[allow(dead_code)]         // コンパイラの警告を抑制
fn unused_helper() { /* ... */ }

#[must_use]                 // 戻り値が破棄された場合に警告
fn compute_checksum(data: &[u8]) -> u32 { /* ... */ }
```

一般的な組み込み属性:

| 属性 | 用途 |
|-----------|---------|
| `#[test]` | テスト関数としてマーク |
| `#[cfg(...)]` | 条件付きコンパイル |
| `#[derive(...)]` | トレイト実装の自動生成 |
| `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` | リント（lint）レベルの制御 |
| `#[must_use]` | 未使用の戻り値に対して警告 |
| `#[inline]` / `#[inline(always)]` | 関数のインライン展開をヒントとして指定 |
| `#[repr(C)]` | C互換のメモリレイアウトを使用（FFI用） |
| `#[no_mangle]` | シンボル名をマングルしない（FFI用） |
| `#[deprecated]` | 任意のメッセージ付きで非推奨としてマーク |

> **C/C++開発者への補足:** 属性は、プリプロセッサディレクティブ（`#pragma`、`__attribute__((...))`）やコンパイラ固有の拡張機能を統合して置き換えるものです。後付けの拡張ではなく、言語文法そのものの一部です。

---

## 手続き的マクロ（概念概要）

手続き的マクロ（「proc macros」）は、コンパイル時に実行されてコードを生成する*独立したRustプログラム*として記述されるマクロです。`macro_rules!` よりも強力ですが、複雑さも増します。

3つの種類があります:

| 種類 | 構文 | 例 | 動作 |
|------|--------|---------|-------------|
| **関数風（Function-like）** | `my_macro!(...)` | `sql!(SELECT * FROM users)` | カスタム構文を解析し、Rustコードを生成 |
| **導出（Derive）** | `#[derive(MyTrait)]` | `#[derive(Serialize)]` | 構造体定義からトレイト実装を生成 |
| **属性（Attribute）** | `#[my_attr]` | `#[tokio::main]`, `#[instrument]` | アノテーションされたアイテムを変換 |

### すでに使用している手続き的マクロ

- `thiserror` の `#[derive(Error)]` — エラー列挙型に対する `Display` および `From` 実装を生成
- `serde` の `#[derive(Serialize, Deserialize)]` — シリアライゼーションコードを生成
- `#[tokio::main]` — `async fn main()` をランタイムのセットアップと `block_on` に変換
- `#[test]` — テストハーネスによって登録される（組み込みの手続き的マクロ）

### 独自の手続き的マクロを作成すべき場面

このコースの中で手続き的マクロを自作する必要はおそらくないでしょう。次のような場合に有用です:
- コンパイル時に構造体のフィールドや列挙型のバリアントを検査する必要がある場合（導出マクロ）
- ドメイン固有言語（DSL）を構築している場合（関数風マクロ）
- 関数のシグネチャを変換する必要がある場合（属性マクロ）

ほとんどのコードでは、`macro_rules!` や通常の関数で十分です。

> **C++ との比較:** 手続き的マクロは、C++においてコードジェネレータ、テンプレートメタプログラミング、`protoc` などの外部ツールが果たしている役割を担います。違いは、手続き的マクロが Cargo のビルドパイプラインの一部として統合されている点です（外部のビルドステップや CMake のカスタムコマンドは不要です）。

---

## 使い分けの基準: マクロ vs 関数 vs ジェネリクス

```text
コードを生成する必要があるか？
  │
  ├── いいえ → 関数またはジェネリック関数を使用する
  │            （よりシンプルで、エラーメッセージも良く、IDEのサポートも得られる）
  │
  └── はい ─┬── 可変個の引数が必要か？
            │     └── はい → macro_rules!（例: println!, vec!）
            │
            ├── 多くの型に対して繰り返しの impl ブロックが必要か？
            │     └── はい → 繰り返し構文を用いた macro_rules!
            │
            ├── 構造体のフィールドを検査する必要があるか？
            │     └── はい → 導出マクロ（手続き的マクロ）
            │
            ├── カスタム構文（DSL）が必要か？
            │     └── はい → 関数風の手続き的マクロ
            │
            └── 関数や構造体を変換する必要があるか？
                  └── はい → 属性型の手続き的マクロ
```

**一般的なガイドライン:** 関数やジェネリクスで実現できるのであれば、マクロは使用しないでください。マクロはエラーメッセージがわかりにくく、マクロ本体内ではIDEの自動補完が効かず、デバッグも難しくなります。

---

## 演習問題

### 🟢 演習 1: `min!` マクロ

以下の要件を満たす `min!` マクロを作成してください:
- `min!(a, b)` は2つの値のうち小さい方を返す
- `min!(a, b, c)` は3つの値のうち最も小さい方を返す
- `PartialOrd` を実装している任意の型で動作する

**ヒント:** `macro_rules!` に2つのマッチアームが必要です。

<details><summary>解答例（クリックして展開）</summary>

```rust
macro_rules! min {
    ($a:expr, $b:expr) => {
        if $a < $b { $a } else { $b }
    };
    ($a:expr, $b:expr, $c:expr) => {
        min!(min!($a, $b), $c)
    };
}

fn main() {
    println!("{}", min!(3, 7));        // 3
    println!("{}", min!(9, 2, 5));     // 2
    println!("{}", min!(1.5, 0.3));    // 0.3
}
```

**注意:** 本番コードでは `std::cmp::min` または `a.min(b)` を優先してください。この演習は複数アームを持つマクロの仕組みを学ぶためのものです。

</details>

### 🟡 演習 2: `hashmap!` を一から作成する

上記の例を見ずに、以下を満たす `hashmap!` マクロを作成してください:
- `key => value` のペアから `HashMap` を生成する
- 末尾のカンマをサポートする
- ハッシュ可能な任意のキー型で動作する

テストコード:
```rust
let m = hashmap! {
    "name" => "Alice",
    "role" => "Engineer",
};
assert_eq!(m["name"], "Alice");
assert_eq!(m.len(), 2);
```

<details><summary>解答例（クリックして展開）</summary>

```rust
use std::collections::HashMap;

macro_rules! hashmap {
    ( $( $key:expr => $val:expr ),* $(,)? ) => {{
        let mut map = HashMap::new();
        $( map.insert($key, $val); )*
        map
    }};
}

fn main() {
    let m = hashmap! {
        "name" => "Alice",
        "role" => "Engineer",
    };
    assert_eq!(m["name"], "Alice");
    assert_eq!(m.len(), 2);
    println!("すべてのテストに合格しました！");
}
```

</details>

### 🟡 演習 3: 浮動小数点比較のための `assert_approx_eq!`

`|a - b| > epsilon` の場合にパニックするマクロ `assert_approx_eq!(a, b, epsilon)` を作成してください。厳密な等値比較が失敗する浮動小数点計算のテストに役立ちます。

テストコード:
```rust
assert_approx_eq!(0.1 + 0.2, 0.3, 1e-10);        // パスするはず
assert_approx_eq!(3.14159, std::f64::consts::PI, 1e-4); // パスするはず
// assert_approx_eq!(1.0, 2.0, 0.5);              // パニックするはず
```

<details><summary>解答例（クリックして展開）</summary>

```rust
macro_rules! assert_approx_eq {
    ($a:expr, $b:expr, $eps:expr) => {
        let (a, b, eps) = ($a as f64, $b as f64, $eps as f64);
        let diff = (a - b).abs();
        if diff > eps {
            panic!(
                "assertion failed: |{} - {}| = {} > {} (epsilon)",
                a, b, diff, eps
            );
        }
    };
}

fn main() {
    assert_approx_eq!(0.1 + 0.2, 0.3, 1e-10);
    assert_approx_eq!(3.14159, std::f64::consts::PI, 1e-4);
    println!("すべての浮動小数点比較に合格しました！");
}
```

</details>

### 🔴 演習 4: `impl_display_for_enum!`

シンプルな C スタイルの列挙型に対して `Display` 実装を生成するマクロを作成してください。以下の入力が与えられたとします:

```rust
impl_display_for_enum! {
    enum Color {
        Red => "red",
        Green => "green",
        Blue => "blue",
    }
}
```

このマクロは、`enum Color { Red, Green, Blue }` の定義と、各バリアントを対応する文字列にマッピングする `impl Display for Color` の両方を生成する必要があります。

**ヒント:** `$( ... ),*` の繰り返しと、複数のフラグメント指定子の両方が必要になります。

<details><summary>解答例（クリックして展開）</summary>

```rust
use std::fmt;

macro_rules! impl_display_for_enum {
    (enum $name:ident { $( $variant:ident => $display:expr ),* $(,)? }) => {
        #[derive(Debug, Clone, Copy, PartialEq)]
        enum $name {
            $( $variant ),*
        }

        impl fmt::Display for $name {
            fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
                match self {
                    $( $name::$variant => write!(f, "{}", $display), )*
                }
            }
        }
    };
}

impl_display_for_enum! {
    enum Color {
        Red => "red",
        Green => "green",
        Blue => "blue",
    }
}

fn main() {
    let c = Color::Green;
    println!("Color: {c}");          // "Color: green"
    println!("Debug: {c:?}");        // "Debug: Green"
    assert_eq!(format!("{}", Color::Red), "red");
    println!("すべてのテストに合格しました！");
}
```

</details>
