# 目次

[はじめに](ch00-introduction.md)

---

# 第I部 — 基礎

- [1. 導入と動機](ch01-introduction-and-motivation.md)
- [2. 入門](ch02-getting-started.md)
    - [重要キーワードリファレンス *(任意)*](ch02-1-essential-keywords-reference.md)
- [3. 組み込み型と変数](ch03-built-in-types-and-variables.md)
    - [真の不変性とレコード型の錯覚](ch03-1-true-immutability-vs-record-illusions.md)
- [4. 制御フロー](ch04-control-flow.md)
- [5. データ構造とコレクション](ch05-data-structures-and-collections.md)
    - [コンストラクタパターン](ch05-1-constructor-patterns.md)
    - [コレクション — Vec、HashMap、イテレータ](ch05-2-collections-vec-hashmap-and-iterators.md)
- [6. 列挙型とパターンマッチング](ch06-enums-and-pattern-matching.md)
    - [網羅的マッチングとNull安全性](ch06-1-exhaustive-matching-and-null-safety.md)
- [7. 所有権と借用](ch07-ownership-and-borrowing.md)
    - [メモリ安全性の詳細](ch07-1-memory-safety-deep-dive.md)
    - [ライフタイムの詳細](ch07-2-lifetimes-deep-dive.md)
    - [スマートポインタ — 単一所有権を超えて](ch07-3-smart-pointers-beyond-single-ownership.md)
- [8. クレートとモジュール](ch08-crates-and-modules.md)
    - [パッケージ管理 — Cargo と NuGet](ch08-1-package-management-cargo-vs-nuget.md)
- [9. エラーハンドリング](ch09-error-handling.md)
    - [クレートレベルのエラー型と Result エイリアス](ch09-1-crate-level-error-types-and-result-alias.md)
- [10. トレイトとジェネリクス](ch10-traits-and-generics.md)
    - [ジェネリック制約](ch10-1-generic-constraints.md)
    - [継承 vs コンポジション](ch10-2-inheritance-vs-composition.md)
- [11. From トレイトと Into トレイト](ch11-from-and-into-traits.md)
- [12. クロージャとイテレータ](ch12-closures-and-iterators.md)
    - [マクロ入門](ch12-1-macros-primer.md)

---

# 第II部 — 並行性とシステム

- [13. 並行性](ch13-concurrency.md)
    - [Async/Await の詳細](ch13-1-asyncawait-deep-dive.md)
- [14. Unsafe Rust と FFI](ch14-unsafe-rust-and-ffi.md)
    - [テスト](ch14-1-testing.md)

---

# 第III部 — マイグレーションとベストプラクティス

- [15. マイグレーションパターンとケーススタディ](ch15-migration-patterns-and-case-studies.md)
    - [C# 開発者のための必須クレート](ch15-1-essential-crates-for-c-developers.md)
    - [段階的導入戦略](ch15-2-incremental-adoption-strategy.md)
- [16. ベストプラクティス](ch16-best-practices.md)
    - [パフォーマンス比較と移行](ch16-1-performance-comparison-and-migration.md)
    - [学習パスとリソース](ch16-2-learning-path-and-resources.md)
    - [Rust ツールエコシステム](ch16-3-rust-tooling-ecosystem.md)

---

# 総合演習（Capstone）

- [17. 総合演習プロジェクト: CLI 天気予報ツールの作成](ch17-capstone-project.md)
