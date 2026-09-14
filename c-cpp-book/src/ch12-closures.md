## Rust のクロージャ

> **学べること:** 無名関数としてのクロージャ、3 つのキャプチャトレイト（`Fn`、`FnMut`、`FnOnce`）、`move` クロージャ、そして C++ ラムダとの比較（手動での `[&]` や `[=]` の指定ではなく、自動的なキャプチャ解析が行われます）。

- クロージャは、周囲の環境をキャプチャできる無名関数です
    - C++ における対応物: ラムダ式（`[&](int x) { return x + 1; }`）
    - 主な違い: Rust のクロージャには **3 つ** のキャプチャトレイト（`Fn`、`FnMut`、`FnOnce`）があり、コンパイラが自動的に選択します
    - C++ のキャプチャモード（`[=]`, `[&]`, `[this]`）は手動で指定するため、エラーが発生しやすいです（ダングリング `[&]` など！）
    - Rust の借用チェッカーは、ダングリングキャプチャをコンパイル時に防ぎます
- クロージャは `||` 記号によって識別されます。引数の型は `||` の中に囲まれ、型推論を利用できます
- クロージャはイテレータ（次のトピック）と組み合わせて頻繁に使用されます
```rust
fn add_one(x: u32) -> u32 {
    x + 1
}
fn main() {
    let add_one_v1 = |x : u32| {x + 1}; // 明示的に型を指定
    let add_one_v2 = |x| {x + 1};   // 呼び出し元から型が推論される
    let add_one_v3 = |x| x+1;   // 1 行の関数の場合に許可される記法
    println!("{} {} {} {}", add_one(42), add_one_v1(42), add_one_v2(42), add_one_v3(42) );
}
```


# 演習: クロージャとキャプチャ

🟡 **中級**

- 外側のスコープから `String` をキャプチャし、そこに文字列を追加するクロージャを作成してください（ヒント: `move` を使用）
- クロージャのベクタ `Vec<Box<dyn Fn(i32) -> i32>>` を作成してください。これには、1 を加算する、2 を掛ける、入力を 2 乗するクロージャを含めます。ベクタをイテレートし、数値 5 に対して各クロージャを適用してください

<details><summary>解答（クリックして展開）</summary>

```rust
fn main() {
    // パート 1: String をキャプチャして追加するクロージャ
    let mut greeting = String::from("Hello");
    let mut append = |suffix: &str| {
        greeting.push_str(suffix);
    };
    append(", world");
    append("!");
    println!("{greeting}");  // "Hello, world!"

    // パート 2: クロージャのベクタ
    let operations: Vec<Box<dyn Fn(i32) -> i32>> = vec![
        Box::new(|x| x + 1),      // 1 を加算
        Box::new(|x| x * 2),      // 2 を乗算
        Box::new(|x| x * x),      // 2 乗
    ];

    let input = 5;
    for (i, op) in operations.iter().enumerate() {
        println!("Operation {i} on {input}: {}", op(input));
    }
}
// 出力:
// Hello, world!
// Operation 0 on 5: 6
// Operation 1 on 5: 10
// Operation 2 on 5: 25
```

</details>

# Rust のイテレータ
- イテレータは Rust の最も強力な機能の 1 つです。フィルタリング（`filter()`）、変換（`map()`）、フィルタリングと変換の同時実行（`filter_map()`）、検索（`find()`）など、コレクションに対する操作を行うための非常にエレガントなメソッドを提供します
- 以下の例において、`|&x| *x >= 42` は同じ比較を実行するクロージャです。`|x| println!("{x}")` も別のクロージャです
```rust
fn main() {
    let a = [0, 1, 2, 3, 42, 43];
    for x in &a {
        if *x >= 42 {
            println!("{x}");
        }
    }
    // 上記と同じ処理
    a.iter().filter(|&x| *x >= 42).for_each(|x| println!("{x}"))
}
```

# Rust のイテレータ
- イテレータの重要な特徴は、そのほとんどが `遅延評価 (lazy)` されることです。つまり、評価されるまで何も実行しません。例えば、`a.iter().filter(|&x| *x >= 42);` は `for_each` がなければ *何も* 行いません。Rust コンパイラはこのような状況を検出すると明示的な警告を発します
```rust
fn main() {
    let a = [0, 1, 2, 3, 42, 43];
    // 各要素に 1 を足して出力
    let _ = a.iter().map(|x|x + 1).for_each(|x|println!("{x}"));
    let found = a.iter().find(|&x|*x == 42);
    println!("{found:?}");
    // 要素数をカウント
    let count = a.iter().count();
    println!("{count}");
}
```

# Rust のイテレータ
- `collect()` メソッドを使用すると、結果を別のコレクションに集約できます
    - 以下において、`Vec<_>` 内の `_` は `map` から返される型のワイルドカードに相当します。例えば、`map` から `String` を返すことも可能です
```rust
fn main() {
    let a = [0, 1, 2, 3, 42, 43];
    let squared_a : Vec<_> = a.iter().map(|x|x*x).collect();
    for x in &squared_a {
        println!("{x}");
    }
    let squared_a_strings : Vec<_> = a.iter().map(|x|(x*x).to_string()).collect();
    // これらは実際には文字列表現です
    for x in &squared_a_strings {
        println!("{x}");
    }
}
```

# 演習: Rust のイテレータ

🟢 **初級 (Starter)**
- 奇数と偶数の要素で構成される整数の配列を作成してください。配列をイテレートし、それぞれ偶数と奇数の要素を含む 2 つの異なるベクタに分割してください
- これは 1 回の走査（ワンパス）で実行できますか？（ヒント: `partition()` を使用）

<details><summary>解答（クリックして展開）</summary>

```rust
fn main() {
    let numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // アプローチ 1: 手動でのイテレーション
    let mut evens = Vec::new();
    let mut odds = Vec::new();
    for n in numbers {
        if n % 2 == 0 {
            evens.push(n);
        } else {
            odds.push(n);
        }
    }
    println!("Evens: {evens:?}");
    println!("Odds:  {odds:?}");

    // アプローチ 2: partition() による 1 パスでの処理
    let (evens, odds): (Vec<i32>, Vec<i32>) = numbers
        .into_iter()
        .partition(|n| *n % 2 == 0);
    println!("Evens (partition): {evens:?}");
    println!("Odds  (partition): {odds:?}");
}
// 出力:
// Evens: [2, 4, 6, 8, 10]
// Odds:  [1, 3, 5, 7, 9]
// Evens (partition): [2, 4, 6, 8, 10]
// Odds  (partition): [1, 3, 5, 7, 9]
```

</details>

> **本番でのパターン**: 本番の Rust コードで使用される実際のイテレータチェーン（`.map().collect()`, `.filter().collect()`, `.find_map()`）については、[クロージャによる代入ピラミッドの解消](ch17-3-collapsing-assignment-pyramids.md#collapsing-assignment-pyramids-with-closures) を参照してください。

### イテレータの強力なツール群: C++ のループを置き換えるメソッド

以下のイテレータアダプタは、本番の Rust コードで *非常によく* 使われます。C++ には `<algorithm>` や C++20 の ranges がありますが、Rust のイテレータチェーンはより合成可能（コンポーザブル）であり、より日常的に使われています。

#### `enumerate` — インデックス + 値（`for (int i = 0; ...)` の置き換え）

```rust
let sensors = vec!["temp0", "temp1", "temp2"];
for (idx, name) in sensors.iter().enumerate() {
    println!("Sensor {idx}: {name}");
}
// Sensor 0: temp0
// Sensor 1: temp1
// Sensor 2: temp2
```

C++ の同等の処理: `for (size_t i = 0; i < sensors.size(); ++i) { auto& name = sensors[i]; ... }`

#### `zip` — 2 つのイテレータの要素をペアにする（並列インデックスループの置き換え）

```rust
let names = ["gpu0", "gpu1", "gpu2"];
let temps = [72.5, 68.0, 75.3];

let report: Vec<String> = names.iter()
    .zip(temps.iter())
    .map(|(name, temp)| format!("{name}: {temp}°C"))
    .collect();
println!("{report:?}");
// ["gpu0: 72.5°C", "gpu1: 68.0°C", "gpu2: 75.3°C"]

// 短い方のイテレータで停止 — 境界外アクセスのリスクなし
```

C++ の同等の処理: `for (size_t i = 0; i < std::min(names.size(), temps.size()); ++i) { ... }`

#### `flat_map` — map + ネストしたコレクションの平坦化

```rust
// 各 GPU は複数の PCIe BDF を持つ。すべての GPU にわたるすべての BDF を収集
let gpu_bdfs = vec![
    vec!["0000:01:00.0", "0000:02:00.0"],
    vec!["0000:41:00.0"],
    vec!["0000:81:00.0", "0000:82:00.0"],
];

let all_bdfs: Vec<&str> = gpu_bdfs.iter()
    .flat_map(|bdfs| bdfs.iter().copied())
    .collect();
println!("{all_bdfs:?}");
// ["0000:01:00.0", "0000:02:00.0", "0000:41:00.0", "0000:81:00.0", "0000:82:00.0"]
```

C++ の同等の処理: 単一の vector に push するネストした `for` ループ。

#### `chain` — 2 つのイテレータの連結

```rust
let critical_gpus = vec!["gpu0", "gpu3"];
let warning_gpus = vec!["gpu1", "gpu5"];

// フラグが立てられたすべての GPU を処理（クリティカルなものを優先）
for gpu in critical_gpus.iter().chain(warning_gpus.iter()) {
    println!("Flagged: {gpu}");
}
```

#### `windows` と `chunks` — スライスに対するスライディング / 固定サイズのビュー

```rust
let temps = [70, 72, 75, 73, 71, 68, 65];

// windows(3): サイズ 3 のスライディングウィンドウ — トレンドを検出
let rising = temps.windows(3)
    .any(|w| w[0] < w[1] && w[1] < w[2]);
println!("Rising trend detected: {rising}"); // true (70 < 72 < 75)

// chunks(2): 固定サイズのグループ — ペアで処理
for pair in temps.chunks(2) {
    println!("Pair: {pair:?}");
}
// Pair: [70, 72]
// Pair: [75, 73]
// Pair: [71, 68]
// Pair: [65]       ← 最後のチャンクは小さくなる場合がある
```

C++ の同等の処理: `i` や `i+1`/`i+2` による手動のインデックス計算。

#### `fold` — 単一の値への集約（`std::accumulate` の置き換え）

```rust
let errors = vec![
    ("gpu0", 3u32),
    ("gpu1", 0),
    ("gpu2", 7),
    ("gpu3", 1),
];

// 総エラー数をカウントし、1 回の走査でサマリーを構築
let (total, summary) = errors.iter().fold(
    (0u32, String::new()),
    |(count, mut s), (name, errs)| {
        if *errs > 0 {
            s.push_str(&format!("{name}:{errs} "));
        }
        (count + errs, s)
    },
);
println!("Total errors: {total}, details: {summary}");
// Total errors: 11, details: gpu0:3 gpu2:7 gpu3:1
```

#### `scan` — 状態を持つ変換（累積合計、差分検出）

```rust
let readings = [100, 105, 103, 110, 108];

// 連続する測定値間の差分を計算
let deltas: Vec<i32> = readings.iter()
    .scan(None::<i32>, |prev, &val| {
        let delta = prev.map(|p| val - p);
        *prev = Some(val);
        Some(delta)
    })
    .flatten()  // 最初の None を除去
    .collect();
println!("Deltas: {deltas:?}"); // [5, -2, 7, -2]
```

#### クイックリファレンス: C++ のループ → Rust のイテレータ

| **C++ パターン** | **Rust のイテレータ** | **例** |
|----------------|------------------|------------|
| `for (int i = 0; i < v.size(); i++)` | `.enumerate()` | `v.iter().enumerate()` |
| インデックスを用いた並列反復 | `.zip()` | `a.iter().zip(b.iter())` |
| ネストしたループ → 平坦な結果 | `.flat_map()` | `vecs.iter().flat_map(\|v\| v.iter())` |
| 2 つのコンテナの連結 | `.chain()` | `a.iter().chain(b.iter())` |
| スライディングウィンドウ `v[i..i+n]` | `.windows(n)` | `v.windows(3)` |
| 固定サイズのグループで処理 | `.chunks(n)` | `v.chunks(4)` |
| `std::accumulate` / 手動のアキュムレータ | `.fold()` | `.fold(init, \|acc, x\| ...)` |
| 累積合計 / 差分追跡 | `.scan()` | `.scan(state, \|s, x\| ...)` |
| `while (it != end && count < n) { ++it; ++count; }` | `.take(n)` | `.iter().take(5)` |
| `while (it != end && !pred(*it)) { ++it; }` | `.skip_while()` | `.skip_while(\|x\| x < &threshold)` |
| `std::any_of` | `.any()` | `.iter().any(\|x\| x > &limit)` |
| `std::all_of` | `.all()` | `.iter().all(\|x\| x.is_valid())` |
| `std::none_of` | `!.any()` | `!iter.any(\|x\| x.failed())` |
| `std::count_if` | `.filter().count()` | `.filter(\|x\| x > &0).count()` |
| `std::min_element` / `std::max_element` | `.min()` / `.max()` | `.iter().max()` → `Option<&T>` |
| `std::unique` | `.dedup()` (ソート済みに対して) | `v.dedup()` (Vec をその場で変更) |

### 演習: イテレータチェーン

`Vec<(String, f64)>`（センサー名、温度）として与えられたセンサーデータに対し、以下を行う **単一のイテレータチェーン** を作成してください：
1. 温度 > 80.0 のセンサーをフィルタリングする
2. 温度の降順にソートする
3. 各要素を `"{name}: {temp}°C [ALARM]"` の形式でフォーマットする
4. `Vec<String>` に集約（collect）する

ヒント: ソートには `Vec` が必要なため、`.sort_by()` の前に `.collect()` を呼び出す必要があります。

<details><summary>解答（クリックして展開）</summary>

```rust
fn alarm_report(sensors: &[(String, f64)]) -> Vec<String> {
    let mut hot: Vec<_> = sensors.iter()
        .filter(|(_, temp)| *temp > 80.0)
        .collect();
    hot.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    hot.iter()
        .map(|(name, temp)| format!("{name}: {temp}°C [ALARM]"))
        .collect()
}

fn main() {
    let sensors = vec![
        ("gpu0".to_string(), 72.5),
        ("gpu1".to_string(), 85.3),
        ("gpu2".to_string(), 91.0),
        ("gpu3".to_string(), 78.0),
        ("gpu4".to_string(), 88.7),
    ];
    for line in alarm_report(&sensors) {
        println!("{line}");
    }
}
// 出力:
// gpu2: 91°C [ALARM]
// gpu4: 88.7°C [ALARM]
// gpu1: 85.3°C [ALARM]
```

</details>

----

# Rust のイテレータ
- `Iterator` トレイトは、ユーザー定義型に対する反復処理を実装するために使用されます (https://doc.rust-lang.org/std/iter/trait.IntoIterator.html)
    - この例では、1, 1, 2, ... で始まり、後続の数値が直前の 2 つの数値の和となるフィボナッチ数列のイテレータを実装します
    - `Iterator` 内の `関連型`（`type Item = u32;`）は、イテレータが出力する型（`u32`）を定義します
    - `next()` メソッドには、イテレータを実装するためのロジックを記述するだけです。この場合、すべての状態情報は `Fibonacci` 構造体内に保持されます
    - より特化したイテレータのために `into_iter()` メソッドを実装する `IntoIterator` という別のトレイトを実装することもできます
    - [▶ Rust Playground で試す](https://play.rust-lang.org/)
