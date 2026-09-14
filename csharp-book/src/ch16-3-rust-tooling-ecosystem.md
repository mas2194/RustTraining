## C#開発者のための必須 Rust ツール環境

> **学習内容:** C# のツールに対応付けた Rust の開発ツール — Clippy（Roslyn アナライザー）、rustfmt（dotnet format）、cargo doc（XML ドキュメント）、cargo watch（dotnet watch）、および VS Code 拡張機能。
>
> **難易度:** 🟢 初級

### ツールの比較

| C# のツール | Rust の同等ツール | インストール方法 | 目的 |
|---------|----------------|---------|---------|
| Roslyn アナライザー | **Clippy** | `rustup component add clippy` | リント ＋ スタイル提案 |
| `dotnet format` | **rustfmt** | `rustup component add rustfmt` | 自動フォーマット |
| XML ドキュメントコメント | **`cargo doc`** | 標準搭載 | HTML ドキュメント生成 |
| OmniSharp / Roslyn | **rust-analyzer** | VS Code 拡張機能 | IDE サポート |
| `dotnet watch` | **cargo-watch** | `cargo install cargo-watch` | 保存時の自動再ビルド |
| — | **cargo-expand** | `cargo install cargo-expand` | マクロ展開結果の確認 |
| `dotnet audit` | **cargo-audit** | `cargo install cargo-audit` | セキュリティ脆弱性スキャン |

### Clippy: 自動コードレビュアー
```bash
# プロジェクトに対して Clippy を実行
cargo clippy

# 警告をエラーとして扱う（CI/CD 向け）
cargo clippy -- -D warnings

# 修正提案を自動適用
cargo clippy --fix
```

```rust
// Clippy は数百ものアンチパターンを検出します:

// Clippy 適用前:
if x == true { }           // 警告: bool との等値比較
let _ = vec.len() == 0;    // 警告: 代わりに .is_empty() を使用してください
for i in 0..vec.len() { }  // 警告: 代わりに .iter().enumerate() を使用してください

// Clippy の提案適用後:
if x { }
let _ = vec.is_empty();
for (i, item) in vec.iter().enumerate() { }
```

### rustfmt: 一貫したコードフォーマット
```bash
# すべてのファイルをフォーマット
cargo fmt

# 変更を加えずにフォーマットを検証（CI/CD 向け）
cargo fmt -- --check
```

```toml
# rustfmt.toml — フォーマットのカスタマイズ（.editorconfig に類似）
max_width = 100
tab_spaces = 4
use_field_init_shorthand = true
```

### cargo doc: ドキュメント生成
```bash
# ドキュメントを生成してブラウザで開く（依存関係を含む）
cargo doc --open

# ドキュメンテーションテストを実行
cargo test --doc
```

```rust
/// 円の面積を計算します。
///
/// # 引数
/// * `radius` - 円の半径（0以上である必要があります）
///
/// # 例
/// ```
/// let area = my_crate::circle_area(5.0);
/// assert!((area - 78.54).abs() < 0.01);
/// ```
///
/// # パニック
/// `radius` が負の場合にパニックします。
pub fn circle_area(radius: f64) -> f64 {
    assert!(radius >= 0.0, "radius must be non-negative");
    std::f64::consts::PI * radius * radius
}
// /// ``` ブロック内のコードは `cargo test` の実行時にコンパイルされテストされます！
```

### cargo watch: ファイル変更時の自動再ビルド
```bash
# ファイルの変更を検知して再ビルド（dotnet watch に類似）
cargo watch -x check          # 型チェックのみ実行（最速）
cargo watch -x test           # 保存時にテストを実行
cargo watch -x 'run -- args'  # 保存時にプログラムを実行
cargo watch -x clippy         # 保存時にリントを実行
```

### cargo expand: マクロが生成するコードの確認
```bash
# derive マクロの展開結果を確認
cargo expand --lib            # lib.rs を展開
cargo expand module_name      # 特定のモジュールを展開
```

### おすすめの VS Code 拡張機能

| 拡張機能 | 目的 |
|-----------|---------|
| **rust-analyzer** | コード補完、インラインエラー表示、リファクタリング |
| **CodeLLDB** | デバッガ（Visual Studio デバッガに類似） |
| **Even Better TOML** | Cargo.toml のシンタックスハイライト |
| **crates** | Cargo.toml 内でクレートの最新バージョンを表示 |
| **Error Lens** | エラーや警告をコード行内にインライン表示 |

***

本ガイドで言及した発展的トピックについてさらに深く掘り下げるには、以下の関連トレーニング教材を参照してください:

- **[Rust パターン集](../../rust-patterns-book/src/SUMMARY.md)** — Pin 射影（projections）、カスタムアロケータ、アリーナパターン、ロックフリーデータ構造、高度な Unsafe パターン
- **[非同期 Rust トレーニング](../../async-book/src/SUMMARY.md)** — Tokio の深層、非同期キャンセレーション安全性、ストリーム処理、本番環境向けの非同期アーキテクチャ
- **[C++開発者のための Rust トレーニング](../../c-cpp-book/src/SUMMARY.md)** — チームに C++ の経験者がいる場合に有用。ムーブセマンティクスの対応関係、RAII の違い、テンプレートとジェネリクスの比較
- **[C言語開発者のための Rust トレーニング](../../c-cpp-book/src/SUMMARY.md)** — 相互運用（FFI）シナリオに関連。FFI パターン、組み込み Rust のデバッグ、`no_std` プログラミング