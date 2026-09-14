## 段階的な導入戦略

> **ここで学ぶこと:** C# / .NET 組織に Rust を導入するためのフェーズ分けされたアプローチ — 学習と実験（1〜4週目）から、パフォーマンスが重要なコンポーネントの置き換え（5〜8週目）、新規マイクロサービスの開発（9〜12週目）まで、具体的なチーム導入タイムラインとともに解説します。
>
> **難易度:** 🟡 中級

### フェーズ1: 学習と実験（1〜4週目）

```rust
// コマンドラインツールやユーティリティから始める
// 例: ログファイルアナライザ
use std::fs;
use std::collections::HashMap;
use clap::Parser;

#[derive(Parser)]
#[command(author, version, about)]
struct Args {
    #[arg(short, long)]
    file: String,
    
    #[arg(short, long, default_value = "10")]
    top: usize,
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let args = Args::parse();
    
    let content = fs::read_to_string(&args.file)?;
    let mut word_count = HashMap::new();
    
    for line in content.lines() {
        for word in line.split_whitespace() {
            let word = word.to_lowercase();
            *word_count.entry(word).or_insert(0) += 1;
        }
    }
    
    let mut sorted: Vec<_> = word_count.into_iter().collect();
    sorted.sort_by(|a, b| b.1.cmp(&a.1));
    
    for (word, count) in sorted.into_iter().take(args.top) {
        println!("{}: {}", word, count);
    }
    
    Ok(())
}
```

### フェーズ2: パフォーマンスが重要なコンポーネントの置き換え（5〜8週目）

```rust
// CPU集約型のデータ処理を置き換える
// 例: 画像処理マイクロサービス
use image::{DynamicImage, ImageBuffer, Rgb};
use serde::{Deserialize, Serialize};
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use warp::Filter;

#[derive(Serialize, Deserialize)]
struct ProcessingRequest {
    image_data: Vec<u8>,
    operation: String,
    parameters: serde_json::Value,
}

#[derive(Serialize)]
struct ProcessingResponse {
    processed_image: Vec<u8>,
    processing_time_ms: u64,
}

async fn process_image(request: ProcessingRequest) -> Result<ProcessingResponse, Box<dyn std::error::Error + Send + Sync>> {
    let start = std::time::Instant::now();
    
    let img = image::load_from_memory(&request.image_data)?;
    
    let processed = match request.operation.as_str() {
        "blur" => {
            let radius = request.parameters["radius"].as_f64().unwrap_or(2.0) as f32;
            img.blur(radius)
        }
        "grayscale" => img.grayscale(),
        "resize" => {
            let width = request.parameters["width"].as_u64().unwrap_or(100) as u32;
            let height = request.parameters["height"].as_u64().unwrap_or(100) as u32;
            img.resize(width, height, image::imageops::FilterType::Lanczos3)
        }
        _ => return Err("Unknown operation".into()),
    };
    
    let mut buffer = Vec::new();
    processed.write_to(&mut std::io::Cursor::new(&mut buffer), image::ImageOutputFormat::Png)?;
    
    Ok(ProcessingResponse {
        processed_image: buffer,
        processing_time_ms: start.elapsed().as_millis() as u64,
    })
}

#[tokio::main]
async fn main() {
    let process_route = warp::path("process")
        .and(warp::post())
        .and(warp::body::json())
        .and_then(|req: ProcessingRequest| async move {
            match process_image(req).await {
                Ok(response) => Ok(warp::reply::json(&response)),
                Err(e) => Err(warp::reject::custom(ProcessingError(e.to_string()))),
            }
        });

    warp::serve(process_route)
        .run(([127, 0, 0, 1], 3030))
        .await;
}

#[derive(Debug)]
struct ProcessingError(String);
impl warp::reject::Reject for ProcessingError {}
```

### フェーズ3: 新規マイクロサービスの構築（9〜12週目）

```rust
// Rust でゼロから新しいサービスを構築する
// 例: 認証サービス
use axum::{
    extract::{Query, State},
    http::StatusCode,
    response::Json,
    routing::{get, post},
    Router,
};
use jsonwebtoken::{encode, decode, Header, Validation, EncodingKey, DecodingKey};
use serde::{Deserialize, Serialize};
use sqlx::{Pool, Postgres};
use uuid::Uuid;
use bcrypt::{hash, verify, DEFAULT_COST};

#[derive(Clone)]
struct AppState {
    db: Pool<Postgres>,
    jwt_secret: String,
}

#[derive(Serialize, Deserialize)]
struct Claims {
    sub: String,
    exp: usize,
}

#[derive(Deserialize)]
struct LoginRequest {
    email: String,
    password: String,
}

#[derive(Serialize)]
struct LoginResponse {
    token: String,
    user_id: Uuid,
}

async fn login(
    State(state): State<AppState>,
    Json(request): Json<LoginRequest>,
) -> Result<Json<LoginResponse>, StatusCode> {
    // 注: sqlx::query!() はコンパイル時に検証されるため、ビルド時に稼働中のデータベースを
    // 指す DATABASE_URL が必要です。実行時検証クエリの場合は、代わりに sqlx::query() や
    // sqlx::query_as() を使用します。
    let user = sqlx::query!(
        "SELECT id, password_hash FROM users WHERE email = $1",
        request.email
    )
    .fetch_optional(&state.db)
    .await
    .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let user = user.ok_or(StatusCode::UNAUTHORIZED)?;

    if !verify(&request.password, &user.password_hash)
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
    {
        return Err(StatusCode::UNAUTHORIZED);
    }

    let claims = Claims {
        sub: user.id.to_string(),
        exp: (chrono::Utc::now() + chrono::Duration::hours(24)).timestamp() as usize,
    };

    let token = encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(state.jwt_secret.as_ref()),
    )
    .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(LoginResponse {
        token,
        user_id: user.id,
    }))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let database_url = std::env::var("DATABASE_URL")?;
    let jwt_secret = std::env::var("JWT_SECRET")?;
    
    let pool = sqlx::postgres::PgPoolOptions::new()
        .max_connections(20)
        .connect(&database_url)
        .await?;

    let app_state = AppState {
        db: pool,
        jwt_secret,
    };

    let app = Router::new()
        .route("/login", post(login))
        .with_state(app_state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app).await?;
    
    Ok(())
}
```

***

## チーム導入タイムライン

### 1か月目: 基礎固め

**1〜2週目: 構文と所有権**
- C# との基本的な構文の違い
- 所有権、借用、ライフタイムの理解
- 小さな演習: CLI ツール、ファイル処理

**3〜4週目: エラー処理と型システム**
- 例外 vs `Result<T, E>`
- null 許容型 vs `Option<T>`
- パターンマッチングと網羅的チェック

**推奨される演習課題:**

```rust
// 1〜2週目: ファイルプロセッサ
fn process_log_file(path: &str) -> Result<Vec<String>, std::io::Error> {
    let content = std::fs::read_to_string(path)?;
    let errors: Vec<String> = content
        .lines()
        .filter(|line| line.contains("ERROR"))
        .map(|line| line.to_string())
        .collect();
    Ok(errors)
}

// 3〜4週目: エラー処理を伴う JSON プロセッサ
use serde::{Deserialize, Serialize};

#[derive(Deserialize, Serialize, Debug)]
struct LogEntry {
    timestamp: String,
    level: String,
    message: String,
}

fn parse_log_entries(json_str: &str) -> Result<Vec<LogEntry>, Box<dyn std::error::Error>> {
    let entries: Vec<LogEntry> = serde_json::from_str(json_str)?;
    Ok(entries)
}
```

### 2か月目: 実践的な応用

**5〜6週目: トレイトとジェネリクス**
- インターフェース vs トレイトシステム
- ジェネリクスの制約（境界）
- 一般的なパターンとイディオム

**7〜8週目: 非同期プログラミングと並行性**
- `async`/`await` の類似点と相違点
- スレッド間通信のためのチャネル（Channel）
- スレッド安全性の保証機構

**推奨されるプロジェクト課題:**

```rust
// 5〜6週目: ジェネリックデータプロセッサ
trait DataProcessor<T> {
    type Output;
    type Error;
    
    fn process(&self, data: T) -> Result<Self::Output, Self::Error>;
}

struct JsonProcessor;

impl DataProcessor<&str> for JsonProcessor {
    type Output = serde_json::Value;
    type Error = serde_json::Error;
    
    fn process(&self, data: &str) -> Result<Self::Output, Self::Error> {
        serde_json::from_str(data)
    }
}

// 7〜8週目: 非同期 Web クライアント
async fn fetch_and_process_data(urls: Vec<&str>) -> Result<(), Box<dyn std::error::Error>> {
    let client = reqwest::Client::new();
    
    let tasks: Vec<_> = urls
        .into_iter()
        .map(|url| {
            let client = client.clone();
            tokio::spawn(async move {
                let response = client.get(url).send().await?;
                let text = response.text().await?;
                println!("{} から {} バイトを取得しました", url, text.len());
                Ok::<(), reqwest::Error>(())
            })
        })
        .collect();
    
    for task in tasks {
        task.await??;
    }
    
    Ok(())
}
```

### 3か月目以降: 本番環境への統合

**9〜12週目: 実際のプロジェクトでの開発**
- 非クリティカルなコンポーネントを選定して書き直しを実施
- 包括的なエラー処理の実装
- ロギング、メトリクス、テストの追加
- パフォーマンスプロファイリングと最適化

**継続的取り組み: チームレビューとメンタリング**
- Rust のイディオムに焦点を当てたコードレビュー
- ペアプログラミングセッション
- ナレッジ共有セッション

***
