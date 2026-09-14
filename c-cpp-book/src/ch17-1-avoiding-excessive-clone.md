## 過剰な clone() の回避

> **学習内容:** Rustにおいて `.clone()` がなぜコードの不吉な臭い（コードスメル）となるのか、不要なコピーを排除するために所有権をどう再設計すべきか、そして所有権の設計上の問題を示す具体的なパターンについて学びます。

- C++出身者にとって、`.clone()` は「とりあえずコピーしておけばいい」という安全なデフォルトのように感じられがちです。しかし、過剰なクローンは所有権に関する設計上の問題を隠蔽し、パフォーマンスを低下させます。
- **経験則**: 借用チェッカを満足させるためだけにクローンしているなら、代わりに所有権の構造を見直すべきである可能性が高いです。

### clone() が不適切なケース

```rust
// 悪い例: 読み取るだけの関数に渡すためだけに String をクローンしている
fn log_message(msg: String) {  // 不要に所有権を要求している
    println!("[LOG] {}", msg);
}
let message = String::from("GPUテストに合格しました");
log_message(message.clone());  // 無駄: 新しい String のためのメモリを丸ごと確保している
log_message(message);           // 元の値が消費される — クローンした意味がなかった
```

```rust
// 良い例: 借用を受け取る — メモリ確保はゼロ
fn log_message(msg: &str) {    // 所有せず借用する
    println!("[LOG] {}", msg);
}
let message = String::from("GPUテストに合格しました");
log_message(&message);          // クローンなし、メモリ確保なし
log_message(&message);          // 再度呼び出し可能 — message は消費されていない
```

### 実例: クローンする代わりに `&str` を返す
```rust
// 実装例: healthcheck.rs — 借用ビューを返す、メモリ確保ゼロ
pub fn serial_or_unknown(&self) -> &str {
    self.serial.as_deref().unwrap_or(UNKNOWN_VALUE)
}

pub fn model_or_unknown(&self) -> &str {
    self.model.as_deref().unwrap_or(UNKNOWN_VALUE)
}
```
C++でこれに相当するものは `const std::string&` や `std::string_view` を返すことですが、C++ではどちらもライフタイムのチェックが行われません。Rustでは、借用チェッカによって返された `&str` が `self` より長く生存できないことが保証されます。

### 実例: 静的文字列スライス — ヒープを一切使用しない
```rust
// 実装例: healthcheck.rs — コンパイル時文字列テーブル
const HBM_SCREEN_RECIPES: &[&str] = &[
    "hbm_ds_ntd", "hbm_ds_ntd_gfx", "hbm_dt_ntd", "hbm_dt_ntd_gfx",
    "hbm_burnin_8h", "hbm_burnin_24h",
];
```
C++では通常 `std::vector<std::string>`（初回使用時にヒープ確保）になるでしょう。Rustの `&'static [&'static str]` は読み取り専用メモリに配置されるため、実行時コストはゼロです。

### clone() が「適切」なケース

| **状況** | **クローンが問題ない理由** | **例** |
|--------------|--------------------|-----------|
| スレッド間共有のための `Arc::clone()` | 参照カウントをインクリメントするだけ（約1ナノ秒）で、データはコピーしない | `let flag = stop_flag.clone();` |
| 生成されたスレッドへのデータの移動 | スレッドが独自のコピーを必要とするため | `let ctx = ctx.clone(); thread::spawn(move \|\| { ... })` |
| `&self` フィールドからの抽出 | 借用からムーブすることはできないため | 所有型 `String` を返す場合の `self.name.clone()` |
| `Option` でラップされた小さな `Copy` 型 | `.clone()` よりも `.copied()` の方が明確 | `Option<&u32>` → `Option<u32>` のための `opt.get(0).copied()` |

### 実例: スレッド共有のための Arc::clone
```rust
// 実装例: workload.rs — Arc::clone は安価（参照カウントをインクリメントするだけ）
let stop_flag = Arc::new(AtomicBool::new(false));
let stop_flag_clone = stop_flag.clone();   // 約1ナノ秒、データコピーなし
let ctx_clone = ctx.clone();               // スレッド内へムーブするためにコンテキストをクローン

let sensor_handle = thread::spawn(move || {
    // ...stop_flag_clone と ctx_clone を使用
});
```

### チェックリスト: クローンすべきか否か？
1. **`String` / `T` の代わりに `&str` / `&T` を受け取ることはできないか？** → クローンせず、借用する
2. **2つの所有者を必要としないように構造を再設計できないか？** → 参照渡しにするか、スコープを利用する
3. **これは `Arc::clone()` か？** → それならば問題ありません。計算量は O(1) です
4. **スレッドやクロージャ内にデータをムーブしようとしているか？** → クローンが必要です
5. **ホットループ内でクローンしているか？** → プロファイリングを行い、借用や `Cow<T>` の使用を検討する

----

## `Cow<'a, T>`: Clone-on-Write — 可能な限り借用し、必要な時だけクローンする

`Cow`（Clone on Write：書き込み時クローン）は、借用参照**または**所有値の**いずれか**を保持する enum です。「可能な限りメモリ確保を避け、変更が必要な場合にのみ確保する」というアプローチを実現するRustの機能です。C++に直接の対応物はありませんが、最も近いのは状況に応じて `const std::string&` を返したり `std::string` を返したりする関数です。

### なぜ `Cow` が存在するのか

```rust
// Cow を使わない場合 — 常に借用するか、常にクローンするかの二者択一になる
fn normalize(s: &str) -> String {          // 常にメモリ確保が発生！
    if s.contains(' ') {
        s.replace(' ', "_")               // 新しい String（メモリ確保が必要）
    } else {
        s.to_string()                     // 不要なメモリ確保！
    }
}

// Cow を使う場合 — 変更がなければ借用し、変更時のみメモリ確保する
use std::borrow::Cow;

fn normalize(s: &str) -> Cow<'_, str> {
    if s.contains(' ') {
        Cow::Owned(s.replace(' ', "_"))    // メモリ確保（変更が必要なため）
    } else {
        Cow::Borrowed(s)                   // メモリ確保ゼロ（そのまま通過）
    }
}
```

### `Cow` の仕組み

```rust
use std::borrow::Cow;

// Cow<'a, str> の本質的な構造:
// enum Cow<'a, str> {
//     Borrowed(&'a str),     // ゼロコストの参照
//     Owned(String),          // ヒープに確保された所有値
// }

fn greet(name: &str) -> Cow<'_, str> {
    if name.is_empty() {
        Cow::Borrowed("stranger")         // 静的文字列 — メモリ確保なし
    } else if name.starts_with(' ') {
        Cow::Owned(name.trim().to_string()) // 変更あり — メモリ確保が必要
    } else {
        Cow::Borrowed(name)               // そのまま通過 — メモリ確保なし
    }
}

fn main() {
    let g1 = greet("Alice");     // Cow::Borrowed("Alice")
    let g2 = greet("");          // Cow::Borrowed("stranger")
    let g3 = greet(" Bob ");     // Cow::Owned("Bob")
    
    // Cow<str> は Deref<Target = str> を実装しているため、&str として扱えます:
    println!("こんにちは、{g1}！");    // 動作する — Cow は自動的に &str へ参照外しされる
    println!("こんにちは、{g2}！");
    println!("こんにちは、{g3}！");
}
```

### 実世界のユースケース: 設定値の正規化

```rust
use std::borrow::Cow;

/// SKU名を正規化する: 空白のトリムと小文字化。
/// 既に正規化されている場合は Cow::Borrowed を返す（メモリ確保ゼロ）。
fn normalize_sku(sku: &str) -> Cow<'_, str> {
    let trimmed = sku.trim();
    if trimmed == sku && sku.chars().all(|c| c.is_lowercase() || !c.is_alphabetic()) {
        Cow::Borrowed(sku)   // 既に正規化済み — メモリ確保なし
    } else {
        Cow::Owned(trimmed.to_lowercase())  // 変更が必要 — メモリ確保
    }
}

fn main() {
    let s1 = normalize_sku("server-x1");   // Borrowed — メモリ確保ゼロ
    let s2 = normalize_sku("  Server-X1 "); // Owned — メモリ確保が発生
    println!("{s1}, {s2}"); // "server-x1, server-x1"
}
```

### `Cow` を使うべき場面

| **状況** | **`Cow` を使うべきか？** |
|--------------|---------------|
| ほとんどの場合に入力をそのまま返す関数 | ✅ Yes — 不要なクローンを回避できる |
| 文字列のパースや正規化（trim, 小文字化, 置換など） | ✅ Yes — 入力がすでに有効な場合が多い |
| 常に変更される（すべてのコードパスでメモリ確保が発生する） | ❌ No — 素直に `String` を返せばよい |
| 単なるパススルー（一切変更しない） | ❌ No — 素直に `&str` を返せばよい |
| 構造体に長期間データを保持する | ❌ No — 所有型の `String` を使用する |

> **C++との比較**: `Cow<str>` は、`std::variant<std::string_view, std::string>` を返す関数に似ています — ただし、自動的な参照外し（Deref）が機能し、値へのアクセスに定型文（ボイラープレート）を必要としません。

----

## `Weak<T>`: 循環参照の解消 — Rustにおける `weak_ptr`

`Weak<T>` は、C++の `std::weak_ptr<T>` に相当するRustの型です。`Rc<T>` または `Arc<T>` の値に対する非所有の参照を保持します。`Weak` 参照が存在していても元の値は解放される可能性があり、値が既に解放されている場合に `upgrade()` を呼び出すと `None` が返されます。

### なぜ `Weak` が存在するのか

2つの値が互いを指し合っている場合、`Rc<T>` や `Arc<T>` は循環参照を引き起こします。参照カウントがゼロにならなくなるため、どちらもドロップされずメモリリークが発生します。`Weak` はこの循環を解消します：

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: String,
    parent: RefCell<Weak<Node>>,      // Weak — 親がドロップされるのを妨げない
    children: RefCell<Vec<Rc<Node>>>,  // Strong — 親が子を所有する
}

impl Node {
    fn new(value: &str) -> Rc<Node> {
        Rc::new(Node {
            value: value.to_string(),
            parent: RefCell::new(Weak::new()),
            children: RefCell::new(Vec::new()),
        })
    }

    fn add_child(parent: &Rc<Node>, child: &Rc<Node>) {
        // 子は親への弱い参照（Weak）を取得（循環なし）
        *child.parent.borrow_mut() = Rc::downgrade(parent);
        // 親は子への強い参照（Strong）を取得
        parent.children.borrow_mut().push(Rc::clone(child));
    }
}

fn main() {
    let root = Node::new("root");
    let child = Node::new("child");
    Node::add_child(&root, &child);

    // upgrade() を介して子から親にアクセス
    if let Some(parent) = child.parent.borrow().upgrade() {
        println!("子の親: {}", parent.value); // "root"
    }
    
    println!("Root の強参照カウント: {}", Rc::strong_count(&root));  // 1
    println!("Root の弱参照カウント: {}", Rc::weak_count(&root));      // 1
}
```

### C++との比較

```cpp
// C++ — shared_ptr の循環を解消するための weak_ptr
struct Node {
    std::string value;
    std::weak_ptr<Node> parent;                  // Weak — 所有権なし
    std::vector<std::shared_ptr<Node>> children;  // Strong — 子を所有

    static auto create(const std::string& v) {
        return std::make_shared<Node>(Node{v, {}, {}});
    }
};

auto root = Node::create("root");
auto child = Node::create("child");
child->parent = root;          // weak_ptr の代入
root->children.push_back(child);

if (auto p = child->parent.lock()) {   // lock() → shared_ptr または nullptr
    std::cout << "親: " << p->value << std::endl;
}
```

| C++ | Rust | 備考 |
|-----|------|-------|
| `shared_ptr<T>` | `Rc<T>`（シングルスレッド） / `Arc<T>`（マルチスレッド） | 同等のセマンティクス |
| `weak_ptr<T>` | `Rc::downgrade()` / `Arc::downgrade()` による `Weak<T>` | 同等のセマンティクス |
| `weak_ptr::lock()` → `shared_ptr` または null | `Weak::upgrade()` → `Option<Rc<T>>` | ドロップ済みなら `None` |
| `shared_ptr::use_count()` | `Rc::strong_count()` | 同じ意味 |

### `Weak` を使うべき場面

| **状況** | **パターン** |
|--------------|-----------|
| 親 ↔ 子のツリー関係 | 親が `Rc<Child>` を保持し、子が `Weak<Parent>` を保持する |
| オブザーバーパターン / イベントリスナー | イベント発生元が `Weak<Observer>` を保持し、オブザーバーが `Rc<Source>` を保持する |
| メモリ解放を妨げないキャッシュ | `HashMap<Key, Weak<Value>>` — エントリが自然に無効化される |
| グラフ構造における循環の解消 | 相互リンクに `Weak` を使い、ツリーのエッジに `Rc`/`Arc` を使う |

> **新規コードにおけるツリー構造には、`Rc/Weak` よりもアリーナパターン**（ケーススタディ2）を優先してください。`Vec<T>` + インデックスによる構成の方がシンプルで高速であり、参照カウントのオーバーヘッドも一切ありません。`Rc/Weak` は動的なライフタイムを持つ所有権の共有が真に必要な場合に使用してください。

----

## Copy 対 Clone、PartialEq 対 Eq — 何をいつ derive すべきか

- **Copy ≈ C++の trivially copyable（独自のコピーコンストラクタ/デストラクタを持たない型）。** `int`、`enum`、単純なPOD構造体のような型であり、コンパイラが自動的にビット単位の `memcpy` を生成します。Rustにおいて `Copy` も同様の概念です。代入 `let b = a;` によって暗黙的なビットコピーが行われ、両方の変数が有効なまま残ります。
- **Clone ≈ C++のコピーコンストラクタ / `operator=` によるディープコピー。** C++のクラスが独自のコピーコンストラクタを持つ場合（例: `std::vector` メンバのディープコピーなど）、Rustでそれに相当するのが `Clone` の実装です。`.clone()` を明示的に呼び出す必要があります — Rustではコストの高いコピーが `=` の背後に暗黙的に隠されることは決してありません。
- **重要な違い:** C++では、単純なコピーもディープコピーも同じ `=` 構文で暗黙的に発生します。Rustでは選択を強制されます。`Copy` 型は暗黙的にコピーされ（安価）、非 `Copy` 型はデフォルトで**ムーブ**され、高コストな複製を作成したい場合は明示的に `.clone()` を呼び出す必要があります。
- 同様に、C++の `operator==` は、`a == a` が常に成り立つ型（整数など）と成り立たない型（NaN を含む `float` など）を区別しません。Rustではこれを `PartialEq` と `Eq` で区別して表現します。

### Copy 対 Clone

| | **Copy** | **Clone** |
|---|---------|----------|
| **動作** | ビット単位の memcpy（暗黙的） | カスタムロジック（明示的な `.clone()`） |
| **発生タイミング** | 代入時: `let b = a;` | `.clone()` を明示的に呼び出した時のみ |
| **コピー/クローン後** | `a` と `b` の双方が有効 | `a` と `b` の双方が有効 |
| **どちらも未実装の場合** | `let b = a;` は `a` を**ムーブ**する（`a` は使用不可） | `let b = a;` は `a` を**ムーブ**する（`a` は使用不可） |
| **実装可能な型** | ヒープデータを持たない型 | 任意の型 |
| **C++における対応** | トリビアルにコピー可能な型 / POD型（独自のコピーコンストラクタなし） | 独自のコピーコンストラクタ（ディープコピー） |

### 実例: Copy — 単純な enum
```rust
// fan_diag/src/sensor.rs より — すべてユニットバリアントで、1バイトに収まる
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize, Default)]
pub enum FanStatus {
    #[default]
    Normal,
    Low,
    High,
    Missing,
    Failed,
    Unknown,
}

let status = FanStatus::Normal;
let copy = status;   // 暗黙的なコピー — status は依然として有効
println!("{:?} {:?}", status, copy);  // 両方とも動作する
```

### 実例: Copy — 整数ペイロードを持つ enum
```rust
// 実装例: healthcheck.rs — u32 ペイロードは Copy なので、enum 全体も Copy にできる
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum HealthcheckStatus {
    Pass,
    ProgramError(u32),
    DmesgError(u32),
    RasError(u32),
    OtherError(u32),
    Unknown,
}
```

### 実例: Clone のみ — ヒープデータを持つ構造体
```rust
// 実装例: components.rs — String があるため Copy にはできない
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FruData {
    pub technology: DeviceTechnology,
    pub physical_location: String,      // ← String: ヒープ確保されるため Copy 不可
    pub expected: bool,
    pub removable: bool,
}
// let a = fru_data;   → ムーブする（fru_data は使用不可）
// let a = fru_data.clone();  → クローンする（fru_data は有効なまま、新たなヒープ確保が発生）
```

### ルール: Copy にできるか？
```text
その型は String, Vec, Box, HashMap,
Rc, Arc などのヒープを所有する型を含んでいるか？
    はい  → Clone のみ（Copy にはできない）
    いいえ → Copy を derive 可能（型が小さければ推奨）
```

### PartialEq 対 Eq

| | **PartialEq** | **Eq** |
|---|--------------|-------|
| **提供される機能** | `==` および `!=` 演算子 | マーカー: 「同値性が反射的である」ことの表明 |
| **反射律（a == a）** | 保証されない | **保証される** |
| **なぜ重要か** | `f32::NAN != f32::NAN` となるため | `HashMap` のキーには `Eq` が**必須** |
| **derive すべき時** | ほぼ常に | その型が `f32`/`f64` フィールドを持たない場合 |
| **C++における対応** | `operator==` | 直接の対応物なし（C++はチェックしない） |

### 実例: Eq — HashMapのキーとして使用
```rust
// hms_trap/src/cpu_handler.rs より — Hash は Eq を要求する
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum CpuFaultType {
    InvalidFaultType,
    CpuCperFatalErr,
    CpuLpddr5UceErr,
    CpuC2CUceFatalErr,
    // ...
}
// 使用例: HashMap<CpuFaultType, FaultHandler>
// HashMapのキーは Eq + Hash でなければならない — PartialEq だけではコンパイルエラーになる
```

### 実例: Eq を実装できない例 — 型が f32 を含む
```rust
// 実装例: types.rs — f32 があるため Eq は不可
#[derive(Debug, Clone, Serialize, Deserialize, Default)]
pub struct TemperatureSensors {
    pub warning_threshold: Option<f32>,   // ← f32 は NaN ≠ NaN となる
    pub critical_threshold: Option<f32>,  // ← Eq を derive できない
    pub sensor_names: Vec<String>,
}
// HashMap のキーとして使用できない。Eq を derive できない。
// 理由: f32::NAN == f32::NAN が false となり、反射律を満たさないため。
```

### PartialOrd 対 Ord

| | **PartialOrd** | **Ord** |
|---|---------------|--------|
| **提供される機能** | `<`, `>`, `<=`, `>=` 演算子 | `.sort()`, `BTreeMap` のキー |
| **全順序か？** | いいえ（比較不可能なペアが存在し得る） | **はい**（あらゆるペアが比較可能） |
| **f32/f64 の場合** | PartialOrd のみ（NaN が順序を壊す） | Ord を derive できない |

### 実例: Ord — 重要度（深刻度）の順位付け
```rust
// hms_trap/src/fault.rs より — バリアントの定義順が重要度を決定する
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum FaultSeverity {
    Info,      // 最低  (判別子 0)
    Warning,   //       (判別子 1)
    Error,     //       (判別子 2)
    Critical,  // 最高  (判別子 3)
}
// FaultSeverity::Info < FaultSeverity::Critical → true
// 可能になる表現: if severity >= FaultSeverity::Error { escalate(); }
```

### 実例: Ord — 診断レベルの比較
```rust
// 実装例: orchestration.rs
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Default)]
pub enum GpuDiagLevel {
    #[default]
    Quick,     // 最低
    Standard,
    Extended,
    Full,      // 最高
}
// 可能になる表現: if requested_level >= GpuDiagLevel::Extended { run_extended_tests(); }
```

### Derive 決定木

```text
                         作成した新しい型
                                │
                    String/Vec/Box を含むか？
                       /              \
                     はい            いいえ
                      │                  │
                 Clone のみ         Clone + Copy
                      │                  │
                f32/f64 を含むか？  f32/f64 を含むか？
                 /          \         /          \
               はい        いいえ    はい        いいえ
                │             │      │             │
          PartialEq       PartialEq  PartialEq  PartialEq
          のみ            + Eq       のみ       + Eq
                           │                      │
                     ソートが必要か？       ソートが必要か？
                       /       \               /       \
                     はい     いいえ         はい     いいえ
                      │          │              │          │
                PartialOrd      完了       PartialOrd     完了
                + Ord                     + Ord
                      │                        │
                 マップのキー            マップのキー
                 として必要か？          として必要か？
                   │                        │
                 + Hash                   + Hash
```

### クイックリファレンス: プロダクション環境で頻出する derive の組み合わせ

| **型のカテゴリ** | **典型的な derive** | **例** |
|-------------------|--------------------|------------|
| 単純なステータス enum | `Copy, Clone, PartialEq, Eq, Default` | `FanStatus` |
| HashMapのキーとして使う enum | `Copy, Clone, PartialEq, Eq, Hash` | `CpuFaultType`, `SelComponent` |
| ソート可能な重要度 enum | `Copy, Clone, PartialEq, Eq, PartialOrd, Ord` | `FaultSeverity`, `GpuDiagLevel` |
| 文字列を含むデータ構造体 | `Clone, Debug, Serialize, Deserialize` | `FruData`, `OverallSummary` |
| シリアライズ可能な設定 | `Clone, Debug, Default, Serialize, Deserialize` | `DiagConfig` |

----
