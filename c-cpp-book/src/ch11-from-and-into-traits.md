# Rust の From トレイトと Into トレイト

> **学べること:** Rust の型変換トレイト — 失敗しない変換のための `From<T>` と `Into<T>`、失敗する可能性のある変換のための `TryFrom` と `TryInto`。`From` を実装すれば自動的に `Into` も得られます。C++ の型変換演算子や変換コンストラクタを置き換える仕組みです。

- `From` と `Into` は型変換を容易にするための相補的なトレイトです
- 通常、型には `From` トレイトを実装します。`String::from("Rust")` は `&str` を `String` に変換します。
`From<T> for U` が存在する場合、Rust は自動的に `Into<U> for T` も提供するため、`let s: String = "Rust".into();` のようにも記述できます。
```rust
struct Point {x: u32, y: u32}
// タプルから Point を構築
impl From<(u32, u32)> for Point {
    fn from(xy : (u32, u32)) -> Self {
        Point {x : xy.0, y: xy.1}       // タプルの要素を使用して Point を構築
    }
}
fn main() {
    let s = String::from("Rust");
    let x = u32::from(true);
    let p = Point::from((40, 42));
    // let p : Point = (40,42).into(); // 上記の別の表現方法
    println!("s: {s} x:{x} p.x:{} p.y {}", p.x, p.y);   
}
```

# 演習: From と Into
- `Point` から `TransposePoint` という型へ変換するための `From` トレイトを実装してください。`TransposePoint` は `Point` の `x` 要素と `y` 要素を入れ替えます

<details><summary>解答（クリックして展開）</summary>

```rust
struct Point { x: u32, y: u32 }
struct TransposePoint { x: u32, y: u32 }

impl From<Point> for TransposePoint {
    fn from(p: Point) -> Self {
        TransposePoint { x: p.y, y: p.x }
    }
}

fn main() {
    let p = Point { x: 10, y: 20 };
    let tp = TransposePoint::from(p);
    println!("Transposed: x={}, y={}", tp.x, tp.y);  // x=20, y=10

    // .into() を使用 — From が実装されていれば自動的に動作します
    let p2 = Point { x: 3, y: 7 };
    let tp2: TransposePoint = p2.into();
    println!("Transposed: x={}, y={}", tp2.x, tp2.y);  // x=7, y=3
}
// 出力:
// Transposed: x=20, y=10
// Transposed: x=7, y=3
```

</details>

# Rust の Default トレイト
- `Default` を使用して、型のデフォルト値を実装できます
    - 型は `Default` を `derive` マクロで使用するか、カスタム実装を提供できます
```rust
#[derive(Default, Debug)]
struct Point {x: u32, y: u32}
#[derive(Debug)]
struct CustomPoint {x: u32, y: u32}
impl Default for CustomPoint {
    fn default() -> Self {
        CustomPoint {x: 42, y: 42}
    }
}
fn main() {
    let x = Point::default();   // Point{0, 0} を作成
    println!("{x:?}");
    let y = CustomPoint::default();
    println!("{y:?}");
}
```

### Default トレイトの活用法
- `Default` トレイトには次のようなユースケースがあります：
    - 一部のフィールドのみを更新し、残りをデフォルト値で初期化する
    - `unwrap_or_default()` などのメソッドにおける `Option` 型のデフォルトの代替値
```rust
#[derive(Debug)]
struct CustomPoint {x: u32, y: u32}
impl Default for CustomPoint {
    fn default() -> Self {
        CustomPoint {x: 42, y: 42}
    }
}
fn main() {
    let x = CustomPoint::default();
    // y を上書きし、残りの要素はデフォルトのままにする
    let y = CustomPoint {y: 43, ..CustomPoint::default()};
    println!("{x:?} {y:?}");
    let z : Option<CustomPoint> = None;
    // unwrap_or_default() を unwrap() に変更してみてください
    println!("{:?}", z.unwrap_or_default());
}
```

### Rust におけるその他の型変換
- Rust は暗黙の型変換をサポートしておらず、`明示的` な変換には `as` を使用できます
- `as` は縮小型変換（narrowing）などによるデータ損失の可能性があるため、慎重に使用する必要があります。一般的には、可能な限り `into()` または `from()` を使用することが推奨されます
```rust
fn main() {
    let f = 42u8;
    // let g : u32 = f;    // コンパイルエラーになります
    let g = f as u32;      // 動作しますが非推奨です。縮小変換に関するルールの影響を受けます
    let g : u32 = f.into(); // 最も推奨される形式です。絶対に失敗せず、コンパイラによってチェックされます
    // let k : u8 = g.into();  // コンパイルエラーになります。縮小変換はデータ損失を引き起こす可能性があります
    
    // 縮小変換を試みるには try_into を使用する必要があります
    if let Ok(k) = TryInto::<u8>::try_into(g) {
        println!("{k}");
    }
}
```
