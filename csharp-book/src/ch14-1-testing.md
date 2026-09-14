## RustとC#におけるテスト

> **ここで学ぶこと:** 組み込みの `#[test]` と xUnit の比較、`rstest` によるパラメータ化テスト（`[Theory]` に相当）、`proptest` によるプロパティテスト、`mockall` によるモック化、非同期テストのパターン。
>
> **難易度:** 🟡 中級

### 単体テスト

```csharp
// C# — xUnit
using Xunit;

public class CalculatorTests
{
    [Fact]
    public void Add_ReturnsSum()
    {
        var calc = new Calculator();
        Assert.Equal(5, calc.Add(2, 3));
    }

    [Theory]
    [InlineData(1, 2, 3)]
    [InlineData(0, 0, 0)]
    [InlineData(-1, 1, 0)]
    public void Add_Theory(int a, int b, int expected)
    {
        Assert.Equal(expected, new Calculator().Add(a, b));
    }
}
```

```rust
// Rust — 組み込みのテスト機能、外部フレームワークは不要
pub fn add(a: i32, b: i32) -> i32 { a + b }

#[cfg(test)]  // `cargo test` 実行時のみコンパイルされる
mod tests {
    use super::*;  // 親モジュールからインポート

    #[test]
    fn add_returns_sum() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn add_negative_numbers() {
        assert_eq!(add(-1, 1), 0);
    }

    #[test]
    #[should_panic(expected = "overflow")]
    fn add_overflow_panics() {
        let _ = add(i32::MAX, 1); // デバッグモードでパニックする
    }
}
```

### パラメータ化テスト（`[Theory]` に相当）

```rust
// パラメータ化テストには `rstest` クレートを使用
use rstest::rstest;

#[rstest]
#[case(1, 2, 3)]
#[case(0, 0, 0)]
#[case(-1, 1, 0)]
fn test_add(#[case] a: i32, #[case] b: i32, #[case] expected: i32) {
    assert_eq!(add(a, b), expected);
}

// フィクスチャ — テストのセットアップメソッドに相当
#[rstest]
fn test_with_fixture(#[values(1, 2, 3)] x: i32) {
    assert!(x > 0);
}
```

### アサーションの比較

| C# (xUnit) | Rust | 備考 |
|-------------|------|-------|
| `Assert.Equal(expected, actual)` | `assert_eq!(expected, actual)` | 失敗時に差分を出力 |
| `Assert.NotEqual(a, b)` | `assert_ne!(a, b)` | |
| `Assert.True(condition)` | `assert!(condition)` | |
| `Assert.Contains("sub", str)` | `assert!(str.contains("sub"))` | |
| `Assert.Throws<T>(() => ...)` | `#[should_panic]` | または `std::panic::catch_unwind` を使用 |
| `Assert.Null(obj)` | `assert!(option.is_none())` | null は存在しない — `Option` を使用 |

### テストの構成・配置

```text
my_crate/
├── src/
│   ├── lib.rs          # #[cfg(test)] mod tests { } 内の単体テスト
│   └── parser.rs       # 各モジュールが独自のテストモジュールを持てる
├── tests/              # 結合テスト（各ファイルが独立したクレートとなる）
│   ├── parser_test.rs  # 外部の利用者としてパブリックAPIをテストする
│   └── api_test.rs
└── benches/            # ベンチマーク（criterion クレート等を使用）
    └── my_benchmark.rs
```

```rust
// tests/parser_test.rs — 結合テスト
// パブリックAPIにのみアクセス可能（アセンブリ外部からのテストと同様）
use my_crate::parser;

#[test]
fn test_parse_valid_input() {
    let result = parser::parse("valid input");
    assert!(result.is_ok());
}
```

### 非同期テスト

```csharp
// C# — xUnit による非同期テスト
[Fact]
public async Task GetUser_ReturnsUser()
{
    var service = new UserService();
    var user = await service.GetUserAsync(1);
    Assert.Equal("Alice", user.Name);
}
```

```rust
// Rust — tokio による非同期テスト
#[tokio::test]
async fn get_user_returns_user() {
    let service = UserService::new();
    let user = service.get_user(1).await.unwrap();
    assert_eq!(user.name, "Alice");
}
```

### mockall によるモック化

```rust
use mockall::automock;

#[automock]                         // MockUserRepo 構造体を自動生成
trait UserRepo {
    fn find_by_id(&self, id: u32) -> Option<User>;
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn service_returns_user_from_repo() {
        let mut mock = MockUserRepo::new();
        mock.expect_find_by_id()
            .with(mockall::predicate::eq(1))
            .returning(|_| Some(User { name: "Alice".into() }));

        let service = UserService::new(mock);
        let user = service.get_user(1).unwrap();
        assert_eq!(user.name, "Alice");
    }
}
```

```csharp
// C# — Moq の同等コード
var mock = new Mock<IUserRepo>();
mock.Setup(r => r.FindById(1)).Returns(new User { Name = "Alice" });
var service = new UserService(mock.Object);
Assert.Equal("Alice", service.GetUser(1).Name);
```

<details>
<summary><strong>🏋️ 演習: 包括的なテストの作成</strong> (クリックして展開)</summary>

**課題**: 以下の関数に対して、正常系、空の入力、数値文字列、Unicode を網羅するテストを記述してください。

```rust
pub fn title_case(input: &str) -> String {
    input.split_whitespace()
        .map(|word| {
            let mut chars = word.chars();
            match chars.next() {
                Some(c) => format!("{}{}", c.to_uppercase(), chars.as_str().to_lowercase()),
                None => String::new(),
            }
        })
        .collect::<Vec<_>>()
        .join(" ")
}
```

<details>
<summary>🔑 解答例</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn happy_path() {
        assert_eq!(title_case("hello world"), "Hello World");
    }

    #[test]
    fn empty_input() {
        assert_eq!(title_case(""), "");
    }

    #[test]
    fn single_word() {
        assert_eq!(title_case("rust"), "Rust");
    }

    #[test]
    fn already_title_case() {
        assert_eq!(title_case("Hello World"), "Hello World");
    }

    #[test]
    fn all_caps() {
        assert_eq!(title_case("HELLO WORLD"), "Hello World");
    }

    #[test]
    fn extra_whitespace() {
        // split_whitespace は連続する空白も適切に処理する
        assert_eq!(title_case("  hello   world  "), "Hello World");
    }

    #[test]
    fn unicode() {
        assert_eq!(title_case("café résumé"), "Café Résumé");
    }

    #[test]
    fn numeric_words() {
        assert_eq!(title_case("hello 42 world"), "Hello 42 World");
    }
}
```

**重要なポイント**: Rust の組み込みテストフレームワークは、ほとんどの単体テストのニーズをカバーしています。パラメータ化テストには `rstest`、モック化には `mockall` を使用すれば十分であり、xUnit のような大規模なテストフレームワークを導入する必要はありません。

</details>
</details>


<!-- ch14a.1: Property Testing with proptest -->
## プロパティテスト: 大規模な正当性の検証

**FsCheck** に親しんでいる C# 開発者なら、プロパティベーステスト（Property-based testing）の考え方に馴染みがあるでしょう。個々のテストケースを手作業で書く代わりに、**あらゆる可能な入力**に対して成立しなければならない「性質（プロパティ）」を記述し、フレームワークが数千ものランダムな入力を生成してその性質を破ろうと試みます。

### なぜプロパティテストが重要なのか

```csharp
// C# — 手書きの単体テストは特定のケースを検証する
[Fact]
public void Reverse_Twice_Returns_Original()
{
    var list = new List<int> { 1, 2, 3 };
    list.Reverse();
    list.Reverse();
    Assert.Equal(new[] { 1, 2, 3 }, list);
}
// しかし、空リストや単一要素、10,000要素、負の数などはどうでしょうか？
// 手作業で何十ものケースを書く必要があります。
```

```rust
// Rust — proptest が何千もの入力を自動生成する
use proptest::prelude::*;

fn reverse<T: Clone>(v: &[T]) -> Vec<T> {
    v.iter().rev().cloned().collect()
}

proptest! {
    #[test]
    fn reverse_twice_is_identity(ref v in prop::collection::vec(any::<i32>(), 0..1000)) {
        let reversed_twice = reverse(&reverse(v));
        prop_assert_eq!(v, &reversed_twice);
    }
    // proptest は数百・数千のランダムな Vec<i32> の値でこれを実行します:
    // []、[0]、[i32::MIN, i32::MAX]、[42; 999]、ランダムなシーケンスなど...
    // 失敗した場合、失敗を引き起こす最小の入力へと自動的に「縮約（shrink）」します！
}
```

### proptest の導入

```toml
# Cargo.toml
[dev-dependencies]
proptest = "1.4"
```

### C#開発者向けの一般的なパターン

```rust
use proptest::prelude::*;

// 1. ラウンドトリップ（往復）プロパティ: シリアライズ → デシリアライズ = 同一
// (JsonSerializer.Serialize → Deserialize のテストと同様)
proptest! {
    #[test]
    fn json_roundtrip(name in "[a-zA-Z]{1,50}", age in 0u32..150) {
        let user = User { name: name.clone(), age };
        let json = serde_json::to_string(&user).unwrap();
        let parsed: User = serde_json::from_str(&json).unwrap();
        prop_assert_eq!(user, parsed);
    }
}

// 2. 不変条件プロパティ: 出力が常に特定の条件を満たす
proptest! {
    #[test]
    fn sort_output_is_sorted(ref v in prop::collection::vec(any::<i32>(), 0..500)) {
        let mut sorted = v.clone();
        sorted.sort();
        // 隣接するすべてのペアが整列順になっている必要がある
        for window in sorted.windows(2) {
            prop_assert!(window[0] <= window[1]);
        }
    }
}

// 3. オラクルプロパティ: 2つの実装を比較する
proptest! {
    #[test]
    fn fast_path_matches_slow_path(input in "[0-9a-f]{1,100}") {
        let result_fast = parse_hex_fast(&input);
        let result_slow = parse_hex_slow(&input);
        prop_assert_eq!(result_fast, result_slow);
    }
}

// 4. カスタムストラテジー: ドメイン固有のテストデータを生成する
fn valid_email() -> impl Strategy<Value = String> {
    ("[a-z]{1,20}", "[a-z]{1,10}", prop::sample::select(vec!["com", "org", "io"]))
        .prop_map(|(user, domain, tld)| format!("{}@{}.{}", user, domain, tld))
}

proptest! {
    #[test]
    fn email_parsing_accepts_valid_emails(email in valid_email()) {
        let result = Email::new(&email);
        prop_assert!(result.is_ok(), "パースに失敗しました: {}", email);
    }
}
```

### proptest と FsCheck の比較

| 機能 | C# FsCheck | Rust proptest |
|---------|-----------|---------------|
| ランダム入力生成 | `Arb.Generate<T>()` | `any::<T>()` |
| カスタムジェネレータ | `Arb.Register<T>()` | `impl Strategy<Value = T>` |
| 失敗時の自動縮約（Shrinking） | 自動 | 自動 |
| 文字列パターン | 手動 | `"[regex]"` ストラテジー |
| コレクション生成 | `Gen.ListOf` | `prop::collection::vec(strategy, range)` |
| ジェネレータの合成 | `Gen.Select` | `.prop_map()`, `.prop_flat_map()` |
| 設定（テスト実行数） | `Config.MaxTest` | `proptest!` ブロック内で `#![proptest_config(ProptestConfig::with_cases(10000))]` |

### プロパティテストと単体テストの使い分け

| **単体テスト** を使うべき場合 | **proptest** を使うべき場合 |
|------------------------|----------------------|
| 特定のエッジケースのテスト | すべての入力にわたる不変条件の検証 |
| エラーメッセージやエラーコードのテスト | ラウンドトリッププロパティ（パース ↔ フォーマット） |
| 結合テストやモックを用いたテスト | 2つの実装の振る舞い比較 |
| 振る舞いが厳密な値に依存する場合 | 「すべてのXに対して、性質Pが成り立つ」の検証 |

---

## 結合テスト: `tests/` ディレクトリ

単体テストは `src/` 内に `#[cfg(test)]` と共に配置されます。一方、結合テストは独立した `tests/` ディレクトリに配置し、クレートの**パブリックAPI**をテストします。これは、C# の結合テストが対象プロジェクトを外部アセンブリとして参照する構成と同じです。

```
my_crate/
├── src/
│   ├── lib.rs          // パブリックAPI
│   └── internal.rs     // プライベートな実装
├── tests/
│   ├── smoke.rs        // 各ファイルが独立したテストバイナリとなる
│   ├── api_tests.rs
│   └── common/
│       └── mod.rs      // 共有テストヘルパー
└── Cargo.toml
```

### 結合テストの記述

`tests/` 内の各ファイルは、自作のライブラリに依存する独立したクレートとしてコンパイルされます。

```rust
// tests/smoke.rs — my_crate の pub アイテムにのみアクセス可能
use my_crate::{process_order, Order, OrderResult};

#[test]
fn process_valid_order_returns_confirmation() {
    let order = Order::new("SKU-001", 3);
    let result = process_order(order);
    assert!(matches!(result, OrderResult::Confirmed { .. }));
}
```

### 共通テストヘルパー

共通のセットアップコードは `tests/common/mod.rs` に配置します（`tests/common.rs` と名付けると、それ自体が独立したテストファイルとして扱われてしまうため注意してください）。

```rust
// tests/common/mod.rs
use my_crate::Config;

pub fn test_config() -> Config {
    Config::builder()
        .database_url("sqlite::memory:")
        .build()
        .expect("テスト用の設定は有効でなければなりません")
}
```

```rust
// tests/api_tests.rs
mod common;

use my_crate::App;

#[test]
fn app_starts_with_test_config() {
    let config = common::test_config();
    let app = App::new(config);
    assert!(app.is_healthy());
}
```

### 特定のテスト種別の実行

```bash
cargo test                  # すべてのテストを実行（単体 + 結合）
cargo test --lib            # 単体テストのみ実行（dotnet test --filter Category=Unit に相当）
cargo test --test smoke     # tests/smoke.rs のみ実行
cargo test --test api_tests # tests/api_tests.rs のみ実行
```

**C# との主な違い:** 結合テストファイルからは、クレートの `pub` な API にしかアクセスできません。プライベートな関数は一切見えないため、強制的にパブリックインターフェース経由でテストすることになり、結果としてより健全なテスト設計が促されます。

***
