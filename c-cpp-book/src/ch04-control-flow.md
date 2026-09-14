# Rustの if キーワード

> **学習目標:** Rustの制御フロー構文 — 式としての `if`/`else`、`loop`/`while`/`for`、`match`、そしてそれらがC/C++の対応機能とどう異なるかを学びます。重要なポイント：Rustの制御フローの多くは値を返します。

- Rustにおいて、`if` は実際には式（expression）です。つまり、値の代入に使用できますが、文（statement）のようにも振る舞います。[▶ 試してみる](https://play.rust-lang.org/)

```rust
fn main() {
    let x = 42;
    if x < 42 {
        println!("人生の秘密より小さいです");
    } else if x == 42 {
        println!("人生の秘密と等しいです");
    } else {
        println!("人生の秘密より大きいです");
    }
    let is_secret_of_life = if x == 42 {true} else {false};
    println!("{}", is_secret_of_life);
}
```

# while と for を使ったRustのループ
- `while` キーワードを使用すると、式が真（true）である間ループできます
```rust
fn main() {
    let mut x = 40;
    while x != 42 {
        x += 1;
    }
}
```
- `for` キーワードを使用して、範囲（レンジ）に対する反復処理ができます
```rust
fn main() {
    // 43は出力されません。最後の要素を含めるには 40..=43 を使用します
    for x in 40..43 {
        println!("{}", x);
    } 
}
```

# loop を使ったRustのループ
- `loop` キーワードは、`break` に到達するまでの無限ループを作成します
```rust
fn main() {
    let mut x = 40;
    // ループに任意のラベルを指定するには以下を 'here: loop に変更します
    loop {
        if x == 42 {
            break; // xの値を返すには break x; を使用します
        }
        x += 1;
    }
}
```
- `break` 文には任意の式を含めることができ、`loop` 式全体の評価値として代入できます
- `continue` キーワードを使用すると、`loop` の先頭に戻ることができます
- ループラベルは `break` や `continue` と組み合わせて使用でき、ネストしたループを扱う際に役立ちます

# Rustの式ブロック
- Rustの式ブロックは、単に `{}` で囲まれた一連の式です。ブロックの評価値は、そのブロック内の最後の式になります
```rust
fn main() {
    let x = {
        let y = 40;
        y + 2 // 注: ; は省略する必要があります
    };
    // Pythonスタイルの出力形式に注目してください
    println!("{x}");
}
```
- Rustスタイルでは、これを利用して関数内の `return` キーワードを省略します
```rust
fn is_secret_of_life(x: u32) -> bool {
    // if x == 42 {true} else {false} と同じです
    x == 42 // 注: ; は省略する必要があります
}
fn main() {
    println!("{}", is_secret_of_life(42));
}
```
