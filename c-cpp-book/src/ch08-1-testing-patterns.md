## C++プログラマのためのテストパターン

> **学習目標:** Rustの組み込みテストフレームワーク（`#[test]`、`#[should_panic]`、`Result` を返すテスト、テストデータ向けのビルダーパターン、トレイトベースのモック、`proptest` によるプロパティベーステスト、`insta` によるスナップショットテスト、統合テストの構成）を学びます。Google Test + CMake を置き換える、設定不要（ゼロコンフィグ）のテスト環境です。

C++のテストは通常、外部フレームワーク（Google Test、Catch2、Boost.Test など）と複雑なビルド統合に依存しています。Rustのテストフレームワークは**言語とツールチェーンに組み込まれて**おり、外部依存関係、CMake統合、テストランナーの設定などは一切不要です。

### `#[test]` 以外のテスト属性

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn basic_pass() {
        assert_eq!(2 + 2, 4);
    }

    // パニックを期待する — GTestの EXPECT_DEATH に相当
    #[test]
    #[should_panic]
    fn out_of_bounds_panics() {
        let v = vec![1, 2, 3];
        let _ = v[10]; // パニックする — テストは合格
    }

    // 特定のメッセージ部分文字列を含むパニックを期待する
    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn specific_panic_message() {
        let v = vec![1, 2, 3];
        let _ = v[10];
    }

    // Result<(), E> を返すテスト — unwrap() の代わりに ? を使用
    #[test]
    fn test_with_result() -> Result<(), String> {
        let value: u32 = "42".parse().map_err(|e| format!("{e}"))?;
        assert_eq!(value, 42);
        Ok(())
    }

    // デフォルトで低速なテストを無視（スキップ）する — cargo test -- --ignored で実行
    #[test]
    #[ignore]
    fn slow_integration_test() {
        std::thread::sleep(std::time::Duration::from_secs(10));
    }
}
```

```bash
cargo test                          # 無視されていないすべてのテストを実行
cargo test -- --ignored             # 無視されたテストのみを実行
cargo test -- --include-ignored     # 無視されたテストを含めすべてのテストを実行
cargo test test_name                # 名前のパターンに一致するテストを実行
cargo test -- --nocapture           # テスト中の println! の出力を表示
cargo test -- --test-threads=1      # テストを直列（逐次）実行（共有状態を扱う場合）
```

### テストヘルパー: テストデータ向けのビルダーパターン

C++では、Google Testのフィクスチャ（`class MyTest : public ::testing::Test`）を使用することが多いでしょう。Rustでは、ビルダー関数や `Default` トレイトを使用します:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // ビルダー関数 — 適切なデフォルト値を持つテストデータを作成
    fn make_gpu_event(severity: Severity, fault_code: u32) -> DiagEvent {
        DiagEvent {
            source: "accel_diag".to_string(),
            severity,
            message: format!("Test event FC:{fault_code}"),
            fault_code,
        }
    }

    // 再利用可能なテストフィクスチャ — 事前に構築されたイベントのセット
    fn sample_events() -> Vec<DiagEvent> {
        vec![
            make_gpu_event(Severity::Critical, 67956),
            make_gpu_event(Severity::Warning, 32709),
            make_gpu_event(Severity::Info, 10001),
        ]
    }

    #[test]
    fn filter_critical_events() {
        let events = sample_events();
        let critical: Vec<_> = events.iter()
            .filter(|e| e.severity == Severity::Critical)
            .collect();
        assert_eq!(critical.len(), 1);
        assert_eq!(critical[0].fault_code, 67956);
    }
}
```

### トレイトによるモック

C++では、モックを作成するために Google Mock などのフレームワークや手動での仮想関数オーバーライドが必要です。Rustでは、依存関係に対してトレイトを定義し、テスト時に実装を差し替えます:

```rust
// 本番トレイト
trait SensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String>;
}

// 本番実装
struct HwSensorReader;
impl SensorReader for HwSensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String> {
        // 実際のハードウェア呼び出し...
        Ok(72.5)
    }
}

// テスト用モック — 予測可能な値を返す
#[cfg(test)]
struct MockSensorReader {
    temperatures: std::collections::HashMap<u32, f64>,
}

#[cfg(test)]
impl SensorReader for MockSensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String> {
        self.temperatures.get(&sensor_id)
            .copied()
            .ok_or_else(|| format!("Unknown sensor {sensor_id}"))
    }
}

// テスト対象の関数 — reader に対してジェネリック
fn check_overtemp(reader: &impl SensorReader, ids: &[u32], threshold: f64) -> Vec<u32> {
    ids.iter()
        .filter(|&&id| reader.read_temperature(id).unwrap_or(0.0) > threshold)
        .copied()
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn detect_overtemp_sensors() {
        let mut mock = MockSensorReader { temperatures: Default::default() };
        mock.temperatures.insert(0, 72.5);
        mock.temperatures.insert(1, 91.0);  // 閾値超過
        mock.temperatures.insert(2, 65.0);

        let hot = check_overtemp(&mock, &[0, 1, 2], 80.0);
        assert_eq!(hot, vec![1]);
    }
}
```

### テストにおける一時ファイルと一時ディレクトリ

C++のテストでは、プラットフォーム固有の一時ディレクトリがよく使用されます。Rustには `tempfile` クレートがあります:

```rust
// Cargo.toml: [dev-dependencies]
// tempfile = "3"

#[cfg(test)]
mod tests {
    use super::*;
    use tempfile::NamedTempFile;
    use std::io::Write;

    #[test]
    fn parse_config_from_file() -> Result<(), Box<dyn std::error::Error>> {
        // ドロップ時に自動削除される一時ファイルを作成
        let mut file = NamedTempFile::new()?;
        writeln!(file, r#"{{"sku": "ServerNode", "level": "Quick"}}"#)?;

        let config = load_config(file.path().to_str().unwrap())?;
        assert_eq!(config.sku, "ServerNode");
        Ok(())
        // file はここで削除される — クリーンアップコードは不要
    }
}
```

### `proptest` によるプロパティベーステスト

個別のテストケースを記述する代わりに、すべての入力に対して成立すべき**プロパティ（性質）**を記述します。`proptest` はランダムな入力を生成し、失敗する最小のケース（反例）を探索します:

```rust
// Cargo.toml: [dev-dependencies]
// proptest = "1"

#[cfg(test)]
mod tests {
    use proptest::prelude::*;

    fn parse_and_format(n: u32) -> String {
        format!("{n}")
    }

    proptest! {
        #[test]
        fn roundtrip_u32(n: u32) {
            // プロパティ: フォーマットした後にパースすると元の値に戻るはず
            let formatted = parse_and_format(n);
            let parsed: u32 = formatted.parse().unwrap();
            prop_assert_eq!(n, parsed);
        }

        #[test]
        fn string_contains_no_null(s in "[a-zA-Z0-9 ]{0,100}") {
            // プロパティ: 文字列にヌル文字が含まれないこと
            prop_assert!(!s.contains('\0'));
        }
    }
}
```

### `insta` によるスナップショットテスト

複雑な出力（JSON、フォーマットされた文字列など）を生成するテストの場合、`insta` は基準となるスナップショットを自動生成して管理します:

```rust
// Cargo.toml: [dev-dependencies]
// insta = { version = "1", features = ["json"] }

#[cfg(test)]
mod tests {
    use insta::assert_json_snapshot;

    #[test]
    fn der_entry_format() {
        let entry = DerEntry {
            fault_code: 67956,
            component: "GPU".to_string(),
            message: "ECC error detected".to_string(),
        };
        // 初回実行時: tests/snapshots/ にスナップショットファイルを作成
        // 2回目以降の実行時: 保存されたスナップショットと比較
        assert_json_snapshot!(entry);
    }
}
```

```bash
cargo insta test              # テストを実行し、新規/変更されたスナップショットを確認
cargo insta review            # スナップショットの変更を対話的にレビュー
```

### C++とRustのテスト比較

| **C++ (Google Test)** | **Rust** | **備考** |
|----------------------|---------|----------|
| `TEST(Suite, Name) { }` | `#[test] fn name() { }` | スイートやクラスの階層は不要 |
| `ASSERT_EQ(a, b)` | `assert_eq!(a, b)` | 組み込みマクロ、フレームワーク不要 |
| `ASSERT_NEAR(a, b, eps)` | `assert!((a - b).abs() < eps)` | または `approx` クレートを使用 |
| `EXPECT_THROW(expr, type)` | `#[should_panic(expected = "...")]` | 詳細な制御には `catch_unwind` も利用可 |
| `EXPECT_DEATH(expr, "msg")` | `#[should_panic(expected = "msg")]` | |
| `class Fixture : public ::testing::Test` | ビルダー関数 + `Default` | 継承は不要 |
| Google Mock `MOCK_METHOD` | トレイト + テスト用実装 | より明示的で、マクロの黒魔術が不要 |
| `INSTANTIATE_TEST_SUITE_P` (パラメータ化) | `proptest!` またはマクロ生成テスト | |
| `SetUp()` / `TearDown()` | `Drop` による RAII — クリーンアップは自動 | テスト終了時に変数がドロップされる |
| 個別のテストバイナリ + CMake | `cargo test` — ゼロコンフィグ | |
| `ctest --output-on-failure` | `cargo test -- --nocapture` | |

----

### 統合テスト: `tests/` ディレクトリ

ユニットテストはコードと同じファイル内の `#[cfg(test)]` モジュールに配置されます。**統合テスト**はクレートルートの独立した `tests/` ディレクトリに配置され、外部の利用者の視点からライブラリの公開APIをテストします:

```
my_crate/
├── src/
│   └── lib.rs          # ライブラリのコード
├── tests/
│   ├── smoke.rs        # 各.rsファイルが個別のテストバイナリになる
│   ├── regression.rs
│   └── common/
│       └── mod.rs      # 共有テストヘルパー（それ自体はテストではない）
└── Cargo.toml
```

```rust
// tests/smoke.rs — 外部ユーザーの視点でクレートをテストする
use my_crate::DiagEngine;  // 公開APIのみアクセス可能

#[test]
fn engine_starts_successfully() {
    let engine = DiagEngine::new("test_config.json");
    assert!(engine.is_ok());
}

#[test]
fn engine_rejects_invalid_config() {
    let engine = DiagEngine::new("nonexistent.json");
    assert!(engine.is_err());
}
```

```rust
// tests/common/mod.rs — 共有ヘルパー（テストバイナリとしてはコンパイルされない）
pub fn setup_test_environment() -> tempfile::TempDir {
    let dir = tempfile::tempdir().unwrap();
    std::fs::write(dir.path().join("config.json"), r#"{"log_level": "debug"}"#).unwrap();
    dir
}
```

```rust
// tests/regression.rs — 共有ヘルパーを使用可能
mod common;

#[test]
fn regression_issue_42() {
    let env = common::setup_test_environment();
    let engine = my_crate::DiagEngine::new(
        env.path().join("config.json").to_str().unwrap()
    );
    assert!(engine.is_ok());
}
```

**統合テストの実行:**
```bash
cargo test                          # ユニットテストと統合テストの両方を実行
cargo test --test smoke             # tests/smoke.rs のみを実行
cargo test --test regression        # tests/regression.rs のみを実行
cargo test --lib                    # ユニットテストのみを実行（統合テストをスキップ）
```

> **ユニットテストとの主な違い**: 統合テストは非公開関数や `pub(crate)` アイテムにアクセスできません。これにより、公開APIだけで十分であるかを検証せざるを得なくなり、API設計上の有益な指針となります。C++の用語で言えば、`friend` アクセスを持たずに公開ヘッダーに対してのみテストを行うようなものです。

----
