# 5. チャンネルとメッセージパッシング 🟢

> **学習内容:**
> - `std::sync::mpsc` の基本と crossbeam-channel への移行時期
> - 複数ソースのメッセージを処理する `select!` によるチャンネル選択
> - 有界チャンネル vs 無界チャンネルとバックプレッシャー戦略
> - 並行状態をカプセル化するアクターパターン

## std::sync::mpsc — 標準チャンネル

Rust の標準ライブラリは、複数プロデューサー・単一コンシューマー（multi-producer, single-consumer）チャンネルを提供します：

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    // チャンネルを作成: tx (送信側) と rx (受信側)
    let (tx, rx) = mpsc::channel();

    // プロデューサースレッドを生成
    let tx1 = tx.clone(); // 複数のプロデューサー用にクローン
    thread::spawn(move || {
        for i in 0..5 {
            tx1.send(format!("producer-1: msg {i}")).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
    });

    // 2つ目のプロデューサー
    thread::spawn(move || {
        for i in 0..5 {
            tx.send(format!("producer-2: msg {i}")).unwrap();
            thread::sleep(Duration::from_millis(150));
        }
    });

    // コンシューマー: すべてのメッセージを受信
    for msg in rx {
        // すべての送信側がドロップされると rx イテレータが終了する
        println!("受信: {msg}");
    }
    println!("すべてのプロデューサーが完了しました。");
}
```

> **注意:** `.send()` に対する `.unwrap()` は簡潔さのために使用しています。受信側がドロップされている場合はパニックします。プロダクションコードでは `SendError` を適切に処理すべきです。

**主な特性**:
- デフォルトで**無界（Unbounded）**（コンシューマーが遅い場合、メモリを消費し続ける可能性があります）
- `mpsc::sync_channel(N)` はバックプレッシャーを持つ**有界（Bounded）**チャンネルを作成します
- `rx.recv()` はメッセージが届くまでカレントスレッドをブロックします
- `rx.try_recv()` は準備完了したメッセージがない場合、即座に `Err(TryRecvError::Empty)` を返します
- すべての `Sender` がドロップされるとチャンネルはクローズします

```rust
// バックプレッシャーを持つ有界チャンネル:
let (tx, rx) = mpsc::sync_channel(10); // 10メッセージのバッファ

thread::spawn(move || {
    for i in 0..1000 {
        tx.send(i).unwrap(); // バッファがいっぱいの場合はブロック — 自然なバックプレッシャー
    }
});
```

> **注意:** `.unwrap()` は簡潔さのために使用しています。プロダクション環境では、パニックする代わりに `SendError`（受信側がドロップされた）を処理してください。

### crossbeam-channel — プロダクションにおける主力

`crossbeam-channel` は、プロダクション環境でのチャンネル利用における事実上の標準です。`std::sync::mpsc` よりも高速で、複数コンシューマー（`mpmc`）をサポートしています：

```rust,ignore
// Cargo.toml:
//   [dependencies]
//   crossbeam-channel = "0.5"
use crossbeam_channel::{bounded, unbounded, select, Sender, Receiver};
use std::thread;
use std::time::Duration;

fn main() {
    // 有界 MPMC チャンネル
    let (tx, rx) = bounded::<String>(100);

    // 複数のプロデューサー
    for id in 0..4 {
        let tx = tx.clone();
        thread::spawn(move || {
            for i in 0..10 {
                tx.send(format!("worker-{id}: item-{i}")).unwrap();
            }
        });
    }
    drop(tx); // チャンネルがクローズできるようにオリジナルの送信側をドロップ

    // 複数のコンシューマー（std::sync::mpsc では不可！）
    let rx2 = rx.clone();
    let consumer1 = thread::spawn(move || {
        while let Ok(msg) = rx.recv() {
            println!("[consumer-1] {msg}");
        }
    });
    let consumer2 = thread::spawn(move || {
        while let Ok(msg) = rx2.recv() {
            println!("[consumer-2] {msg}");
        }
    });

    consumer1.join().unwrap();
    consumer2.join().unwrap();
}
```

### チャンネル選択 (select!)

複数のチャンネルを同時にリッスンする — Go 言語の `select` と同様です：

```rust,ignore
use crossbeam_channel::{bounded, tick, after, select};
use std::time::Duration;

fn main() {
    let (work_tx, work_rx) = bounded::<String>(10);
    let ticker = tick(Duration::from_secs(1));        // 定期的なティック
    let deadline = after(Duration::from_secs(10));     // ワンショットタイムアウト

    // プロデューサー
    let tx = work_tx.clone();
    std::thread::spawn(move || {
        for i in 0..100 {
            tx.send(format!("job-{i}")).unwrap();
            std::thread::sleep(Duration::from_millis(500));
        }
    });
    drop(work_tx);

    loop {
        select! {
            recv(work_rx) -> msg => {
                match msg {
                    Ok(job) => println!("処理中: {job}"),
                    Err(_) => {
                        println!("ワークチャンネルがクローズしました");
                        break;
                    }
                }
            },
            recv(ticker) -> _ => {
                println!("Tick — ハートビート");
            },
            recv(deadline) -> _ => {
                println!("デッドラインに到達 — シャットダウンします");
                break;
            },
        }
    }
}
```

> **Go との比較**: これは Go のチャンネルに対する `select` 文とまったく同じです。
> crossbeam の `select!` マクロは、Go と同様に飢餓（スターベーション）を防ぐために順序をランダム化します。

### 有界 vs 無界とバックプレッシャー

| 種類 | バッファ満杯時の動作 | メモリ | ユースケース |
|------|-------------------|--------|----------|
| **無界（Unbounded）** | 決してブロックしない（ヒープを拡張） | 無制限 ⚠️ | 稀 — プロデューサーがコンシューマーより遅い場合のみ |
| **有界（Bounded）** | 空きができるまで `send()` がブロック | 固定 | プロダクションのデフォルト — OOM を防止 |
| **ランデブー（Rendezvous）** (bounded(0)) | 受信側の準備ができるまで `send()` がブロック | なし | 同期 / ハンドオフ |

```rust
// ランデブーチャンネル — 容量ゼロ、直接の受け渡し
let (tx, rx) = crossbeam_channel::bounded(0);
// tx.send(x) は rx.recv() が呼ばれるまでブロックし、逆も同様です。
// これにより、2つのスレッドを正確に同期させます。
```

**ルール**: プロデューサーがコンシューマーの処理速度を上回ることが決してないと証明できる場合を除き、プロダクションでは常に有界チャンネルを使用してください。

### チャンネルを用いたアクターパターン

アクターパターンは、チャンネルを使用して可変状態へのアクセスを直列化します — Mutex は不要です：

```rust
use std::sync::mpsc;
use std::thread;

// アクターが受信できるメッセージ
enum CounterMsg {
    Increment,
    Decrement,
    Get(mpsc::Sender<i64>), // 返信用チャンネル
}

struct CounterActor {
    count: i64,
    rx: mpsc::Receiver<CounterMsg>,
}

impl CounterActor {
    fn new(rx: mpsc::Receiver<CounterMsg>) -> Self {
        CounterActor { count: 0, rx }
    }

    fn run(mut self) {
        while let Ok(msg) = self.rx.recv() {
            match msg {
                CounterMsg::Increment => self.count += 1,
                CounterMsg::Decrement => self.count -= 1,
                CounterMsg::Get(reply) => {
                    let _ = reply.send(self.count);
                }
            }
        }
    }
}

// アクターハンドル — クローンが安価で、Send + Sync
#[derive(Clone)]
struct Counter {
    tx: mpsc::Sender<CounterMsg>,
}

impl Counter {
    fn spawn() -> Self {
        let (tx, rx) = mpsc::channel();
        thread::spawn(move || CounterActor::new(rx).run());
        Counter { tx }
    }

    fn increment(&self) { let _ = self.tx.send(CounterMsg::Increment); }
    fn decrement(&self) { let _ = self.tx.send(CounterMsg::Decrement); }

    fn get(&self) -> i64 {
        let (reply_tx, reply_rx) = mpsc::channel();
        self.tx.send(CounterMsg::Get(reply_tx)).unwrap();
        reply_rx.recv().unwrap()
    }
}

fn main() {
    let counter = Counter::spawn();

    // 複数のスレッドがカウンターを安全に使用可能 — Mutex は不要！
    let handles: Vec<_> = (0..10).map(|_| {
        let counter = counter.clone();
        thread::spawn(move || {
            for _ in 0..1000 {
                counter.increment();
            }
        })
    }).collect();

    for h in handles { h.join().unwrap(); }
    println!("最終カウント: {}", counter.get()); // 10000
}
```

> **アクター vs Mutex の使い分け**: アクターは、状態に複雑な不変条件がある場合、操作に長い時間がかかる場合、またはロックの順序を意識せずにアクセスを直列化したい場合に最適です。短いクリティカルセクションには Mutex の方がシンプルです。

> **重要なポイント — チャンネル**
> - `crossbeam-channel` はプロダクションにおける主力であり、`std::sync::mpsc` よりも高速で機能が豊富です
> - `select!` は複雑な複数ソースのポーリングを宣言的なチャンネル選択に置き換えます
> - 有界チャンネルは自然なバックプレッシャーを提供し、無界チャンネルは OOM のリスクがあります

> **関連項目:** スレッド、Mutex、共有状態については [第6章 — 並行性](ch06-concurrency-vs-parallelism-vs-threads.md) を参照してください。非同期チャンネル（`tokio::sync::mpsc`）については [第15章 — 非同期](ch16-asyncawait-essentials.md) を参照してください。

---

### 演習: チャンネルベースのワーカープール ★★★（約45分）

チャンネルを使用してワーカープールを構築してください：
- ディスパッチャーがチャンネルを通じて `Job` 構造体を送信する
- N 個のワーカーがジョブを消費し、結果を送り返す
- 共有ワークキューに `Arc<Mutex<Receiver>>` とともに `std::sync::mpsc` を使用する

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

    let job_rx = std::sync::Arc::new(std::sync::Mutex::new(job_rx));

    let mut handles = Vec::new();
    for worker_id in 0..num_workers {
        let job_rx = job_rx.clone();
        let result_tx = result_tx.clone();
        handles.push(thread::spawn(move || {
            loop {
                let job = {
                    let rx = job_rx.lock().unwrap();
                    rx.recv()
                };
                match job {
                    Ok(job) => {
                        let output = format!("processed '{}' by worker {worker_id}", job.data);
                        result_tx.send(JobResult {
                            job_id: job.id, output, worker_id,
                        }).unwrap();
                    }
                    Err(_) => break,
                }
            }
        }));
    }
    drop(result_tx);

    let num_jobs = jobs.len();
    for job in jobs {
        job_tx.send(job).unwrap();
    }
    drop(job_tx);

    let results: Vec<_> = result_rx.into_iter().collect();
    assert_eq!(results.len(), num_jobs);

    for h in handles { h.join().unwrap(); }
    results
}

fn main() {
    let jobs: Vec<Job> = (0..20).map(|i| Job {
        id: i, data: format!("task-{i}"),
    }).collect();

    let results = worker_pool(jobs, 4);
    for r in &results {
        println!("[worker {}] job {}: {}", r.worker_id, r.job_id, r.output);
    }
}
```

</details>

***
