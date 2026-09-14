# Rust のジェネリクス

> **学べること:** ジェネリック型パラメータ、単相化（ゼロコストジェネリクス）、トレイト境界、そして C++ テンプレートとの比較（Rust のジェネリクスはより優れたエラーメッセージを提供し、SFINAE が不要です）。

- ジェネリクスを使用すると、同じアルゴリズムやデータ構造を複数のデータ型にわたって再利用できます
    - ジェネリックパラメータは `<>` 内の識別子として記述されます（例: `<T>`）。パラメータには任意の有効な識別子名を使用できますが、通常は簡潔にするために短く保たれます
    - コンパイラはコンパイル時に単相化（monomorphization）を実行します。つまり、出現した `T` のバリエーションごとに新しい型を生成します
```rust
// 型 <T> の left と right から構成される型 <T> のタプルを返す
fn pick<T>(x: u32, left: T, right: T) -> (T, T) {
   if x == 42 {
    (left, right) 
   } else {
    (right, left)
   }
}
fn main() {
    let a = pick(42, true, false);
    let b = pick(42, "hello", "world");
    println!("{a:?}, {b:?}");
}
```

# Rust のジェネリクス
- ジェネリクスはデータ型や関連メソッドにも適用できます。特定の `<T>`（例: `f32` と `u32`）に対して実装を特殊化することも可能です
```rust
#[derive(Debug)] // これについては後で詳しく説明します
struct Point<T> {
    x : T,
    y : T,
}
impl<T> Point<T> {
    fn new(x: T, y: T) -> Self {
        Point {x, y}
    }
    fn set_x(&mut self, x: T) {
         self.x = x;       
    }
    fn set_y(&mut self, y: T) {
         self.y = y;       
    }
}
impl Point<f32> {
    fn is_secret(&self) -> bool {
        self.x == 42.0
    }    
}
fn main() {
    let mut p = Point::new(2, 4); // i32
    let q = Point::new(2.0, 4.0); // f32
    p.set_x(42);
    p.set_y(43);
    println!("{p:?} {q:?} {}", q.is_secret());
}
```

# 演習: ジェネリクス

🟢 **初級 (Starter)**
- `Point` 型を変更し、x と y に 2 つの異なる型（`T` と `U`）を使用できるようにしてください

<details><summary>解答（クリックして展開）</summary>

```rust
#[derive(Debug)]
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn new(x: T, y: U) -> Self {
        Point { x, y }
    }
}

fn main() {
    let p1 = Point::new(42, 3.14);        // Point<i32, f64>
    let p2 = Point::new("hello", true);   // Point<&str, bool>
    let p3 = Point::new(1u8, 1000u64);    // Point<u8, u64>
    println!("{p1:?}");
    println!("{p2:?}");
    println!("{p3:?}");
}
// 出力:
// Point { x: 42, y: 3.14 }
// Point { x: "hello", y: true }
// Point { x: 1, y: 1000 }
```

</details>

### Rust のトレイトとジェネリクスの組み合わせ
- トレイトを使用してジェネリック型に制限（制約 / 境界）を課すことができます
- 制約は、ジェネリック型パラメータの後に `:` を付けるか、`where` 句を使用して指定します。以下では、`ComputeArea` トレイトを実装している任意の型 `T` を受け取るジェネリック関数 `get_area` を定義しています
```rust
    trait ComputeArea {
        fn area(&self) -> u64;
    }
    fn get_area<T: ComputeArea>(t: &T) -> u64 {
        t.area()
    }
```
- [▶ Rust Playground で試す](https://play.rust-lang.org/)

### Rust のトレイトとジェネリクスの組み合わせ
- 複数のトレイト制約を指定することも可能です
```rust
trait Fish {}
trait Mammal {}
struct Shark;
struct Whale;
impl Fish for Shark {}
impl Fish for Whale {}
impl Mammal for Whale {}
fn only_fish_and_mammals<T: Fish + Mammal>(_t: &T) {}
fn main() {
    let w = Whale {};
    only_fish_and_mammals(&w);
    let _s = Shark {};
    // コンパイルエラーになります
    only_fish_and_mammals(&_s);
}
```

### データ型におけるトレイト制約
- トレイト制約はデータ型のジェネリクスと組み合わせることができます
- 次の例では、`PrintDescription` トレイトと、そのトレイトで制約されたメンバを持つジェネリックな `struct` `Shape` を定義しています
```rust
trait PrintDescription {
    fn print_description(&self);
}
struct Shape<S: PrintDescription> {
    shape: S,
}
// PrintDescription を実装する任意の型に対するジェネリックな Shape の実装
impl<S: PrintDescription> Shape<S> {
    fn print(&self) {
        self.shape.print_description();
    }
}
```
- [▶ Rust Playground で試す](https://play.rust-lang.org/)

# 演習: トレイト制約とジェネリクス

🟡 **中級**
- `CipherText` を実装するジェネリックなメンバ `cipher` を持つ `struct` を実装してください
```rust
trait CipherText {
    fn encrypt(&self);
}
// TO DO
//struct Cipher<>

```
- 次に、`cipher` の `encrypt` を呼び出す `encrypt` メソッドを `struct` の `impl` に実装してください
```rust
// TO DO
impl for Cipher<> {}
```
- 続いて、`CipherOne` と `CipherTwo` という 2 つの構造体に `CipherText` を実装してください（中身は `println!()` だけで構いません）。`CipherOne` と `CipherTwo` を作成し、`Cipher` を使ってそれぞれを呼び出してください

<details><summary>解答（クリックして展開）</summary>

```rust
trait CipherText {
    fn encrypt(&self);
}

struct Cipher<T: CipherText> {
    cipher: T,
}

impl<T: CipherText> Cipher<T> {
    fn encrypt(&self) {
        self.cipher.encrypt();
    }
}

struct CipherOne;
struct CipherTwo;

impl CipherText for CipherOne {
    fn encrypt(&self) {
        println!("CipherOne encryption applied");
    }
}

impl CipherText for CipherTwo {
    fn encrypt(&self) {
        println!("CipherTwo encryption applied");
    }
}

fn main() {
    let c1 = Cipher { cipher: CipherOne };
    let c2 = Cipher { cipher: CipherTwo };
    c1.encrypt();
    c2.encrypt();
}
// 出力:
// CipherOne encryption applied
// CipherTwo encryption applied
```

</details>

### Rust の型状態（タイプステート）パターンとジェネリクス
- Rust の型を使用することで、*コンパイル時* にステートマシンの状態遷移を強制できます
    - 例えば、`Idle` と `Flying` という 2 つの状態を持つ `Drone` を考えてみましょう。`Idle` 状態では許可されるメソッドは `takeoff()` のみです。`Flying` 状態では `land()` を許可します
    
- アプローチの 1 つとして、次のようにステートマシンをモデル化することが考えられます
```rust
enum DroneState {
    Idle,
    Flying
}
struct Drone {x: u64, y: u64, z: u64, state: DroneState}  // x, y, z は座標
```
- この方法では、ステートマシンのセマンティクスを強制するために多くの実行時チェックが必要になります — なぜそうなるのかは [▶ 試してみる](https://play.rust-lang.org/) で確認してください

### Rust の型状態パターンとジェネリクスの活用
- ジェネリクスを使用すると、ステートマシンを *コンパイル時* に強制できます。これには `PhantomData<T>` と呼ばれる特殊なジェネリクスを使用する必要があります
- `PhantomData<T>` は `サイズゼロ (zero-sized)` のマーカーデータ型です。ここでは `Idle` と `Flying` の状態を表すために使用しますが、実行時のサイズは `ゼロ` です
- `takeoff` メソッドと `land` メソッドがパラメータとして `self` を受け取っている点に注目してください。これは `消費 (consuming)` と呼ばれます（借用を使用する `&self` との対比）。基本的に、`Drone<Idle>` に対し `takeoff()` を呼び出すと、戻り値として `Drone<Flying>` のみを受け取ることができ、その逆も同様です
```rust
struct Drone<T> {x: u64, y: u64, z: u64, state: PhantomData<T> }
impl Drone<Idle> {
    fn takeoff(self) -> Drone<Flying> {...}
}
impl Drone<Flying> {
    fn land(self) -> Drone<Idle> { ...}
}
```
    - [▶ Rust Playground で試す](https://play.rust-lang.org/)

### 型状態パターンの重要ポイント
- 重要ポイント:
    - 状態は構造体（サイズゼロ）を使って表現できる
    - 状態 `T` を `PhantomData<T>`（サイズゼロ）と組み合わせることができる
    - ステートマシンの特定の段階に対するメソッドの実装は、単に `impl Drone<T>`（または `impl State<T>`）とするだけ
    - ある状態から別の状態へ遷移するには、`self` を消費するメソッドを使用する
    - これにより `ゼロコスト抽象化 (zero-cost abstractions)` が得られる。コンパイラがコンパイル時にステートマシンを強制できるため、状態が正しくない限りメソッドを呼び出すことは不可能になる

### Rust のビルダーパターン
- `self` の消費は、ビルダーパターンにも有用です
- 数十本のピンを持つ GPIO 設定を考えてみましょう。各ピンは High または Low に設定できます（デフォルトは Low）
```rust
#[derive(default)]
enum PinState {
    #[default]
    Low,
    High,
} 
#[derive(default)]
struct GPIOConfig {
    pin0: PinState,
    pin1: PinState
    ... 
}
```
- メソッドチェーンによって GPIO 設定を構築するためにビルダーパターンを使用できます — [▶ 試してみる](https://play.rust-lang.org/)
