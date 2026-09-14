# C/C++ プログラマのための Rust ブートストラップコース

## コース概要
- コース概要
    - Rust を学ぶ動機（C と C++ 両方の視点から）
    - ローカル環境のインストール
    - 型、関数、制御フロー、パターンマッチング
    - モジュール、cargo
    - トレイト、ジェネリクス
    - コレクション、エラー処理
    - クロージャ、メモリ管理、ライフタイム、スマートポインタ
    - 並行性
    - Unsafe Rust（FFI（外部関数インターフェース）を含む）
    - ファームウェアチーム向けの `no_std` および組み込み Rust の基礎
    - ケーススタディ: 実践的な C++ から Rust への移行パターン
- このコースでは非同期（`async`）Rust は扱いません。Future、エグゼキュータ、`Pin`、tokio、実践的な非同期パターンについては、姉妹編の [非同期 Rust トレーニング](../async-book/) を参照してください。


---

# 独習ガイド

この教材は、講師による講義用としても、独習用としても利用できます。一人で学習を進める場合は、最大限に活用するために以下のガイドを参考にしてください。

**学習ペースの目安:**

| 章 | トピック | 推奨期間 | チェックポイント |
|---|---|---|---|
| 1–4 | セットアップ、型、制御フロー | 1日 | CLI 温度変換ツールを作成できる |
| 5–7 | データ構造、所有権 | 1–2日 | `let s2 = s1` が `s1` を無効化する*理由*を説明できる |
| 8–9 | モジュール、エラー処理 | 1日 | `?` を使ってエラーを伝播する複数ファイルのプロジェクトを作成できる |
| 10–12 | トレイト、ジェネリクス、クロージャ | 1–2日 | トレイト境界を持つジェネリック関数を作成できる |
| 13–14 | 並行性、Unsafe/FFI | 1日 | `Arc<Mutex<T>>` を使用してスレッドセーフなカウンタを作成できる |
| 15–16 | 深掘り | 自分のペースで | リファレンス資料 — 必要なときに参照 |
| 17–19 | ベストプラクティス＆リファレンス | 自分のペースで | 実際のコードを書く際に参照 |

**演習問題の使い方:**
- 各章には難易度別の実践的な演習問題が含まれています: 🟢 初級（Starter）、🟡 中級（Intermediate）、🔴 チャレンジ（Challenge）
- **解答を展開する前に、必ず自分で挑戦してください。** ボローチェッカ（借用チェッカ）と格闘することも学習の一部です。コンパイラのエラーメッセージこそが最良の指導者です。
- 15分以上詰まったら、解答を展開して内容を理解し、一度閉じてから最初から自力で書き直してみてください。
- [Rust Playground](https://play.rust-lang.org/) を使用すると、ローカルにインストールしなくてもブラウザ上でコードを実行できます。

**壁にぶつかったときは:**
- コンパイラのエラーメッセージを注意深く読んでください。Rust のエラーメッセージは非常に丁寧で有益です。
- 該当セクションを読み返してください。所有権（第7章）などの概念は、2回目で理解できることがよくあります。
- [Rust 標準ライブラリのドキュメント](https://doc.rust-lang.org/std/) は非常に充実しています。任意の型やメソッドを検索してみてください。
- 非同期パターンについては、姉妹編の [非同期 Rust トレーニング](../async-book/) を参照してください。

---

# 目次

## 第I部 — 基礎

### 1. 導入と動機
- [講師紹介とアプローチ](ch01-introduction-and-motivation.md#speaker-intro-and-general-approach)
- [Rust を採用する動機](ch01-introduction-and-motivation.md#the-case-for-rust)
- [Rust はこれらの問題にどう対処するか？](ch01-introduction-and-motivation.md#how-does-rust-address-these-issues)
- [Rust のその他の特長と強み](ch01-introduction-and-motivation.md#other-rust-usps-and-features)
- [クイックリファレンス: Rust vs C/C++](ch01-introduction-and-motivation.md#quick-reference-rust-vs-cc)
- [なぜC/C++開発者にRustが必要なのか](ch01-1-why-c-cpp-developers-need-rust.md)
  - [Rust が排除する問題 — 完全リスト](ch01-1-why-c-cpp-developers-need-rust.md#what-rust-eliminates--the-complete-list)
  - [C と C++ に共通する問題](ch01-1-why-c-cpp-developers-need-rust.md#the-problems-shared-by-c-and-c)
  - [C++ がさらに抱える問題](ch01-1-why-c-cpp-developers-need-rust.md#c-adds-more-problems-on-top)
  - [Rust はこれらにどう対処するか](ch01-1-why-c-cpp-developers-need-rust.md#how-rust-addresses-all-of-this)

### 2. 環境構築とファーストステップ
- [講釈は十分: コードを見てみよう](ch02-getting-started.md#enough-talk-already-show-me-some-code)
- [Rust のローカル環境インストール](ch02-getting-started.md#rust-local-installation)
- [Rust パッケージ（クレート）](ch02-getting-started.md#rust-packages-crates)
- [例: cargo とクレート](ch02-getting-started.md#example-cargo-and-crates)

### 3. 基本的な型と変数
- [Rust の組み込み型](ch03-built-in-types.md#built-in-rust-types)
- [Rust の型指定と代入](ch03-built-in-types.md#rust-type-specification-and-assignment)
- [Rust の型指定と型推論](ch03-built-in-types.md#rust-type-specification-and-inference)
- [Rust の変数と可変性](ch03-built-in-types.md#rust-variables-and-mutability)

### 4. 制御フロー
- [Rust の if キーワード](ch04-control-flow.md#rust-if-keyword)
- [while と for によるループ](ch04-control-flow.md#rust-loops-using-while-and-for)
- [loop によるループ](ch04-control-flow.md#rust-loops-using-loop)
- [Rust の式ブロック](ch04-control-flow.md#rust-expression-blocks)

### 5. データ構造とコレクション
- [Rust の配列型](ch05-data-structures.md#rust-array-type)
- [Rust のタプル](ch05-data-structures.md#rust-tuples)
- [Rust の参照](ch05-data-structures.md#rust-references)
- [C++ の参照 vs Rust の参照 — 主な違い](ch05-data-structures.md#c-references-vs-rust-references--key-differences)
- [Rust のスライス](ch05-data-structures.md#rust-slices)
- [Rust の定数と静的変数](ch05-data-structures.md#rust-constants-and-statics)
- [Rust の文字列: String vs &str](ch05-data-structures.md#rust-strings-string-vs-str)
- [Rust の構造体](ch05-data-structures.md#rust-structs)
- [Rust の Vec\<T\>](ch05-data-structures.md#rust-vec-type)
- [Rust の HashMap](ch05-data-structures.md#rust-hashmap-type)
- [演習: Vec と HashMap](ch05-data-structures.md#exercise-vec-and-hashmap)

### 6. パターンマッチングと列挙型
- [Rust の列挙型（enum）](ch06-enums-and-pattern-matching.md#rust-enum-types)
- [Rust の match 文](ch06-enums-and-pattern-matching.md#rust-match-statement)
- [演習: match と enum を用いた加算と減算の実装](ch06-enums-and-pattern-matching.md#exercise-implement-add-and-subtract-using-match-and-enum)

### 7. 所有権とメモリ管理
- [Rust のメモリ管理](ch07-ownership-and-borrowing.md#rust-memory-management)
- [Rust の所有権、借用、ライフタイム](ch07-ownership-and-borrowing.md#rust-ownership-borrowing-and-lifetimes)
- [Rust のムーブセマンティクス](ch07-ownership-and-borrowing.md#rust-move-semantics)
- [Rust の Clone](ch07-ownership-and-borrowing.md#rust-clone)
- [Rust の Copy トレイト](ch07-ownership-and-borrowing.md#rust-copy-trait)
- [Rust の Drop トレイト](ch07-ownership-and-borrowing.md#rust-drop-trait)
- [演習: Move、Copy、Drop](ch07-ownership-and-borrowing.md#exercise-move-copy-and-drop)
- [Rust のライフタイムと借用](ch07-1-lifetimes-and-borrowing-deep-dive.md#rust-lifetime-and-borrowing)
- [Rust のライフタイム注釈](ch07-1-lifetimes-and-borrowing-deep-dive.md#rust-lifetime-annotations)
- [演習: ライフタイムを持つスライスの格納](ch07-1-lifetimes-and-borrowing-deep-dive.md#exercise-slice-storage-with-lifetimes)
- [ライフタイム省略規則の詳細](ch07-1-lifetimes-and-borrowing-deep-dive.md#lifetime-elision-rules-deep-dive)
- [Rust の Box\<T\>](ch07-2-smart-pointers-and-interior-mutability.md#rust-boxt)
- [内部可変性: Cell\<T\> と RefCell\<T\>](ch07-2-smart-pointers-and-interior-mutability.md#interior-mutability-cellt-and-refcellt)
- [共有所有権: Rc\<T\>](ch07-2-smart-pointers-and-interior-mutability.md#shared-ownership-rct)
- [演習: 共有所有権と内部可変性](ch07-2-smart-pointers-and-interior-mutability.md#exercise-shared-ownership-and-interior-mutability)

### 8. モジュールとクレート
- [Rust のクレートとモジュール](ch08-crates-and-modules.md#rust-crates-and-modules)
- [演習: モジュールと関数](ch08-crates-and-modules.md#exercise-modules-and-functions)
- [ワークスペースとクレート（パッケージ）](ch08-crates-and-modules.md#workspaces-and-crates-packages)
- [演習: ワークスペースとパッケージ依存関係の利用](ch08-crates-and-modules.md#exercise-using-workspaces-and-package-dependencies)
- [crates.io からのコミュニティクレートの利用](ch08-crates-and-modules.md#using-community-crates-from-cratesio)
- [クレートの依存関係と SemVer](ch08-crates-and-modules.md#crates-dependencies-and-semver)
- [演習: rand クレートの利用](ch08-crates-and-modules.md#exercise-using-the-rand-crate)
- [Cargo.toml と Cargo.lock](ch08-crates-and-modules.md#cargotoml-and-cargolock)
- [Cargo の test 機能](ch08-crates-and-modules.md#cargo-test-feature)
- [その他の Cargo 機能](ch08-crates-and-modules.md#other-cargo-features)
- [テストパターン](ch08-1-testing-patterns.md)

### 9. エラー処理
- [列挙型と Option、Result の関係](ch09-error-handling.md#connecting-enums-to-option-and-result)
- [Rust の Option 型](ch09-error-handling.md#rust-option-type)
- [Rust の Result 型](ch09-error-handling.md#rust-result-type)
- [演習: Option を用いた log() 関数の実装](ch09-error-handling.md#exercise-log-function-implementation-with-option)
- [Rust のエラー処理](ch09-error-handling.md#rust-error-handling)
- [演習: エラー処理](ch09-error-handling.md#exercise-error-handling)
- [エラー処理のベストプラクティス](ch09-1-error-handling-best-practices.md)

### 10. トレイトとジェネリクス
- [Rust のトレイト](ch10-traits.md#rust-traits)
- [C++ 演算子オーバーロード → Rust std::ops トレイト](ch10-traits.md#c-operator-overloading--rust-stdops-traits)
- [演習: Logger トレイトの実装](ch10-traits.md#exercise-logger-trait-implementation)
- [enum と dyn Trait の使い分け](ch10-traits.md#when-to-use-enum-vs-dyn-trait)
- [演習: 移植前の設計検討（Think Before You Translate）](ch10-traits.md#exercise-think-before-you-translate)
- [Rust のジェネリクス](ch10-1-generics.md#rust-generics)
- [演習: ジェネリクス](ch10-1-generics.md#exercise-generics)
- [Rust のトレイトとジェネリクスの組み合わせ](ch10-1-generics.md#combining-rust-traits-and-generics)
- [データ型における Rust トレイト境界](ch10-1-generics.md#rust-traits-constraints-in-data-types)
- [演習: トレイト境界とジェネリクス](ch10-1-generics.md#exercise-traits-constraints-and-generics)
- [Rust の型状態（タイプステート）パターンとジェネリクス](ch10-1-generics.md#rust-type-state-pattern-and-generics)
- [Rust のビルダーパターン](ch10-1-generics.md#rust-builder-pattern)

### 11. 高度な型システム機能
- [Rust の From トレイトと Into トレイト](ch11-from-and-into-traits.md#rust-from-and-into-traits)
- [演習: From と Into](ch11-from-and-into-traits.md#exercise-from-and-into)
- [Rust の Default トレイト](ch11-from-and-into-traits.md#rust-default-trait)
- [Rust のその他の型変換](ch11-from-and-into-traits.md#other-rust-type-conversions)

### 12. 関数型プログラミング
- [Rust のクロージャ](ch12-closures.md#rust-closures)
- [演習: クロージャとキャプチャ](ch12-closures.md#exercise-closures-and-capturing)
- [Rust のイテレータ](ch12-closures.md#rust-iterators)
- [演習: Rust イテレータ](ch12-closures.md#exercise-rust-iterators)
- [イテレータの高度な活用法リファレンス](ch12-1-iterator-power-tools.md#iterator-power-tools-reference)

### 13. 並行性
- [Rust の並行性](ch13-concurrency.md#rust-concurrency)
- [なぜ Rust はデータレースを防げるのか: Send と Sync](ch13-concurrency.md#why-rust-prevents-data-races-send-and-sync)
- [演習: マルチスレッド単語カウント](ch13-concurrency.md#exercise-multi-threaded-word-count)

### 14. Unsafe Rust と FFI
- [Unsafe Rust](ch14-unsafe-rust-and-ffi.md#unsafe-rust)
- [シンプルな FFI の例](ch14-unsafe-rust-and-ffi.md#simple-ffi-example-rust-library-function-consumed-by-c)
- [複雑な FFI の例](ch14-unsafe-rust-and-ffi.md#complex-ffi-example)
- [Unsafe コードの正当性の保証](ch14-unsafe-rust-and-ffi.md#ensuring-correctness-of-unsafe-code)
- [演習: 安全な FFI ラッパーの作成](ch14-unsafe-rust-and-ffi.md#exercise-writing-a-safe-ffi-wrapper)

## 第II部 — 深掘り

### 15. no_std — ベアメタル向け Rust
- [no_std とは？](ch15-no_std-rust-without-the-standard-library.md#what-is-no_std)
- [no_std と std の使い分け](ch15-no_std-rust-without-the-standard-library.md#when-to-use-no_std-vs-std)
- [演習: no_std リングバッファ](ch15-no_std-rust-without-the-standard-library.md#exercise-no_std-ring-buffer)
- [組み込み開発の詳細](ch15-1-embedded-deep-dive.md)

### 16. ケーススタディ: 実践的な C++ から Rust への移行
- [ケーススタディ 1: 継承階層 → enum ディスパッチ](ch16-case-studies.md#case-study-1-inheritance-hierarchy--enum-dispatch)
- [ケーススタディ 2: shared_ptr ツリー → アリーナ/インデックスパターン](ch16-case-studies.md#case-study-2-shared_ptr-tree--arenaindex-pattern)
- [ケーススタディ 3: フレームワーク間通信 → ライフタイム借用](ch16-1-case-study-lifetime-borrowing.md#case-study-3-framework-communication--lifetime-borrowing)
- [ケーススタディ 4: 神オブジェクト（God Object） → 合成可能な状態](ch16-1-case-study-lifetime-borrowing.md#case-study-4-god-object--composable-state)
- [ケーススタディ 5: トレイトオブジェクト — 適切な使いどころ](ch16-1-case-study-lifetime-borrowing.md#case-study-5-trait-objects--when-they-are-right)

## 第III部 — ベストプラクティス＆リファレンス

### 17. ベストプラクティス
- [Rust ベストプラクティスまとめ](ch17-best-practices.md#rust-best-practices-summary)
- [過剰な clone() の回避](ch17-1-avoiding-excessive-clone.md#avoiding-excessive-clone)
- [境界チェックなしインデックスアクセスの回避](ch17-2-avoiding-unchecked-indexing.md#avoiding-unchecked-indexing)
- [代入ピラミッドの解消](ch17-3-collapsing-assignment-pyramids.md#collapsing-assignment-pyramids)
- [総合演習: 診断イベントパイプライン](ch17-3-collapsing-assignment-pyramids.md#capstone-exercise-diagnostic-event-pipeline)
- [ロギングとトレーシングのエコシステム](ch17-4-logging-and-tracing-ecosystem.md#logging-and-tracing-ecosystem)

### 18. C++ → Rust セマンティクスの深掘り
- [キャスト、プリプロセッサ、モジュール、volatile、static、constexpr、SFINAE など](ch18-cpp-rust-semantic-deep-dives.md)

### 19. Rust マクロ
- [宣言的マクロ（`macro_rules!`）](ch19-macros.md#declarative-macros-with-macro_rules)
- [標準ライブラリの一般的なマクロ](ch19-macros.md#common-standard-library-macros)
- [Derive マクロ](ch19-macros.md#derive-macros)
- [属性マクロ](ch19-macros.md#attribute-macros)
- [手続き型マクロ（概念概要）](ch19-macros.md#procedural-macros-conceptual-overview)
- [使い分けの指針: マクロ vs 関数 vs ジェネリクス](ch19-macros.md#when-to-use-what-macros-vs-functions-vs-generics)
- [演習問題](ch19-macros.md#exercises)
