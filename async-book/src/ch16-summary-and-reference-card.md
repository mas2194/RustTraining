# まとめとリファレンスカード

## クイックリファレンスカード

### 非同期のメンタルモデル

```text
┌─────────────────────────────────────────────────────┐
│  async fn → 状態機械（enum） → impl Future          │
│  .await   → 内部の Future を poll()                 │
│  エグゼキュータ → loop { poll(); sleep_until_woken(); } │
│  waker    → 「エグゼキュータさん、もう一度ポーリングして」 │
│  Pin      → 「メモリ上で移動しないことを保証します」    │
└─────────────────────────────────────────────────────┘
```

### よくあるパターンのチートシート

| 目的 | 使用するもの |
|------|--------------|
| 2つのFutureを並行に実行する | `tokio::join!(a, b)` |
| 2つのFutureを競合させる（レース） | `tokio::select! { ... }` |
| バックグラウンドタスクをスポーンする | `tokio::spawn(async { ... })` |
| 非同期内でブロッキングコードを実行する | `tokio::task::spawn_blocking(\|\| { ... })` |
| 並行数を制限する | `Semaphore::new(N)` |
| 多数のタスク結果を収集する | `JoinSet` |
| タスク間で状態を共有する | `Arc<Mutex<T>>` またはチャネル |
| グレースフルシャットダウン | `watch::channel` + `select!` |
| ストリームをN個ずつ処理する | `.buffer_unordered(N)` |
| Futureにタイムアウトを設定する | `tokio::time::timeout(dur, fut)` |
| バックオフ付きリトライ | カスタムコンビネータ（第13章参照） |

### Pinning（ピン留め）クイックリファレンス

| 状況 | 使用するもの |
|------|--------------|
| Futureをヒープ上でピン留めする | `Box::pin(fut)` |
| Futureをスタック上でピン留めする | `tokio::pin!(fut)` |
| `Unpin` な型をピン留めする | `Pin::new(&mut val)` — 安全、コストゼロ |
| ピン留めされたトレイトオブジェクトを返す | `-> Pin<Box<dyn Future<Output = T> + Send>>` |

### チャネル選択ガイド

| チャネル | 送信者（Producers） | 受信者（Consumers） | 値 | 使用場面 |
|---------|--------------------|-------------------|-----|----------|
| `mpsc` | N | 1 | ストリーム | 作業キュー、イベントバス |
| `oneshot` | 1 | 1 | 単一 | リクエスト/レスポンス、完了通知 |
| `broadcast` | N | N | 全員がすべて受信 | ファンアウト通知、シャットダウンシグナル |
| `watch` | 1 | N | 最新値のみ | 設定の更新、ヘルス状態 |

### Mutex選択ガイド

| Mutex | 使用場面 |
|-------|----------|
| `std::sync::Mutex` | ロック保持時間が短く、決して `.await` を跨がない |
| `tokio::sync::Mutex` | ロックを `.await` を跨いで保持する必要がある |
| `parking_lot::Mutex` | 競合が激しく、`.await` を含まず、パフォーマンスが要求される |
| `tokio::sync::RwLock` | 読み取りが多く書き込みが少ない、ロックが `.await` を跨ぐ |

### 意思決定クイックリファレンス

```text
並行性が必要？
├── I/Oバウンド → async/await
├── CPUバウンド → rayon / std::thread
└── 混在 → CPU処理部分に spawn_blocking

ランタイムの選択？
├── サーバーアプリ → tokio
├── ライブラリ → ランタイム非依存（futures クレート）
├── 組み込み → embassy
└── 最小構成 → smol

並行なFutureが必要？
├── 'static + Send にできる → tokio::spawn
├── 'static + !Send にできる → LocalSet
├── 'static にできない → FuturesUnordered
└── 追跡/中断が必要 → JoinSet
```

### よくあるエラーメッセージと修正方法

| エラー | 原因 | 修正方法 |
|-------|------|----------|
| `future is not Send` | `!Send` 型を `.await` を跨いで保持している | 値のスコープを狭めて `.await` の前にドロップされるようにするか、`current_thread` ランタイムを使用する |
| スポーン時の `borrowed value does not live long enough` | `tokio::spawn` に `'static` が要求される | `Arc`、`clone()`、または `FuturesUnordered` を使用する |
| `the trait Future is not implemented for ()` | `.await` の付け忘れ | 非同期呼び出しに `.await` を追加する |
| poll 内での `cannot borrow as mutable` | 自己参照による借用エラー | `Pin<&mut Self>` を正しく使用する（第4章参照） |
| プログラムが警告なしにハングする | `waker.wake()` の呼び出し忘れ | すべての `Pending` パスで Waker が登録され、トリガーされることを確認する |

### 参考文献・さらなる学習

| リソース | 推奨理由 |
|----------|----------|
| [Tokio Tutorial](https://tokio.rs/tokio/tutorial) | 公式ハンズオンガイド — 最初のプロジェクトに最適 |
| [Async Book（公式）](https://rust-lang.github.io/async-book/) | 言語レベルでの `Future`、`Pin`、`Stream` の解説 |
| [Jon Gjengset — Crust of Rust: async/await](https://www.youtube.com/watch?v=ThjvMReOXYM) | ライブコーディングによる内部構造の2時間徹底解説 |
| [Alice Ryhl — Actors with Tokio](https://ryhl.io/blog/actors-with-tokio/) | ステートフルサービス向けの本番アーキテクチャパターン |
| [Without Boats — Pin, Unpin, and why Rust needs them](https://without.boats/blog/pin/) | 言語デザイナーによる導入動機の解説 |
| [Tokio mini-Redis](https://github.com/tokio-rs/mini-redis) | 完全な非同期Rustプロジェクト — 学習に適した本番品質コード |
| [Tower ドキュメント](https://docs.rs/tower) | axum、tonic、hyper で使われるミドルウェア/サービスアーキテクチャ |

---

*非同期Rustトレーニングガイド 完*
