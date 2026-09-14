## C#開発者のためのベストプラクティス

> **ここで学ぶこと:** 5つの重要なマインドセットの転換（GC→所有権、例外→Result、継承→コンポジションなど）、イディオマティックなプロジェクト構成、エラー処理戦略、テストパターン、そして C# 開発者が Rust で陥りがちな最も一般的なミスとその対策。
>
> **難易度:** 🟡 中級

### 1. **マインドセットの転換**
- **GC から所有権へ**: データは誰が所有しており、いつ解放されるのかを常に意識する
- **例外から Result へ**: エラー処理を明示的かつ型として可視化する
- **継承からコンポジションへ**: トレイトを活用して振る舞いを合成する
- **Null から Option へ**: 値の不在を型システム上で明示する

### 2. **コードの構成・プロジェクト構造**

```rust
// C# ソリューションのようにプロジェクトを構成する
src/
├── main.rs          // Program.cs に相当
├── lib.rs           // ライブラリのエントリポイント
├── models/          // C# の Models/ フォルダに相当
│   ├── mod.rs
│   ├── user.rs
│   └── product.rs
├── services/        // Services/ フォルダに相当
│   ├── mod.rs
│   ├── user_service.rs
│   └── product_service.rs
├── controllers/     // Controllers/ フォルダに相当（Web アプリ用）
├── repositories/    // Repositories/ フォルダに相当
└── utils/          // Utilities/ フォルダに相当
```

### 3. **エラー処理戦略**

```rust
// アプリケーション共通の Result 型を定義
pub type AppResult<T> = Result<T, AppError>;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),
    
    #[error("Validation error: {message}")]
    Validation { message: String },
    
    #[error("Business logic error: {message}")]
    Business { message: String },
}

// アプリケーション全体で使用
pub async fn create_user(data: CreateUserRequest) -> AppResult<User> {
    validate_user_data(&data)?;  // AppError::Validation を返す
    let user = repository.create_user(data).await?;  // AppError::Database を返す
    Ok(user)
}
```

### 4. **テストパターン**

```rust
// C# の単体テストのようにテストを構成
#[cfg(test)]
mod tests {
    use super::*;
    use rstest::*;  // C# の [Theory] のようなパラメータ化テスト用
    
    #[test]
    fn test_basic_functionality() {
        // Arrange (準備)
        let input = "test data";
        
        // Act (実行)
        let result = process_data(input);
        
        // Assert (検証)
        assert_eq!(result, "expected output");
    }
    
    #[rstest]
    #[case(1, 2, 3)]
    #[case(5, 5, 10)]
    #[case(0, 0, 0)]
    fn test_addition(#[case] a: i32, #[case] b: i32, #[case] expected: i32) {
        assert_eq!(add(a, b), expected);
    }
    
    #[tokio::test]  // 非同期テスト用
    async fn test_async_functionality() {
        let result = async_function().await;
        assert!(result.is_ok());
    }
}
```

### 5. **避けるべきよくあるミス**

```rust
// 【誤り】継承を実装しようとしない
// 以下のように書くのではなく:
// struct Manager : Employee  // Rust にこのような構文は存在しない

// 【正解】トレイトによる合成（コンポジション）を使用する
trait Employee {
    fn get_salary(&self) -> u32;
}

trait Manager: Employee {
    fn get_team_size(&self) -> usize;
}

// 【誤り】至る所で unwrap() を使わない（例外をもみ消すようなもの）
let value = might_fail().unwrap();  // パニックする可能性がある！

// 【正解】エラーを適切に処理する
let value = match might_fail() {
    Ok(v) => v,
    Err(e) => {
        log::error!("Operation failed: {}", e);
        return Err(e.into());
    }
};

// 【誤り】何でも clone() しない（不要にオブジェクトをコピーするようなもの）
let data = expensive_data.clone();  // コストが高い！

// 【正解】可能な限り借用を使用する
let data = &expensive_data;  // 単なる参照

// 【誤り】至る所で RefCell を使わない（何でも可変にするようなもの）
struct Data {
    value: RefCell<i32>,  // 内部可変性 - 慎重に利用すること
}

// 【正解】所有データまたは借用データを優先する
struct Data {
    value: i32,  // シンプルで明確
}
```

本ガイドは、C# 開発者がこれまでに培った知識が Rust にどのようにマッピングされるのかを包括的に理解できるようにし、類似点と根本的なアプローチの違いの両方を浮き彫りにします。鍵となるのは、Rust の制約（所有権など）が、初期の学習コストと引き換えに、C# では発生し得るバグの分類そのものを根本から排除するように設計されていることを理解することです。

---

### 6. **過剰な `clone()` の回避** 🟡

C# 開発者は、GC がコストを処理してくれるため無意識にデータを複製（clone）しがちです。しかし Rust では、`.clone()` を呼ぶたびに明示的なアロケーションが発生します。その多くは借用を用いることで排除できます。

```rust
// 【誤り】C# の癖: 文字列を受け渡すたびにクローンする
fn greet(name: String) {
    println!("Hello, {name}");
}

let user_name = String::from("Alice");
greet(user_name.clone());  // 不要なメモリ割り当て
greet(user_name.clone());  // 再び不要な割り当て

// 【正解】代わりに借用する — アロケーションゼロ
fn greet(name: &str) {
    println!("Hello, {name}");
}

let user_name = String::from("Alice");
greet(&user_name);  // 借用
greet(&user_name);  // 再び借用 — 追加コストなし
```

**`clone()` が適切なケース:**
- スレッド間や `'static` クロージャにデータを移動する場合（`Arc::clone` は軽量であり、単に参照カウンタをインクリメントするだけです）
- キャッシュ処理: 純粋に独立したコピーが必要な場合
- プロトタイピング: まずは動作させ、後から不要な clone を削る場合

**判断チェックリスト:**
1. 代わりに `&T` や `&str` を渡せないか？ → 可能ならそうする
2. 呼び出し先で所有権が必要か？ → clone ではなくムーブ（move）で渡す
3. 複数スレッド間で共有されているか？ → `Arc<T>` を使用する（clone は参照カウントの増加のみ）
4. 上記のいずれにも当てはまらないか？ → その場合に初めて `clone()` が正当化される

---

### 7. **本番コードでの `unwrap()` の回避** 🟡

C# で例外を無視してコードを書くのと、Rust で至る所に `.unwrap()` を書くのは同等に危険です。

```rust
// 【誤り】「あとで直す」の罠
let config = std::fs::read_to_string("config.toml").unwrap();
let port: u16 = config_value.parse().unwrap();
let conn = db_pool.get().await.unwrap();

// 【正解】アプリケーションコードでは ? で伝播する
let config = std::fs::read_to_string("config.toml")?;
let port: u16 = config_value.parse()?;
let conn = db_pool.get().await?;

// 【正解】失敗が真にバグである場合にのみ expect() を使用する
let home = std::env::var("HOME")
    .expect("HOME 環境変数が設定されていなければなりません");  // 不変条件を文書化する
```

**使い分けの目安:**
| メソッド | 使用すべき場面 |
|--------|------------|
| `?` | アプリケーション / ライブラリコード — 呼び出し元へエラーを伝播する |
| `expect("reason")` | 起動時のアサーション、絶対に成立していなければならない不変条件 |
| `unwrap()` | テストコード内、または事前に `is_some()` / `is_ok()` で確認済みの場合のみ |
| `unwrap_or(default)` | 妥当なフォールバック（デフォルト値）がある場合 |
| `unwrap_or_else(|| ...)` | フォールバック値の計算コストが高い場合 |

---

### 8. **借用チェッカーとの格闘（およびその解消法）** 🟡

すべての C# 開発者は、一見正しそうに見えるコードが借用チェッカーによって拒絶されるフェーズを経験します。その解決策は、回避策を探すことではなく、構造を見直すことです。

```rust
// 【誤り】反復処理中に変更しようとする（C# の foreach 中の変更パターン）
let mut items = vec![1, 2, 3, 4, 5];
for item in &items {
    if *item > 3 {
        items.push(*item * 2);  // エラー: items を可変として借用できない
    }
}

// 【正解】先に収集してから変更する
let extras: Vec<i32> = items.iter()
    .filter(|&&x| x > 3)
    .map(|&x| x * 2)
    .collect();
items.extend(extras);
```

```rust
// 【誤り】ローカル変数への参照を返す（C# では GC により参照を自由に戻せる）
fn get_greeting() -> &str {
    let s = String::from("hello");
    &s  // エラー: s は関数の終了時に破棄（ドロップ）される
}

// 【正解】所有されたデータを返す
fn get_greeting() -> String {
    String::from("hello")  // 呼び出し元が所有権を持つ
}
```

**借用チェッカーの衝突を解消する一般的なパターン:**

| C# の習慣 | Rust での解決策 |
|----------|--------------|
| 構造体に参照を保持する | 所有データを保持するか、ライフタイムパラメータを付与する |
| 共有状態を自由に変更する | `Arc<Mutex<T>>` を使用するか、共有を避ける設計に再構成する |
| ローカル変数への参照を返す | 所有された値（Owned value）を返す |
| 反復処理中にコレクションを変更する | 変更内容を別途収集してから一括適用する |
| 複数の可変参照を同時に要求する | 構造体を互いに独立したフィールド群に分割する |

---

### 9. **ネストした判定ピラミッドの解消** 🟢

C# 開発者は `if (x != null) { if (x.Value > 0) { ... } }` のようなネストした条件分岐を書きがちです。Rust の `match`、`if let`、そして `?` はこれらを平坦化します。

```rust
// 【誤り】C# 流のネストした null チェック
fn process(input: Option<String>) -> Option<usize> {
    match input {
        Some(s) => {
            if !s.is_empty() {
                match s.parse::<usize>() {
                    Ok(n) => {
                        if n > 0 {
                            Some(n * 2)
                        } else {
                            None
                        }
                    }
                    Err(_) => None,
                }
            } else {
                None
            }
        }
        None => None,
    }
}

// 【正解】コンビネータで平坦化する
fn process(input: Option<String>) -> Option<usize> {
    input
        .filter(|s| !s.is_empty())
        .and_then(|s| s.parse::<usize>().ok())
        .filter(|&n| n > 0)
        .map(|n| n * 2)
}
```

**すべての C# 開発者が知っておくべき主要なコンビネータ:**

| コンビネータ | 機能 | C# の相当機能 |
|-----------|-------------|---------------|
| `map` | 内部の値を変換する | `Select` / null 条件演算子 `?.` |
| `and_then` | Option/Result を返す操作を連鎖させる | `SelectMany` / `?.Method()` |
| `filter` | 述語（条件）を満たす場合のみ値を保持する | `Where` |
| `unwrap_or` | デフォルト値を提供する | `?? defaultValue` |
| `ok()` | `Result` を `Option` に変換する（エラーは破棄） | — |
| `transpose` | `Option<Result>` と `Result<Option>` を相互変換する | — |
