## 総合演習

### 演習 1: 型安全な状態機械 ★★ (約30分)

型状態（タイプステート）パターンを用いて信号機の状態機械を構築してください。信号は `Red → Green → Yellow → Red` の順でのみ遷移し、他の順序での遷移は不可能なように設計します。

<details>
<summary>🔑 解答例</summary>

```rust
use std::marker::PhantomData;

struct Red;
struct Green;
struct Yellow;

struct TrafficLight<State> {
    _state: PhantomData<State>,
}

impl TrafficLight<Red> {
    fn new() -> Self {
        println!("🔴 赤 — 停止");
        TrafficLight { _state: PhantomData }
    }

    fn go(self) -> TrafficLight<Green> {
        println!("🟢 青 — 進行");
        TrafficLight { _state: PhantomData }
    }
}

impl TrafficLight<Green> {
    fn caution(self) -> TrafficLight<Yellow> {
        println!("🟡 黄 — 注意");
        TrafficLight { _state: PhantomData }
    }
}

impl TrafficLight<Yellow> {
    fn stop(self) -> TrafficLight<Red> {
        println!("🔴 赤 — 停止");
        TrafficLight { _state: PhantomData }
    }
}

fn main() {
    let light = TrafficLight::new(); // Red
    let light = light.go();          // Green
    let light = light.caution();     // Yellow
    let light = light.stop();        // Red

    // light.caution(); // ❌ コンパイルエラー: Red には caution メソッドが存在しない
    // TrafficLight::new().stop(); // ❌ コンパイルエラー: Red には stop メソッドが存在しない
}
```

**重要ポイント**: 不正な遷移は実行時パニックではなく、コンパイルエラーとして検出されます。

</details>

---

### 演習 2: PhantomData による計量単位システム ★★ (約30分)

第4章の計量単位パターンを拡張し、以下をサポートしてください:
- `Meters`、`Seconds`、`Kilograms`
- 同一単位同士の加算
- 乗算: `Meters * Meters = SquareMeters`
- 除算: `Meters / Seconds = MetersPerSecond`

<details>
<summary>🔑 解答例</summary>

```rust
use std::marker::PhantomData;
use std::ops::{Add, Mul, Div};

#[derive(Clone, Copy)]
struct Meters;
#[derive(Clone, Copy)]
struct Seconds;
#[derive(Clone, Copy)]
struct Kilograms;
#[derive(Clone, Copy)]
struct SquareMeters;
#[derive(Clone, Copy)]
struct MetersPerSecond;

#[derive(Debug, Clone, Copy)]
struct Qty<U> {
    value: f64,
    _unit: PhantomData<U>,
}

impl<U> Qty<U> {
    fn new(v: f64) -> Self { Qty { value: v, _unit: PhantomData } }
}

impl<U> Add for Qty<U> {
    type Output = Qty<U>;
    fn add(self, rhs: Self) -> Self::Output { Qty::new(self.value + rhs.value) }
}

impl Mul<Qty<Meters>> for Qty<Meters> {
    type Output = Qty<SquareMeters>;
    fn mul(self, rhs: Qty<Meters>) -> Qty<SquareMeters> {
        Qty::new(self.value * rhs.value)
    }
}

impl Div<Qty<Seconds>> for Qty<Meters> {
    type Output = Qty<MetersPerSecond>;
    fn div(self, rhs: Qty<Seconds>) -> Qty<MetersPerSecond> {
        Qty::new(self.value / rhs.value)
    }
}

fn main() {
    let width = Qty::<Meters>::new(5.0);
    let height = Qty::<Meters>::new(3.0);
    let area = width * height; // Qty<SquareMeters>
    println!("面積: {:.1} m²", area.value);

    let dist = Qty::<Meters>::new(100.0);
    let time = Qty::<Seconds>::new(9.58);
    let speed = dist / time;
    println!("速度: {:.2} m/s", speed.value);

    let sum = width + height; // 同一単位 ✅
    println!("合計: {:.1} m", sum.value);

    // let bad = width + time; // ❌ コンパイルエラー: Meters と Seconds は加算できない
}
```

</details>

---

### 演習 3: チャンネルベースのワーカープール ★★★ (約45分)

チャンネルを用いたワーカープールを構築してください。要件は以下の通りです:
- ディスパッチャがチャンネル経由で `Job` 構造体を送信する
- N 個のワーカーがジョブを消費し、結果を送り返す
- `crossbeam-channel` を使用する（利用できない場合は `std::sync::mpsc` を使用）

<details>
<summary>🔑 解答例</summary>

```rust
use std::sync::mpsc;
use std::thread;

struct Job {
    id: u64,
    data: String,
}

struct JobResult {
    job_id: u64,
    output: String,
    worker_id: usize,
}

fn worker_pool(jobs: Vec<Job>, num_workers: usize) -> Vec<JobResult> {
    let (job_tx, job_rx) = mpsc::channel::<Job>();
    let (result_tx, result_rx) = mpsc::channel::<JobResult>();

    // ワーカー間で共有するためにレシーバを Arc<Mutex> でラップする
    let job_rx = std::sync::Arc::new(std::sync::Mutex::new(job_rx));

    // ワーカーを生成
    let mut handles = Vec::new();
    for worker_id in 0..num_workers {
        let job_rx = job_rx.clone();
        let result_tx = result_tx.clone();
        handles.push(thread::spawn(move || {
            loop {
                // ロック取得、受信、ロック解除 — クリティカルセクションを最小化
                let job = {
                    let rx = job_rx.lock().unwrap();
                    rx.recv() // ジョブを受信するかチャンネルが閉じるまでブロック
                };
                match job {
                    Ok(job) => {
                        let output = format!("ワーカー {worker_id} により '{}' を処理完了", job.data);
                        result_tx.send(JobResult {
                            job_id: job.id,
                            output,
                            worker_id,
                        }).unwrap();
                    }
                    Err(_) => break, // チャンネルが閉じた — 終了
                }
            }
        }));
    }
    drop(result_tx); // ワーカー完了時に結果チャンネルが閉じるよう、自身のコピーをドロップ

    // ジョブをディスパッチ
    let num_jobs = jobs.len();
    for job in jobs {
        job_tx.send(job).unwrap();
    }
    drop(job_tx); // ジョブチャンネルを閉じる — ワーカーはすべてのジョブを処理した後に終了する

    // 結果を収集
    let mut results = Vec::new();
    for result in result_rx {
        results.push(result);
    }
    assert_eq!(results.len(), num_jobs);

    for h in handles { h.join().unwrap(); }
    results
}

fn main() {
    let jobs: Vec<Job> = (0..20).map(|i| Job {
        id: i,
        data: format!("task-{i}"),
    }).collect();

    let results = worker_pool(jobs, 4);
    for r in &results {
        println!("[ワーカー {}] ジョブ {}: {}", r.worker_id, r.job_id, r.output);
    }
}
```

</details>

---

### 演習 4: 高階コンビネータパイプライン ★★ (約25分)

変換処理をチェーンする `Pipeline` 構造体を作成してください。変換を追加する `.pipe(f)` と、チェーン全体を実行する `.execute(input)` をサポートする必要があります。

<details>
<summary>🔑 解答例</summary>

```rust
struct Pipeline<T> {
    transforms: Vec<Box<dyn Fn(T) -> T>>,
}

impl<T: 'static> Pipeline<T> {
    fn new() -> Self {
        Pipeline { transforms: Vec::new() }
    }

    fn pipe(mut self, f: impl Fn(T) -> T + 'static) -> Self {
        self.transforms.push(Box::new(f));
        self
    }

    fn execute(self, input: T) -> T {
        self.transforms.into_iter().fold(input, |val, f| f(val))
    }
}

fn main() {
    let result = Pipeline::new()
        .pipe(|s: String| s.trim().to_string())
        .pipe(|s| s.to_uppercase())
        .pipe(|s| format!(">>> {s} <<<"))
        .execute("  hello world  ".to_string());

    println!("{result}"); // >>> HELLO WORLD <<<

    // 数値パイプライン:
    let result = Pipeline::new()
        .pipe(|x: i32| x * 2)
        .pipe(|x| x + 10)
        .pipe(|x| x * x)
        .execute(5);

    println!("{result}"); // (5*2 + 10)^2 = 400
}
```

**発展**: ステージ間で型が変化するジェネリックパイプラインは異なる設計になります。各 `.pipe()` が異なる出力型を持つ `Pipeline` を返す形となり、より高度なジェネリクスの配線が必要となります。

</details>

---

### 演習 5: thiserror によるエラー階層の設計 ★★ (約30分)

I/O、パース（JSON および CSV）、およびバリデーションの各フェーズで失敗する可能性があるファイル処理アプリケーション向けのエラー型階層を設計してください。`thiserror` を使用し、`?` 演算子によるエラー伝播を実証してください。

<details>
<summary>🔑 解答例</summary>

```rust,ignore
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("I/O エラー: {0}")]
    Io(#[from] std::io::Error),

    #[error("JSON パースエラー: {0}")]
    Json(#[from] serde_json::Error),

    #[error("CSV エラー ({line} 行目): {message}")]
    Csv { line: usize, message: String },

    #[error("バリデーションエラー: {field} — {reason}")]
    Validation { field: String, reason: String },
}

fn read_file(path: &str) -> Result<String, AppError> {
    Ok(std::fs::read_to_string(path)?) // #[from] により io::Error → AppError::Io に自動変換
}

fn parse_json(content: &str) -> Result<serde_json::Value, AppError> {
    Ok(serde_json::from_str(content)?) // serde_json::Error → AppError::Json に自動変換
}

fn validate_name(value: &serde_json::Value) -> Result<String, AppError> {
    let name = value.get("name")
        .and_then(|v| v.as_str())
        .ok_or_else(|| AppError::Validation {
            field: "name".into(),
            reason: "null 以外の文字列である必要があります".into(),
        })?;

    if name.is_empty() {
        return Err(AppError::Validation {
            field: "name".into(),
            reason: "空文字にすることはできません".into(),
        });
    }

    Ok(name.to_string())
}

fn process_file(path: &str) -> Result<String, AppError> {
    let content = read_file(path)?;
    let json = parse_json(&content)?;
    let name = validate_name(&json)?;
    Ok(name)
}

fn main() {
    match process_file("config.json") {
        Ok(name) => println!("名前: {name}"),
        Err(e) => eprintln!("エラー: {e}"),
    }
}
```

</details>

---

### 演習 6: 関連型を持つジェネリックトレイト ★★★ (約40分)

関連型 `Error` および `Id` を持つ `Repository` トレイトを設計してください。インメモリストアに対してこれを実装し、コンパイル時の型安全性を実証してください。

<details>
<summary>🔑 解答例</summary>

```rust
use std::collections::HashMap;

trait Repository {
    type Item;
    type Id;
    type Error;

    fn get(&self, id: &Self::Id) -> Result<Option<&Self::Item>, Self::Error>;
    fn insert(&mut self, item: Self::Item) -> Result<Self::Id, Self::Error>;
    fn delete(&mut self, id: &Self::Id) -> Result<bool, Self::Error>;
}

#[derive(Debug, Clone)]
struct User {
    name: String,
    email: String,
}

struct InMemoryUserRepo {
    data: HashMap<u64, User>,
    next_id: u64,
}

impl InMemoryUserRepo {
    fn new() -> Self {
        InMemoryUserRepo { data: HashMap::new(), next_id: 1 }
    }
}

// エラー型は Infallible — インメモリ操作は決して失敗しない
impl Repository for InMemoryUserRepo {
    type Item = User;
    type Id = u64;
    type Error = std::convert::Infallible;

    fn get(&self, id: &u64) -> Result<Option<&User>, Self::Error> {
        Ok(self.data.get(id))
    }

    fn insert(&mut self, item: User) -> Result<u64, Self::Error> {
        let id = self.next_id;
        self.next_id += 1;
        self.data.insert(id, item);
        Ok(id)
    }

    fn delete(&mut self, id: &u64) -> Result<bool, Self::Error> {
        Ok(self.data.remove(id).is_some())
    }
}

// 任意の Repository で動作するジェネリック関数:
fn create_and_fetch<R: Repository>(repo: &mut R, item: R::Item) -> Result<(), R::Error>
where
    R::Item: std::fmt::Debug,
    R::Id: std::fmt::Debug,
{
    let id = repo.insert(item)?;
    println!("ID を付与して挿入完了: {id:?}");
    let retrieved = repo.get(&id)?;
    println!("取得結果: {retrieved:?}");
    Ok(())
}

fn main() {
    let mut repo = InMemoryUserRepo::new();
    create_and_fetch(&mut repo, User {
        name: "Alice".into(),
        email: "alice@example.com".into(),
    }).unwrap();
}
```

</details>

---

### 演習 7: Unsafe をカプセル化する安全なラッパー（第11章） ★★★ (約45分)

固定容量のスタック割り当てベクターである `FixedVec<T, const N: usize>` を実装してください。
要件:
- `push(&mut self, value: T) -> Result<(), T>` は満杯時に `Err(value)` を返す
- `pop(&mut self) -> Option<T>` は最後の要素を取り出して削除する
- `as_slice(&self) -> &[T]` は初期化済みの要素を参照として借用する
- すべての公開メソッドは安全（safe）でなければならず、すべての unsafe コードは `SAFETY:` コメントでカプセル化すること
- `Drop` は初期化済みの要素を適切に破棄・クリーンアップすること

**ヒント**: `MaybeUninit<T>` および `[const { MaybeUninit::uninit() }; N]` を使用してください。

<details>
<summary>🔑 解答例</summary>

```rust
use std::mem::MaybeUninit;

pub struct FixedVec<T, const N: usize> {
    data: [MaybeUninit<T>; N],
    len: usize,
}

impl<T, const N: usize> FixedVec<T, N> {
    pub fn new() -> Self {
        FixedVec {
            data: [const { MaybeUninit::uninit() }; N],
            len: 0,
        }
    }

    pub fn push(&mut self, value: T) -> Result<(), T> {
        if self.len >= N { return Err(value); }
        // SAFETY: len < N であるため、data[len] は境界内に収まる。
        self.data[self.len] = MaybeUninit::new(value);
        self.len += 1;
        Ok(())
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 { return None; }
        self.len -= 1;
        // SAFETY: data[len] は初期化済みである（デクリメント前は len > 0 であった）。
        Some(unsafe { self.data[self.len].assume_init_read() })
    }

    pub fn as_slice(&self) -> &[T] {
        // SAFETY: data[0..len] はすべて初期化済みであり、MaybeUninit<T>
        // は T と同じメモリレイアウトを持つ。
        unsafe { std::slice::from_raw_parts(self.data.as_ptr() as *const T, self.len) }
    }

    pub fn len(&self) -> usize { self.len }
    pub fn is_empty(&self) -> bool { self.len == 0 }
}

impl<T, const N: usize> Drop for FixedVec<T, N> {
    fn drop(&mut self) {
        // SAFETY: data[0..len] は初期化済みであるため、それぞれをドロップする。
        for i in 0..self.len {
            unsafe { self.data[i].assume_init_drop(); }
        }
    }
}

fn main() {
    let mut v = FixedVec::<String, 4>::new();
    v.push("hello".into()).unwrap();
    v.push("world".into()).unwrap();
    assert_eq!(v.as_slice(), &["hello", "world"]);
    assert_eq!(v.pop(), Some("world".into()));
    assert_eq!(v.len(), 1);
    // Drop が残りの "hello" をクリーンアップする
}
```

</details>

---

### 演習 8: 宣言的マクロ — `map!`（第12章） ★ (約15分)

キー・バリューのペアから `HashMap` を生成する、`vec![]` に似た `map!` マクロを作成してください:

```rust
let m = map! {
    "host" => "localhost",
    "port" => "8080",
};
assert_eq!(m.get("host"), Some(&"localhost"));
assert_eq!(m.len(), 2);
```

要件:
- 末尾のカンマ（trailing comma）をサポートする
- 空の呼び出し `map!{}` をサポートする
- 柔軟性を最大化するため、`Into<K>` および `Into<V>` を実装した任意の型で動作するようにする

<details>
<summary>🔑 解答例</summary>

```rust
macro_rules! map {
    // 空の場合
    () => {
        std::collections::HashMap::new()
    };
    // 1つ以上の key => value のペア（末尾カンマは任意）
    ( $( $key:expr => $val:expr ),+ $(,)? ) => {{
        let mut m = std::collections::HashMap::new();
        $( m.insert($key, $val); )+
        m
    }};
}

fn main() {
    // 基本的な使用法:
    let config = map! {
        "host" => "localhost",
        "port" => "8080",
        "timeout" => "30",
    };
    assert_eq!(config.len(), 3);
    assert_eq!(config["host"], "localhost");

    // 空のマップ:
    let empty: std::collections::HashMap<String, String> = map!();
    assert!(empty.is_empty());

    // 異なる型:
    let scores = map! {
        1 => 100,
        2 => 200,
    };
    assert_eq!(scores[&1], 100);
}
```

</details>

---

### 演習 9: カスタム serde デシリアライゼーション（第10章） ★★★ (約45分)

カスタム serde デシリアライザーを用いて、`"30s"`、`"5m"`、`"2h"` のような人間が読める文字列からデシリアライズする `Duration` ラッパーを設計してください。また、同一フォーマットへと再シリアライズできるようにしてください。

<details>
<summary>🔑 解答例</summary>

```rust,ignore
use serde::{Deserialize, Deserializer, Serialize, Serializer};
use std::fmt;

#[derive(Debug, Clone, PartialEq)]
struct HumanDuration(std::time::Duration);

impl HumanDuration {
    fn from_str(s: &str) -> Result<Self, String> {
        let s = s.trim();
        if s.is_empty() { return Err("空の期間文字列です".into()); }

        let (num_str, suffix) = s.split_at(
            s.find(|c: char| !c.is_ascii_digit()).unwrap_or(s.len())
        );
        let value: u64 = num_str.parse()
            .map_err(|_| format!("無効な数値です: {num_str}"))?;

        let duration = match suffix {
            "s" | "sec"  => std::time::Duration::from_secs(value),
            "m" | "min"  => std::time::Duration::from_secs(value * 60),
            "h" | "hr"   => std::time::Duration::from_secs(value * 3600),
            "ms"         => std::time::Duration::from_millis(value),
            other        => return Err(format!("未知の接尾辞です: {other}")),
        };
        Ok(HumanDuration(duration))
    }
}

impl fmt::Display for HumanDuration {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let secs = self.0.as_secs();
        if secs == 0 {
            write!(f, "{}ms", self.0.as_millis())
        } else if secs % 3600 == 0 {
            write!(f, "{}h", secs / 3600)
        } else if secs % 60 == 0 {
            write!(f, "{}m", secs / 60)
        } else {
            write!(f, "{}s", secs)
        }
    }
}

impl Serialize for HumanDuration {
    fn serialize<S: Serializer>(&self, serializer: S) -> Result<S::Ok, S::Error> {
        serializer.serialize_str(&self.to_string())
    }
}

impl<'de> Deserialize<'de> for HumanDuration {
    fn deserialize<D: Deserializer<'de>>(deserializer: D) -> Result<Self, D::Error> {
        let s = String::deserialize(deserializer)?;
        HumanDuration::from_str(&s).map_err(serde::de::Error::custom)
    }
}

#[derive(Debug, Deserialize, Serialize)]
struct Config {
    timeout: HumanDuration,
    retry_interval: HumanDuration,
}

fn main() {
    let json = r#"{ "timeout": "30s", "retry_interval": "5m" }"#;
    let config: Config = serde_json::from_str(json).unwrap();

    assert_eq!(config.timeout.0, std::time::Duration::from_secs(30));
    assert_eq!(config.retry_interval.0, std::time::Duration::from_secs(300));

    // 正しく相互変換（ラウンドトリップ）される:
    let serialized = serde_json::to_string(&config).unwrap();
    assert!(serialized.contains("30s"));
    assert!(serialized.contains("5m"));
    println!("Config: {serialized}");
}
```

</details>

### 演習 10 — タイムアウト付き並行フェッチャー ★★ (約25分)

それぞれ `tokio::time::sleep` でネットワーク呼び出しをシミュレートする3つの `tokio::spawn` タスクを起動する非同期関数 `fetch_all` を作成してください。3つのタスクすべてを `tokio::try_join!` で待ち合わせ、全体を `tokio::time::timeout(Duration::from_secs(5), ...)` でラップします。いずれかのタスクが失敗するか期限が切れた場合はエラーを返し、成功時は `Result<Vec<String>, ...>` を返すようにしてください。

**学習目標**: `tokio::spawn`、`try_join!`、`timeout`、タスク境界を越えたエラー伝播。

<details>
<summary>ヒント</summary>

生成された各タスクは `Result<String, _>` を返します。`try_join!` は3つすべてを展開します。`try_join!` 全体を `timeout()` でラップします。`Elapsed` エラーは制限時間に達したことを意味します。

</details>

<details>
<summary>解答例</summary>

```rust,ignore
use tokio::time::{sleep, timeout, Duration};

async fn fake_fetch(name: &'static str, delay_ms: u64) -> Result<String, String> {
    sleep(Duration::from_millis(delay_ms)).await;
    Ok(format!("{name}: OK"))
}

async fn fetch_all() -> Result<Vec<String>, Box<dyn std::error::Error>> {
    let deadline = Duration::from_secs(5);

    let (a, b, c) = timeout(deadline, async {
        let h1 = tokio::spawn(fake_fetch("svc-a", 100));
        let h2 = tokio::spawn(fake_fetch("svc-b", 200));
        let h3 = tokio::spawn(fake_fetch("svc-c", 150));
        tokio::try_join!(h1, h2, h3)
    })
    .await??; // 最初の ? = タイムアウトの判定、2番目の ? = join の結果

    Ok(vec![a?, b?, c?]) // 内部の Result をアンラップ
}

#[tokio::main]
async fn main() {
    let results = fetch_all().await.unwrap();
    for r in &results {
        println!("{r}");
    }
}
```

</details>

### 演習 11 — 非同期チャンネルパイプライン ★★★ (約40分)

`tokio::sync::mpsc` を用いて、プロデューサー → トランスフォーマー → コンシューマー のパイプラインを構築してください:

1. **Producer**: 整数 1..=20 をチャンネル A（容量 4）に送信します。
2. **Transformer**: チャンネル A から読み取り、各値を2乗してチャンネル B に送信します。
3. **Consumer**: チャンネル B から読み取り、`Vec<u64>` に収集して返します。

すべての3つのステージは並行な `tokio::spawn` タスクとして実行されます。バックプレッシャーを実証するために有界（bounded）チャンネルを使用してください。最終的なベクターが `[1, 4, 9, ..., 400]` と等しいことをアサートしてください。

**学習目標**: `mpsc::channel`、有界バックプレッシャー、move クロージャを伴う `tokio::spawn`、チャンネルクローズによるグレースフルシャットダウン。

<details>
<summary>解答例</summary>

```rust,ignore
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx_a, mut rx_a) = mpsc::channel::<u64>(4); // 有界 — バックプレッシャー
    let (tx_b, mut rx_b) = mpsc::channel::<u64>(4);

    // プロデューサー
    let producer = tokio::spawn(async move {
        for i in 1..=20u64 {
            tx_a.send(i).await.unwrap();
        }
        // tx_a はここでドロップされる → チャンネル A が閉じる
    });

    // トランスフォーマー
    let transformer = tokio::spawn(async move {
        while let Some(val) = rx_a.recv().await {
            tx_b.send(val * val).await.unwrap();
        }
        // tx_b はここでドロップされる → チャンネル B が閉じる
    });

    // コンシューマー
    let consumer = tokio::spawn(async move {
        let mut results = Vec::new();
        while let Some(val) = rx_b.recv().await {
            results.push(val);
        }
        results
    });

    producer.await.unwrap();
    transformer.await.unwrap();
    let results = consumer.await.unwrap();

    let expected: Vec<u64> = (1..=20).map(|x: u64| x * x).collect();
    assert_eq!(results, expected);
    println!("パイプライン完了: {results:?}");
}
```

</details>

***
