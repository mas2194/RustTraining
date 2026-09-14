# 16. 非同期処理（Async/Await）のエッセンス 🔴

> **学ぶこと:**
> - Rust の `Future` トレイトが Go の goroutine や Python の asyncio とどのように異なるか
> - Tokio クイックスタート: タスクの生成（spawn）、`join!`、およびランタイム設定
> - よくある非同期処理の落とし穴とその修正方法
> - `spawn_blocking` でブロッキング処理をオフロードすべきタイミング

## Future、ランタイム、そして `async fn`

Rust の非同期モデルは、Go の goroutine や Python の `asyncio` とは**根本的に異なります**。
まずは以下の3つの概念を理解すれば、スムーズに始めることができます:

1. **`Future` は遅延評価される状態機械（ステートマシン）である** — `async fn` を呼び出しても即座には何も実行されません。ポーリング（poll）される必要のある `Future` が返されるだけです。
2. **Future をポーリングするにはランタイムが必要である** — `tokio`、`async-std`、`smol` など。標準ライブラリは `Future` を定義していますが、ランタイムは提供していません。
3. **`async fn` は糖衣構文（構文糖）である** — コンパイラはこれを `Future` を実装した状態機械へと変換（脱糖）します。

```rust
// Future は単なるトレイト:
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

// async fn の脱糖（desugar）後:
// fn fetch_data(url: &str) -> impl Future<Output = Result<Vec<u8>, Error>>
async fn fetch_data(url: &str) -> Result<Vec<u8>, reqwest::Error> {
    let response = reqwest::get(url).await?;  // .await は準備ができるまで処理を一時中断（yield）する
    let bytes = response.bytes().await?;
    Ok(bytes.to_vec())
}
```

### Tokio クイックスタート

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

```rust,ignore
use tokio::time::{sleep, Duration};
use tokio::task;

#[tokio::main]
async fn main() {
    // 並行タスクを生成（軽量スレッドのようなもの）:
    let handle_a = task::spawn(async {
        sleep(Duration::from_millis(100)).await;
        "タスク A 完了"
    });

    let handle_b = task::spawn(async {
        sleep(Duration::from_millis(50)).await;
        "タスク B 完了"
    });

    // 両方を .await — 逐次実行ではなく並行に実行される:
    let (a, b) = tokio::join!(handle_a, handle_b);
    println!("{}, {}", a.unwrap(), b.unwrap());
}
```

### 非同期処理でよくある落とし穴

| 落とし穴 | 発生原因 | 修正方法 |
|---------|---------------|-----|
| 非同期内でのブロッキング | `std::thread::sleep` や CPU 負荷の高い処理がエグゼキュータをブロックする | `tokio::task::spawn_blocking` または `rayon` を使用する |
| `Send` 境界エラー | `.await` を跨いで保持された Future が `!Send` 型（例: `Rc`, `MutexGuard`）を含んでいる | 非 Send の値を `.await` の前にドロップするように構造を見直す |
| Future がポーリングされない | `.await` も spawn も行わずに `async fn` を呼び出している（何も実行されない） | 返された future に対して必ず `.await` または `tokio::spawn` を行う |
| `.await` を跨いで `MutexGuard` を保持 | `std::sync::MutexGuard` は `!Send` である。非同期タスクは別のスレッドで再開される可能性がある | `tokio::sync::Mutex` を使用するか、`.await` の前にガードをドロップする |
| 意図しない逐次実行 | `let a = foo().await; let b = bar().await;` は順次実行される | 並行実行には `tokio::join!` または `tokio::spawn` を使用する |

```rust
// ❌ 非同期エグゼキュータのブロッキング:
async fn bad() {
    std::thread::sleep(std::time::Duration::from_secs(5)); // スレッド全体をブロックしてしまう！
}

// ✅ ブロッキング処理をオフロードする:
async fn good() {
    tokio::task::spawn_blocking(|| {
        std::thread::sleep(std::time::Duration::from_secs(5)); // 専用のブロッキングスレッドプール上で実行
    }).await.unwrap();
}
```

> **包括的な非同期処理の解説**: `Stream`、`select!`、キャンセル安全性、構造化並行性、および `tower` ミドルウェアの詳細については、専用の **Async Rust Training** ガイドを参照してください。本セクションでは、基本的な非同期コードを読み書きするのに十分な内容を網羅しています。

### タスク生成と構造化並行性

Tokio の `spawn` は新しい非同期タスクを生成します — `thread::spawn` と似ていますが、はるかに軽量です:

```rust,ignore
use tokio::task;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // 3つの並行タスクを生成
    let h1 = task::spawn(async {
        sleep(Duration::from_millis(200)).await;
        "ユーザープロファイル取得完了"
    });

    let h2 = task::spawn(async {
        sleep(Duration::from_millis(100)).await;
        "注文履歴取得完了"
    });

    let h3 = task::spawn(async {
        sleep(Duration::from_millis(150)).await;
        "レコメンデーション取得完了"
    });

    // 3つすべてを並行に待機（逐次待機ではない！）
    let (r1, r2, r3) = tokio::join!(h1, h2, h3);
    println!("{}", r1.unwrap());
    println!("{}", r2.unwrap());
    println!("{}", r3.unwrap());
}
```

**`join!` vs `try_join!` vs `select!`**:

| マクロ | 挙動 | 使用場面 |
|-------|----------|----------|
| `join!` | **すべての** Future の完了を待機 | 全タスクが完了する必要がある場合 |
| `try_join!` | すべてを待機するが、最初の `Err` で即座に終了（ショートサーキット） | タスクが `Result` を返す場合 |
| `select!` | **最初の** Future が完了した時点で制御を戻す | タイムアウト、キャンセル処理 |

```rust,ignore
use tokio::time::{timeout, Duration};

async fn fetch_with_timeout() -> Result<String, Box<dyn std::error::Error>> {
    let result = timeout(Duration::from_secs(5), async {
        // 時間のかかるネットワーク呼び出しをシミュレート
        tokio::time::sleep(Duration::from_millis(100)).await;
        Ok::<_, Box<dyn std::error::Error>>("data".to_string())
    }).await??; // 最初の ? で Elapsed をアンラップし、2番目の ? で内部の Result をアンラップ

    Ok(result)
}
```

### `Send` 境界と Future が `Send` でなければならない理由

`tokio::spawn` で Future を生成すると、その Future は別の OS スレッド上で再開される可能性があります。
これは、Future が `Send` でなければならないことを意味します。よくある落とし穴:

```rust,ignore
use std::rc::Rc;

async fn not_send() {
    let rc = Rc::new(42); // Rc は !Send
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{}", rc); // rc は .await を跨いで保持される — Future は !Send となる
}

// 修正案 1: .await の前にドロップする
async fn fixed_drop() {
    let data = {
        let rc = Rc::new(42);
        *rc // 値をコピーして取り出す
    }; // rc はここでドロップされる
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{}", data); // 単なる i32 であり、Send である
}

// 修正案 2: Rc の代わりに Arc を使用する
async fn fixed_arc() {
    let arc = std::sync::Arc::new(42); // Arc は Send
    tokio::time::sleep(std::time::Duration::from_millis(10)).await;
    println!("{}", arc); // ✅ Future は Send である
}
```

> **包括的な非同期処理の解説**: `Stream`、`select!`、キャンセル安全性、構造化並行性、および `tower` ミドルウェアの詳細については、専用の **Async Rust Training** ガイドを参照してください。本セクションでは、基本的な非同期コードを読み書きするのに十分な内容を網羅しています。

> **関連情報:** 同期チャンネルについては「[第5章 チャンネルとメッセージパッシング](ch05-channels-and-message-passing.md)」、OS スレッドと非同期タスクの比較については「[第6章 並行性 vs 並列性 vs スレッド](ch06-concurrency-vs-parallelism-vs-threads.md)」を参照してください。

> **重要ポイント — 非同期処理**
> - `async fn` は遅延評価される `Future` を返す — `.await` または spawn するまで何も実行されない
> - 非同期コンテキスト内での CPU 負荷の高い処理やブロッキング処理には `tokio::task::spawn_blocking` を使用する
> - `.await` を跨いで `std::sync::MutexGuard` を保持しない — 代わりに `tokio::sync::Mutex` を使用する
> - spawn される Future は `Send` でなければならない — `.await` ポイントの前に `!Send` 型をドロップする

---

### 演習: タイムアウト付き並行フェッチャー ★★ (約25分)

それぞれ `tokio::time::sleep` を用いてネットワーク呼び出しをシミュレートする3つのタスクを `tokio::spawn` で起動する非同期関数 `fetch_all` を作成してください。3つのタスクすべてを `tokio::try_join!` で待ち合わせ、全体を `tokio::time::timeout(Duration::from_secs(5), ...)` でラップします。いずれかのタスクが失敗するか期限が切れた場合はエラーを返し、成功時は `Result<Vec<String>, ...>` を返すようにしてください。

<details>
<summary>🔑 解答例</summary>

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
    .await??;

    Ok(vec![a?, b?, c?])
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

***
