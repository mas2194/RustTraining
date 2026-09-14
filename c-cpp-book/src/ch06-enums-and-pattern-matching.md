# Rustの列挙型（enum）

> **学習目標:** 判別共用体（タグ付き共用体の洗練形）としてのRustのenum、網羅的なパターンマッチングを行う `match`、そしてenumがC++のクラス階層やCのタグ付き共用体をコンパイラが強制する安全性によってどのように置き換えるかを学びます。

- 列挙型（Enum）は判別共用体（discriminated union）です。すなわち、特定のバリアントを識別するタグを備えた、複数の異なる可能性のある型の直和型（sum type）です
    - Cプログラマ向け: Rustのenumはデータを保持できます（適切に設計されたタグ付き共用体 — コンパイラが現在どのアクティブなバリアントであるかを追跡します）
    - C++プログラマ向け: Rustのenumは `std::variant` に似ていますが、網羅的なパターンマッチングを備え、`std::get` のような例外も `std::visit` のようなボイラープレートも不要です
    - `enum` のサイズは、可能性のある最大のバリアントのサイズになります。個々のバリアント同士は無関係であり、完全に異なる型を持つことができます
    - `enum` 型はRustの最も強力な機能の1つであり、C++におけるクラス階層全体を置き換えることができます（ケーススタディで詳しく後述します）
```rust
fn main() {
    enum Numbers {
        Zero,
        SmallNumber(u8),
        BiggerNumber(u32),
        EvenBiggerNumber(u64),
    }
    let a = Numbers::Zero;
    let b = Numbers::SmallNumber(42);
    let c : Numbers = a; // OK -- aの型はNumbersです
    let d : Numbers = b; // OK -- bの型はNumbersです
}
```
----
# Rustの match 式
- Rustの `match` は、C言語の「switch文」を強力に強化したものです
    - `match` は単純なデータ型、`struct`、`enum` に対するパターンマッチングに使用できます
    - `match` 式は網羅的（exhaustive）でなければなりません。つまり、指定された `型` のすべての可能なケースをカバーする必要があります。`_` は「それ以外のすべて」を表すワイルドカードとして使用できます
    - `match` は値を返すことができますが、すべてのアーム（`=>`）は同じ型の値を返す必要があります

```rust
fn main() {
    let x = 42;
    // この場合、_ は明示的にリストされていないすべての数値をカバーします
    let is_secret_of_life = match x {
        42 => true, // 戻り値の型はブール値
        _ => false, // 戻り値の型はブール値
        // 戻り値の型がブール値ではないため、以下はコンパイルエラーになります
        // _ => 0  
    };
    println!("{is_secret_of_life}");
}
```

# Rustの match 式
- `match` は範囲、ブール条件によるフィルタリング、および `if` ガード文をサポートします
```rust
fn main() {
    let x = 42;
    match x {
        // =41 により末尾を含める範囲指定になります
        0..=41 => println!("人生の秘密より小さいです"),
        42 => println!("人生の秘密です"),
        _ => println!("人生の秘密より大きいです"),
    }
    let y = 100;
    match y {
        100 if x == 43 => println!("yは100%人生の秘密ではありません"),
        100 if x == 42 => println!("yは100%人生の秘密です"),
        _ => (),    // 何もしない
    }
}
```

# Rustの match 式
- `match` と `enum` は頻繁に組み合わせて使用されます
    - match式は、バリアントに含まれる値を内部の変数に「束縛（バインド）」できます。値を使用しない場合は `_` を使用します
    - `matches!` マクロを使用すると、特定のバリアントと一致するかどうかを判定できます
```rust
fn main() {
    enum Numbers {
        Zero,
        SmallNumber(u8),
        BiggerNumber(u32),
        EvenBiggerNumber(u64),
    }
    let b = Numbers::SmallNumber(42);
    match b {
        Numbers::Zero => println!("ゼロ"),
        Numbers::SmallNumber(value) => println!("小さな数値 {value}"),
        Numbers::BiggerNumber(_) | Numbers::EvenBiggerNumber(_) => println!("BiggerNumber または EvenBiggerNumber です"),
    }
    
    // 特定のバリアントに対するブール値テスト
    if matches!(b, Numbers::Zero | Numbers::SmallNumber(_)) {
        println!("Zero または小さな数値にマッチしました");
    }
}
```

# Rustの match 式
- `match` は分配束縛やスライスを使用したマッチングも実行できます
```rust
fn main() {
    struct Foo {
        x: (u32, bool),
        y: u32
    }
    let f = Foo {x: (42, true), y: 100};
    match f {
        // x の値を tuple という名前の変数にキャプチャ
        Foo{y: 100, x : tuple} => println!("x にマッチしました: {tuple:?}"),
        _ => ()
    }
    let a = [40, 41, 42];
    match a {
        // スライスの最後の要素が 42 である必要がある。@ はマッチした部分を束縛するために使用される
        [rest @ .., 42] => println!("{rest:?}"),
        // スライスの最初の要素が 42 である必要がある。@ はマッチした部分を束縛するために使用される
        [42, rest @ ..] => println!("{rest:?}"),
        _ => (),
    }
}
```

# 演習: match と enum を使った加算と減算の実装

🟢 **初級課題**

- 符号なし64ビット整数に対する算術演算を実装する関数を記述してください
- **ステップ 1**: 演算を表す enum を定義します:
```rust
enum Operation {
    Add(u64, u64),
    Subtract(u64, u64),
}
```
- **ステップ 2**: 結果を表す enum を定義します:
```rust
enum CalcResult {
    Ok(u64),                    // 成功した結果
    Invalid(String),            // 無効な演算のエラーメッセージ
}
```
- **ステップ 3**: `calculate(op: Operation) -> CalcResult` を実装します
    - Add の場合: Ok(sum) を返す
    - Subtract の場合: 最初の数値 >= 2番目の数値 であれば Ok(差) を返し、そうでなければ Invalid("Underflow") を返す
- **ヒント**: 関数内でパターンマッチングを使用します:
```rust
match op {
    Operation::Add(a, b) => { /* コードを記述 */ },
    Operation::Subtract(a, b) => { /* コードを記述 */ },
}
```

<details><summary>解答（クリックして展開）</summary>

```rust
enum Operation {
    Add(u64, u64),
    Subtract(u64, u64),
}

enum CalcResult {
    Ok(u64),
    Invalid(String),
}

fn calculate(op: Operation) -> CalcResult {
    match op {
        Operation::Add(a, b) => CalcResult::Ok(a + b),
        Operation::Subtract(a, b) => {
            if a >= b {
                CalcResult::Ok(a - b)
            } else {
                CalcResult::Invalid("Underflow".to_string())
            }
        }
    }
}

fn main() {
    match calculate(Operation::Add(10, 20)) {
        CalcResult::Ok(result) => println!("10 + 20 = {result}"),
        CalcResult::Invalid(msg) => println!("エラー: {msg}"),
    }
    match calculate(Operation::Subtract(5, 10)) {
        CalcResult::Ok(result) => println!("5 - 10 = {result}"),
        CalcResult::Invalid(msg) => println!("エラー: {msg}"),
    }
}
// 出力:
// 10 + 20 = 30
// エラー: Underflow
```

</details>

# Rustの関連メソッド
- `impl` を使用して、`struct` や `enum` などの型に関連付けられたメソッドを定義できます
    - メソッドはオプションでパラメータとして `self` を取ることができます。`self` は概念的には、C言語で構造体へのポインタを第1引数として渡すことや、C++の `this` に似ています
    - `self` への参照は、不変（デフォルト: `&self`）、可変（`&mut self`）、または所有権の移動を伴う値渡し（`self`）のいずれかにできます
    - `Self` キーワードは、その型自身を表すショートカットとして使用できます
```rust
struct Point {x: u32, y: u32}
impl Point {
    fn new(x: u32, y: u32) -> Self {
        Point {x, y}
    }
    fn increment_x(&mut self) {
        self.x += 1;
    }
}
fn main() {
    let mut p = Point::new(10, 20);
    p.increment_x();
}
```

# 演習: Pointの加算と変換（add and transform）

🟡 **中級課題** — メソッドシグネチャにおけるムーブと借用の違いを理解している必要があります
- `Point` に対して以下の関連メソッドを実装してください
    - `add()` は別の `Point` への参照を受け取り、その場で x と y の値を加算（インクリメント）します（ヒント: `&mut self` を使用）
    - `transform()` は既存の `Point` を消費し（ヒント: `self` を使用）、x と y を二乗した新しい `Point` を返します

<details><summary>解答（クリックして展開）</summary>

```rust
struct Point { x: u32, y: u32 }

impl Point {
    fn new(x: u32, y: u32) -> Self {
        Point { x, y }
    }
    fn add(&mut self, other: &Point) {
        self.x += other.x;
        self.y += other.y;
    }
    fn transform(self) -> Point {
        Point { x: self.x * self.x, y: self.y * self.y }
    }
}

fn main() {
    let mut p1 = Point::new(2, 3);
    let p2 = Point::new(10, 20);
    p1.add(&p2);
    println!("加算後: x={}, y={}", p1.x, p1.y);           // x=12, y=23
    let p3 = p1.transform();
    println!("変換後: x={}, y={}", p3.x, p3.y);     // x=144, y=529
    // transform() が消費したため、p1 にはアクセスできなくなります
}
```

</details>

----
