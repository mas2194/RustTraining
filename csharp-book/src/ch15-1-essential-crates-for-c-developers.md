## C#開発者のための必須クレート

> **ここで学ぶこと:** 一般的な .NET ライブラリに対応する Rust クレート群 — serde（JSON.NET）、reqwest（HttpClient）、tokio（Task / async）、sqlx（Entity Framework）など。さらに、`System.Text.Json` と比較した serde の属性システムの詳細な解説。
>
> **難易度:** 🟡 中級

### 主要機能の対応クレート

```toml
# C# 開発者向けの Cargo.toml 依存関係
[dependencies]
# シリアライゼーション（Newtonsoft.Json や System.Text.Json に相当）
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# HTTP クライアント（HttpClient に相当）
reqwest = { version = "0.11", features = ["json"] }

# 非同期ランタイム（Task.Run や async/await に相当）
tokio = { version = "1.0", features = ["full"] }

# エラー処理（カスタム例外に相当）
thiserror = "1.0"
anyhow = "1.0"

# ロギング（ILogger や Serilog に相当）
log = "0.4"
env_logger = "0.10"

# 日時（DateTime に相当）
chrono = { version = "0.4", features = ["serde"] }

# UUID（System.Guid に相当）
uuid = { version = "1.0", features = ["v4", "serde"] }

# コレクション（List<T> や Dictionary<K,V> に相当）
# 基本的なものは std に組み込み済み。より高度なコレクション用:
indexmap = "2.0"  # 順序付き HashMap

# 設定管理（IConfiguration に相当）
config = "0.13"

# データベース（Entity Framework に相当）
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }

# テスト（xUnit や NUnit に相当）
# 基本的なものは std に組み込み済み。拡張機能用:
rstest = "0.18"  # パラメータ化テスト

# モック化（Moq に相当）
mockall = "0.11"

# 並列処理（Parallel.ForEach に相当）
rayon = "1.7"
```

### 実装パターンの例

```rust
use serde::{Deserialize, Serialize};
use reqwest;
use tokio;
use thiserror::Error;
use chrono::{DateTime, Utc};
use uuid::Uuid;

// データモデル（属性付きの C# POCO に相当）
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct User {
    pub id:标志: Uuid,
    pub name: String,
    pub email: String,
    #[serde(with = "chrono::serde::ts_seconds")]
    pub created_at: DateTime<Utc>,
}

// カスタムエラー型（カスタム例外に相当）
#[derive(Error, Debug)]
pub enum ApiError {
    #[error("HTTP request failed: {0}")]
    Http(#[from] reqwest::Error),
    
    #[error("Serialization failed: {0}")]
    Serialization(#[from] serde_json::Error),
    
    #[error("User not found: {id}")]
    UserNotFound { id: Uuid },
    
    #[error("Validation failed: {message}")]
    Validation { message: String },
}

// サービスクラスに相当
pub struct UserService {
    client: reqwest::Client,
    base_url: String,
}

impl UserService {
    pub fn new(base_url: String) -> Self {
        let client = reqwest::Client::builder()
            .timeout(std::time::Duration::from_secs(30))
            .build()
            .expect("HTTPクライアントの作成に失敗しました");
            
        UserService { client, base_url }
    }
    
    // 非同期メソッド（C# の async Task<User> に相当）
    pub async fn get_user(&self, id: Uuid) -> Result<User, ApiError> {
        let url = format!("{}/users/{}", self.base_url, id);
        
        let response = self.client
            .get(&url)
            .send()
            .await?;
        
        if response.status() == 404 {
            return Err(ApiError::UserNotFound { id });
        }
        
        let user = response.json::<User>().await?;
        Ok(user)
    }
    
    // ユーザー作成（C# の async Task<User> に相当）
    pub async fn create_user(&self, name: String, email: String) -> Result<User, ApiError> {
        if name.trim().is_empty() {
            return Err(ApiError::Validation {
                message: "Name cannot be empty".to_string(),
            });
        }
        
        let new_user = User {
            id: Uuid::new_v4(),
            name,
            email,
            created_at: Utc::now(),
        };
        
        let response = self.client
            .post(&format!("{}/users", self.base_url))
            .json(&new_user)
            .send()
            .await?;
        
        let created_user = response.json::<User>().await?;
        Ok(created_user)
    }
}

// 使用例（C# の Main メソッドに相当）
#[tokio::main]
async fn main() -> Result<(), ApiError> {
    // ロギングの初期化（ILogger の設定に相当）
    env_logger::init();
    
    let service = UserService::new("https://api.example.com".to_string());
    
    // ユーザー作成
    let user = service.create_user(
        "John Doe".to_string(),
        "john@example.com".to_string(),
    ).await?;
    
    println!("ユーザーを作成しました: {:?}", user);
    
    // ユーザー取得
    let retrieved_user = service.get_user(user.id).await?;
    println!("ユーザーを取得しました: {:?}", retrieved_user);
    
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[tokio::test]  // C# の [Test] や [Fact] に相当
    async fn test_user_creation() {
        let service = UserService::new("http://localhost:8080".to_string());
        
        let result = service.create_user(
            "Test User".to_string(),
            "test@example.com".to_string(),
        ).await;
        
        assert!(result.is_ok());
        let user = result.unwrap();
        assert_eq!(user.name, "Test User");
        assert_eq!(user.email, "test@example.com");
    }
    
    #[test]
    fn test_validation() {
        // 同期テスト
        let error = ApiError::Validation {
            message: "Invalid input".to_string(),
        };
        
        assert_eq!(error.to_string(), "Validation failed: Invalid input");
    }
}
```

***


<!-- ch15.1a: Serde Deep Dive for C# Developers -->
## Serde徹底解説: C#開発者のためのJSONシリアライゼーション

C# 開発者は `System.Text.Json` や `Newtonsoft.Json` を頻繁に利用します。Rust では、**serde**（serialize / deserialize）がデファクトスタンダードのフレームワークです。その属性（attribute）システムを理解することで、大半のデータ処理シナリオにスムーズに対応できるようになります。

### 基本的な Derive: まずはここから

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct User {
    name: String,
    age: u32,
    email: String,
}

let user = User { name: "Alice".into(), age: 30, email: "alice@co.com".into() };
let json = serde_json::to_string_pretty(&user)?;
let parsed: User = serde_json::from_str(&json)?;
```

```csharp
// C# の同等コード
public class User
{
    public string Name { get; set; }
    public int Age { get; set; }
    public string Email { get; set; }
}
var json = JsonSerializer.Serialize(user, new JsonSerializerOptions { WriteIndented = true });
var parsed = JsonSerializer.Deserialize<User>(json);
```

### フィールドレベルの属性（`[JsonProperty]` に相当）

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct ApiResponse {
    // JSON 出力時のフィールド名を変更（[JsonPropertyName("user_id")] に相当）
    #[serde(rename = "user_id")]
    id: u64,

    // シリアライズ時とデシリアライズ時で異なる名前を使用
    #[serde(rename(serialize = "userName", deserialize = "user_name"))]
    name: String,

    // このフィールドを完全に除外（[JsonIgnore] に相当）
    #[serde(skip)]
    internal_cache: Option<String>,

    // シリアライズ時のみ除外
    #[serde(skip_serializing)]
    password_hash: String,

    // JSON に存在しない場合のデフォルト値（デフォルトコンストラクタの値に相当）
    #[serde(default)]
    is_active: bool,

    // カスタムデフォルト値
    #[serde(default = "default_role")]
    role: String,

    // ネストした構造体を親構造体にフラットに展開（[JsonExtensionData] に相当）
    #[serde(flatten)]
    metadata: Metadata,

    // 値が None の場合はスキップ（null フィールドを省略）
    #[serde(skip_serializing_if = "Option::is_none")]
    nickname: Option<String>,
}

fn default_role() -> String { "viewer".into() }

#[derive(Serialize, Deserialize, Debug)]
struct Metadata {
    created_at: String,
    version: u32,
}
```

```csharp
// C# の対応する属性
public class ApiResponse
{
    [JsonPropertyName("user_id")]
    public ulong Id { get; set; }

    [JsonIgnore]
    public string? InternalCache { get; set; }

    [JsonExtensionData]
    public Dictionary<string, JsonElement>? Metadata { get; set; }
}
```

### 列挙型（Enum）の表現形式（C#との決定的な違い）

Rust の serde は、列挙型（Enum）に対して**4種類の異なる JSON 表現形式**をサポートしています。C# の enum は常に整数または文字列であるため、これは C# には直接的な同等物が存在しない重要な概念です。

```rust
use serde::{Deserialize, Serialize};

// 1. 外部タグ付き（Externally tagged、デフォルト） — 最も一般的
#[derive(Serialize, Deserialize)]
enum Message {
    Text(String),
    Image { url: String, width: u32 },
    Ping,
}
// Text バリアント:  {"Text": "hello"}
// Image バリアント: {"Image": {"url": "...", "width": 100}}
// Ping バリアント:  "Ping"

// 2. 内部タグ付き（Internally tagged） — 他言語の判別共用体（Discriminated Unions）と同様
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Event {
    Created { id: u64, name: String },
    Deleted { id: u64 },
    Updated { id: u64, fields: Vec<String> },
}
// {"type": "Created", "id": 1, "name": "Alice"}
// {"type": "Deleted", "id": 1}

// 3. 隣接タグ付き（Adjacently tagged） — タグとコンテンツを別々のフィールドに分離
#[derive(Serialize, Deserialize)]
#[serde(tag = "t", content = "c")]
enum ApiResult {
    Success(UserData),
    Error(String),
}
// {"t": "Success", "c": {"name": "Alice"}}
// {"t": "Error", "c": "not found"}

// 4. タグなし（Untagged） — serde が各バリアントを順番に試行
#[derive(Serialize, Deserialize)]
#[serde(untagged)]
enum FlexibleValue {
    Integer(i64),
    Float(f64),
    Text(String),
    Bool(bool),
}
// 42, 3.14, "hello", true — serde がバリアントを自動判別
```

### カスタムシリアライゼーション（`JsonConverter` に相当）

```rust
use serde::{Deserialize, Deserializer, Serialize, Serializer};

// 特定のフィールドに対するカスタムシリアライゼーション
#[derive(Serialize, Deserialize)]
struct Config {
    #[serde(serialize_with = "serialize_duration", deserialize_with = "deserialize_duration")]
    timeout: std::time::Duration,
}

fn serialize_duration<S: Serializer>(dur: &std::time::Duration, s: S) -> Result<S::Ok, S::Error> {
    s.serialize_u64(dur.as_millis() as u64)
}

fn deserialize_duration<'de, D: Deserializer<'de>>(d: D) -> Result<std::time::Duration, D::Error> {
    let ms = u64::deserialize(d)?;
    Ok(std::time::Duration::from_millis(ms))
}
// JSON: {"timeout": 5000}  ↔  Config { timeout: Duration::from_millis(5000) }
```

### コンテナレベルの属性

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]  // JSON 内の全フィールドが camelCase になる
struct UserProfile {
    first_name: String,      // → "firstName"
    last_name: String,       // → "lastName"
    email_address: String,   // → "emailAddress"
}

#[derive(Serialize, Deserialize)]
#[serde(deny_unknown_fields)]  // 未定義の余分なフィールドがある JSON を拒絶する（厳密なパース）
struct StrictConfig {
    port: u16,
    host: String,
}
// serde_json::from_str::<StrictConfig>(r#"{"port":8080,"host":"localhost","extra":true}"#)
// → エラー: 未知のフィールド `extra` が存在します
```

### クイックリファレンス: Serde 属性一覧

| 属性 | レベル | C# の相当機能 | 目的 |
|-----------|-------|---------------|---------|
| `#[serde(rename = "...")]` | フィールド | `[JsonPropertyName]` | JSON 内のプロパティ名変更 |
| `#[serde(skip)]` | フィールド | `[JsonIgnore]` | 完全に対象外とする |
| `#[serde(default)]` | フィールド | デフォルト値 | 存在しない場合に `Default::default()` を使用 |
| `#[serde(flatten)]` | フィールド | `[JsonExtensionData]` | ネストした構造体を親に統合 |
| `#[serde(skip_serializing_if = "...")]` | フィールド | `JsonIgnoreCondition` | 条件付きでスキップ |
| `#[serde(rename_all = "camelCase")]` | コンテナ | `JsonSerializerOptions.PropertyNamingPolicy` | 命名規則の一括適用 |
| `#[serde(deny_unknown_fields)]` | コンテナ | — | 厳格なデシリアライズ（未知のフィールドを禁止） |
| `#[serde(tag = "type")]` | 列挙型 | ディスクリミネータパターン | 内部タグ付け（Internal tagging） |
| `#[serde(untagged)]` | 列挙型 | — | 各バリアントを順番に検証 |
| `#[serde(with = "...")]` | フィールド | `[JsonConverter]` | カスタムシリアライズ / デシリアライズ |

### JSONを超えて: あらゆる形式に対応する Serde

```rust
// クレートを変更するだけで、同じ derive があらゆるフォーマットで動作する
let user = User { name: "Alice".into(), age: 30, email: "a@b.com".into() };

let json  = serde_json::to_string(&user)?;        // JSON
let toml  = toml::to_string(&user)?;               // TOML（設定ファイル向け）
let yaml  = serde_yaml::to_string(&user)?;          // YAML
let cbor  = serde_cbor::to_vec(&user)?;             // CBOR（バイナリ、コンパクト）
let msgpk = rmp_serde::to_vec(&user)?;              // MessagePack（バイナリ）

// 1つの #[derive(Serialize, Deserialize)] で、すべてのフォーマットが追加コストなしで利用可能
```

***
