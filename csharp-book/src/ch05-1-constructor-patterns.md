## コンストラクタパターン

> **学習内容:** 従来のコンストラクタを持たない Rust で構造体をインスタンス化する方法 — `new()` の慣例、`Default` トレイト、ファクトリメソッド、そして複雑な初期化のための Builder パターンについて学びます。
>
> **難易度:** 🟢 初級

### C# のコンストラクタパターン
```csharp
public class Configuration
{
    public string DatabaseUrl { get; set; }
    public int MaxConnections { get; set; }
    public bool EnableLogging { get; set; }
    
    // デフォルトコンストラクタ
    public Configuration()
    {
        DatabaseUrl = "localhost";
        MaxConnections = 10;
        EnableLogging = false;
    }
    
    // パラメータ付きコンストラクタ
    public Configuration(string databaseUrl, int maxConnections)
    {
        DatabaseUrl = databaseUrl;
        MaxConnections = maxConnections;
        EnableLogging = false;
    }
    
    // ファクトリメソッド
    public static Configuration ForProduction()
    {
        return new Configuration("prod.db.server", 100)
        {
            EnableLogging = true
        };
    }
}
```

### Rust のコンストラクタパターン
```rust
#[derive(Debug)]
pub struct Configuration {
    pub database_url: String,
    pub max_connections: u32,
    pub enable_logging: bool,
}

impl Configuration {
    // デフォルトコンストラクタ（慣例的な new 関数）
    pub fn new() -> Configuration {
        Configuration {
            database_url: "localhost".to_string(),
            max_connections: 10,
            enable_logging: false,
        }
    }
    
    // パラメータ付きコンストラクタ
    pub fn with_database(database_url: String, max_connections: u32) -> Configuration {
        Configuration {
            database_url,
            max_connections,
            enable_logging: false,
        }
    }
    
    // ファクトリメソッド
    pub fn for_production() -> Configuration {
        Configuration {
            database_url: "prod.db.server".to_string(),
            max_connections: 100,
            enable_logging: true,
        }
    }
    
    // Builder パターンのメソッド
    pub fn enable_logging(mut self) -> Configuration {
        self.enable_logging = true;
        self  // メソッドチェーンのために self を返す
    }
    
    pub fn max_connections(mut self, count: u32) -> Configuration {
        self.max_connections = count;
        self
    }
}

// Default トレイトの実装
impl Default for Configuration {
    fn default() -> Self {
        Self::new()
    }
}

fn main() {
    // さまざまな構築パターン
    let config1 = Configuration::new();
    let config2 = Configuration::with_database("localhost:5432".to_string(), 20);
    let config3 = Configuration::for_production();
    
    // Builder パターン
    let config4 = Configuration::new()
        .enable_logging()
        .max_connections(50);
    
    // Default トレイトの使用
    let config5 = Configuration::default();
    
    println!("{:?}", config4);
}
```

### Builder パターンの実装
```rust
// より複雑な Builder パターン
#[derive(Debug)]
pub struct DatabaseConfig {
    host: String,
    port: u16,
    username: String,
    password: Option<String>,
    ssl_enabled: bool,
    timeout_seconds: u64,
}

pub struct DatabaseConfigBuilder {
    host: Option<String>,
    port: Option<u16>,
    username: Option<String>,
    password: Option<String>,
    ssl_enabled: bool,
    timeout_seconds: u64,
}

impl DatabaseConfigBuilder {
    pub fn new() -> Self {
        DatabaseConfigBuilder {
            host: None,
            port: None,
            username: None,
            password: None,
            ssl_enabled: false,
            timeout_seconds: 30,
        }
    }
    
    pub fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }
    
    pub fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }
    
    pub fn username(mut self, username: impl Into<String>) -> Self {
        self.username = Some(username.into());
        self
    }
    
    pub fn password(mut self, password: impl Into<String>) -> Self {
        self.password = Some(password.into());
        self
    }
    
    pub fn enable_ssl(mut self) -> Self {
        self.ssl_enabled = true;
        self
    }
    
    pub fn timeout(mut self, seconds: u64) -> Self {
        self.timeout_seconds = seconds;
        self
    }
    
    pub fn build(self) -> Result<DatabaseConfig, String> {
        let host = self.host.ok_or("ホスト名は必須です")?;
        let port = self.port.ok_or("ポート番号は必須です")?;
        let username = self.username.ok_or("ユーザー名は必須です")?;
        
        Ok(DatabaseConfig {
            host,
            port,
            username,
            password: self.password,
            ssl_enabled: self.ssl_enabled,
            timeout_seconds: self.timeout_seconds,
        })
    }
}

fn main() {
    let config = DatabaseConfigBuilder::new()
        .host("localhost")
        .port(5432)
        .username("admin")
        .password("secret123")
        .enable_ssl()
        .timeout(60)
        .build()
        .expect("設定の構築に失敗しました");
    
    println!("{:?}", config);
}
```

---

## 演習問題

<details>
<summary><strong>🏋️ 演習: バリデーション付き Builder</strong> (クリックして展開)</summary>

以下の要件を満たす `EmailBuilder` を作成してください:
1. `to` と `subject` を必須とする（これらがないとビルドできないようにする — 型状態（タイプステート）パターンを使用するか、`build()` 内で検証する）
2. 任意の `body` と `cc`（アドレスの Vec）を持つ
3. `build()` は `Result<Email, String>` を返す — 空の `to` や `subject` は拒絶する
4. 不正な入力が拒絶されることを証明するテストを作成する

<details>
<summary>🔑 解答</summary>

```rust
#[derive(Debug)]
struct Email {
    to: String,
    subject: String,
    body: Option<String>,
    cc: Vec<String>,
}

#[derive(Default)]
struct EmailBuilder {
    to: Option<String>,
    subject: Option<String>,
    body: Option<String>,
    cc: Vec<String>,
}

impl EmailBuilder {
    fn new() -> Self { Self::default() }

    fn to(mut self, to: impl Into<String>) -> Self {
        self.to = Some(to.into()); self
    }
    fn subject(mut self, subject: impl Into<String>) -> Self {
        self.subject = Some(subject.into()); self
    }
    fn body(mut self, body: impl Into<String>) -> Self {
        self.body = Some(body.into()); self
    }
    fn cc(mut self, addr: impl Into<String>) -> Self {
        self.cc.push(addr.into()); self
    }
    fn build(self) -> Result<Email, String> {
        let to = self.to.filter(|s| !s.is_empty())
            .ok_or("'to' は必須です")?;
        let subject = self.subject.filter(|s| !s.is_empty())
            .ok_or("'subject' は必須です")?;
        Ok(Email { to, subject, body: self.body, cc: self.cc })
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn valid_email() {
        let email = EmailBuilder::new()
            .to("alice@example.com")
            .subject("Hello")
            .build();
        assert!(email.is_ok());
    }
    #[test]
    fn missing_to_fails() {
        let email = EmailBuilder::new().subject("Hello").build();
        assert!(email.is_err());
    }
}
```

</details>
</details>

***
