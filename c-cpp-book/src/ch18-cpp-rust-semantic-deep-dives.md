## C++ から Rust へのセマンティクス詳解

> **学習目標:** 明白な1対1の対応物を持たないC++の概念（4つの名前付きキャスト、SFINAEとトレイト境界、CRTPと関連型、その他移植時によく直面する摩擦点など）について、Rustへの詳細なマッピングを学びます。

以下のセクションでは、明確な1対1のRust対応物を持たないC++の概念をマッピングします。これらの違いは、C++プログラマがコードを移植する際によくつまずくポイントとなります。

### キャストの階層: 4つのC++キャストとRustにおける対応物

C++には4つの名前付きキャストがあります。Rustではこれらを、異なる、より明示的なメカニズムに置き換えています:

```cpp
// C++ のキャスト階層
int i = static_cast<int>(3.14);            // 1. 数値キャスト / アップキャスト
Derived* d = dynamic_cast<Derived*>(base); // 2. 実行時ダウンキャスト
int* p = const_cast<int*>(cp);              // 3. const のキャスト除去
auto* raw = reinterpret_cast<char*>(&obj); // 4. ビットレベルの再解釈
```

| C++ キャスト | Rust での対応物 | 安全性 | 備考 |
|----------|----------------|--------|-------|
| `static_cast` (数値) | `as` キーワード | 安全だが切り捨て/ラップが発生しうる | `let i = 3.14_f64 as i32;` — 3 に切り捨て |
| `static_cast` (数値、チェックあり) | `From`/`Into` | 安全、コンパイル時に検証 | `let i: i32 = 42_u8.into();` — 拡大変換のみ |
| `static_cast` (数値、失敗しうる変換) | `TryFrom`/`TryInto` | 安全、`Result` を返す | `let i: u8 = 300_u16.try_into()?;` — Err を返す |
| `dynamic_cast` (ダウンキャスト) | 列挙型に対する `match` / `Any::downcast_ref` | 安全 | 列挙型にはパターンマッチング、トレイトオブジェクトには `Any` |
| `const_cast` | 対応物なし | | 安全なコードで `&` を `&mut` にキャストする方法はありません。内部可変性には `Cell` や `RefCell` を使用します |
| `reinterpret_cast` | `std::mem::transmute` | **`unsafe`** | ビットパターンを再解釈。ほとんどの場合不適切 — `from_le_bytes()` などを推奨 |

```rust
// Rust での対応物:

// 1. 数値キャスト — as よりも From/Into を優先
let widened: u32 = 42_u8.into();             // 失敗しない拡大変換 — 常にこちらを推奨
let truncated = 300_u16 as u8;                // ⚠ 44 にラップ（桁あふれ）！暗黙のデータ損失
let checked: Result<u8, _> = 300_u16.try_into(); // Err — 安全な失敗しうる変換

// 2. ダウンキャスト: 列挙型（推奨）または Any（型消去が必要な場合）
use std::any::Any;

fn handle_any(val: &dyn Any) {
    if let Some(s) = val.downcast_ref::<String>() {
        println!("文字列を取得: {s}");
    } else if let Some(n) = val.downcast_ref::<i32>() {
        println!("整数を取得: {n}");
    }
}

// 3. "const_cast" → 内部可変性（unsafe 不要）
use std::cell::Cell;
struct Sensor {
    read_count: Cell<u32>,  // &self 経由で変更可能
}
impl Sensor {
    fn read(&self) -> f64 {
        self.read_count.set(self.read_count.get() + 1); // &mut self ではなく &self
        42.0
    }
}

// 4. reinterpret_cast → transmute（ほぼ不要）
// 安全な代替手段を推奨:
let bytes: [u8; 4] = 0x12345678_u32.to_ne_bytes();  // ✅ 安全
let val = u32::from_ne_bytes(bytes);                   // ✅ 安全
// unsafe { std::mem::transmute::<u32, [u8; 4]>(val) } // ❌ 回避すべき
```

> **ガイドライン**: イディオマティックなRustにおいて、`as` の使用は稀であるべきです（拡大変換には `From`/`Into`、縮小変換には `TryFrom`/`TryInto` を使用）。`transmute` は極めて例外的な場合のみに限定し、`const_cast` は内部可変性型によって不要となるため対応物自体が存在しません。

---

### プリプロセッサ → `cfg`、機能フラグ（Feature Flags）、`macro_rules!`

C++は条件付きコンパイル、定数定義、コード生成をプリプロセッサに大きく依存しています。Rustはこれらすべてを第一級言語機能へと置き換えています。

#### `#define` による定数 → `const` または `const fn`

```cpp
// C++
#define MAX_RETRIES 5
#define BUFFER_SIZE (1024 * 64)
#define SQUARE(x) ((x) * (x))  // マクロ — 単なる文字列置換、型安全性なし
```

```rust
// Rust — 型安全、スコープ制限あり、文字列置換ではない
const MAX_RETRIES: u32 = 5;
const BUFFER_SIZE: usize = 1024 * 64;
const fn square(x: u32) -> u32 { x * x }  // コンパイル時に評価

// const コンテキストで使用可能:
const AREA: u32 = square(12);  // コンパイル時に計算
static BUFFER: [u8; BUFFER_SIZE] = [0; BUFFER_SIZE];
```

#### `#ifdef` / `#if` → `#[cfg()]` および `cfg!()`

```cpp
// C++
#ifdef DEBUG
    log_verbose("Step 1 complete");
#endif

#if defined(LINUX) && !defined(ARM)
    use_x86_path();
#else
    use_generic_path();
#endif
```

```rust
// Rust — 属性ベースの条件付きコンパイル
#[cfg(debug_assertions)]
fn log_verbose(msg: &str) { eprintln!("[VERBOSE] {msg}"); }

#[cfg(not(debug_assertions))]
fn log_verbose(_msg: &str) { /* リリースビルドではコンパイル時に除外される */ }

// 条件の組み合わせ:
#[cfg(all(target_os = "linux", target_arch = "x86_64"))]
fn use_x86_path() { /* ... */ }

#[cfg(not(all(target_os = "linux", target_arch = "x86_64")))]
fn use_generic_path() { /* ... */ }

// 実行時チェック（条件判定自体はコンパイル時だが、式の中で使用可能）:
if cfg!(target_os = "windows") {
    println!("Windows で実行中");
}
```

#### Cargo.toml における機能フラグ（Feature Flags）

```toml
# Cargo.toml — #ifdef FEATURE_FOO を置き換える
[features]
default = ["json"]
json = ["dep:serde_json"]       # オプショナルな依存関係
verbose-logging = []            # 追加の依存関係を持たないフラグ
gpu-support = ["dep:cuda-sys"]  # オプショナルなGPUサポート
```

```rust
// 機能フラグに基づく条件付きコード:
#[cfg(feature = "json")]
pub fn parse_config(data: &str) -> Result<Config, Error> {
    serde_json::from_str(data).map_err(Error::from)
}

#[cfg(feature = "verbose-logging")]
macro_rules! verbose {
    ($($arg:tt)*) => { eprintln!("[VERBOSE] {}", format!($($arg)*)); }
}
#[cfg(not(feature = "verbose-logging"))]
macro_rules! verbose {
    ($($arg:tt)*) => { }; // 何も出力しないようコンパイルされる
}
```

#### `#define MACRO(x)` → `macro_rules!`

```cpp
// C++ — 単なる文字列置換、エラーを引き起こしやすいことで悪名高い
#define DIAG_CHECK(cond, msg) \
    do { if (!(cond)) { log_error(msg); return false; } } while(0)
```

```rust
// Rust — 健全（hygienic）、型チェック済み、構文木（AST）を操作
macro_rules! diag_check {
    ($cond:expr, $msg:expr) => {
        if !($cond) {
            log_error($msg);
            return Err(DiagError::CheckFailed($msg.to_string()));
        }
    };
}

fn run_test() -> Result<(), DiagError> {
    diag_check!(temperature < 85.0, "GPU too hot");
    diag_check!(voltage > 0.8, "Rail voltage too low");
    Ok(())
}
```

| C++ プリプロセッサ | Rust での対応物 | メリット |
|-----------------|----------------|-----------|
| `#define PI 3.14` | `const PI: f64 = 3.14;` | 型付けされている、スコープがある、デバッガから参照可能 |
| `#define MAX(a,b) ((a)>(b)?(a):(b))` | `macro_rules!` またはジェネリック関数 `fn max<T: Ord>` | 二重評価のバグが発生しない |
| `#ifdef DEBUG` | `#[cfg(debug_assertions)]` | コンパイラが検査するため、スペルミスのリスクがない |
| `#ifdef FEATURE_X` | `#[cfg(feature = "x")]` | Cargo が機能を管理し、依存関係を認識できる |
| `#include "header.h"` | `mod module;` + `use module::Item;` | インクルードガードが不要、循環インクルードが発生しない |
| `#pragma once` | 不要 | 各 `.rs` ファイルがモジュールとなり、正確に1度だけコンパイルされる |

---

### ヘッダファイルと `#include` → モジュールと `use`

C++のコンパイルモデルは、テキストのインクルード（挿入）を中心に構成されています:

```cpp
// widget.h — Widget を使用するすべての翻訳単位がこれをインクルードする
#pragma once
#include <string>
#include <vector>

class Widget {
public:
    Widget(std::string name);
    void activate();
private:
    std::string name_;
    std::vector<int> data_;
};
```

```cpp
// widget.cpp — 分割された実装定義
#include "widget.h"
Widget::Widget(std::string name) : name_(std::move(name)) {}
void Widget::activate() { /* ... */ }
```

Rustには、**ヘッダファイルも、前方宣言も、インクルードガードも一切ありません**:

```rust
// src/widget.rs — 宣言と定義が単一ファイル内に共存
pub struct Widget {
    name: String,         // デフォルトでプライベート
    data: Vec<i32>,
}

impl Widget {
    pub fn new(name: String) -> Self {
        Widget { name, data: Vec::new() }
    }
    pub fn activate(&self) { /* ... */ }
}
```

```rust
// src/main.rs — モジュールパスによるインポート
mod widget;  // src/widget.rs を含めるようコンパイラに指示
use widget::Widget;

fn main() {
    let w = Widget::new("sensor".to_string());
    w.activate();
}
```

| C++ | Rust | 優れている理由 |
|-----|------|-----------------|
| `#include "foo.h"` | 親モジュールでの `mod foo;` + `use foo::Item;` | テキスト挿入がなく、ODR（単一定義規則）違反が発生しない |
| `#pragma once` / インクルードガード | 不要 | 各 `.rs` ファイルがモジュールであり、1度だけコンパイルされる |
| 前方宣言 | 不要 | コンパイラがクレート全体を把握するため、定義順序は問われない |
| `class Foo;`（不完全型） | 不要 | 宣言と定義の分離が不要 |
| クラスごとの `.h` + `.cpp` | 単一の `.rs` ファイル | 宣言と定義の不一致によるバグが起きない |
| `using namespace std;` | `use std::collections::HashMap;` | 常に明示的 — グローバル名前空間の汚染がない |
| ネストした `namespace a::b` | ネストした `mod a { mod b { } }` または `a/b.rs` | ファイルシステムがモジュールツリーを反映 |

---

### `friend` とアクセス制御 → モジュール可視性

C++では `friend` を使用して特定のクラスや関数にプライベートメンバへのアクセスを許可します。
Rustには `friend` キーワードは存在しません。代わりに、**プライバシーはモジュールスコープ単位**で管理されます:

```cpp
// C++
class Engine {
    friend class Car;   // Car はプライベートメンバにアクセス可能
    int rpm_;
    void set_rpm(int r) { rpm_ = r; }
public:
    int rpm() const { return rpm_; }
};
```

```rust
// Rust — 同一モジュール内のアイテムはすべてのフィールドにアクセス可能（friend 不要）
mod vehicle {
    pub struct Engine {
        rpm: u32,  // モジュールに対してプライベート（構造体に対してではない！）
    }

    impl Engine {
        pub fn new() -> Self { Engine { rpm: 0 } }
        pub fn rpm(&self) -> u32 { self.rpm }
    }

    pub struct Car {
        engine: Engine,
    }

    impl Car {
        pub fn new() -> Self { Car { engine: Engine::new() } }
        pub fn accelerate(&mut self) {
            self.engine.rpm = 3000; // ✅ 同一モジュール — フィールドに直接アクセス可能
        }
        pub fn rpm(&self) -> u32 {
            self.engine.rpm  // ✅ 同一モジュール — プライベートフィールドを読み取り可能
        }
    }
}

fn main() {
    let mut car = vehicle::Car::new();
    car.accelerate();
    // car.engine.rpm = 9000;  // ❌ コンパイルエラー: `engine` はプライベート
    println!("RPM: {}", car.rpm()); // ✅ Car のパブリックメソッド
}
```

| C++ のアクセス指定 | Rust での対応物 | スコープ |
|-----------|----------------|-------|
| `private` | （デフォルト、キーワードなし） | 同一モジュール内からのみアクセス可能 |
| `protected` | 直接の対応物なし | 親モジュールからのアクセスの場合は `pub(super)` を使用 |
| `public` | `pub` | どこからでもアクセス可能 |
| `friend class Foo` | `Foo` を同一モジュール内に配置 | モジュールレベルのプライバシーが friend を代替 |
| — | `pub(crate)` | クレート内全体から可視だが、外部依存クレートからは不可視 |
| — | `pub(super)` | 親モジュールからのみ可視 |
| — | `pub(in crate::path)` | 指定したモジュールサブツリー内からのみ可視 |

> **重要なポイント**: C++のプライバシーはクラス単位です。Rustのプライバシーはモジュール単位です。
> つまり、どの型を同じモジュールに配置するかによってアクセスを制御します。
> 同一モジュールに同居する型同士は、お互いのプライベートフィールドに自由にアクセスできます。

---

### `volatile` → アトミックと `read_volatile` / `write_volatile`

C++において、`volatile` は読み取りや書き込みを最適化によって省略しないようコンパイラに指示します（主にメモリマップドI/Oレジスタなどに使用されます）。**Rustには `volatile` キーワードはありません。**

```cpp
// C++: ハードウェアレジスタに対する volatile
volatile uint32_t* const GPIO_REG = reinterpret_cast<volatile uint32_t*>(0x4002'0000);
*GPIO_REG = 0x01;              // 書き込みが最適化で削除されない
uint32_t val = *GPIO_REG;     // 読み取りが最適化で削除されない
```

```rust
// Rust: 明示的な volatile 操作 — unsafe コード内でのみ使用可能
use std::ptr;

const GPIO_REG: *mut u32 = 0x4002_0000 as *mut u32;

// SAFETY: GPIO_REG は有効なメモリマップド I/O アドレス。
unsafe {
    ptr::write_volatile(GPIO_REG, 0x01);   // 書き込みが最適化で削除されない
    let val = ptr::read_volatile(GPIO_REG); // 読み取りが最適化で削除されない
}
```

**並行処理における共有状態**（C++ で `volatile` が使われがちなもう一つの用途）に対しては、Rustではアトミック（atomics）を使用します:

```cpp
// C++: volatile はスレッドセーフティには不十分（よくある誤り！）
volatile bool stop_flag = false;  // ❌ データ競合 — C++11以降では未定義動作（UB）

// 正しい C++:
std::atomic<bool> stop_flag{false};
```

```rust
// Rust: スレッド間で可変状態を共有する唯一の安全な手段がアトミック
use std::sync::atomic::{AtomicBool, Ordering};

static STOP_FLAG: AtomicBool = AtomicBool::new(false);

// 別のスレッドから:
STOP_FLAG.store(true, Ordering::Release);

// チェック:
if STOP_FLAG.load(Ordering::Acquire) {
    println!("停止処理中");
}
```

| C++ での用途 | Rust での対応物 | 備考 |
|-----------|----------------|-------|
| ハードウェアレジスタに対する `volatile` | `ptr::read_volatile` / `ptr::write_volatile` | `unsafe` が必要 — MMIO に対して正しいアプローチ |
| スレッド間シグナリングに対する `volatile` | `AtomicBool` / `AtomicU32` など | C++ でも volatile をこれに使うのは誤り！ |
| `std::atomic<T>` | `std::sync::atomic::AtomicT` | 同一のセマンティクス、同一のメモリ順序 |
| `std::atomic<T>::load(memory_order_acquire)` | `AtomicT::load(Ordering::Acquire)` | 1対1のマッピング |

---

### `static` 変数 → `static`、`const`、`LazyLock`、`OnceLock`

#### 基本的な `static` と `const`

```cpp
// C++
const int MAX_RETRIES = 5;                    // コンパイル時定数
static std::string CONFIG_PATH = "/etc/app";  // 静的初期化 — 実行順序は未定義！
```

```rust
// Rust
const MAX_RETRIES: u32 = 5;                   // コンパイル時定数、インライン展開される
static CONFIG_PATH: &str = "/etc/app";         // 'static ライフタイム、固定アドレス
```

#### 静的初期化順序の崩壊（Static Initialization Order Fiasco）

C++には、異なる翻訳単位におけるグローバルコンストラクタが**未規定の順序**で実行されるという有名な問題があります。Rustはこの問題を完全に回避しています。`static` 値はコンパイル時定数でなければなりません（コンストラクタ実行なし）。

実行時に初期化されるグローバル変数には、`LazyLock`（Rust 1.80以上）または `OnceLock` を使用します:

```rust
use std::sync::LazyLock;

// C++ の static std::regex に相当 — 初回アクセス時に初期化、スレッドセーフ
static CONFIG_REGEX: LazyLock<regex::Regex> = LazyLock::new(|| {
    regex::Regex::new(r"^[a-z]+_diag$").expect("invalid regex")
});

fn is_valid_diag(name: &str) -> bool {
    CONFIG_REGEX.is_match(name)  // 初回呼び出し時に初期化され、以降の呼び出しは高速
}
```

```rust
use std::sync::OnceLock;

// OnceLock: 1度だけ初期化され、実行時データから設定可能
static DB_CONN: OnceLock<String> = OnceLock::new();

fn init_db(connection_string: &str) {
    DB_CONN.set(connection_string.to_string())
        .expect("DB_CONN はすでに初期化されています");
}

fn get_db() -> &'static str {
    DB_CONN.get().expect("DB が初期化されていません")
}
```

| C++ | Rust | 備考 |
|-----|------|-------|
| `const int X = 5;` | `const X: i32 = 5;` | どちらもコンパイル時。Rustでは型注釈が必須 |
| `constexpr int X = 5;` | `const X: i32 = 5;` | Rust の `const` は常に constexpr 相当 |
| `static int count = 0;` (ファイルスコープ) | `static COUNT: AtomicI32 = AtomicI32::new(0);` | 可変な static には `unsafe` またはアトミックが必要 |
| `static std::string s = "hi";` | `static S: &str = "hi";` または `LazyLock<String>` | 単純なケースでは実行時コンストラクタ不要 |
| `static MyObj obj;` (複雑な初期化) | `static OBJ: LazyLock<MyObj> = LazyLock::new(\|\| { ... });` | スレッドセーフ、遅延初期化、初期化順序問題なし |
| `thread_local` | `thread_local! { static X: Cell<u32> = Cell::new(0); }` | 同一のセマンティクス |

---

### `constexpr` → `const fn`

C++の `constexpr` は、コンパイル時評価を行う関数や変数を示します。Rustでは同じ目的のために `const fn` と `const` を使用します:

```cpp
// C++
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
constexpr int val = factorial(5);  // コンパイル時に計算 → 120
```

```rust
// Rust
const fn factorial(n: u32) -> u32 {
    if n <= 1 { 1 } else { n * factorial(n - 1) }
}
const VAL: u32 = factorial(5);  // コンパイル時に計算 → 120

// 配列サイズや match パターン内でも使用可能:
const LOOKUP: [u32; 5] = [factorial(1), factorial(2), factorial(3),
                           factorial(4), factorial(5)];
```

| C++ | Rust | 備考 |
|-----|------|-------|
| `constexpr int f()` | `const fn f() -> i32` | 同じ目的 — コンパイル時に評価可能 |
| `constexpr` 変数 | `const` 変数 | Rust の `const` は常にコンパイル時 |
| `consteval` (C++20) | 対応物なし | `const fn` は実行時にも実行可能 |
| `if constexpr` (C++17) | 対応物なし（`cfg!` やジェネリクスを使用） | トレイトの特殊化（specialization）等で一部のユースケースを代替 |
| `constinit` (C++20) | const 初期化子を持つ `static` | Rust の `static` はデフォルトで const 初期化が必須 |

> **`const fn` の現在の制約事項**（Rust 1.82時点）:
> - トレイトメソッドは使用不可（const コンテキストで `Vec` の `.len()` などを呼び出せない）
> - ヒープ割り当ては不可（`Box::new` や `Vec::new` は const ではない）
> - ~~浮動小数点演算の不可~~ — **Rust 1.82 で安定化**
> - `for` ループは使用不可（再帰、または手動インデックス付きの `while` ループを使用）

---

### SFINAE と `enable_if` → トレイト境界と `where` 節

C++において、SFINAE（Substitution Failure Is Not An Error: 置き換え失敗はエラーではない）は条件付きジェネリックプログラミングを支える仕組みです。強力ですが、極めて可読性が低いことで悪名高いです。Rustはこれを**トレイト境界（trait bounds）**で完全に置き換えます:

```cpp
// C++: SFINAE ベースの条件付き関数（C++20 以前）
template<typename T,
         std::enable_if_t<std::is_integral_v<T>, int> = 0>
T double_it(T val) { return val * 2; }

template<typename T,
         std::enable_if_t<std::is_floating_point_v<T>, int> = 0>
T double_it(T val) { return val * 2.0; }

// C++20 コンセプト — よりクリーンだが依然として冗長:
template<std::integral T>
T double_it(T val) { return val * 2; }
```

```rust
// Rust: トレイト境界 — 高い可読性、合成可能、優れたエラーメッセージ
use std::ops::Mul;

fn double_it<T: Mul<Output = T> + From<u8>>(val: T) -> T {
    val * T::from(2)
}

// 複雑な境界には where 節を使用:
fn process<T>(val: T) -> String
where
    T: std::fmt::Display + Clone + Send,
{
    format!("処理中: {}", val)
}

// 個別の impl による条件分岐挙動（SFINAE のオーバーロードを代替）:
trait Describable {
    fn describe(&self) -> String;
}

impl Describable for u32 {
    fn describe(&self) -> String { format!("integer: {self}") }
}

impl Describable for f64 {
    fn describe(&self) -> String { format!("float: {self:.2}") }
}
```

| C++ テンプレートメタプログラミング | Rust での対応物 | 可読性 |
|-----------------------------|----------------|-------------|
| `std::enable_if_t<cond>` | `where T: Trait` | 🟢 明瞭な英語構文 |
| `std::is_integral_v<T>` | 数値トレイトまたは特定型に対する境界 | 🟢 `_v` / `_t` のようなサフィックス不要 |
| SFINAE のオーバーロードセット | 具象型ごとに独立した `impl Trait for ConcreteType` ブロック | 🟢 各 impl が完全に独立 |
| `if constexpr (std::is_same_v<T, int>)` | トレイト impl による特殊化 | 🟢 コンパイル時ディスパッチ |
| C++20 `concept` | `trait` | 🟢 意図はほぼ同一 |
| `requires` 節 | `where` 節 | 🟢 配置場所・構文ともに類似 |
| テンプレートの深部でコンパイルエラー発生 | 呼び出し元でトレイト不一致としてコンパイルエラー | 🟢 200行にも及ぶエラーの連鎖がない |

> **重要なポイント**: C++のコンセプト（C++20）は、Rustのトレイトに最も近い概念です。
> C++20のコンセプトに慣れているなら、Rustのトレイトとは、ダックタイピングではなく首尾一貫した実装モデル（トレイト実装）を備え、1.0から第一級言語機能として存在しているコンセプトのようなものだと捉えるとよいでしょう。

---

### `std::function` → 関数ポインタ、`impl Fn`、`Box<dyn Fn>`

C++の `std::function<R(Args...)>` は型消去された呼び出し可能オブジェクト（callable）です。Rustには3つの選択肢があり、それぞれトレードオフが異なります:

```cpp
// C++: すべてをカバーする万能型（ヒープ割り当て、型消去）
#include <functional>
std::function<int(int)> make_adder(int n) {
    return [n](int x) { return x + n; };
}
```

```rust
// Rust 選択肢 1: fn ポインタ — シンプル、キャプチャなし、メモリ割り当てなし
fn add_one(x: i32) -> i32 { x + 1 }
let f: fn(i32) -> i32 = add_one;
println!("{}", f(5)); // 6

// Rust 選択肢 2: impl Fn — 単相化、オーバーヘッドゼロ、環境のキャプチャ可能
fn apply(val: i32, f: impl Fn(i32) -> i32) -> i32 { f(val) }
let n = 10;
let result = apply(5, |x| x + n);  // クロージャが `n` をキャプチャ

// Rust 選択肢 3: Box<dyn Fn> — 型消去、ヒープ割り当て（std::function と同様）
fn make_adder(n: i32) -> Box<dyn Fn(i32) -> i32> {
    Box::new(move |x| x + n)
}
let adder = make_adder(10);
println!("{}", adder(5));  // 15

// 異種（ヘテロジニアス）な呼び出し可能オブジェクトの格納（vector<function<int(int)>> に相当）:
let callbacks: Vec<Box<dyn Fn(i32) -> i32>> = vec![
    Box::new(|x| x + 1),
    Box::new(|x| x * 2),
    Box::new(make_adder(100)),
];
for cb in &callbacks {
    println!("{}", cb(5));  // 6, 10, 105
}
```

| 使用する場面 | C++ での対応物 | Rust での選択 |
|------------|---------------|-------------|
| トップレベル関数、キャプチャなし | 関数ポインタ | `fn(Args) -> Ret` |
| 呼び出し可能オブジェクトを受け取るジェネリック関数 | テンプレートパラメータ | `impl Fn(Args) -> Ret` (静的ディスパッチ) |
| ジェネリクスにおけるトレイト境界 | `template<typename F>` | `F: Fn(Args) -> Ret` |
| 呼び出し可能オブジェクトの保存、型消去 | `std::function<R(Args)>` | `Box<dyn Fn(Args) -> Ret>` |
| 状態を変更するコールバック | 可変（mutable）ラムダを持つ `std::function` | `Box<dyn FnMut(Args) -> Ret>` |
| 1回限りのコールバック（消費される） | ムーブされる `std::function` | `Box<dyn FnOnce(Args) -> Ret>` |

> **パフォーマンスに関する注意**: `impl Fn` はオーバーヘッドがゼロです（C++テンプレートと同様に単相化されます）。
> `Box<dyn Fn>` は `std::function` と同じオーバーヘッド（仮想関数テーブル + ヒープ割り当て）が発生します。
> 異種の呼び出し可能オブジェクトをコレクションに保存する必要がある場合を除き、`impl Fn` を優先してください。

---

### コンテナのマッピング: C++ STL → Rust `std::collections`

| C++ STL コンテナ | Rust での対応物 | 備考 |
|------------------|----------------|-------|
| `std::vector<T>` | `Vec<T>` | ほぼ同一の API。Rust はデフォルトで境界チェックを行う |
| `std::array<T, N>` | `[T; N]` | スタック割り当ての固定長配列 |
| `std::deque<T>` | `std::collections::VecDeque<T>` | リングバッファ。両端での効率的な push/pop |
| `std::list<T>` | `std::collections::LinkedList<T>` | Rust では滅多に使用されない — ほぼ常に `Vec` の方が高速 |
| `std::forward_list<T>` | 対応物なし | `Vec` または `VecDeque` を使用 |
| `std::unordered_map<K, V>` | `std::collections::HashMap<K, V>` | デフォルトで `SipHash` を使用（DoS 攻撃耐性） |
| `std::map<K, V>` | `std::collections::BTreeMap<K, V>` | B木。キーはソート済み。`K: Ord` が必須 |
| `std::unordered_set<T>` | `std::collections::HashSet<T>` | `T: Hash + Eq` が必須 |
| `std::set<T>` | `std::collections::BTreeSet<T>` | ソート済みセット。`T: Ord` が必須 |
| `std::priority_queue<T>` | `std::collections::BinaryHeap<T>` | デフォルトで最大ヒープ（C++ と同様） |
| `std::stack<T>` | `.push()` / `.pop()` を持つ `Vec<T>` | 専用のスタック型は不要 |
| `std::queue<T>` | `.push_back()` / `.pop_front()` を持つ `VecDeque<T>` | 専用のキュー型は不要 |
| `std::string` | `String` | UTF-8 保証、null 終端ではない |
| `std::string_view` | `&str` | 借用された UTF-8 スライス |
| `std::span<T>` (C++20) | `&[T]` / `&mut [T]` | Rust のスライスは 1.0 から第一級の型 |
| `std::tuple<A, B, C>` | `(A, B, C)` | 第一級の言語構文、分解（構造化束縛）可能 |
| `std::pair<A, B>` | `(A, B)` | 単なる2要素のタプル |
| `std::bitset<N>` | 標準ライブラリに対応物なし | `bitvec` クレートまたは `[u8; N/8]` を使用 |

**主な相違点**:
- Rustの `HashMap` / `HashSet` は `K: Hash + Eq` を要求します。C++ではハッシュ化できないキーを指定するとSTLの奥深くでテンプレートエラーが発生しますが、Rustではコンパイラが型レベルでこれを強制します
- `Vec` のインデックスアクセス（`v[i]`）は、範囲外アクセスの際にデフォルトでパニックします。`Option<&T>` を返す `.get(i)` や、境界チェックを完全に回避できるイテレータを使用してください
- `std::multimap` や `std::multiset` に相当する標準型はありません — `HashMap<K, Vec<V>>` や `BTreeMap<K, Vec<V>>` を使用します

---

### 例外安全性 → パニック安全性

C++では、3段階の例外安全性（エイブラハムズの保証: Abrahams guarantees）が定義されています:

| C++ のレベル | 意味 | Rust での対応物 |
|----------|---------|----------------|
| **No-throw（非送出保証）** | 例外を決して投げない | 決してパニックしない（`Result` を返す） |
| **Strong（強い保証 / コミットまたはロールバック）** | 例外が投げられた場合、状態は変化しない | 所有権モデルにより自然に達成される — `?` で早期リターンした場合、構築途中の値は自動破棄される |
| **Basic（基本保証）** | 例外が投げられた場合でも不変条件は維持される | Rustのデフォルト — `Drop` が実行され、リソースリークは発生しない |

#### Rustの所有権モデルがもたらす利点

```rust
// 追加コストなしで強い保証を獲得 — file.write() が失敗しても config は変更されない
fn update_config(config: &mut Config, path: &str) -> Result<(), Error> {
    let new_data = fetch_from_network()?; // Err → 早期リターン、config は手付かずのまま
    let validated = validate(new_data)?;   // Err → 早期リターン、config は手付かずのまま
    *config = validated;                   // 成功時のみ到達（コミット）
    Ok(())
}
```

C++では、強い保証を達成するために手動のロールバック処理や copy-and-swap イディオムが必要です。Rustでは、`?` 演算子による伝播を用いることで、ほとんどのコードでデフォルトで強い保証が得られます。

#### `catch_unwind` — Rust における `catch(...)` 相当の機能

```rust
use std::panic;

// パニックを捕捉（C++ の catch(...) 相当）— 通常は滅多に必要ない
let result = panic::catch_unwind(|| {
    // パニックする可能性のあるコード
    let v = vec![1, 2, 3];
    v[10]  // パニック！（インデックス範囲外）
});

match result {
    Ok(val) => println!("取得: {val}"),
    Err(_) => eprintln!("パニックを捕捉 — クリーンアップ完了"),
}
```

#### `UnwindSafe` — パニック安全な型であることを示すマーカー

```rust
use std::panic::UnwindSafe;

// &mut 配下の型はデフォルトでは UnwindSafe ではない — パニックによって
// 中途半端に変更された状態のまま残される可能性があるため
fn safe_execute<F: FnOnce() + UnwindSafe>(f: F) {
    let _ = std::panic::catch_unwind(f);
}

// コードの安全性を検証済みの場合、AssertUnwindSafe を使ってオーバーライド:
use std::panic::AssertUnwindSafe;
let mut data = vec![1, 2, 3];
let _ = std::panic::catch_unwind(AssertUnwindSafe(|| {
    data.push(4);
}));
```

| C++ の例外パターン | Rust での対応物 |
|-----------------------|-----------------|
| `throw MyException()` | `return Err(MyError::...)`（推奨）または `panic!("...")` |
| `try { } catch (const E& e)` | `match result { Ok(v) => ..., Err(e) => ... }` または `?` |
| `catch (...)` | `std::panic::catch_unwind(...)` |
| `noexcept` | `-> Result<T, E>`（エラーは例外ではなく値） |
| スタック巻き戻し時の RAII クリーンアップ | パニックの巻き戻し（unwinding）中に `Drop::drop()` が実行される |
| `std::uncaught_exceptions()` | `std::thread::panicking()` |
| `-fno-exceptions` コンパイルフラグ | Cargo.toml の [profile] で `panic = "abort"` |

> **要約**: Rustでは、ほとんどのコードが例外の代わりに `Result<T, E>` を使用するため、エラー経路が明示的かつ合成可能になります。`panic!` はバグ（`assert!` の失敗など）のために予約されており、日常的なエラーには使用されません。このため「例外安全性」が問題になることはほとんどなく、所有権システムによって自動的にクリーンアップが行われます。

---

## C++ から Rust への移行パターン

### クイックリファレンス: C++ → Rust イディオム対応表

| **C++ のパターン** | **Rust のイディオム** | **備考** |
|----------------|---------------|----------|
| `class Derived : public Base` | `enum Variant { A {...}, B {...} }` | 閉じた（既知の）集合には列挙型を推奨 |
| `virtual void method() = 0` | `trait MyTrait { fn method(&self); }` | オープン/拡張可能なインターフェースに使用 |
| `dynamic_cast<Derived*>(ptr)` | `match value { Variant::A(data) => ..., }` | 網羅的（exhaustive）、実行時失敗なし |
| `vector<unique_ptr<Base>>` | `Vec<Box<dyn Trait>>` | 真にポリモーフィズムが必要な場合のみ |
| `shared_ptr<T>` | `Rc<T>` または `Arc<T>` | まずは `Box<T>` や所有値を優先 |
| `enable_shared_from_this<T>` | アリーナパターン（`Vec<T>` + インデックス） | 循環参照を完全に排除 |
| 各クラス内の `Base* m_pFramework` | `fn execute(&mut self, ctx: &mut Context)` | ポインタを保持せず、コンテキストを渡す |
| `try { } catch (...) { }` | `match result { Ok(v) => ..., Err(e) => ... }` | 伝播には `?` を使用 |
| `std::optional<T>` | `Option<T>` | `match` が必須、None の処理忘れを防止 |
| `const std::string&` 引数 | `&str` 引数 | `String` と `&str` の両方を受け付け可能 |
| `enum class Foo { A, B, C }` | `enum Foo { A, B, C }` | Rust の列挙型はデータを持つことも可能 |
| `auto x = std::move(obj)` | `let x = obj;` | ムーブがデフォルト、`std::move` は不要 |
| CMake + make + lint | `cargo build / test / clippy / fmt` | 1つのツールですべてを完結 |

### 移行戦略
1. **データ型から始める**: 最初に構造体と列挙型を移植します。これにより所有権について考えることを余儀なくされます
2. **ファクトリを列挙型に変換する**: ファクトリが異なる派生型を生成している場合、多くは `enum` + `match` に置き換えるべきです
3. **神オブジェクト（God Object）を合成構造体に分割する**: 関連するフィールドを目的特化の構造体にまとめます
4. **ポインタを借用に置き換える**: `Base*` のような保持ポインタを、ライフタイム境界を持つ `&'a T` 借用に変換します
5. **`Box<dyn Trait>` は控えめに使用する**: プラグイン機構やテスト用のモックでのみ使用します
6. **コンパイラの案内に従う**: Rustのエラーメッセージは極めて優秀です。注意深く読みましょう
