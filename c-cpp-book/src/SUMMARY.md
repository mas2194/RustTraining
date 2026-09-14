# 目次

[はじめに](ch00-introduction.md)

---

# 第I部 — 基礎

- [1. 導入と動機](ch01-introduction-and-motivation.md)
    - [なぜC/C++開発者にRustが必要なのか](ch01-1-why-c-cpp-developers-need-rust.md)
- [2. 環境構築とファーストステップ](ch02-getting-started.md)
- [3. 組み込み型](ch03-built-in-types.md)
- [4. 制御フロー](ch04-control-flow.md)
- [5. データ構造](ch05-data-structures.md)
- [6. 列挙型とパターンマッチング](ch06-enums-and-pattern-matching.md)
- [7. 所有権と借用](ch07-ownership-and-borrowing.md)
    - [ライフタイムと借用の詳細](ch07-1-lifetimes-and-borrowing-deep-dive.md)
    - [スマートポインタと内部可変性](ch07-2-smart-pointers-and-interior-mutability.md)
- [8. クレートとモジュール](ch08-crates-and-modules.md)
    - [テストパターン](ch08-1-testing-patterns.md)
- [9. エラー処理](ch09-error-handling.md)
    - [エラー処理のベストプラクティス](ch09-1-error-handling-best-practices.md)
- [10. トレイト](ch10-traits.md)
    - [ジェネリクス](ch10-1-generics.md)
- [11. From トレイトと Into トレイト](ch11-from-and-into-traits.md)
- [12. クロージャ](ch12-closures.md)
    - [イテレータの高度な活用法](ch12-1-iterator-power-tools.md)
- [13. 並行性](ch13-concurrency.md)
- [14. Unsafe Rust と FFI](ch14-unsafe-rust-and-ffi.md)

---

# 第II部 — 深掘り

- [15. no_std — 標準ライブラリなしのRust](ch15-no_std-rust-without-the-standard-library.md)
    - [組み込み開発の詳細](ch15-1-embedded-deep-dive.md)
- [16. ケーススタディ: 実践的なC++からRustへの移行](ch16-case-studies.md)
    - [ケーススタディ3〜5: ライフタイム、合成可能性、トレイトオブジェクト](ch16-cases-3-5-lifetime-borrowing.md)

---

# 第III部 — ベストプラクティス＆リファレンス

- [17. ベストプラクティス](ch17-best-practices.md)
    - [過剰な clone() の回避](ch17-1-avoiding-excessive-clone.md)
    - [境界チェックなしインデックスアクセスの回避](ch17-2-avoiding-unchecked-indexing.md)
    - [代入ピラミッドの解消](ch17-3-collapsing-assignment-pyramids.md)
    - [ロギングとトレーシングのエコシステム](ch17-4-logging-and-tracing-ecosystem.md)
- [18. C++ → Rust セマンティクスの深掘り](ch18-cpp-rust-semantic-deep-dives.md)
- [19. Rustマクロ: プリプロセッサからメタプログラミングへ](ch19-macros.md)
