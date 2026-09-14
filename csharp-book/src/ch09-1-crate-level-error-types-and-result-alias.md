## クレートレベルのエラー型と Result エイリアス

> **学べること:** `thiserror` を用いてクレートごとのエラー列挙型（enum）を定義する実践的なパターン、`Result<T>` 型エイリアスの作成、そして `thiserror`（ライブラリ向け）と `anyhow`（アプリケーション向け）の使い分け。
>
> **難易度:** 🟡 中級

本番環境の Rust における極めて重要なパターン：クレートごとのエラー列挙型（enum）と `Result` 型エイリアスを定義することで、ボイラープレートを排除します。

### このパターン
```rust
// src/error.rs
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("データベースエラー: {0}")]
    Database(#[from] sqlx::Error),

    #[error("HTTP エラー: {0}")]
    Http(#[from] reqwest::Error),

    #[error("シリアライズエラー: {0}")]
    Serialization(#[from] serde_json::Error),

    #[error("バリデーションエラー: {message}")]
    Validation { message: String },

    #[error("見つかりません: {entity} (id: {id})")]
    NotFound { entity: String, id: String },
}

/// クレート全体で共通の Result エイリアス — すべての関数がこれを返します
pub type Result<T> = std::result::Result<T, AppError>;
```

### クレート全体での使用法
```rust
use crate::error::{AppError, Result};

// データベース接続プールが利用可能であることを想定（例）:
// async fn get_user(pool: &PgPool, id: Uuid) -> Result<User>
// ここでは簡略化のため `pool` をそのまま使用するパターンを示しています。
pub async fn get_user(id: Uuid) -> Result<User> {
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
        .fetch_optional(&pool)
        .await?;  // #[from] により sqlx::Error → AppError::Database に自動変換

    user.ok_or_else(|| AppError::NotFound {
        entity: "User".into(),
        id: id.to_string(),
    })
}

pub async fn create_user(req: CreateUserRequest) -> Result<User> {
    if req.name.trim().is_empty() {
        return Err(AppError::Validation {
            message: "名前を空にすることはできません".into(),
        });
    }
    // ...
}
```

### C# との比較
```csharp
// C# における同等のパターン
public class AppException : Exception
{
    public string ErrorCode { get; }
    public AppException(string code, string message) : base(message)
    {
        ErrorCode = code;
    }
}

// しかし C# では、呼び出し側はどんな例外が発生するかコードを見るまで分かりません！
// Rust では、エラー型が関数のシグネチャに明示されます。
```

### なぜこれが重要なのか
- **`thiserror`** は `Display` および `Error` の実装を自動生成します
- **`#[from]`** により、`?` 演算子でライブラリのエラーを自動変換できます
- `Result<T>` エイリアスにより、すべての関数シグネチャを `fn foo() -> Result<Bar>` のように簡潔に保てます
- **C# の例外とは異なり**、呼び出し側は型を通じて発生しうるすべてのエラーバリアントを把握できます


### thiserror vs anyhow：使い分けの指針

Rust のエラーハンドリングでは主に 2 つのクレートが使われます。どちらを選択するかが最初の設計判断となります：

| | `thiserror` | `anyhow` |
|---|---|---|
| **用途** | **ライブラリ**向けに構造化されたエラー型を定義 | **アプリケーション**向けの手軽なエラーハンドリング |
| **出力** | 開発者が制御する独自の列挙型（enum） | 内部が隠蔽された `anyhow::Error` ラッパー |
| **呼び出し側に見える情報** | 型内のすべてのエラーバリアント | 単に `anyhow::Error` のみ（不透明） |
| **最適な適用先** | ライブラリクレート、API、利用者が存在する任意のコード | バイナリ、スクリプト、プロトタイプ、CLI ツール |
| **ダウンキャスト** | バリアントに対する直接の `match` | `error.downcast_ref::<MyError>()` |

```rust
// thiserror — ライブラリ向け（呼び出し側がエラーバリアントを match 分岐する必要がある場合）
use thiserror::Error;

#[derive(Error, Debug)]
pub enum StorageError {
    #[error("ファイルが見つかりません: {path}")]
    NotFound { path: String },

    #[error("アクセスが拒否されました: {0}")]
    PermissionDenied(String),

    #[error(transparent)]
    Io(#[from] std::io::Error),
}

pub fn read_config(path: &str) -> Result<String, StorageError> {
    std::fs::read_to_string(path).map_err(|e| match e.kind() {
        std::io::ErrorKind::NotFound => StorageError::NotFound { path: path.into() },
        std::io::ErrorKind::PermissionDenied => StorageError::PermissionDenied(path.into()),
        _ => StorageError::Io(e),
    })
}
```

```rust
// anyhow — アプリケーション向け（エラー型を定義せず、単に伝播させるだけの場合）
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let config = std::fs::read_to_string("config.toml")
        .context("設定ファイルの読み込みに失敗しました")?;

    let port: u16 = config.parse()
        .context("ポート番号のパースに失敗しました")?;

    println!("ポート {port} でリッスン中");
    Ok(())
}
// anyhow::Result<T> = Result<T, anyhow::Error>
// .context() は任意のエラーに人間が読めるコンテキストを追加します
```

```csharp
// C# との比較:
// thiserror ≈ 特定のプロパティを持つカスタム例外クラスを定義することに相当
// anyhow ≈ Exception をキャッチしてメッセージでラップすることに相当:
//   throw new InvalidOperationException("Failed to read config", ex);
```

**指針**: 作成しているコードが**ライブラリ**（他のコードから呼び出されるもの）である場合は `thiserror` を使用します。**アプリケーション**（最終的なバイナリ）である場合は `anyhow` を使用します。多くのプロジェクトでは、ライブラリクレートの公開 API には `thiserror` を使い、`main()` バイナリでは `anyhow` を使うというように両方を併用しています。

### エラー回復のパターン

C# 開発者は特定の例外から回復するために `try/catch` ブロックをよく利用します。Rust では同様の目的のために `Result` のコンビネータを使用します：

```rust
use std::fs;

// パターン 1: フォールバック値で回復する
let config = fs::read_to_string("config.toml")
    .unwrap_or_else(|_| String::from("port = 8080"));  // ファイルが存在しない場合のデフォルト値

// パターン 2: 特定のエラーから回復し、それ以外は伝播する
fn read_or_create(path: &str) -> Result<String, std::io::Error> {
    match fs::read_to_string(path) {
        Ok(content) => Ok(content),
        Err(e) if e.kind() == std::io::ErrorKind::NotFound => {
            let default = String::from("# new file");
            fs::write(path, &default)?;
            Ok(default)
        }
        Err(e) => Err(e),  // 権限エラーなどは伝播する
    }
}

// パターン 3: 伝播する前にコンテキストを追加する
use anyhow::Context;

fn load_config() -> anyhow::Result<Config> {
    let text = fs::read_to_string("config.toml")
        .context("config.toml の読み込みに失敗しました")?;
    let config: Config = toml::from_str(&text)
        .context("config.toml のパースに失敗しました")?;
    Ok(config)
}

// パターン 4: ドメインのエラー型にマッピングする
fn parse_port(s: &str) -> Result<u16, AppError> {
    s.parse::<u16>()
        .map_err(|_| AppError::Validation {
            message: format!("無効なポート番号: {s}"),
        })
}
```

```csharp
// C# における同等の表現:
try { config = File.ReadAllText("config.toml"); }
catch (FileNotFoundException) { config = "port = 8080"; }  // パターン 1

try { /* ... */ }
catch (FileNotFoundException) { /* ファイルを作成 */ }        // パターン 2
catch { throw; }                                            // それ以外は再スロー
```

**回復するか伝播するかの判断基準:**
- 適切なデフォルト値やリトライ戦略が存在する場合は**回復**する
- 呼び出し側が対処を決定すべきである場合は **`?` で伝播**する
- エラーの追跡履歴を構築するために、モジュールの境界で**コンテキストを追加**（`.context()`）する

---

## 演習

<details>
<summary><strong>🏋️ 演習：クレートエラー型の設計</strong>（クリックして展開）</summary>

ユーザー登録サービスを構築していると仮定します。`thiserror` を使ってエラー型を設計してください：

1. バリアント `DuplicateEmail(String)`, `WeakPassword(String)`, `DatabaseError(#[from] sqlx::Error)`, `RateLimited { retry_after_secs: u64 }` を持つ `RegistrationError` を定義する
2. `type Result<T> = std::result::Result<T, RegistrationError>;` エイリアスを作成する
3. `?` によるエラー伝播と明示的なエラー構築を示す `register_user(email: &str, password: &str) -> Result<()>` を実装する

<details>
<summary>🔑 解答例</summary>

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum RegistrationError {
    #[error("このメールアドレスは既に登録されています: {0}")]
    DuplicateEmail(String),

    #[error("パスワードが脆弱です: {0}")]
    WeakPassword(String),

    #[error("データベースエラー")]
    Database(#[from] sqlx::Error),

    #[error("レート制限中 — {retry_after_secs}秒後に再試行してください")]
    RateLimited { retry_after_secs: u64 },
}

pub type Result<T> = std::result::Result<T, RegistrationError>;

pub fn register_user(email: &str, password: &str) -> Result<()> {
    if password.len() < 8 {
        return Err(RegistrationError::WeakPassword(
            "8文字以上である必要があります".into(),
        ));
    }

    // この ? により sqlx::Error → RegistrationError::Database へ自動変換されます
    // db.check_email_unique(email).await?;

    // これはドメインロジックのための明示的なエラー構築です
    if email.contains("+spam") {
        return Err(RegistrationError::DuplicateEmail(email.to_string()));
    }

    Ok(())
}
```

**重要パターン**: `#[from]` はライブラリのエラーに対して `?` を有効にし、ドメインロジックには明示的な `Err(...)` を使用します。Result エイリアスによってすべての関数シグネチャが簡潔に保たれます。

</details>
</details>

***
