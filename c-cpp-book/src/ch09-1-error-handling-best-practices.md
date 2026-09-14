# Rust における Option と Result の重要ポイント

> **学べること:** イディオマティックなエラーハンドリングパターン — `unwrap()` に代わる安全な方法、エラー伝播のための `?` 演算子、カスタムエラー型、そして本番コードにおける `anyhow` と `thiserror` の使い分け。

- `Option` と `Result` は、イディオマティックな Rust に不可欠な構成要素です
- **`unwrap()` の安全な代替手段**:
```rust
// Option<T> の安全な代替手段
let value = opt.unwrap_or(default);              // フォールバック値を提供する
let value = opt.unwrap_or_else(|| compute());    // フォールバック用の遅延評価
let value = opt.unwrap_or_default();             // Default トレイトの実装を使用
let value = opt.expect("descriptive message");   // パニックが許容される場合のみ使用

// Result<T, E> の安全な代替手段  
let value = result.unwrap_or(fallback);          // エラーを無視し、フォールバックを使用
let value = result.unwrap_or_else(|e| handle(e)); // エラーを処理し、フォールバックを返す
let value = result.unwrap_or_default();          // Default トレイトを使用
```
- **明示的な制御のためのパターンマッチング**:
```rust
match some_option {
    Some(value) => println!("取得値: {}", value),
    None => println!("値が見つかりません"),
}

match some_result {
    Ok(value) => process(value),
    Err(error) => log_error(error),
}
```
- **エラー伝播のための `?` 演算子の使用**: 早期リターン（ショートサーキット）してエラーを上位に伝播
```rust
fn process_file(path: &str) -> Result<String, std::io::Error> {
    let content = std::fs::read_to_string(path)?; // エラー時は自動的にリターン
    Ok(content.to_uppercase())
}
```
- **変換メソッド**:
    - `map()`: 成功値を変換 `Ok(T)` -> `Ok(U)` または `Some(T)` -> `Some(U)`
    - `map_err()`: エラー型を変換 `Err(E)` -> `Err(F)`
    - `and_then()`: 失敗する可能性のある処理をチェーン
- **独自の API での活用**: 例外やエラーコードよりも `Result<T, E>` を優先
- **参考リンク**: [Option ドキュメント](https://doc.rust-lang.org/std/option/enum.Option.html) | [Result ドキュメント](https://doc.rust-lang.org/std/result/enum.Result.html)

# Rust のよくある落とし穴とデバッグのヒント

- **借用の問題**: 初心者が最もよく遭遇するミス
    - "cannot borrow as mutable" -> 同時に許可される可変参照は 1 つだけ
    - "borrowed value does not live long enough" -> 参照が指しているデータの寿命よりも参照が長く生きている
    - **解決策**: スコープ `{}` を使って参照のライフタイムを制限するか、必要に応じてデータをクローンする
- **トレイト実装の不足**: "method not found" エラー
    - **解決策**: 一般的なトレイトに対して `#[derive(Debug, Clone, PartialEq)]` を追加する
    - `cargo run` よりも `cargo check` を使用して、より分かりやすいエラーメッセージを取得する
- **デバッグモードでの整数オーバーフロー**: Rust はオーバーフロー時にパニックする
    - **解決策**: 明示的な挙動を指定するために `wrapping_add()`、`saturating_add()`、または `checked_add()` を使用する
- **String と &str の混同**: 用途に応じた異なる型
    - `&str` は文字列スライス（借用）用、`String` は所有権を持つ文字列用に使用する
    - **解決策**: `.to_string()` または `String::from()` を使用して `&str` を `String` に変換する
- **借用チェッカーとの戦い**: 借用チェッカーを出し抜こうとしない
    - **解決策**: 所有権のルールに逆らうのではなく、それに従うようにコードを再構築する
    - 複雑な共有シナリオでは、慎重に `Rc<RefCell<T>>` の使用を検討する

## エラーハンドリングの例: 良い例 vs 悪い例

```rust
// [ERROR] 悪い例: 予期せずパニックする可能性がある
fn bad_config_reader() -> String {
    let config = std::env::var("CONFIG_FILE").unwrap(); // 未設定の場合はパニック！
    std::fs::read_to_string(config).unwrap()           // ファイルが存在しない場合はパニック！
}

// [OK] 良い例: エラーを適切に処理する
fn good_config_reader() -> Result<String, ConfigError> {
    let config_path = std::env::var("CONFIG_FILE")
        .unwrap_or_else(|_| "default.conf".to_string()); // デフォルトにフォールバック
    
    let content = std::fs::read_to_string(config_path)
        .map_err(ConfigError::FileRead)?;                // エラーを変換して伝播
    
    Ok(content)
}

// [OK] さらに良い例: 適切なエラー型を定義する
use thiserror::Error;

#[derive(Error, Debug)]
enum ConfigError {
    #[error("Failed to read config file: {0}")]
    FileRead(#[from] std::io::Error),
    
    #[error("Invalid configuration: {message}")]
    Invalid { message: String },
}
```

ここで何が起きているのかを詳しく見てみましょう。`ConfigError` には **2 つのバリアント** しかありません — 1 つは I/O エラー用、もう 1 つはバリデーションエラー用です。これはほとんどのモジュールにとって最適な出発点となります：

| `ConfigError` バリアント | 保持するデータ | 生成元 |
|----------------------|-------|-----------|
| `FileRead(io::Error)` | 元の I/O エラー | `#[from]` により `?` 経由で自動変換 |
| `Invalid { message }` | 人間が読める説明 | バリデーションコード |

これで、`Result<T, ConfigError>` を返す関数を作成できます：

```rust
fn read_config(path: &str) -> Result<String, ConfigError> {
    let content = std::fs::read_to_string(path)?;  // io::Error → ConfigError::FileRead
    if content.is_empty() {
        return Err(ConfigError::Invalid {
            message: "設定ファイルが空です".to_string(),
        });
    }
    Ok(content)
}
```

> **🟢 自習チェックポイント:** 先に進む前に、以下の質問に答えられることを確認してください：
> 1. なぜ `read_to_string` の呼び出しに対する `?` が機能するのでしょうか？（`#[from]` が `impl From<io::Error> for ConfigError` を生成するため）
> 2. 3 つ目のバリアント `MissingKey(String)` を追加した場合、コードのどこを変更する必要がありますか？（バリアントを追加するだけでよく、既存のコードはそのままコンパイルが通ります）

## クレートレベルのエラー型と Result エイリアス

プロジェクトが単一ファイルを超えて成長するにつれて、複数のモジュールレベルのエラーを **クレートレベルのエラー型** にまとめることになります。これは本番環境の Rust における標準的なパターンです。上記の `ConfigError` をベースに構築してみましょう。

実際の Rust プロジェクトでは、すべてのクレート（または重要なモジュール）が独自の `Error` 列挙型と `Result` 型エイリアスを定義します。これはイディオマティックなパターンであり、C++ でライブラリごとに例外階層と `using Result = std::expected<T, Error>;` を定義することに似ています。

### パターン

```rust
// src/error.rs  (または lib.rs の先頭)
use thiserror::Error;

/// このクレートが発生させうるすべてのエラー
#[derive(Error, Debug)]
pub enum Error {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),          // From 経由で自動変換

    #[error("JSON parse error: {0}")]
    Json(#[from] serde_json::Error),     // From 経由で自動変換

    #[error("Invalid sensor id: {0}")]
    InvalidSensor(u32),                  // ドメイン固有のバリアント

    #[error("Timeout after {ms} ms")]
    Timeout { ms: u64 },
}

/// クレート全体の Result エイリアス — クレート内でのタイピング量を削減
pub type Result<T> = core::result::Result<T, Error>;
```

### すべての関数がどのようにシンプルになるか

エイリアスがない場合、次のように書く必要があります：

```rust
// 冗長 — エラー型が至る所で繰り返される
fn read_sensor(id: u32) -> Result<f64, crate::Error> { ... }
fn parse_config(path: &str) -> Result<Config, crate::Error> { ... }
```

エイリアスを使用した場合：

```rust
// クリーン — 単に `Result<T>` と書くだけ
use crate::{Error, Result};

fn read_sensor(id: u32) -> Result<f64> {
    if id > 128 {
        return Err(Error::InvalidSensor(id));
    }
    let raw = std::fs::read_to_string(format!("/dev/sensor/{id}"))?; // io::Error → Error::Io
    let value: f64 = raw.trim().parse()
        .map_err(|_| Error::InvalidSensor(id))?;
    Ok(value)
}
```

`Io` に付けられた `#[from]` 属性によって、次の `impl` が自動生成されます：

```rust
// thiserror の #[from] によって自動生成されるコード
impl From<std::io::Error> for Error {
    fn from(source: std::io::Error) -> Self {
        Error::Io(source)
    }
}
```

これが `?` を機能させる仕組みです。ある関数が `std::io::Error` を返し、自作の関数が `Result<T>`（自作のエイリアス）を返す場合、コンパイラは `From::from()` を呼び出して自動的に変換します。

### モジュールレベルのエラーの合成

より大きなクレートでは、モジュールごとにエラーを分割し、クレートのルートでそれらを合成します：

```rust
// src/config/error.rs
#[derive(thiserror::Error, Debug)]
pub enum ConfigError {
    #[error("Missing key: {0}")]
    MissingKey(String),
    #[error("Invalid value for '{key}': {reason}")]
    InvalidValue { key: String, reason: String },
}

// src/error.rs  (クレートレベル)
#[derive(thiserror::Error, Debug)]
pub enum Error {
    #[error(transparent)]               // Display の出力を内部のエラーに委譲
    Config(#[from] crate::config::ConfigError),

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
}
pub type Result<T> = core::result::Result<T, Error>;
```

呼び出し側では、引き続き特定の設定エラーに対してマッチングを行うことができます：

```rust
match result {
    Err(Error::Config(ConfigError::MissingKey(k))) => eprintln!("設定に '{k}' を追加してください"),
    Err(e) => eprintln!("その他のエラー: {e}"),
    Ok(v) => use_value(v),
}
```

### C++ との比較

| 概念 | C++ | Rust |
|---------|-----|------|
| エラー階層 | `class AppError : public std::runtime_error` | `#[derive(thiserror::Error)] enum Error { ... }` |
| エラーの返却 | `std::expected<T, Error>` または `throw` | `fn foo() -> Result<T>` |
| エラーの変換 | 手動の `try/catch` + 再スロー | `#[from]` + `?` — ボイラープレート不要 |
| Result エイリアス | `template<class T> using Result = std::expected<T, Error>;` | `pub type Result<T> = core::result::Result<T, Error>;` |
| エラーメッセージ | `what()` のオーバーライド | `#[error("...")]` — `Display` 実装にコンパイルされる |
