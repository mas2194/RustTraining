## 学習ロードマップと次のステップ

> **学習内容:** 体系的な学習ロードマップ（1〜2週目、1〜2ヶ月目、3ヶ月目以降）、おすすめの書籍とリソース、C#開発者が陥りやすい落とし穴（所有権の混同、ボローチェッカーとの格闘）、および `tracing` と `ILogger` の比較による構造化オブザーバビリティ。
>
> **難易度:** 🟢 初級

### 最初のステップ（1〜2週目）
1. **環境のセットアップ**
   - [rustup.rs](https://rustup.rs/) から Rust をインストールする
   - VS Code に rust-analyzer 拡張機能を設定する
   - 最初のプロジェクトを作成する: `cargo new hello_world`

2. **基本の習得**
   - 簡単な演習で所有権の練習をする
   - さまざまなパラメータ型（`&str`、`String`、`&mut`）を持つ関数を書く
   - 基本的な構造体とメソッドを実装する

3. **エラー処理の練習**
   - C# の try-catch コードを Result ベースのパターンに変換する
   - `?` 演算子と `match` 式の練習をする
   - カスタムエラー型を実装する

### 中期目標（1〜2ヶ月目）
1. **コレクションとイテレータ**
   - `Vec<T>`、`HashMap<K,V>`、`HashSet<T>` を使いこなす
   - イテレータメソッド（`map`、`filter`、`collect`、`fold`）を学ぶ
   - `for` ループとイテレータチェーンの使い分けを練習する

2. **トレイトとジェネリクス**
   - 一般的なトレイトを実装する: `Debug`、`Clone`、`PartialEq`
   - ジェネリックな関数や構造体を書く
   - トレイト境界（trait bounds）と where 節を理解する

3. **プロジェクト構造**
   - コードをモジュールに分割して整理する
   - `pub` による可視性を理解する
   - crates.io の外部クレートを利用する

### 発展的なトピック（3ヶ月目以降）
1. **並行性（Concurrency）**
   - `Send` トレイトと `Sync` トレイトについて学ぶ
   - 基本的な並列処理に `std::thread` を使用する
   - 非同期プログラミングのために `tokio` を探求する

2. **メモリ管理**
   - 共有所有権のための `Rc<T>` と `Arc<T>` を理解する
   - ヒープ割り当てに `Box<T>` を使用すべき場面を学ぶ
   - 複雑なシナリオに対応するライフタイムを習得する

3. **実践的なプロジェクト**
   - `clap` を使って CLI ツールを構築する
   - `axum` や `warp` で Web API を作成する
   - ライブラリを作成して crates.io に公開する

### おすすめの学習リソース

#### 書籍
- **"The Rust Programming Language"**（オンライン無料）- 公式ブック（通称「The Book」）
- **"Rust by Example"**（オンライン無料）- 実践的なコード例集
- **"Programming Rust"**（Jim Blandy 著）- 深い技術的解説

#### オンラインリソース
- [Rust Playground](https://play.rust-lang.org/) - ブラウザ上でコードを試せる環境
- [Rustlings](https://github.com/rust-lang/rustlings) - インタラクティブな練習問題集
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) - 実践的なサンプルコード

#### 実践プロジェクトのアイデア
1. **コマンドライン計算機** - 列挙型とパターンマッチングの練習
2. **ファイル整理ツール** - ファイルシステム操作とエラー処理の習得
3. **JSON プロセッサ** - serde とデータ変換の学習
4. **HTTP サーバー** - 非同期プログラミングとネットワークの理解
5. **データベースライブラリ** - トレイト、ジェネリクス、エラー処理の習得

### C#開発者が陥りやすい落とし穴

#### 所有権の混同
```rust
// 非推奨: ムーブされた値を使おうとする
fn wrong_way() {
    let s = String::from("hello");
    takes_ownership(s);
    // println!("{}", s); // エラー: s はムーブされています
}

// 推奨: 参照を使うか、必要に応じて clone する
fn right_way() {
    let s = String::from("hello");
    borrows_string(&s);
    println!("{}", s); // OK: ここでも s の所有権は保持されている
}

fn takes_ownership(s: String) { /* ここで s がムーブされる */ }
fn borrows_string(s: &str) { /* ここで s が借用される */ }
```

#### ボローチェッカーとの格闘
```rust
// 非推奨: 複数の可変参照
fn wrong_borrowing() {
    let mut v = vec![1, 2, 3];
    let r1 = &mut v;
    // let r2 = &mut v; // エラー: 2回以上可変として借用することはできません
}

// 推奨: 可変借用のスコープを限定する
fn right_borrowing() {
    let mut v = vec![1, 2, 3];
    {
        let r1 = &mut v;
        r1.push(4);
    } // ここで r1 はスコープを抜ける
    
    let r2 = &mut v; // OK: 他に可変借用は存在しない
    r2.push(5);
}
```

#### null値を期待してしまう
```rust
// 非推奨: null のような振る舞いを期待する
fn no_null_in_rust() {
    // let s: String = null; // Rust に null はありません！
}

// 推奨: Option<T> を明示的に使う
fn use_option_instead() {
    let maybe_string: Option<String> = None;
    
    match maybe_string {
        Some(s) => println!("文字列を取得: {}", s),
        None => println!("文字列はありません"),
    }
}
```

### 最後に

1. **コンパイラを受け入れる** - Rust のコンパイラエラーは敵ではなく、親切な案内役です
2. **小さく始める** - シンプルなプログラムから始めて、徐々に複雑さを増やしていきましょう
3. **他の人のコードを読む** - GitHub で人気のあるクレートのコードを読んで学びましょう
4. **助けを求める** - Rust コミュニティは温かく協力的です
5. **定期的に練習する** - 練習を重ねることで、Rust の概念が自然に身につきます

覚えておいてください: Rust には学習曲線がありますが、メモリ安全性、優れたパフォーマンス、恐れなき並行性（fearless concurrency）という大きな見返りがあります。最初は制限のように感じられる所有権システムも、正しく効率的なプログラムを書くための強力な武器になります。

---

**おめでとうございます！** C# から Rust への移行に向けた強固な基盤が整いました。まずは小さなプロジェクトから始め、学習プロセスを焦らず、徐々により複雑なアプリケーションに挑戦していきましょう。Rust がもたらす安全性とパフォーマンスの利点は、最初の学習投資に見合うだけの十分な価値があります。


<!-- ch16.2a: Structured Observability with tracing -->
## 構造化オブザーバビリティ: `tracing` vs ILogger / Serilog

C# 開発者は、ログメッセージに型付きのキー・バリュープロパティを持たせる `ILogger`、**Serilog**、**NLog** などの**構造化ロギング**に親しんでいます。Rust の `log` クレートは基本的なレベル別ロギングを提供しますが、スパン、非同期の考慮、分散トレーシングのサポートを備えた構造化オブザーバビリティの本番標準となっているのは **`tracing`** です。

### なぜ `log` ではなく `tracing` なのか

| 機能 | `log` クレート | `tracing` クレート | C# の同等機能 |
|---------|------------|-----------------|----------------|
| レベル別メッセージ | ✅ `info!()`, `error!()` | ✅ `info!()`, `error!()` | `ILogger.LogInformation()` |
| 構造化フィールド | ❌ 文字列補間のみ | ✅ 型付きキー・バリューフィールド | Serilog `Log.Information("{User}", user)` |
| スパン（スコープ付きコンテキスト） | ❌ | ✅ `#[instrument]`, `span!()` | `ILogger.BeginScope()` |
| 非同期対応 | ❌ `.await` をまたぐとコンテキスト喪失 | ✅ `.await` を越えてスパンが追随 | `Activity` / `DiagnosticSource` |
| 分散トレーシング | ❌ | ✅ OpenTelemetry 統合 | `System.Diagnostics.Activity` |
| 多彩な出力形式 | 基本的 | JSON、pretty、compact、OTLP | Serilog シンク（Sink） |

### はじめに
```toml
# Cargo.toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

### 基本的な使い方: 構造化ロギング
```csharp
// C# Serilog
Log.Information("Processing order {OrderId} for {Customer}, total {Total:C}",
    orderId, customer.Name, order.Total);
// 出力: Processing order 12345 for Alice, total $99.95
// JSON:  {"OrderId": 12345, "Customer": "Alice", "Total": 99.95, ...}
```

```rust
use tracing::{info, warn, error, debug, instrument};

// 構造化フィールド — 文字列補間ではなく型付き
info!(order_id = 12345, customer = "Alice", total = 99.95,
      "Processing order");
// 出力: INFO Processing order order_id=12345 customer="Alice" total=99.95
// JSON:  {"order_id": 12345, "customer": "Alice", "total": 99.95, ...}

// 動的な値
let order_id = 12345;
info!(order_id, "Order received");  // フィールド名 = 変数名 の省略記法

// 条件付きフィールド
if let Some(promo) = promo_code {
    info!(order_id, promo_code = %promo, "Promo applied");
    //                        ^ % は Display フォーマットを使用することを意味する
    //                        ? は Debug フォーマットを使用することを意味する
}
```

### スパン: 非同期コードのためのキラー機能

スパンは、関数呼び出しや `.await` ポイントをまたいでフィールドを引き継ぐスコープ付きコンテキストです。`ILogger.BeginScope()` に似ていますが、非同期セーフです。

```csharp
// C# — Activity / BeginScope
using var activity = new Activity("ProcessOrder").Start();
activity.SetTag("order_id", orderId);

using (_logger.BeginScope(new Dictionary<string, object> { ["OrderId"] = orderId }))
{
    _logger.LogInformation("Starting processing");
    await ProcessPaymentAsync();
    _logger.LogInformation("Payment complete");  // OrderId はスコープ内に残る
}
```

```rust
use tracing::{info, instrument, Instrument};

// #[instrument] は関数の引数をフィールドとして自動的にスパンを作成します
#[instrument(skip(db), fields(customer_name))]
async fn process_order(order_id: u64, db: &Database) -> Result<(), AppError> {
    let order = db.get_order(order_id).await?;
    
    // 現在のスパンに動的にフィールドを追加する
    tracing::Span::current().record("customer_name", &order.customer_name.as_str());
    
    info!("Starting processing");
    process_payment(&order).await?;        // .await をまたいでもスパンのコンテキストが維持される！
    info!(items = order.items.len(), "Payment complete");
    Ok(())
}
// この関数内のすべてのログメッセージに自動的に以下が含まれます:
//   order_id=12345 customer_name="Alice"
// ネストされた非同期呼び出しでも維持されます！

// 手動でのスパン作成（BeginScope に類似）
async fn batch_process(orders: Vec<u64>, db: &Database) {
    for order_id in orders {
        let span = tracing::info_span!("process_order", order_id);
        
        // .instrument(span) は Future にスパンを関連付けます
        process_order(order_id, db)
            .instrument(span)
            .await
            .unwrap_or_else(|e| error!("Failed: {e}"));
    }
}
```

### Subscriber の設定（Serilog の Sink に相当）

```rust
use tracing_subscriber::{fmt, EnvFilter, layer::SubscriberExt, util::SubscriberInitExt};

fn init_tracing() {
    // 開発環境: 人間が読みやすいカラー出力
    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| "my_app=debug,tower_http=info".into()))
        .with(fmt::layer().pretty())  // カラー化されインデントされたスパン
        .init();
}

fn init_tracing_production() {
    // 本番環境: ログ集約のための JSON 出力（Serilog の JSON Sink に類似）
    tracing_subscriber::registry()
        .with(EnvFilter::new("my_app=info"))
        .with(fmt::layer().json())  // 構造化 JSON
        .init();
    // 出力: {"timestamp":"...","level":"INFO","fields":{"order_id":123},...}
}
```

```bash
# 環境変数でログレベルを制御（Serilog の MinimumLevel に類似）
RUST_LOG=my_app=debug,hyper=warn cargo run
RUST_LOG=trace cargo run  # すべて出力
```

### Serilog → tracing 移行チートシート

| Serilog / ILogger | tracing | 備考 |
|-------------------|---------|-------|
| `Log.Information("{Key}", val)` | `info!(key = val, "message")` | フィールドは文字列補間ではなく型付き |
| `Log.ForContext("Key", val)` | `span.record("key", val)` | 現在のスパンにフィールドを追加 |
| `using BeginScope(...)` | `#[instrument]` または `info_span!()` | `#[instrument]` で自動化 |
| `.WriteTo.Console()` | `fmt::layer()` | 人間が読みやすい形式 |
| `.WriteTo.Seq()` / `.File()` | `fmt::layer().json()` + ファイルリダイレクト | または `tracing-appender` を使用 |
| `.Enrich.WithProperty()` | `span!(Level::INFO, "name", key = val)` | スパンのフィールド |
| `LogEventLevel.Debug` | `tracing::Level::DEBUG` | 同じ概念 |
| `{@Object}` の分解（デストラクチャリング） | `field = ?value` (Debug) または `%value` (Display) | `?` = Debug, `%` = Display |

### OpenTelemetry との統合
```toml
# 分散トレーシング用（System.Diagnostics + OTLP エクスポーターに類似）
[dependencies]
tracing-opentelemetry = "0.22"
opentelemetry = "0.21"
opentelemetry-otlp = "0.14"
```

```rust
// コンソール出力と並行して OpenTelemetry レイヤーを追加
use tracing_opentelemetry::OpenTelemetryLayer;

fn init_otel() {
    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(opentelemetry_otlp::new_exporter().tonic())
        .install_batch(opentelemetry_sdk::runtime::Tokio)
        .expect("Failed to create OTLP tracer");

    tracing_subscriber::registry()
        .with(OpenTelemetryLayer::new(tracer))  // スパンを Jaeger / Tempo に送信
        .with(fmt::layer())                      // コンソールにも出力
        .init();
}
// これで #[instrument] スパンが自動的に分散トレースになります！
```

***
