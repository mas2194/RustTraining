## 演習問題

### 演習 1: 非同期エコーサーバー

複数のクライアントを並行して処理する TCP エコーサーバーを構築してください。

**要件**:
- `127.0.0.1:8080` でリッスンする
- 接続を受け入れ、各行をそのまま送り返す（エコー）
- クライアントの切断を適切に処理する
- クライアントの接続時および切断時にログを出力する

<details>
<summary>🔑 解答</summary>

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("エコーサーバーが :8080 でリッスン中");

    loop {
        let (socket, addr) = listener.accept().await?;
        println!("[{addr}] 接続されました");

        tokio::spawn(async move {
            let (reader, mut writer) = socket.into_split();
            let mut reader = BufReader::new(reader);
            let mut line = String::new();

            loop {
                line.clear();
                match reader.read_line(&mut line).await {
                    Ok(0) => {
                        println!("[{addr}] 切断されました");
                        break;
                    }
                    Ok(_) => {
                        print!("[{addr}] エコー: {line}");
                        if writer.write_all(line.as_bytes()).await.is_err() {
                            println!("[{addr}] 書き込みエラー。切断します");
                            break;
                        }
                    }
                    Err(e) => {
                        eprintln!("[{addr}] 読み込みエラー: {e}");
                        break;
                    }
                }
            }
        });
    }
}
```

</details>

---

### 演習 2: レート制限付き並行URLフェッチャー

最大5つの並行リクエストで、URLリストを並行してフェッチしてください。

<details>
<summary>🔑 解答</summary>

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{sleep, Duration};

async fn fetch_urls(urls: Vec<String>) -> Vec<Result<String, String>> {
    // buffer_unordered(5) により、最大5つのFutureが並行してポーリングされることが保証される
    // — ここで別途 Semaphore を用意する必要はない。
    let results: Vec<_> = stream::iter(urls)
        .map(|url| {
            async move {
                println!("フェッチ中: {url}");

                match reqwest::get(&url).await {
                    Ok(resp) => match resp.text().await {
                        Ok(body) => Ok(body),
                        Err(e) => Err(format!("{url}: {e}")),
                    },
                    Err(e) => Err(format!("{url}: {e}")),
                }
            }
        })
        .buffer_unordered(5) // ← これだけで並行数を5に制限できる
        .collect()
        .await;

    results
}

// 注意: 独立してスポーンされたタスク（tokio::spawn）間で並行数を制限したい場合は Semaphore を使用します。
// ストリームを処理する場合は buffer_unordered を使用します。同じ制限に対して両方を重複して組み合わせないでください。
```

</details>

---

### 演習 3: ワーカープールによるグレースフルシャットダウン

以下の機能を持つタスクプロセッサを構築してください：
- チャネルベースの作業キュー
- キューから処理を取り出すN個のワーカタスク
- Ctrl+C によるグレースフルシャットダウン：新規受付を停止し、処理中の作業を完了させる

<details>
<summary>🔑 解答</summary>

```rust
use tokio::sync::{mpsc, watch};
use tokio::time::{sleep, Duration};

struct WorkItem {
    id: u64,
    payload: String,
}

#[tokio::main]
async fn main() {
    let (work_tx, work_rx) = mpsc::channel::<WorkItem>(100);
    let (shutdown_tx, shutdown_rx) = watch::channel(false);

    // 4つのワーカーをスポーン
    let mut worker_handles = Vec::new();
    let work_rx = std::sync::Arc::new(tokio::sync::Mutex::new(work_rx));

    for id in 0..4 {
        let rx = work_rx.clone();
        let mut shutdown = shutdown_rx.clone();
        let handle = tokio::spawn(async move {
            loop {
                let item = {
                    let mut rx = rx.lock().await;
                    tokio::select! {
                        item = rx.recv() => item,
                        _ = shutdown.changed() => {
                            if *shutdown.borrow() { None } else { continue }
                        }
                    }
                };

                match item {
                    Some(work) => {
                        println!("ワーカー {id}: アイテム {} を処理中", work.id);
                        sleep(Duration::from_millis(200)).await; // 処理のシミュレーション
                        println!("ワーカー {id}: アイテム {} の処理完了", work.id);
                    }
                    None => {
                        println!("ワーカー {id}: チャネルがクローズしました。終了します");
                        break;
                    }
                }
            }
        });
        worker_handles.push(handle);
    }

    // プロデューサー: 作業を投入
    let producer = tokio::spawn(async move {
        for i in 0..20 {
            let _ = work_tx.send(WorkItem {
                id: i,
                payload: format!("task-{i}"),
            }).await;
            sleep(Duration::from_millis(50)).await;
        }
    });

    // Ctrl+C を待機
    tokio::signal::ctrl_c().await.unwrap();
    println!("\nシャットダウンシグナルを受信しました！");
    shutdown_tx.send(true).unwrap();
    producer.abort(); // プロデューサータスクをキャンセル

    // ワーカーの終了を待機
    for handle in worker_handles {
        let _ = handle.await;
    }
    println!("すべてのワーカーが終了しました。終了します！");
}
```

</details>

---

### 演習 4: ゼロからのシンプルな非同期Mutexの実装

（`tokio::sync::Mutex` を使わずに）チャネルまたはセマフォを用いて、非同期対応のMutexを実装してください。

*ヒント*: アクセスを直列化するために、パーミット数1の `tokio::sync::Semaphore` を使用します。

<details>
<summary>🔑 解答</summary>

```rust
use std::cell::UnsafeCell;
use std::sync::Arc;
use tokio::sync::{OwnedSemaphorePermit, Semaphore};

pub struct SimpleAsyncMutex<T> {
    data: Arc<UnsafeCell<T>>,
    semaphore: Arc<Semaphore>,
}

// SAFETY: T へのアクセスはセマフォ（最大パーミット数1）によって直列化されている。
unsafe impl<T: Send> Send for SimpleAsyncMutex<T> {}
unsafe impl<T: Send> Sync for SimpleAsyncMutex<T> {}

pub struct SimpleGuard<T> {
    data: Arc<UnsafeCell<T>>,
    _permit: OwnedSemaphorePermit, // ガードのドロップ時にドロップされる → ロックを解放
}

impl<T> SimpleAsyncMutex<T> {
    pub fn new(value: T) -> Self {
        SimpleAsyncMutex {
            data: Arc::new(UnsafeCell::new(value)),
            semaphore: Arc::new(Semaphore::new(1)),
        }
    }

    pub async fn lock(&self) -> SimpleGuard<T> {
        let permit = self.semaphore.clone().acquire_owned().await.unwrap();
        SimpleGuard {
            data: self.data.clone(),
            _permit: permit,
        }
    }
}

impl<T> std::ops::Deref for SimpleGuard<T> {
    type Target = T;
    fn deref(&self) -> &T {
        // SAFETY: 唯一のセマフォパーミットを保持しているため、他に
        // SimpleGuard は存在せず、排他アクセスが保証される。
        unsafe { &*self.data.get() }
    }
}

impl<T> std::ops::DerefMut for SimpleGuard<T> {
    fn deref_mut(&mut self) -> &mut T {
        // SAFETY: 同様の理由 — 単一パーミットにより排他性が保証される。
        unsafe { &mut *self.data.get() }
    }
}

// SimpleGuard がドロップされると _permit がドロップされ、
// セマフォパーミットが解放されるため、別の lock() が進行可能になる。

// 使用例:
// let mutex = SimpleAsyncMutex::new(vec![1, 2, 3]);
// {
//     let mut guard = mutex.lock().await;
//     guard.push(4);
// } // ここでパーミットが解放される
```

**重要なポイント**: 非同期Mutexは通常、セマフォをベースに構築されます。セマフォが非同期の待機機構を提供します。ロックされている場合、`acquire()` はパーミットが解放されるまでタスクを中断（サスペンド）します。これは `tokio::sync::Mutex` の内部動作とまさに同じです。

> **なぜ `std::sync::Mutex` ではなく `UnsafeCell` なのか？** この演習の以前のバージョンでは、`Deref`/`DerefMut` が `.lock().unwrap()` を呼び出す `Arc<Mutex<T>>` を使用していました。しかし、それではコンパイルが通りません。返される `&T` が、即座にドロップされる一時的な `MutexGuard` を借用してしまうためです。`UnsafeCell` を使用することで中間ガードを回避でき、セマフォによる直列化によって `unsafe` の健全性（soundness）が保たれます。

</details>

---

### 演習 5: ストリームパイプライン

ストリームを使用して、データ処理パイプラインを構築してください：
1. 1..=100 の数値を生成する
2. 偶数のみにフィルタリングする
3. 各数値を2乗（二乗）する
4. 一度に10個ずつ並行処理する（sleepでシミュレート）
5. 結果を収集する

<details>
<summary>🔑 解答</summary>

```rust
use futures::stream::{self, StreamExt};
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let results: Vec<u64> = stream::iter(1u64..=100)
        // ステップ 2: 偶数をフィルタリング
        .filter(|x| futures::future::ready(x % 2 == 0))
        // ステップ 3: 各数値を2乗
        .map(|x| x * x)
        // ステップ 4: 並行処理（非同期処理をシミュレート）
        .map(|x| async move {
            sleep(Duration::from_millis(50)).await;
            println!("処理完了: {x}");
            x
        })
        .buffer_unordered(10) // 10並行
        // ステップ 5: 収集
        .collect()
        .await;

    println!("{} 個の結果を取得しました", results.len());
    println!("合計: {}", results.iter().sum::<u64>());
}
```

</details>

---

### 演習 6: タイムアウト付きSelectの実装

`tokio::select!` や `tokio::time::timeout` を使わずに、Futureと期限を競合させ、タイムアウト時には `Either::Left(result)` または `Either::Right(())` を返す関数を実装してください。

*ヒント*: 第6章の `Select` コンビネータおよび同章の `TimerFuture` をベースに構築してください。

<details>
<summary>🔑 解答</summary>

```rust,ignore
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

pub enum Either<A, B> {
    Left(A),
    Right(B),
}

pub struct Timeout<F> {
    future: F,
    timer: TimerFuture, // 第6章より
}

impl<F: Future + Unpin> Timeout<F> {
    pub fn new(future: F, duration: Duration) -> Self {
        Timeout {
            future,
            timer: TimerFuture::new(duration),
        }
    }
}

impl<F: Future + Unpin> Future for Timeout<F> {
    type Output = Either<F::Output, ()>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // メインのFutureが完了したかチェック
        if let Poll::Ready(val) = Pin::new(&mut self.future).poll(cx) {
            return Poll::Ready(Either::Left(val));
        }

        // タイマーが満了したかチェック
        if let Poll::Ready(()) = Pin::new(&mut self.timer).poll(cx) {
            return Poll::Ready(Either::Right(()));
        }

        Poll::Pending
    }
}

// 使用例:
// match Timeout::new(fetch_data(), Duration::from_secs(5)).await {
//     Either::Left(data) => println!("データを取得: {data}"),
//     Either::Right(()) => println!("タイムアウトしました！"),
// }
```

**重要なポイント**: `select` や `timeout` は、単に2つのFutureをポーリングしてどちらが先に完了するかを確認しているだけです。非同期エコシステム全体が、このシンプルなプリミティブ（poll、Pending/Ready、Waker）の上に構築されています。

</details>

---
