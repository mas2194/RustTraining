## 非同期プログラミング: C# Task vs Rust Future

> **学習内容:** Rust の遅延評価型（lazy）`Future` に対する C# の即時実行型（eager）`Task`、エグゼキュータモデル（tokio）、`CancellationToken` に対する `Drop` + `select!` によるキャンセル、および並行リクエストの実践的なパターン。
>
> **難易度:** 🔴 上級

C# 開発者は `async`/`await` に非常に親しんでいます。Rust でも同じキーワードが使われますが、その実行モデルは根本的に異なります。

### エグゼキュータモデル

```csharp
// C# — ランタイムが組み込みのスレッドプールとタスクスケジューラを提供
// 設定なしで async/await が「そのまま」動作します
public async Task<string> FetchDataAsync(string url)
{
    using var client = new HttpClient();
    return await client.GetStringAsync(url);  // .NET スレッドプールによってスケジュールされます
}
// .NET がスレッドプール、タスクスケジューリング、同期コンテキストを管理します
```

```rust
// Rust — 組み込みの非同期ランタイムはありません。エグゼキュータを自分で選択します。
// 最も人気があるのは tokio です。
async fn fetch_data(url: &str) -> Result<String, reqwest::Error> {
    let body = reqwest::get(url).await?.text().await?;
    Ok(body)
}

// 非同期コードを実行するには、必ずランタイムが必要です:
#[tokio::main]  // このマクロが tokio ランタイムをセットアップします
async fn main() {
    let data = fetch_data("https://example.com").await.unwrap();
    println!("{}", &data[..100]);
}
```

### Future vs Task

| | C# `Task<T>` | Rust `Future<Output = T>` |
|---|---|---|
| **実行方式** | 生成されると即座に開始（Eager） | **遅延評価（Lazy）** — `.await` されるまで何もしない |
| **ランタイム** | 組み込み（CLR スレッドプール） | 外部クレート（tokio、async-std 等） |
| **キャンセル** | `CancellationToken` | `Future` の破棄（Drop）または `tokio::select!` |
| **状態機械** | コンパイラが生成 | コンパイラが生成 |
| **サイズ** | ヒープアロケーション | ボックス化されない限りスタック割り当て |

```rust
// 重要: Rust の Future は遅延評価されます！
async fn compute() -> i32 { println!("Computing!"); 42 }

let future = compute();  // 何も出力されません！Future はまだポーリングされていません。
let result = future.await; // ここで初めて "Computing!" が出力されます
```

```csharp
// C# の Task は即座に開始されます！
var task = ComputeAsync();  // "Computing!" が即座に出力されます
var result = await task;    // 完了を待機するだけです
```

### キャンセル処理: CancellationToken vs Drop / select!

```csharp
// C# — CancellationToken による協調的キャンセル
public async Task ProcessAsync(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        await Task.Delay(1000, ct);  // キャンセルされた場合は例外をスロー
        DoWork();
    }
}

var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
await ProcessAsync(cts.Token);
```

```rust
// Rust — Future の破棄（Drop）または tokio::select! によるキャンセル
use tokio::time::{sleep, Duration};

async fn process() {
    loop {
        sleep(Duration::from_secs(1)).await;
        do_work();
    }
}

// select! を使ったタイムアウトパターン
async fn run_with_timeout() {
    tokio::select! {
        _ = process() => { println!("完了しました"); }
        _ = sleep(Duration::from_secs(5)) => { println!("タイムアウトしました！"); }
    }
    // select! がタイムアウトのブランチを選択すると、process() の Future は破棄（Drop）されます
    // — 自動的にクリーンアップが行われ、CancellationToken は不要です
}
```

### 実践パターン: タイムアウト付きの並行リクエスト

```csharp
// C# — タイムアウト付きの並行 HTTP リクエスト
public async Task<string[]> FetchAllAsync(string[] urls, CancellationToken ct)
{
    var tasks = urls.Select(url => httpClient.GetStringAsync(url, ct));
    return await Task.WhenAll(tasks);
}
```

```rust
// Rust — tokio::join! または futures::join_all による並行リクエスト
use futures::future::join_all;

async fn fetch_all(urls: &[&str]) -> Vec<Result<String, reqwest::Error>> {
    let futures = urls.iter().map(|url| reqwest::get(*url));
    let responses = join_all(futures).await;

    let mut results = Vec::new();
    for resp in responses {
        results.push(resp?.text().await);
    }
    results
}

// タイムアウト付き:
async fn fetch_all_with_timeout(urls: &[&str]) -> Result<Vec<String>, &'static str> {
    tokio::time::timeout(
        Duration::from_secs(10),
        async {
            let futures: Vec<_> = urls.iter()
                .map(|url| async { reqwest::get(*url).await?.text().await })
                .collect();
            let results = join_all(futures).await;
            results.into_iter().collect::<Result<Vec<_>, _>>()
        }
    )
    .await
    .map_err(|_| "リクエストがタイムアウトしました")?
    .map_err(|_| "リクエストが失敗しました")
}
```

<details>
<summary><strong>🏋️ 演習問題: 非同期タイムアウトパターン</strong> (クリックして展開)</summary>

**課題**: 2つの URL から並行してフェッチを行い、先に応答があった方を返して、もう一方をキャンセルする非同期関数を作成してください（これは C# の `Task.WhenAny` に相当します）。

<details>
<summary>🔑 解答例</summary>

```rust
use tokio::time::{sleep, Duration};

// 疑似的な非同期フェッチ
async fn fetch(url: &str, delay_ms: u64) -> String {
    sleep(Duration::from_millis(delay_ms)).await;
    format!("{url} からの応答")
}

async fn fetch_first(url1: &str, url2: &str) -> String {
    tokio::select! {
        result = fetch(url1, 200) => {
            println!("URL 1 が勝ちました");
            result
        }
        result = fetch(url2, 500) => {
            println!("URL 2 が勝ちました");
            result
        }
    }
    // 負けたブランチの Future は自動的に破棄（キャンセル）されます
}

#[tokio::main]
async fn main() {
    let result = fetch_first("https://fast.api", "https://slow.api").await;
    println!("{result}");
}
```

**重要なポイント**: `tokio::select!` は Rust における `Task.WhenAny` の同等機能です — 複数の Future を競合（race）させ、最初のものが完了した時点で終了し、残りを破棄（キャンセル）します。

</details>
</details>

### tokio::spawn による独立したタスクの生成

C# では、`Task.Run` を使用して呼び出し元から独立して実行される作業を開始します。Rust における同等機能は `tokio::spawn` です:

```rust
use tokio::task;

async fn background_work() {
    // 独立して実行される — 呼び出し元の Future が破棄されても継続します
    let handle = task::spawn(async {
        tokio::time::sleep(Duration::from_secs(2)).await;
        42
    });

    // 生成されたタスクが実行されている間に他の作業を行う...
    println!("他の作業を実行中");

    // 結果が必要になった時点で await します
    let result = handle.await.unwrap(); // 42
}
```

```csharp
// C# の同等コード
var task = Task.Run(async () => {
    await Task.Delay(2000);
    return 42;
});
// 他の作業を行う...
var result = await task;
```

**主な違い**: 通常の `async {}` ブロックは遅延評価されます — await されるまで何もしません。一方、`tokio::spawn` は C# の `Task.Run` のように、ランタイム上で即座に実行を開始します。

### Pin: C#には存在しない概念がRustの非同期処理にある理由

C# 開発者が `Pin` に遭遇することはありません — CLR のガベージコレクタがオブジェクトをメモリ上で自由に移動させ、すべての参照を自動的に更新してくれるためです。一方、Rust には GC がありません。コンパイラが `async fn` を状態機械（ステートマシン）の構造体に変換する際、その構造体は自身のフィールドを指す内部ポインタ（自己参照）を保持する場合があります。この構造体がメモリ上で移動（ムーブ）されると、それらのポインタが無効化されてしまいます。

`Pin<T>` は、**「この値はメモリ上で移動（ムーブ）されない」** ことを保証するラッパーです。

```rust
// 以下のようなコンテキストで Pin を目にします:
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
    //           ^^^^^^^^^^^^^^ pin されているため、内部参照の有効性が保たれる
}

// トレイトからボックス化された Future を返す場合:
fn make_future() -> Pin<Box<dyn Future<Output = i32> + Send>> {
    Box::pin(async { 42 })
}
```

**実際には、自分で `Pin` を直接書く機会はほとんどありません。** `async fn` や `.await` 構文が自動的に処理してくれます。Pin に遭遇するのは主に以下のような場合だけです:
- コンパイラのエラーメッセージ（提示されたアドバイスに従えば解決します）
- `tokio::select!`（`pin!()` マクロを使用します）
- `dyn Future` を返すトレイトメソッド（`Box::pin(async { ... })` を使用します）

> **さらに詳しく知りたい方へ:** 姉妹編の [Async Rust Training](../../async-book/src/ch04-pin-and-unpin.md) では、Pin、Unpin、自己参照構造体、構造的 Pin留め（structural pinning）について詳細に解説しています。

***
