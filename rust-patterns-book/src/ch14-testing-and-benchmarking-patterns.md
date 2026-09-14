# 14. テストとベンチマークパターン 🟢

> **学習内容:**
> - Rust の3つのテスト層: 単体テスト、結合テスト、ドキュメンテーションテスト
> - エッジケースを発見するための `proptest` によるプロパティベーステスト
> - 信頼性の高い性能測定を行うための `criterion` によるベンチマーク
> - 重量級フレームワークを使わないモック戦略

## 単体テスト、結合テスト、ドキュメンテーションテスト

Rust には言語機能として3つのテスト層が組み込まれています:

```rust
// --- 単体テスト: テスト対象コードと同じファイル内に記述 ---
pub fn factorial(n: u64) -> u64 {
    (1..=n).product()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_factorial_zero() {
        // (1..=0).product() は 1 を返す — 空の範囲に対する乗法の単位元
        assert_eq!(factorial(0), 1);
    }

    #[test]
    fn test_factorial_five() {
        assert_eq!(factorial(5), 120);
    }

    #[test]
    #[cfg(debug_assertions)] // オーバーフローチェックはデバッグモードでのみ有効
    #[should_panic(expected = "overflow")]
    fn test_factorial_overflow() {
        // ⚠️ このテストはデバッグモード（オーバーフローチェック有効時）のみ成功します。
        // リリースモード（`cargo test --release`）では、u64 の算術演算は
        // パニックを起こさず暗黙的にラップアラウンドします。リリースモードでの安全性を
        // 確保するには `checked_mul` を使うか、プロファイルで `overflow-checks = true` を設定してください。
        factorial(100); // オーバーフロー時にパニックすべき
    }

    #[test]
    fn test_with_result() -> Result<(), Box<dyn std::error::Error>> {
        // テスト関数は Result を返すことも可能 — テスト内で ? 演算子が使えます！
        let value: u64 = "42".parse()?;
        assert_eq!(value, 42);
        Ok(())
    }
}
```

```rust
// --- 結合テスト: tests/ ディレクトリ内に配置 ---
// tests/integration_test.rs
// クレートのパブリック（PUBLIC）API のみをテストします

use my_crate::factorial;

#[test]
fn test_factorial_from_outside() {
    assert_eq!(factorial(10), 3_628_800);
}
```

```rust
// --- ドキュメンテーションテスト: ドキュメントコメント内に記述 ---
/// `n` の階乗を計算します。
///
/// # Examples
///
/// ```
/// use my_crate::factorial;
/// assert_eq!(factorial(5), 120);
/// ```
///
/// # Panics
///
/// 結果が `u64` の表現範囲をオーバーフローした場合にパニックします。
///
/// ```should_panic
/// my_crate::factorial(100);
/// ```
pub fn factorial(n: u64) -> u64 {
    (1..=n).product()
}
// ドキュメンテーションテストは `cargo test` で自動的にコンパイル・実行されます
// これによりドキュメント内のコード例が陳腐化するのを防ぎます。
```

### テストフィクスチャとセットアップ

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // 共通のセットアップ処理 — ヘルパー関数を作成
    fn setup_database() -> TestDb {
        let db = TestDb::new_in_memory();
        db.run_migrations();
        db.seed_test_data();
        db
    }

    #[test]
    fn test_user_creation() {
        let db = setup_database();
        let user = db.create_user("Alice", "alice@test.com").unwrap();
        assert_eq!(user.name, "Alice");
    }

    #[test]
    fn test_user_deletion() {
        let db = setup_database();
        db.create_user("Bob", "bob@test.com").unwrap();
        assert!(db.delete_user("Bob").is_ok());
        assert!(db.get_user("Bob").is_none());
    }

    // Drop によるクリーンアップ（RAII パターン）:
    struct TempDir {
        path: std::path::PathBuf,
    }

    impl TempDir {
        fn new() -> Self {
            // Cargo.toml: rand = "0.8"
            let path = std::env::temp_dir().join(format!("test_{}", rand::random::<u32>()));
            std::fs::create_dir_all(&path).unwrap();
            TempDir { path }
        }
    }

    impl Drop for TempDir {
        fn drop(&mut self) {
            let _ = std::fs::remove_dir_all(&self.path);
        }
    }

    #[test]
    fn test_file_operations() {
        let dir = TempDir::new(); // 作成
        std::fs::write(dir.path.join("test.txt"), "hello").unwrap();
        assert!(dir.path.join("test.txt").exists());
    } // dir がここでドロップ → 一時ディレクトリが自動クリーンアップされる
}
```

### プロパティベーステスト (proptest)

特定の値だけをテストする代わりに、常に成立すべき「性質（プロパティ）」をテストします:

```rust
// Cargo.toml: proptest = "1"
use proptest::prelude::*;

fn reverse(v: &[i32]) -> Vec<i32> {
    v.iter().rev().cloned().collect()
}

proptest! {
    #[test]
    fn test_reverse_twice_is_identity(v in prop::collection::vec(any::<i32>(), 0..100)) {
        // プロパティ: 2回反転させると元のベクタに戻る
        assert_eq!(reverse(&reverse(&v)), v);
    }

    #[test]
    fn test_reverse_preserves_length(v in prop::collection::vec(any::<i32>(), 0..100)) {
        assert_eq!(reverse(&v).len(), v.len());
    }

    #[test]
    fn test_sort_is_idempotent(mut v in prop::collection::vec(any::<i32>(), 0..100)) {
        v.sort();
        let sorted_once = v.clone();
        v.sort();
        assert_eq!(v, sorted_once); // 2回のソート結果は1回のソート結果と等しい（冪等性）
    }

    #[test]
    fn test_parse_roundtrip(x in any::<f64>().prop_filter("finite", |x| x.is_finite())) {
        // プロパティ: 文字列化してから再パースすると元の値と一致する
        let s = format!("{x}");
        let parsed: f64 = s.parse().unwrap();
        prop_assert!((x - parsed).abs() < f64::EPSILON);
    }
}
```

> **proptest を使うべき場合**: 入力空間が広く、人間が思いつかないような境界値やエッジケースに対しても確実に動作することを検証したい場合に使用します。proptest は何百ものランダムな入力を生成し、テストが失敗した場合には最小の再現ケースへと絞り込み（shrink）を行います。

### criterion によるベンチマーク

```rust
// Cargo.toml:
// [dev-dependencies]
// criterion = { version = "0.5", features = ["html_reports"] }
//
// [[bench]]
// name = "my_benchmarks"
// harness = false

// benches/my_benchmarks.rs
use criterion::{criterion_group, criterion_main, Criterion, black_box};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 | 1 => n,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn bench_fibonacci(c: &mut Criterion) {
    c.bench_function("fibonacci 20", |b| {
        b.iter(|| fibonacci(black_box(20)))
    });

    // 異なる入力サイズや実装の比較:
    let mut group = c.benchmark_group("fibonacci_compare");
    for size in [10, 15, 20, 25] {
        group.bench_with_input(
            criterion::BenchmarkId::from_parameter(size),
            &size,
            |b, &size| b.iter(|| fibonacci(black_box(size))),
        );
    }
    group.finish();
}

criterion_group!(benches, bench_fibonacci);
criterion_main!(benches);

// 実行: cargo bench
// target/criterion/ に詳細な HTML レポートが生成されます
```

### フレームワークに依存しないモック戦略

Rust のトレイトシステムは自然な依存性の注入（Dependency Injection）を提供するため、重量級のモックフレームワークは不要です:

```rust
// 振る舞いをトレイトとして定義
trait Clock {
    fn now(&self) -> std::time::Instant;
}

trait HttpClient {
    fn get(&self, url: &str) -> Result<String, String>;
}

// 本番環境用の実装
struct RealClock;
impl Clock for RealClock {
    fn now(&self) -> std::time::Instant { std::time::Instant::now() }
}

// サービスは抽象に依存する
struct CacheService<C: Clock, H: HttpClient> {
    clock: C,
    client: H,
    ttl: std::time::Duration,
}

impl<C: Clock, H: HttpClient> CacheService<C, H> {
    fn fetch(&self, url: &str) -> Result<String, String> {
        // self.clock や self.client を利用 — 差し替え（注入）可能
        self.client.get(url)
    }
}

// モック実装を用いたテスト — フレームワークは一切不要！
#[cfg(test)]
mod tests {
    use super::*;

    struct MockClock {
        fixed_time: std::time::Instant,
    }
    impl Clock for MockClock {
        fn now(&self) -> std::time::Instant { self.fixed_time }
    }

    struct MockHttpClient {
        response: String,
    }
    impl HttpClient for MockHttpClient {
        fn get(&self, _url: &str) -> Result<String, String> {
            Ok(self.response.clone())
        }
    }

    #[test]
    fn test_cache_service() {
        let service = CacheService {
            clock: MockClock { fixed_time: std::time::Instant::now() },
            client: MockHttpClient { response: "cached data".into() },
            ttl: std::time::Duration::from_secs(300),
        };

        assert_eq!(service.fetch("http://example.com").unwrap(), "cached data");
    }
}
```

> **テスト設計の哲学**: 結合テストでは本物の依存関係を使い、単体テストではトレイトベースのテストダブル（モック）を活用します。依存関係グラフが極めて複雑でない限り、専用のモックフレームワークは避けましょう。Rust のトレイトジェネリクスで大半のケースに自然に対処できます。

> **テストの重要ポイント**
> - ドキュメンテーションテスト（`///`）はドキュメントと回帰テストを兼ねており、コンパイルされて実行される
> - `proptest` は手動では書けないようなエッジケースを発見するためのランダム入力を生成する
> - `criterion` は統計的に厳密なベンチマークと HTML レポートを提供する
> - モックフレームワークではなく、トレイトジェネリクスとテストダブルを活用してモック化を行う

> **関連情報:** マクロが生成したコードのテストについては [第13章 — マクロ](ch13-macros-code-that-writes-code.md) を、モジュールの構成がテストの構造化に与える影響については [第15章 — クレートアーキテクチャとAPI設計](ch15-crate-architecture-and-api-design.md) を参照してください。

---

### 演習: proptest によるプロパティベーステスト ★★（約25分）

常にソートされた不変条件を維持する `SortedVec<T: Ord>` ラッパーを作成してください。`proptest` を使用して以下を検証してください:
1. 任意の順序・回数で挿入した後でも、内部のベクタが常にソートされていること
2. `contains()` の結果が標準ライブラリの `Vec::contains()` と完全に一致すること
3. ベクタの長さ（要素数）が挿入した回数と等しいこと

<details>
<summary>🔑 解答例</summary>

```rust,ignore
#[derive(Debug)]
struct SortedVec<T: Ord> {
    inner: Vec<T>,
}

impl<T: Ord> SortedVec<T> {
    fn new() -> Self { SortedVec { inner: Vec::new() } }

    fn insert(&mut self, value: T) {
        let pos = self.inner.binary_search(&value).unwrap_or_else(|p| p);
        self.inner.insert(pos, value);
    }

    fn contains(&self, value: &T) -> bool {
        self.inner.binary_search(value).is_ok()
    }

    fn len(&self) -> usize { self.inner.len() }
    fn as_slice(&self) -> &[T] { &self.inner }
}

#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    proptest! {
        #[test]
        fn always_sorted(values in proptest::collection::vec(-1000i32..1000, 0..100)) {
            let mut sv = SortedVec::new();
            for v in &values {
                sv.insert(*v);
            }
            for w in sv.as_slice().windows(2) {
                prop_assert!(w[0] <= w[1]);
            }
            prop_assert_eq!(sv.len(), values.len());
        }

        #[test]
        fn contains_matches_stdlib(values in proptest::collection::vec(0i32..50, 1..30)) {
            let mut sv = SortedVec::new();
            for v in &values {
                sv.insert(*v);
            }
            for v in &values {
                prop_assert!(sv.contains(v));
            }
            prop_assert!(!sv.contains(&9999));
        }
    }
}
```

</details>

***
