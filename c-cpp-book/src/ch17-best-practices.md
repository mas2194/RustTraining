# Rustのベストプラクティスまとめ

> **学習内容:** イディオマティックなRustを書くための実践的なガイドライン — コードの構成、命名規則、エラー処理パターン、ドキュメント作成。頻繁に振り返ることになるクイックリファレンス章です。

## コードの構成
- **関数は小さく保つ**: テストやロジックの把握が容易になります
- **説明的な名前を使用する**: `calc()` ではなく `calculate_total_price()`
- **関連する機能をグループ化する**: モジュールや分割ファイルを使用します
- **ドキュメントを書く**: 公開APIには `///` を使用します

## エラー処理
- **確実に失敗しない場合を除き `unwrap()` を避ける**: パニックしないと100%確信できる場合にのみ使用します
```rust
// 悪い例: パニックする可能性がある
let value = some_option.unwrap();

// 良い例: None の場合を処理する
let value = some_option.unwrap_or(default_value);
let value = some_option.unwrap_or_else(|| expensive_computation());
let value = some_option.unwrap_or_default(); // Default トレイトを使用

// Result<T, E> の場合
let value = some_result.unwrap_or(fallback_value);
let value = some_result.unwrap_or_else(|err| {
    eprintln!("エラーが発生しました: {err}");
    default_value
});
```
- **説明的なメッセージを伴う `expect()` を使用する**: unwrap が正当化される場合、その理由を説明します
```rust
let config = std::env::var("CONFIG_PATH")
    .expect("環境変数 CONFIG_PATH が設定されている必要があります");
```
- **失敗する可能性のある操作には `Result<T, E>` を返す**: エラーの処理方法は呼び出し元に委ねます
- **カスタムエラー型には `thiserror` を使用する**: 手動で実装するよりもエルゴノミクス（使い勝手）が向上します
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error("IOエラー: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("パースエラー: {message}")]
    Parse { message: String },
    
    #[error("値 {value} は範囲外です")]
    OutOfRange { value: i32 },
}
```
- **`?` 演算子でエラーを連鎖させる**: エラーをコールスタックの上位へと伝播させます
- **`anyhow` よりも `thiserror` を選ぶ**: 私たちのチームの規約では、呼び出し元が特定のバリアントに対してマッチできるように、`#[derive(thiserror::Error)]` を使って明示的なエラー enum を定義します。`anyhow::Error` は手早いプロトタイピングには便利ですが、エラー型が消去（型消去）されてしまうため、呼び出し元で特定のエラーに対処することが難しくなります。ライブラリやプロダクションコードには `thiserror` を使用し、`anyhow` は使い捨てのスクリプトや、エラーを出力するだけで済む最上位のバイナリ用に留めておきましょう。
- **`unwrap()` が許容されるケース**:
  - **単体テスト**: `assert_eq!(result.unwrap(), expected)`
  - **プロトタイピング**: 後で置き換える使い捨てコード
  - **確実に失敗しない操作**: 失敗しないことが証明できる場合
```rust
let numbers = vec![1, 2, 3];
let first = numbers.get(0).unwrap(); // 安全: 要素を持つVecを作成した直後であるため

// より良い方法: 理由を説明した expect() を使用する
let first = numbers.get(0).expect("numbers の Vec は構造上空ではありません");
```
- **フェイルファスト（早期失敗）**: 前提条件を早期にチェックし、直ちにエラーを返します

## メモリ管理
- **クローンよりも借用を優先する**: 可能な限りクローンする代わりに `&T` を使用します
- **`Rc<T>` は控えめに使用する**: 所有権の共有が真に必要な場合にのみ使用します
- **ライフタイムを限定する**: スコープ `{}` を使用して値がドロップされるタイミングを制御します
- **公開APIでの `RefCell<T>` を避ける**: 内部可変性はモジュール内部に留めます

## パフォーマンス
- **最適化の前にプロファイリングを行う**: `cargo bench` やプロファイリングツールを使用します
- **ループよりもイテレータを好む**: 可読性が高く、最適化により高速になることが多いです
- **`String` よりも `&str` を使用する**: 所有権を必要としない場合に使用します
- **巨大なスタックオブジェクトには `Box<T>` を検討する**: 必要に応じてヒープに移動します

## 実装すべき重要なトレイト

### すべての型で検討すべき基本トレイト

独自のカスタム型を作成する際は、Rustネイティブな使い心地にするために、以下の基本的なトレイトの実装を検討してください：

#### **Debug と Display**
```rust
use std::fmt;

#[derive(Debug)]  // デバッグ用の自動実装
struct Person {
    name: String,
    age: u32,
}

// ユーザー向け出力のための手動 Display 実装
impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} ({}歳)", self.name, self.age)
    }
}

// 使用例:
let person = Person { name: "Alice".to_string(), age: 30 };
println!("{:?}", person);  // Debug: Person { name: "Alice", age: 30 }
println!("{}", person);    // Display: Alice (30歳)
```

#### **Clone と Copy**
```rust
// Copy: 小さく単純な型のための暗黙的な複製
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

// Clone: 複雑な型のための明示的な複製
#[derive(Debug, Clone)]
struct Person {
    name: String,  // String は Copy を実装していない
    age: u32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = p1;  // Copy（暗黙的）

let person1 = Person { name: "Bob".to_string(), age: 25 };
let person2 = person1.clone();  // Clone（明示的）
```

#### **PartialEq と Eq**
```rust
#[derive(Debug, PartialEq, Eq)]
struct UserId(u64);

#[derive(Debug, PartialEq)]
struct Temperature {
    celsius: f64,  // f64 は Eq を実装していない（NaN のため）
}

let id1 = UserId(123);
let id2 = UserId(123);
assert_eq!(id1, id2);  // PartialEq により機能する

let temp1 = Temperature { celsius: 20.0 };
let temp2 = Temperature { celsius: 20.0 };
assert_eq!(temp1, temp2);  // PartialEq により機能する
```

#### **PartialOrd と Ord**
```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Priority(u8);

let high = Priority(1);
let low = Priority(10);
assert!(high < low);  // 数値が小さいほど = 優先度が高い

// コレクションでの使用
let mut priorities = vec![Priority(5), Priority(1), Priority(8)];
priorities.sort();  // Priority が Ord を実装しているため機能する
```

#### **Default**
```rust
#[derive(Debug, Default)]
struct Config {
    debug: bool,           // false (デフォルト)
    max_connections: u32,  // 0 (デフォルト)
    timeout: Option<u64>,  // None (デフォルト)
}

// 独自の Default 実装
impl Default for Config {
    fn default() -> Self {
        Config {
            debug: false,
            max_connections: 100,  // 独自のデフォルト値
            timeout: Some(30),     // 独自のデフォルト値
        }
    }
}

let config = Config::default();
let config = Config { debug: true, ..Default::default() };  // 部分的なオーバーライド
```

#### **From と Into**
```rust
struct UserId(u64);
struct UserName(String);

// From を実装すれば、Into は自動的に提供される
impl From<u64> for UserId {
    fn from(id: u64) -> Self {
        UserId(id)
    }
}

impl From<String> for UserName {
    fn from(name: String) -> Self {
        UserName(name)
    }
}

impl From<&str> for UserName {
    fn from(name: &str) -> Self {
        UserName(name.to_string())
    }
}

// 使用例:
let user_id: UserId = 123u64.into();         // Into を使用
let user_id = UserId::from(123u64);          // From を使用
let username = UserName::from("alice");      // &str -> UserName
let username: UserName = "bob".into();       // Into を使用
```

#### **TryFrom と TryInto**
```rust
use std::convert::TryFrom;

struct PositiveNumber(u32);

#[derive(Debug)]
struct NegativeNumberError;

impl TryFrom<i32> for PositiveNumber {
    type Error = NegativeNumberError;
    
    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value >= 0 {
            Ok(PositiveNumber(value as u32))
        } else {
            Err(NegativeNumberError)
        }
    }
}

// 使用例:
let positive = PositiveNumber::try_from(42)?;     // Ok(PositiveNumber(42))
let error = PositiveNumber::try_from(-5);         // Err(NegativeNumberError)
```

#### **Serde（シリアライゼーション用）**
```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

// 自動的なJSONシリアライゼーション / デシリアライゼーション
let user = User {
    id: 1,
    name: "Alice".to_string(),
    email: "alice@example.com".to_string(),
};

let json = serde_json::to_string(&user)?;
let deserialized: User = serde_json::from_str(&json)?;
```

### トレイト実装チェックリスト

新しい型を定義する際は、このチェックリストを検討してください：

```rust
#[derive(
    Debug,          // [OK] デバッグ用に常に実装する
    Clone,          // [OK] 複製可能にすべき型の場合
    PartialEq,      // [OK] 比較可能にすべき型の場合
    Eq,             // [OK] 比較が反射律・推移律を満たす場合
    PartialOrd,     // [OK] 順序関係を持つ型の場合
    Ord,            // [OK] 全順序を持つ場合
    Hash,           // [OK] HashMapのキーとして使用する場合
    Default,        // [OK] 適切なデフォルト値が存在する場合
)]
struct MyType {
    // フィールド...
}

// 検討すべき手動実装:
impl Display for MyType { /* ユーザー向けの表現 */ }
impl From<OtherType> for MyType { /* 便利な型変換 */ }
impl TryFrom<FallibleType> for MyType { /* 失敗する可能性のある型変換 */ }
```

### トレイトを実装すべきでない場合

- **ヒープデータを持つ型には Copy を実装しない**: `String`, `Vec`, `HashMap` など
- **値が NaN になり得る場合は Eq を実装しない**: `f32`/`f64` を含む型
- **妥当なデフォルト値がない場合は Default を実装しない**: ファイルハンドル、ネットワーク接続など
- **クローンのコストが高い場合は Clone を実装しない**: 大規模なデータ構造（代わりに `Rc<T>` などを検討）

### まとめ: トレイトのメリット一覧

| トレイト | メリット | 使用すべき場面 |
|-------|---------|-------------|
| `Debug` | `println!("{:?}", value)` による出力 | ほぼ常時（ごく稀な例外を除く） |
| `Display` | `println!("{}", value)` による出力 | ユーザー向けに表示する型 |
| `Clone` | `value.clone()` による複製 | 明示的な複製に意味がある場合 |
| `Copy` | 代入時の暗黙的な複製 | 小さく単純な型 |
| `PartialEq` | `==` および `!=` 演算子 | ほとんどの型 |
| `Eq` | 反射的な同値性 | 同値性が数学的に厳密に成り立つ場合 |
| `PartialOrd` | `<`, `>`, `<=`, `>=` 演算子 | 自然な順序関係を持つ型 |
| `Ord` | `sort()`, `BinaryHeap` での利用 | 全順序が定義できる場合 |
| `Hash` | `HashMap` のキーとして利用 | マップのキーとして使用される型 |
| `Default` | `Default::default()` による生成 | 明確なデフォルト値が存在する型 |
| `From/Into` | 便利な型変換 | 一般的な型同士の変換 |
| `TryFrom/TryInto` | 失敗する可能性のある変換 | エラーが発生し得る型変換 |

----

----
