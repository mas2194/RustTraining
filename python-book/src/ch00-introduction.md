# PythonプログラマーのためのRust入門：完全トレーニングガイド

Pythonの経験を持つ開発者のための、Rustプログラミング言語の包括的な学習ガイドです。基礎的な構文から高度なパターンに至るまで網羅し、動的型付け・ガベージコレクションを備えた言語から、静的型付けとコンパイル時メモリ安全性を特徴とするシステムプログラミング言語へ移行する際に求められる「思考の転換」に焦点を当てています。

## 本書の読み方

**自習形式**: まずは第I部（1〜6章）から進めてください — これらはすでに知っているPythonの概念と密接に対応しています。第II部（7〜12章）では、所有権やトレイトといったRust特有の概念を学びます。第III部（13〜16章）では、高度なトピックと移行パターンを扱います。

**学習ペースの目安:**

| 章 | トピック | 推奨時間 | チェックポイント |
|---|---|---|---|
| 1–4 | 環境構築、型、制御フロー | 1日 | RustでCLI温度変換ツールが書ける |
| 5–6 | データ構造、列挙型、パターンマッチング | 1–2日 | データを保持する列挙型を定義し、網羅的な `match` が書ける |
| 7 | 所有権と借用 | 1–2日 | `let s2 = s1` がなぜ `s1` を無効化するのかを説明できる |
| 8–9 | モジュール、エラー処理 | 1日 | `?` を使ってエラーを伝播するマルチファイル構成のプロジェクトを作成できる |
| 10–12 | トレイト、ジェネリクス、クロージャ、イテレータ | 1–2日 | リスト内包表記をイテレータチェーンに変換できる |
| 13 | 並行性 | 1日 | `Arc<Mutex<T>>` を使ってスレッドセーフなカウンターが書ける |
| 14 | Unsafe、PyO3、テスト | 1日 | PyO3を介してPythonからRust関数を呼び出せる |
| 15–16 | 移行、ベストプラクティス | 自身のペースで | リファレンス資料 — 実際のコードを書く際に参照する |
| 17 | 総合演習プロジェクト | 2–3日 | すべての知識を統合した完全なCLIアプリケーションを構築できる |

**演習問題の進め方:**
- 各章には、折りたたみ式の `<details>` ブロック内に解答付きの実践演習問題が含まれています
- **解答を展開する前に、必ず自力で解いてみてください。** ボローチェッカー（借用検査器）との格闘こそが学習の肝です — コンパイラのエラーメッセージが最良の指導者となります
- 15分以上手が止まった場合は解答を展開して確認し、理解したら解答を閉じて最初から書き直してみてください
- [Rust Playground](https://play.rust-lang.org/) を使えば、ローカル環境へのインストールなしでコードを実行できます

**難易度表記:**
- 🟢 **初級** — Pythonの概念から直接置き換えられる内容
- 🟡 **中級** — 所有権やトレイトの理解が必要な内容
- 🔴 **上級** — ライフタイム、非同期処理の内部構造、Unsafeコードを含む内容

**行き詰まったときは:**
- コンパイラのエラーメッセージを注意深く読んでください — Rustのエラーは非常に親切で有益です
- 該当するセクションを読み返してください。所有権（第7章）などの概念は、2回目に読んだときに腑に落ちることがよくあります
- [Rust 標準ライブラリドキュメント](https://doc.rust-lang.org/std/) は非常に優れています — 任意の型やメソッドを検索してみてください
- より深い非同期パターンについては、姉妹編の [Async Rust Training](../async-book/) を参照してください

---

## 目次

### 第I部 — 基礎

#### 1. 導入と動機 🟢
- [Python開発者がRustを学ぶ理由](ch01-introduction-and-motivation.md#the-case-for-rust-for-python-developers)
- [Rustが解決するPythonの代表的な課題](ch01-introduction-and-motivation.md#common-python-pain-points-that-rust-addresses)
- [Pythonの代わりにRustを選択すべき状況](ch01-introduction-and-motivation.md#when-to-choose-rust-over-python)

#### 2. はじめの一歩 🟢
- [インストールとセットアップ](ch02-getting-started.md#installation-and-setup)
- [最初のRustプログラム](ch02-getting-started.md#your-first-rust-program)
- [Cargo と pip/Poetry の比較](ch02-getting-started.md#cargo-vs-pippoetry)

#### 3. 組み込み型と変数 🟢
- [変数と可変性](ch03-built-in-types-and-variables.md#variables-and-mutability)
- [プリミティブ型の比較](ch03-built-in-types-and-variables.md#primitive-types-comparison)
- [文字列型: String と &str](ch03-built-in-types-and-variables.md#string-types-string-vs-str)

#### 4. 制御フロー 🟢
- [条件分岐](ch04-control-flow.md#conditional-statements)
- [ループと反復処理](ch04-control-flow.md#loops-and-iteration)
- [式ブロック](ch04-control-flow.md#expression-blocks)
- [関数と型シグネチャ](ch04-control-flow.md#functions-and-type-signatures)

#### 5. データ構造とコレクション 🟢
- [タプル、配列、スライス](ch05-data-structures-and-collections.md#tuples-and-destructuring)
- [構造体とクラスの比較](ch05-data-structures-and-collections.md#structs-vs-classes)
- [Vec と list、HashMap と dict の比較](ch05-data-structures-and-collections.md#vec-vs-list)

#### 6. 列挙型とパターンマッチング 🟡
- [代数的データ型と Union 型の比較](ch06-enums-and-pattern-matching.md#algebraic-data-types-vs-union-types)
- [網羅的パターンマッチング](ch06-enums-and-pattern-matching.md#exhaustive-pattern-matching)
- [None の安全性を実現する Option](ch06-enums-and-pattern-matching.md#option-for-none-safety)

### 第II部 — コアコンセプト

#### 7. 所有権と借用 🟡
- [所有権の仕組みを理解する](ch07-ownership-and-borrowing.md#understanding-ownership)
- [ムーブセマンティクスと参照カウントの比較](ch07-ownership-and-borrowing.md#move-semantics-vs-reference-counting)
- [借用とライフタイム](ch07-ownership-and-borrowing.md#borrowing-and-lifetimes)
- [スマートポインタ](ch07-ownership-and-borrowing.md#smart-pointers)

#### 8. クレートとモジュール 🟢
- [RustのモジュールとPythonパッケージの比較](ch08-crates-and-modules.md#rust-modules-vs-python-packages)
- [クレートと PyPI パッケージの比較](ch08-crates-and-modules.md#crates-vs-pypi-packages)

#### 9. エラー処理 🟡
- [例外と Result の比較](ch09-error-handling.md#exceptions-vs-result)
- [? 演算子](ch09-error-handling.md#the--operator)
- [thiserror によるカスタムエラー型](ch09-error-handling.md#custom-error-types-with-thiserror)

#### 10. トレイトとジェネリクス 🟡
- [トレイトとダックタイピングの比較](ch10-traits-and-generics.md#traits-vs-duck-typing)
- [Protocol (PEP 544) とトレイトの比較](ch10-traits-and-generics.md#protocols-pep-544-vs-traits)
- [ジェネリクスの制約（トレイト境界）](ch10-traits-and-generics.md#generic-constraints)

#### 11. From トレイトと Into トレイト 🟡
- [Rustにおける型変換](ch11-from-and-into-traits.md#type-conversions-in-rust)
- [From、Into、TryFrom](ch11-from-and-into-traits.md#rust-frominto)
- [文字列変換パターン](ch11-from-and-into-traits.md#string-conversions)

#### 12. クロージャとイテレータ 🟡
- [クロージャとラムダ式の比較](ch12-closures-and-iterators.md#rust-closures-vs-python-lambdas)
- [イテレータとジェネレータの比較](ch12-closures-and-iterators.md#iterators-vs-generators)
- [マクロ: コードを生成するコード](ch12-closures-and-iterators.md#why-macros-exist-in-rust)

### 第III部 — 発展的トピックと移行

#### 13. 並行性 🔴
- [GIL の不在: 真の並列処理](ch13-concurrency.md#no-gil-true-parallelism)
- [スレッド安全性: 型システムによる保証](ch13-concurrency.md#thread-safety-type-system-guarantees)
- [async/await の比較](ch13-concurrency.md#asyncawait-comparison)

#### 14. Unsafe Rust、FFI、テスト 🔴
- [Unsafe を使用するタイミングと理由](ch14-unsafe-rust-and-ffi.md#when-and-why-to-use-unsafe)
- [PyO3: PythonのためのRust拡張](ch14-unsafe-rust-and-ffi.md#pyo3-rust-extensions-for-python)
- [ユニットテストと pytest の比較](ch14-unsafe-rust-and-ffi.md#unit-tests-vs-pytest)

#### 15. 移行パターン 🟡
- [Rustにおける一般的なPythonパターン](ch15-migration-patterns.md#common-python-patterns-in-rust)
- [Python開発者のための必須クレート](ch08-crates-and-modules.md#essential-crates-for-python-developers)
- [段階的な導入戦略](ch15-migration-patterns.md#incremental-adoption-strategy)

#### 16. ベストプラクティス 🟡
- [Python開発者のための慣用的なRust (Idiomatic Rust)](ch16-best-practices.md#idiomatic-rust-for-python-developers)
- [よくある落とし穴と解決策](ch16-best-practices.md#common-pitfalls-and-solutions)
- [Python→Rust ロゼッタストーン](ch16-best-practices.md#rosetta-stone-python-to-rust)
- [学習ロードマップと参考リソース](ch16-best-practices.md#learning-path-and-resources)

---

### 第IV部 — 総合演習

#### 17. 総合演習プロジェクト: CLI タスクマネージャー 🔴
- [プロジェクト概要: `rustdo`](ch17-capstone-project.md#the-project-rustdo)
- [データモデル、ストレージ、コマンド、ビジネスロジック](ch17-capstone-project.md#step-1-define-the-data-model-ch-3-6-10-11)
- [テストと発展課題](ch17-capstone-project.md#step-7-tests-ch-14)

***
