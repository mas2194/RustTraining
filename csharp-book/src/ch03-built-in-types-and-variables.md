## 変数と可変性

> **学習内容:** Rust の変数宣言と可変性モデル vs C# の `var`/`const`、
> プリミティブ型の対応関係、重要となる `String` と `&str` の違い、型推論、
> そして Rust が C# とは異なる方法で型キャストや型変換を扱う仕組みについて学びます。
>
> **難易度:** 🟢 初級

### C# の変数宣言
```csharp
// C# - 変数はデフォルトで可変
int count = 0;           // 可変
count = 5;               // ✅ 動作する

// readonly フィールド（クラスレベルのみ、ローカル変数には不可）
// readonly int maxSize = 100;  // 初期化後は不変

const int BUFFER_SIZE = 1024; // コンパイル時定数（ローカル変数としてもフィールドとしても機能）
```

### Rust の変数宣言
```rust
// Rust - 変数はデフォルトで不変
let count = 0;           // デフォルトで不変
// count = 5;            // ❌ コンパイルエラー: cannot assign twice to immutable variable（不変変数への再代入不可）

let mut count = 0;       // 明示的に可変
count = 5;               // ✅ 動作する

const BUFFER_SIZE: usize = 1024; // コンパイル時定数
```

### C# 開発者のための重要な発想の転換
```rust
// 'let' は、すべての変数に適用された C# の readonly フィールドのセマンティクスと考えると理解しやすいです
let name = "John";       // readonly フィールドと同様: 一度設定すると変更不可
let mut age = 30;        // int age = 30; に相当

// 変数のシャドーイング（Rust 固有の機能）
let spaces = "   ";      // 文字列（&str）
let spaces = spaces.len(); // ここで数値（usize）になる
// これは可変（mutation）とは異なります - 新しい変数を再宣言しています
```

### 実践例: カウンタ
```csharp
// C# 版
public class Counter
{
    private int value = 0;
    
    public void Increment()
    {
        value++;  // 変更
    }
    
    public int GetValue() => value;
}
```

```rust
// Rust 版
pub struct Counter {
    value: i32,  // デフォルトで非公開
}

impl Counter {
    pub fn new() -> Counter {
        Counter { value: 0 }
    }
    
    pub fn increment(&mut self) {  // 変更には &mut が必要
        self.value += 1;
    }
    
    pub fn get_value(&self) -> i32 {
        self.value
    }
}
```

***

## データ型の比較

### プリミティブ型

| C# の型 | Rust の型 | サイズ | 範囲 |
|---------|-----------|------|-------|
| `byte` | `u8` | 8 ビット | 0 〜 255 |
| `sbyte` | `i8` | 8 ビット | -128 〜 127 |
| `short` | `i16` | 16 ビット | -32,768 〜 32,767 |
| `ushort` | `u16` | 16 ビット | 0 〜 65,535 |
| `int` | `i32` | 32 ビット | -2³¹ 〜 2³¹-1 |
| `uint` | `u32` | 32 ビット | 0 〜 2³²-1 |
| `long` | `i64` | 64 ビット | -2⁶³ 〜 2⁶³-1 |
| `ulong` | `u64` | 64 ビット | 0 〜 2⁶⁴-1 |
| `float` | `f32` | 32 ビット | IEEE 754 |
| `double` | `f64` | 64 ビット | IEEE 754 |
| `bool` | `bool` | 1 ビット | true/false |
| `char` | `char` | 32 ビット | Unicode スカラー値 |

### ポインタサイズ型（重要！）
```csharp
// C# - int は常に 32 ビット
int arrayIndex = 0;
long fileSize = file.Length;
```

```rust
// Rust - サイズ型はポインタサイズ（32 ビットまたは 64 ビット）に一致
let array_index: usize = 0;    // C の size_t に相当
let file_size: u64 = file.len(); // 明示的な 64 ビット
```

### 型推論
```csharp
// C# - var キーワード
var name = "John";        // string
var count = 42;           // int
var price = 29.99;        // double
```

```rust
// Rust - 自動型推論
let name = "John";        // &str（文字列スライス）
let count = 42;           // i32（整数のデフォルト）
let price = 29.99;        // f64（浮動小数点数のデフォルト）

// 明示的な型注釈
let count: u32 = 42;
let price: f32 = 29.99;
```

### 配列とコレクションの概要
```csharp
// C# - 参照型、ヒープ割り当て
int[] numbers = new int[5];        // 固定長
List<int> list = new List<int>();  // 可変長
```

```rust
// Rust - 複数の選択肢
let numbers: [i32; 5] = [1, 2, 3, 4, 5];  // スタック配列、固定長
let mut list: Vec<i32> = Vec::new();       // ヒープベクター、可変長
```

***

## 文字列型: String vs &str

これは C# 開発者にとって最も混乱しやすい概念の1つであるため、丁寧に分解して解説します。

### C# の文字列処理
```csharp
// C# - シンプルな文字列モデル
string name = "John";           // 文字列リテラル
string greeting = "Hello, " + name;  // 文字列連結
string upper = name.ToUpper();  // メソッド呼び出し
```

### Rust の文字列型
```rust
// Rust - 主に2つの文字列型

// 1. &str（文字列スライス）- C# の ReadOnlySpan<char> に類似
let name: &str = "John";        // 文字列リテラル（不変、借用）

// 2. String - StringBuilder や変更可能な文字列に類似
let mut greeting = String::new();       // 空の文字列
greeting.push_str("Hello, ");          // 追加
greeting.push_str(name);               // 追加

// または直接作成
let greeting = String::from("Hello, John");
let greeting = "Hello, John".to_string();  // &str を String に変換
```

### どちらを使うべきか？

| シナリオ | 使用する型 | C# の対応概念 |
|----------|-----|---------------|
| 文字列リテラル | `&str` | `string` リテラル |
| 関数の引数（読み取り専用） | `&str` | `string` または `ReadOnlySpan<char>` |
| 所有権を持つ変更可能な文字列 | `String` | `StringBuilder` |
| 所有権を持つ文字列を返す | `String` | `string` |

### 実践例
```rust
// 任意の文字列型を受け付ける関数
fn greet(name: &str) {  // String と &str の両方を受け付け可能
    println!("Hello, {}!", name);
}

fn main() {
    let literal = "John";                    // &str
    let owned = String::from("Jane");        // String
    
    greet(literal);                          // 動作する
    greet(&owned);                           // 動作する（String を &str として借用）
    greet("Bob");                            // 動作する
}

// 所有権を持つ文字列を返す関数
fn create_greeting(name: &str) -> String {
    format!("Hello, {}!", name)  // format! マクロは String を返す
}
```

### C# 開発者のための捉え方
```rust
// &str は ReadOnlySpan<char> のようなもの - 文字列データへの参照ビュー
// String は自身が所有し変更可能な char[] のようなもの

let borrowed: &str = "I don't own this data";
let owned: String = String::from("I own this data");

// 相互変換
let owned_copy: String = borrowed.to_string();  // 所有するデータとしてコピー
let borrowed_view: &str = &owned;               // 所有するデータから借用
```

***

## 出力と文字列フォーマット

C# 開発者は `Console.WriteLine` や文字列補間（`$""`）を多用します。Rust のフォーマットシステムも同等に強力ですが、代わりにマクロとフォーマット指定子を使用します。

### 基本的な出力
```csharp
// C# の出力
Console.Write("no newline");
Console.WriteLine("with newline");
Console.Error.WriteLine("to stderr");

// 文字列補間（C# 6+）
string name = "Alice";
int age = 30;
Console.WriteLine($"{name} is {age} years old");
```

```rust
// Rust の出力 — すべてマクロ（! に注目）
print!("no newline");              // → 標準出力、改行なし
println!("with newline");           // → 標準出力 ＋ 改行
eprint!("to stderr");              // → 標準エラー出力、改行なし  
eprintln!("to stderr with newline"); // → 標準エラー出力 ＋ 改行

// 文字列フォーマット（$"" 補間に類似）
let name = "Alice";
let age = 30;
println!("{name} is {age} years old");     // インライン変数キャプチャ（Rust 1.58+）
println!("{} is {} years old", name, age); // 位置指定の引数

// format! は出力する代わりに String を返す
let msg = format!("{name} is {age} years old");
```

### フォーマット指定子
```csharp
// C# のフォーマット指定子
Console.WriteLine($"{price:F2}");         // 固定小数点:    29.99
Console.WriteLine($"{count:D5}");         // ゼロ埋め整数:  00042
Console.WriteLine($"{value,10}");         // 右揃え（幅10）
Console.WriteLine($"{value,-10}");        // 左揃え（幅10）
Console.WriteLine($"{hex:X}");            // 16進数:        FF
Console.WriteLine($"{ratio:P1}");         // パーセンテージ: 85.0%
```

```rust
// Rust のフォーマット指定子
println!("{price:.2}");          // 小数点以下2桁: 29.99
println!("{count:05}");          // ゼロ埋め、幅5: 00042
println!("{value:>10}");         // 右揃え、幅10
println!("{value:<10}");         // 左揃え、幅10
println!("{value:^10}");         // 中央揃え、幅10
println!("{hex:#X}");            // プレフィックス付き16進数: 0xFF
println!("{hex:08X}");           // 16進数ゼロ埋め: 000000FF
println!("{bits:#010b}");        // プレフィックス付き2進数: 0b00001010
println!("{big}", big = 1_000_000); // 名前付き引数
```

### Debug 出力 vs Display 出力
```rust
// {:?}  — Debug トレイト（開発者向け、自動導出可能）
// {:#?} — 整形された Debug 出力（インデント付き複数行）
// {}    — Display トレイト（エンドユーザー向け、手動実装が必要）

#[derive(Debug)] // Debug 出力を自動生成
struct Point { x: f64, y: f64 }

let p = Point { x: 1.5, y: 2.7 };

println!("{:?}", p);   // Point { x: 1.5, y: 2.7 }   — コンパクトなデバッグ出力
println!("{:#?}", p);  // Point {                     — 整形デバッグ出力
                        //     x: 1.5,
                        //     y: 2.7,
                        // }
// println!("{}", p);  // ❌ エラー: Point は Display を実装していません

// ユーザー向け出力用に Display を実装:
use std::fmt;

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
println!("{}", p);    // (1.5, 2.7)  — ユーザーフレンドリーな出力
```

```csharp
// C# における対応:
// {:?}  ≈ object.GetType().ToString() またはリフレクションダンプ
// {}    ≈ object.ToString()
// C# では ToString() をオーバーライドしますが、Rust では Display を実装します
```

### クイックリファレンス

| C# | Rust | 出力内容 |
|----|------|--------|
| `Console.WriteLine(x)` | `println!("{x}")` | Display フォーマット |
| `$"{x}"`（文字列補間） | `format!("{x}")` | `String` を返す |
| `x.ToString()` | `x.to_string()` | `Display` トレイトが必要 |
| `ToString()` のオーバーライド | `impl Display` | ユーザー向け出力 |
| デバッガ表示 | `{:?}` または `dbg!(x)` | 開発者向け出力 |
| `String.Format("{0:F2}", x)` | `format!("{x:.2}")` | フォーマットされた `String` |
| `Console.Error.WriteLine` | `eprintln!()` | 標準エラー出力への書き込み |

***

## 型キャストと型変換

C# には暗黙の型変換、明示的なキャスト `(int)x`、そして `Convert.To*()` があります。Rust はより厳格であり、数値の暗黙の型変換は一切存在しません。

### 数値の型変換
```csharp
// C# — 暗黙的および明示的な変換
int small = 42;
long big = small;              // 暗黙的な拡大変換: OK
double d = small;              // 暗黙的な拡大変換: OK
int truncated = (int)3.14;     // 明示的な縮小変換: 3
byte b = (byte)300;            // 暗黙のオーバーフロー（サイレント）: 44

// 安全な変換
if (int.TryParse("42", out int parsed)) { /* ... */ }
```

```rust
// Rust — すべての数値変換は明示的
let small: i32 = 42;
let big: i64 = small as i64;       // 拡大変換: 'as' による明示的な指定
let d: f64 = small as f64;         // 整数から浮動小数点数へ: 明示的
let truncated: i32 = 3.14_f64 as i32; // 縮小変換: 3（切り捨て）
let b: u8 = 300_u16 as u8;        // オーバーフロー: 44 にラップ（C# の unchecked に類似）

// TryFrom を使用した安全な変換
use std::convert::TryFrom;
let safe: Result<u8, _> = u8::try_from(300_u16); // Err — 範囲外
let ok: Result<u8, _>   = u8::try_from(42_u16);  // Ok(42)

// 文字列パース — bool + out 引数ではなく、Result を返す
let parsed: Result<i32, _> = "42".parse::<i32>();   // Ok(42)
let bad: Result<i32, _>    = "abc".parse::<i32>();  // Err(ParseIntError)

// ターボフィッシュ（turbofish）構文を使用:
let n = "42".parse::<f64>().unwrap(); // 42.0
```

### 文字列の型変換
```csharp
// C#
int n = 42;
string s = n.ToString();          // "42"
string formatted = $"{n:X}";
int back = int.Parse(s);          // 42（失敗時は例外スロー）
bool ok = int.TryParse(s, out int result);
```

```rust
// Rust — Display による to_string()、FromStr による parse()
let n: i32 = 42;
let s: String = n.to_string();            // "42"（Display トレイトを使用）
let formatted = format!("{n:X}");         // "2A"
let back: i32 = s.parse().unwrap();       // 42（失敗時はパニック）
let result: Result<i32, _> = s.parse();   // Ok(42) — 安全なバージョン

// &str ↔ String 間の変換（Rust で最も一般的な変換）
let owned: String = "hello".to_string();    // &str → String
let owned2: String = String::from("hello"); // &str → String（同等）
let borrowed: &str = &owned;               // String → &str（借用するだけでコストゼロ）
```

### 参照の型変換（継承キャストは不可！）
```csharp
// C# — アップキャストとダウンキャスト
Animal a = new Dog();              // アップキャスト（暗黙的）
Dog d = (Dog)a;                    // ダウンキャスト（明示的、例外の可能性あり）
if (a is Dog dog) { /* ... */ }    // パターンマッチによる安全なダウンキャスト
```

```rust
// Rust — 継承はなく、アップキャスト/ダウンキャストもありません
// ポリモフィズムにはトレイトオブジェクトを使用します:
let animal: Box<dyn Animal> = Box::new(Dog);

// 「ダウンキャスト」には Any トレイトが必要（滅多に使われません）:
use std::any::Any;
if let Some(dog) = animal_any.downcast_ref::<Dog>() {
    // dog を使用
}
// 実際の実装ではダウンキャストの代わりに列挙型（enum）を使用します:
enum Animal {
    Dog(Dog),
    Cat(Cat),
}
match animal {
    Animal::Dog(d) => { /* d を使用 */ }
    Animal::Cat(c) => { /* c を使用 */ }
}
```

### クイックリファレンス

| C# | Rust | 備考 |
|----|------|-------|
| `(int)x` | `x as i32` | 切り捨て/ラップを伴うキャスト |
| 暗黙的な拡大変換 | `as` の使用が必須 | 暗黙の数値変換は不可 |
| `Convert.ToInt32(x)` | `i32::try_from(x)` | 安全、`Result` を返す |
| `int.Parse(s)` | `s.parse::<i32>().unwrap()` | 失敗時はパニック |
| `int.TryParse(s, out n)` | `s.parse::<i32>()` | `Result<i32, _>` を返す |
| `(Dog)animal` | 利用不可 | enum または `Any` を使用 |
| `as Dog` / `is Dog` | `downcast_ref::<Dog>()` | `Any` トレイト経由。通常は enum を推奨 |

***

## コメントとドキュメント

### 通常のコメント
```csharp
// C# のコメント
// 単一行コメント
/* 複数行
   コメント */

/// <summary>
/// XML ドキュメントコメント
/// </summary>
/// <param name="name">ユーザー名</param>
/// <returns>挨拶文字列</returns>
public string Greet(string name)
{
    return $"Hello, {name}!";
}
```

```rust
// Rust のコメント
// 単一行コメント
/* 複数行
   コメント */

/// ドキュメントコメント（C# の /// に相当）
/// 指定された名前のユーザーへの挨拶文を生成します。
/// 
/// # 引数
/// 
/// * `name` - 文字列スライスとしてのユーザー名
/// 
/// # 戻り値
/// 
/// 挨拶を含む `String`
/// 
/// # 例
/// 
/// ```
/// let greeting = greet("Alice");
/// assert_eq!(greeting, "Hello, Alice!");
/// ```
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

### ドキュメント生成
```bash
# ドキュメントの生成（C# の XML ドキュメントに相当）
cargo doc --open

# ドキュメントテストの実行
cargo test --doc
```

---

## 演習問題

<details>
<summary><strong>🏋️ 演習: 型安全な温度変換</strong> (クリックして展開)</summary>

以下の要件を満たす Rust プログラムを作成してください:
1. 摂氏の絶対零度（`-273.15`）を表す `const` を宣言する
2. これまでに実行された変換回数を記録する `static` カウンタを宣言する（`AtomicU32` を使用）
3. 摂氏から華氏へ変換する関数 `celsius_to_fahrenheit(c: f64) -> f64` を記述する（絶対零度未満の温度は拒絶し、`f64::NAN` を返す）
4. 文字列 `"98.6"` を `f64` にパースし、華氏に変換する処理をシャドーイングを用いて記述する

<details>
<summary>🔑 解答</summary>

```rust
use std::sync::atomic::{AtomicU32, Ordering};

const ABSOLUTE_ZERO_C: f64 = -273.15;
static CONVERSION_COUNT: AtomicU32 = AtomicU32::new(0);

fn celsius_to_fahrenheit(c: f64) -> f64 {
    if c < ABSOLUTE_ZERO_C {
        return f64::NAN;
    }
    CONVERSION_COUNT.fetch_add(1, Ordering::Relaxed);
    c * 9.0 / 5.0 + 32.0
}

fn main() {
    let temp = "98.6";           // &str
    let temp: f64 = temp.parse().unwrap(); // f64 としてシャドーイング
    let temp = celsius_to_fahrenheit(temp); // 華氏としてシャドーイング
    println!("{temp:.1}°F");
    println!("Conversions: {}", CONVERSION_COUNT.load(Ordering::Relaxed));
}
```

</details>
</details>

***
