# C# プログラマーのための Rust 入門: 完全トレーニングガイド

C# の開発経験を持つエンジニアのための包括的な Rust 学習ガイドです。本ガイドでは、基本的な構文から高度なパターンに至るまで、2つの言語間における概念の転換や実践的な相違点に焦点を当てて解説します。

## コース概要
- **Rust を学ぶ意義** — パフォーマンス、安全性、正確性の観点から C# 開発者にとって Rust が重要である理由
- **入門** — インストール、ツールチェーン、最初のプログラム
- **基本的な構成要素** — 型、変数、制御フロー
- **データ構造** — 配列、タプル、構造体、コレクション
- **パターンマッチングと列挙型** — 代数的データ型と網羅的マッチング
- **所有権と借用** — Rust のメモリ管理モデル
- **モジュールとクレート** — コードの構成と依存関係の管理
- **エラーハンドリング** — Result 型に基づくエラー伝播
- **トレイトとジェネリクス** — Rust の型システム
- **クロージャとイテレータ** — 関数型プログラミングパターン
- **並行性** — 型システムの保証による「恐れなき並行性（Fearless Concurrency）」、async/await の詳細
- **Unsafe Rust と FFI** — 安全な Rust の枠を超えるケースとその手法
- **マイグレーションパターン** — C# から Rust への実践的パターンと段階的導入
- **ベストプラクティス** — C# 開発者のためのイディオマティックな Rust

---

# 自習ガイド

本教材は講師主導のトレーニングコースとしても、自習用としても活用できます。独学で進める場合は、以下の推奨ペースを参考に学習効果を最大限に高めてください。

**推奨ペース:**

| 章 | トピック | 目安時間 | チェックポイント |
|---|---|---|---|
| 1–4 | 環境構築、型、制御フロー | 1日 | Rust で CLI 温度変換ツールを作成できる |
| 5–6 | データ構造、列挙型、パターンマッチング | 1–2日 | データを保持する enum を定義し、`match` で網羅的に分岐できる |
| 7 | 所有権と借用 | 1–2日 | `let s2 = s1` によって *なぜ* `s1` が無効化されるのかを説明できる |
| 8–9 | モジュール、エラーハンドリング | 1日 | `?` 演算子でエラーを伝播するマルチファイル構成のプロジェクトを作成できる |
| 10–12 | トレイト、ジェネリクス、クロージャ、イテレータ | 1–2日 | LINQ メソッドチェーンを Rust のイテレータに変換できる |
| 13 | 並行性と非同期処理 | 1日 | `Arc<Mutex<T>>` を用いたスレッドセーフなカウンタを実装できる |
| 14 | Unsafe Rust、FFI、テスト | 1日 | P/Invoke 経由で C# から Rust の関数を呼び出せる |
| 15–16 | マイグレーション、ベストプラクティス、ツール | 自分のペースで | リファレンスとして、実際のコードを書く際に適宜参照する |
| 17 | 総合演習プロジェクト | 1–2日 | 気象データを取得する実用的な CLI ツールを完成させられる |

**演習問題の取り組み方:**
- 各章には折りたたみ可能な `<details>` ブロック内に解答付きの実践演習が含まれています。
- **必ず解答を展開する前に自力で取り組んでください。** ボローチェッカー（借用チェッカー）との格闘も学習の重要なステップです。コンパイラのエラーメッセージこそが最高のメンターです。
- 15分以上手が止まってしまった場合は、解答を展開して内容を理解した上で、一旦閉じて再度ゼロから書いてみましょう。
- ローカルに環境をインストールしていなくても、[Rust Playground](https://play.rust-lang.org/) を使ってブラウザ上でコードを実行できます。

**難易度アイコン:**
- 🟢 **初級** — C# の概念から直接類推できる内容
- 🟡 **中級** — 所有権やトレイトの理解が必要な内容
- 🔴 **上級** — ライフタイム、非同期の内部構造、または unsafe コードを扱う内容

**壁にぶつかったときは:**
- コンパイラのエラーメッセージを注意深く読んでください — Rust のコンパイラエラーは非常に親切で具体的です。
- 関連するセクションを再読してください。所有権（第7章）などの概念は、2回目に読んだときにすんなり理解できることがよくあります。
- [Rust 標準ライブラリドキュメント](https://doc.rust-lang.org/std/) は非常に充実しています。任意の型やメソッドを検索してみましょう。
- 非同期処理の高度なパターンをさらに深掘りしたい場合は、姉妹教材である [Async Rust Training](../async-book/) を参照してください。

---

# 目次

## 第I部 — 基礎

### 1. 導入と動機 🟢
- [C# 開発者にとっての Rust の価値](ch01-introduction-and-motivation.md#the-case-for-rust-for-c-developers)
- [Rust が解決する C# の代表的な課題](ch01-introduction-and-motivation.md#common-c-pain-points-that-rust-addresses)
- [C# ではなく Rust を選ぶべき場面](ch01-introduction-and-motivation.md#when-to-choose-rust-over-c)
- [言語設計哲学の比較](ch01-introduction-and-motivation.md#language-philosophy-comparison)
- [クイックリファレンス: Rust vs C#](ch01-introduction-and-motivation.md#quick-reference-rust-vs-c)

### 2. 入門 🟢
- [インストールとセットアップ](ch02-getting-started.md#installation-and-setup)
- [最初の Rust プログラム](ch02-getting-started.md#your-first-rust-program)
- [Cargo vs NuGet / MSBuild](ch02-getting-started.md#cargo-vs-nugetmsbuild)
- [入力と CLI 引数の読み取り](ch02-getting-started.md#reading-input-and-cli-arguments)
- [Rust の重要キーワード *(必要に応じて参照する任意のリファレンス)*](ch02-1-essential-keywords-reference.md#essential-rust-keywords-for-c-developers)

### 3. 組み込み型と変数 🟢
- [変数と可変性](ch03-built-in-types-and-variables.md#variables-and-mutability)
- [プリミティブ型の比較](ch03-built-in-types-and-variables.md#primitive-types)
- [文字列型: String vs &str](ch03-built-in-types-and-variables.md#string-types-string-vs-str)
- [出力と文字列フォーマット](ch03-built-in-types-and-variables.md#printing-and-string-formatting)
- [型キャストと型変換](ch03-built-in-types-and-variables.md#type-casting-and-conversions)
- [真の不変性とレコード型の錯覚](ch03-1-true-immutability-vs-record-illusions.md#true-immutability-vs-record-illusions)

### 4. 制御フロー 🟢
- [関数 vs メソッド](ch04-control-flow.md#functions-vs-methods)
- [式（Expression）vs 文（Statement）（重要！）](ch04-control-flow.md#expression-vs-statement-important)
- [条件分岐](ch04-control-flow.md#conditional-statements)
- [ループと反復処理](ch04-control-flow.md#loops)

### 5. データ構造とコレクション 🟢
- [タプルと分割代入](ch05-data-structures-and-collections.md#tuples-and-destructuring)
- [配列とスライス](ch05-data-structures-and-collections.md#arrays-and-slices)
- [構造体 vs クラス](ch05-data-structures-and-collections.md#structs-vs-classes)
- [コンストラクタパターン](ch05-1-constructor-patterns.md#constructor-patterns)
- [`Vec<T>` vs `List<T>`](ch05-2-collections-vec-hashmap-and-iterators.md#vect-vs-listt)
- [HashMap vs Dictionary](ch05-2-collections-vec-hashmap-and-iterators.md#hashmap-vs-dictionary)

### 6. 列挙型とパターンマッチング 🟡
- [代数的データ型 vs C# 共用体](ch06-enums-and-pattern-matching.md#algebraic-data-types-vs-c-unions)
- [網羅的パターンマッチング](ch06-1-exhaustive-matching-and-null-safety.md#exhaustive-pattern-matching-compiler-guarantees-vs-runtime-errors)
- [Null 安全性のための `Option<T>`](ch06-1-exhaustive-matching-and-null-safety.md#null-safety-nullablet-vs-optiont)
- [ガード節と高度なパターン](ch06-enums-and-pattern-matching.md#guards-and-advanced-patterns)

### 7. 所有権と借用 🟡
- [所有権の理解](ch07-ownership-and-borrowing.md#understanding-ownership)
- [ムーブセマンティクス vs 参照セマンティクス](ch07-ownership-and-borrowing.md#move-semantics)
- [借用と参照](ch07-ownership-and-borrowing.md#borrowing-basics)
- [メモリ安全性の詳細](ch07-1-memory-safety-deep-dive.md#references-vs-pointers)
- [ライフタイムの詳細](ch07-2-lifetimes-deep-dive.md#lifetimes-telling-the-compiler-how-long-references-live) 🔴
- [スマートポインタ、Drop、Deref](ch07-3-smart-pointers-beyond-single-ownership.md#smart-pointers-when-single-ownership-isnt-enough) 🔴

### 8. クレートとモジュール 🟢
- [Rust のモジュール vs C# の名前空間](ch08-crates-and-modules.md#rust-modules-vs-c-namespaces)
- [クレート vs .NET アセンブリ](ch08-crates-and-modules.md#crates-vs-net-assemblies)
- [パッケージ管理: Cargo vs NuGet](ch08-1-package-management-cargo-vs-nuget.md#package-management-cargo-vs-nuget)

### 9. エラーハンドリング 🟡
- [例外 vs `Result<T, E>`](ch09-error-handling.md#exceptions-vs-resultt-e)
- [? 演算子](ch09-error-handling.md#the--operator-propagating-errors-concisely)
- [カスタムエラー型](ch06-1-exhaustive-matching-and-null-safety.md#custom-error-types)
- [クレートレベルのエラー型と Result エイリアス](ch09-1-crate-level-error-types-and-result-alias.md#crate-level-error-types-and-result-aliases)
- [エラーリカバリパターン](ch09-1-crate-level-error-types-and-result-alias.md#error-recovery-patterns)

### 10. トレイトとジェネリクス 🟡
- [トレイト vs インターフェース](ch10-traits-and-generics.md#traits---rusts-interfaces)
- [継承 vs コンポジション](ch10-2-inheritance-vs-composition.md#inheritance-vs-composition)
- [ジェネリック制約: where 節 vs トレイト境界](ch10-1-generic-constraints.md#generic-constraints-where-vs-trait-bounds)
- [よく使われる標準ライブラリのトレイト](ch10-traits-and-generics.md#common-standard-library-traits)

### 11. From トレイトと Into トレイト 🟡
- [Rust における型変換](ch11-from-and-into-traits.md#type-conversions-in-rust)
- [カスタム型に対する From の実装](ch11-from-and-into-traits.md#rust-from-and-into)

### 12. クロージャとイテレータ 🟡
- [Rust のクロージャ](ch12-closures-and-iterators.md#rust-closures)
- [LINQ vs Rust のイテレータ](ch12-closures-and-iterators.md#linq-vs-rust-iterators)
- [マクロ入門](ch12-1-macros-primer.md#macros-code-that-writes-code)

---

## 第II部 — 並行性とシステム

### 13. 並行性 🔴
- [スレッド安全性: 慣習 vs 型システムの保証](ch13-concurrency.md#thread-safety-convention-vs-type-system-guarantees)
- [async/await: C# の Task vs Rust の Future](ch13-1-asyncawait-deep-dive.md#async-programming-c-task-vs-rust-future)
- [キャンセルパターン](ch13-1-asyncawait-deep-dive.md#cancellation-cancellationtoken-vs-drop--select)
- [Pin と tokio::spawn](ch13-1-asyncawait-deep-dive.md#pin-why-rust-async-has-a-concept-c-doesnt)

### 14. Unsafe Rust、FFI、テスト 🟡
- [Unsafe を使用するタイミングと理由](ch14-unsafe-rust-and-ffi.md#when-you-need-unsafe)
- [FFI を介した C# との相互運用](ch14-unsafe-rust-and-ffi.md#interop-with-c-via-ffi)
- [Rust と C# のテスト比較](ch14-1-testing.md#testing-in-rust-vs-c)
- [プロパティベーステストとモック](ch14-1-testing.md#property-testing-proving-correctness-at-scale)

---

## 第III部 — マイグレーションとベストプラクティス

### 15. マイグレーションパターンとケーススタディ 🟡
- [Rust における一般的な C# パターン](ch15-migration-patterns-and-case-studies.md#common-c-patterns-in-rust)
- [C# 開発者のための必須クレート](ch15-1-essential-crates-for-c-developers.md#essential-crates-for-c-developers)
- [段階的導入戦略](ch15-2-incremental-adoption-strategy.md#incremental-adoption-strategy)

### 16. ベストプラクティスとリファレンス 🟡
- [C# 開発者のためのイディオマティックな Rust](ch16-best-practices.md#best-practices-for-c-developers)
- [パフォーマンス比較: マネージド vs ネイティブ](ch16-1-performance-comparison-and-migration.md#performance-comparison-managed-vs-native)
- [よくある落とし穴と解決策](ch16-2-learning-path-and-resources.md#common-pitfalls-for-c-developers)
- [学習パスとリソース](ch16-2-learning-path-and-resources.md#learning-path-and-next-steps)
- [Rust ツールエコシステム](ch16-3-rust-tooling-ecosystem.md#essential-rust-tooling-for-c-developers)

---

## 総合演習

### 17. 総合演習プロジェクト 🟡
- [CLI 天気予報ツールの作成](ch17-capstone-project.md#capstone-project-build-a-cli-weather-tool) — 構造体、トレイト、エラーハンドリング、非同期処理、モジュール、serde、テストを統合してひとつの動作するアプリケーションを構築します
