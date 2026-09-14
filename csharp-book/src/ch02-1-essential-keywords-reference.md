## C# 開発者のための Rust 重要キーワードリファレンス

> **学習内容:** Rust のキーワードとそれに対応する C# の構文をまとめたクイックリファレンスです —
> 可視性修飾子、所有権キーワード、制御フロー、型定義、パターンマッチング構文について学びます。
>
> **難易度:** 🟢 初級

Rust のキーワードとその役割を理解することで、C# 開発者はよりスムーズに Rust を使いこなせるようになります。

### 可視性とアクセス制御のキーワード

#### C# のアクセス修飾子
```csharp
public class Example
{
    public int PublicField;           // どこからでもアクセス可能
    private int privateField;        // このクラス内からのみ
    protected int protectedField;    // このクラスおよび派生クラスからのみ
    internal int internalField;      // 同一アセンブリ内からのみ
    protected internal int protectedInternalField; // 両方の組み合わせ
}
```

#### Rust の可視性キーワード
```rust
// pub - アイテムを公開する（C# の public に相当）
pub struct PublicStruct {
    pub public_field: i32,           // 公開フィールド
    private_field: i32,              // デフォルトで非公開（キーワードなし）
}

pub mod my_module {
    pub(crate) fn crate_public() {}     // 現在のクレート内でのみ公開（internal に相当）
    pub(super) fn parent_public() {}    // 親モジュールに対して公開
    pub(self) fn self_public() {}       // 現在のモジュール内でのみ公開（非公開と同じ）
    
    pub use super::PublicStruct;        // 再エクスポート（using エイリアスに類似）
}

// C# の protected に直接対応するものはありません - 代わりにコンポジションを使用します
```

### メモリと所有権のキーワード

#### C# のメモリ関連キーワード
```csharp
// ref - 参照渡し
public void Method(ref int value) { value = 10; }

// out - 出力パラメータ
public bool TryParse(string input, out int result) { /* */ }

// in - 読み取り専用参照（C# 7.2+）
public void ReadOnly(in LargeStruct data) { /* data を変更することはできない */ }
```

#### Rust の所有権キーワード
```rust
// & - 不変参照（C# の in パラメータに類似）
fn read_only(data: &Vec<i32>) {
    println!("Length: {}", data.len()); // 読み取り可能、変更不可
}

// &mut - 可変参照（C# の ref パラメータに類似）
fn modify(data: &mut Vec<i32>) {
    data.push(42); // 変更可能
}

// move - クロージャへの値のムーブキャプチャを強制
let data = vec![1, 2, 3];
let closure = move || {
    println!("{:?}", data); // data はクロージャ内にムーブされる
};
// ここではもはや data にアクセスできない

// Box - ヒープ割り当て（参照型に対する C# の new に類似）
let boxed_data = Box::new(42); // ヒープ上に割り当て
```

### 制御フローキーワード

#### C# の制御フロー
```csharp
// return - 値を返して関数を終了
public int GetValue() { return 42; }

// yield return - イテレータパターン
public IEnumerable<int> GetNumbers()
{
    yield return 1;
    yield return 2;
}

// break/continue - ループ制御
foreach (var item in items)
{
    if (item == null) continue;
    if (item.Stop) break;
}
```

#### Rust の制御フローキーワード
```rust
// return - 明示的な return（通常は不要）
fn get_value() -> i32 {
    return 42; // 明示的な return
    // または単に: 42（暗黙的な戻り値）
}

// break/continue - 戻り値を伴うループ制御
fn find_value() -> Option<i32> {
    loop {
        let value = get_next();
        if value < 0 { continue; }
        if value > 100 { break None; }      // 値を伴う break
        if value == 42 { break Some(value); } // 成功時の値を伴う break
    }
}

// loop - 無限ループ（while(true) に相当）
loop {
    if condition { break; }
}

// while - 条件付きループ
while condition {
    // 処理コード
}

// for - イテレータループ
for item in collection {
    // 処理コード
}
```

### 型定義キーワード

#### C# の型キーワード
```csharp
// class - 参照型
public class MyClass { }

// struct - 値型
public struct MyStruct { }

// interface - コントラクト定義
public interface IMyInterface { }

// enum - 列挙型
public enum MyEnum { Value1, Value2 }

// delegate - 関数ポインタ
public delegate void MyDelegate(int value);
```

#### Rust の型キーワード
```rust
// struct - データ構造（C# の class/struct を統合したような存在）
struct MyStruct {
    field: i32,
}

// enum - 代数的データ型（C# の enum より遥かに強力）
enum MyEnum {
    Variant1,
    Variant2(i32),              // データを保持可能
    Variant3 { x: i32, y: i32 }, // 構造体スタイルのバリアント
}

// trait - インターフェース定義（C# の interface に類似するがより強力）
trait MyTrait {
    fn method(&self);
    
    // デフォルト実装（C# 8+ のインターフェースのデフォルトメソッドに類似）
    fn default_method(&self) {
        println!("Default implementation");
    }
}

// type - 型エイリアス（C# の using エイリアスに類似）
type UserId = u32;
type Result<T> = std::result::Result<T, MyError>;

// impl - 実装ブロック（C# に直接の対応なし - メソッドを型定義と分けて定義）
impl MyStruct {
    fn new() -> MyStruct {
        MyStruct { field: 0 }
    }
}

impl MyTrait for MyStruct {
    fn method(&self) {
        println!("Implementation");
    }
}
```

### 関数定義キーワード

#### C# の関数キーワード
```csharp
// static - クラスメソッド
public static void StaticMethod() { }

// virtual - オーバーライド可能
public virtual void VirtualMethod() { }

// override - 基底メソッドをオーバーライド
public override void VirtualMethod() { }

// abstract - 実装が必須
public abstract void AbstractMethod();

// async - 非同期メソッド
public async Task<int> AsyncMethod() { return await SomeTask(); }
```

#### Rust の関数キーワード
```rust
// fn - 関数定義（C# のメソッドに似ているが独立して定義可能）
fn regular_function() {
    println!("Hello");
}

// const fn - コンパイル時関数（C# の const に似ているが関数に適用）
const fn compile_time_function() -> i32 {
    42 // コンパイル時に評価可能
}

// async fn - 非同期関数（C# の async に相当）
async fn async_function() -> i32 {
    some_async_operation().await
}

// unsafe fn - メモリ安全性を損なう可能性のある関数
unsafe fn unsafe_function() {
    // unsafe な操作を実行可能
}

// extern fn - 外部関数インターフェース（FFI）
extern "C" fn c_compatible_function() {
    // C 言語から呼び出し可能
}
```

### 変数宣言キーワード

#### C# の変数キーワード
```csharp
// var - 型推論
var name = "John"; // string と推論される

// const - コンパイル時定数
const int MaxSize = 100;

// readonly - 実行時定数（ローカル変数ではなくフィールドのみ）
// readonly DateTime createdAt = DateTime.Now;

// static - クラスレベル変数
static int instanceCount = 0;
```

#### Rust の変数キーワード
```rust
// let - 変数バインディング（C# の var に相当）
let name = "John"; // デフォルトで不変

// let mut - 可変変数バインディング
let mut count = 0; // 変更可能
count += 1;

// const - コンパイル時定数（C# の const に相当）
const MAX_SIZE: usize = 100;

// static - グローバル変数（C# の static に相当）
static INSTANCE_COUNT: std::sync::atomic::AtomicUsize = 
    std::sync::atomic::AtomicUsize::new(0);
```

### パターンマッチングキーワード

#### C# のパターンマッチング（C# 8+）
```csharp
// switch 式
string result = value switch
{
    1 => "One",
    2 => "Two",
    _ => "Other"
};

// is パターン
if (obj is string str)
{
    Console.WriteLine(str.Length);
}
```

#### Rust のパターンマッチングキーワード
```rust
// match - パターンマッチング（C# の switch に似ているがより強力）
let result = match value {
    1 => "One",
    2 => "Two",
    3..=10 => "Between 3 and 10", // 範囲パターン
    _ => "Other", // ワイルドカード（C# の _ に相当）
};

// if let - 条件付きパターンマッチング
if let Some(value) = optional {
    println!("Got value: {}", value);
}

// while let - パターンマッチングを伴うループ
while let Some(item) = iterator.next() {
    println!("Item: {}", item);
}

// パターンを伴う let - 分割代入
let (x, y) = point; // タプルの分割代入
let Some(value) = optional else {
    return; // パターンに一致しない場合の早期リターン
};
```

### メモリ安全性のキーワード

#### C# のメモリ関連キーワード
```csharp
// unsafe - 安全性チェックを無効化
unsafe
{
    int* ptr = &variable;
    *ptr = 42;
}

// fixed - マネージドメモリを固定（ピン留め）
unsafe
{
    fixed (byte* ptr = array)
    {
        // ptr を使用
    }
}
```

#### Rust の安全性キーワード
```rust
// unsafe - ボローチェッカーを無効化（使用は最小限に！）
unsafe {
    let ptr = &variable as *const i32;
    let value = *ptr; // 生ポインタの参照解決
}

// 生ポインタ型（C# に直接の対応なし - 通常は不要）
let ptr: *const i32 = &42;  // 不変生ポインタ
let ptr: *mut i32 = &mut 42; // 可変生ポインタ
```

### C# には存在しない一般的な Rust キーワード

```rust
// where - ジェネリック制約（C# の where より柔軟）
fn generic_function<T>() 
where 
    T: Clone + Send + Sync,
{
    // T は Clone、Send、Sync トレイトを実装している必要がある
}

// dyn - 動的トレイトオブジェクト（C# の object に似ているが型安全）
let drawable: Box<dyn Draw> = Box::new(Circle::new());

// Self - 実装対象の型自身を参照（C# の this に似ているが型を指す）
impl MyStruct {
    fn new() -> Self { // Self = MyStruct
        Self { field: 0 }
    }
}

// self - メソッドレシーバ
impl MyStruct {
    fn method(&self) { }        // 不変借用
    fn method_mut(&mut self) { } // 可変借用  
    fn consume(self) { }        // 所有権の移動（消費）
}

// crate - 現在のクレートのルートを参照
use crate::models::User; // クレートルートからの絶対パス

// super - 親モジュールを参照
use super::utils; // 親モジュールからインポート
```

### C# 開発者のためのキーワードまとめ

| 用途 | C# | Rust | 主な相違点 |
|---------|----|----|----------------|
| 可視性 | `public`, `private`, `internal` | `pub`, デフォルトは非公開 | `pub(crate)` などでより細やかな制御が可能 |
| 変数 | `var`, `readonly`, `const` | `let`, `let mut`, `const` | デフォルトで不変 |
| 関数 | `method()` | `fn` | 単独の関数を定義可能 |
| 型 | `class`, `struct`, `interface` | `struct`, `enum`, `trait` | enum は代数的データ型 |
| ジェネリクス | `<T> where T : IFoo` | `<T> where T: Foo` | より柔軟な制約の記述が可能 |
| 参照 | `ref`, `out`, `in` | `&`, `&mut` | コンパイル時の借用チェック |
| パターン | `switch`, `is` | `match`, `if let` | 網羅的マッチングが必須 |

***
