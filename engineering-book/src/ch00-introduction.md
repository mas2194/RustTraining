# Rustエンジニアリングプラクティス — `cargo build`の先へ

## 著者（講師）紹介

- マイクロソフト SCHIE (Silicon and Cloud Hardware Infrastructure Engineering) チーム プリンシパルファームウェアアーキテクト
- セキュリティ、システムプログラミング（ファームウェア、オペレーティングシステム、ハイパーバイザ）、CPUおよびプラットフォームアーキテクチャ、C++システムに関する深い専門知識を持つ業界のベテラン
- 2017年（AWS EC2在籍時）にRustを使い始め、以来この言語に魅了され続けている

---

> 本書は、多くのチームがつまずいてから初めて気づくRustツールチェーンの機能（ビルドスクリプト、クロスコンパイル、ベンチマーク、コードカバレッジ、MiriやValgrindによる安全性検証など）を網羅した実践ガイドです。各章では、実際のハードウェア診断コードベース（大規模なマルチクレートワークスペース）から抽出した具体例を使用しているため、すべてのテクニックを本番コードにそのまま適用できます。

## 本書の使い方

本書は、**個人学習またはチームでのワークショップ**向けに設計されています。各章は概ね独立しているため、最初から順番に読み進めることも、必要なトピックに直接ジャンプすることも可能です。

### 難易度の凡例

| 記号 | レベル | 意味 |
|:------:|-------|---------|
| 🟢 | 初級 (Starter) | 明確なパターンを持つ扱いやすいツール — 今日からすぐに役立つ内容 |
| 🟡 | 中級 (Intermediate) | ツールチェーンの内部構造やプラットフォーム概念の理解が必要 |
| 🔴 | 上級 (Advanced) | 深いツールチェーン知識、Nightly機能、または複数ツールのオーケストレーションが必要 |

### 学習ペースの目安

| 部 | 該当章 | 目安時間 | 主な成果 |
|------|----------|:---------:|-------------|
| **I — ビルド＆シップ** | ch01–02 | 3–4 時間 | ビルドメタデータ、クロスコンパイル、静的バイナリ |
| **II — 計測＆検証** | ch03–05 | 4–5 時間 | 統計的ベンチマーク、カバレッジゲート、Miri/サニタイザ |
| **III — 堅牢化＆最適化** | ch06–10 | 6–8 時間 | サプライチェーンセキュリティ、リリースプロファイル、コンパイル時ツール、`no_std`、Windows |
| **IV — 統合** | ch11–13 | 3–4 時間 | 本番CI/CDパイプライン、現場のTips、総合演習 |
| | | **16–21 時間** | **完全なプロダクションエンジニアリングパイプラインの習得** |

### 演習問題への取り組み方

各章には難易度表記付きの **🏋️ 演習問題** が含まれています。解答は折りたたみ式の `<details>` ブロック内に用意されています。まずは自力で演習に取り組み、その後に解答を確認してください。

- 🟢 の演習は通常 10〜15 分程度で完了できます
- 🟡 の演習は 20〜40 分程度を要し、ローカル環境でツールを実行する場合があります
- 🔴 の演習は本格的なセットアップや実験が必要となります（1時間以上）

## 前提知識

| 概念 | 学習リソース |
|---------|-------------------|
| Cargo ワークスペースの構成 | [The Rust Programming Language 第14章3節](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) |
| 機能フラグ（Feature flags） | [Cargo Reference — Features](https://doc.rust-lang.org/cargo/reference/features.html) |
| `#[cfg(test)]` とテストの基礎 | Rust Patterns 第12章 |
| `unsafe` ブロックと FFI の基礎 | Rust Patterns 第10章 |

## 章の依存関係マップ

```text
                 ┌──────────┐
                 │ ch00     │
                 │ はじめに │
                 └────┬─────┘
        ┌─────┬───┬──┴──┬──────┬──────┐
        ▼     ▼   ▼     ▼      ▼      ▼
      ch01  ch03 ch04  ch05   ch06   ch09
     ビルド ベンチ カバレッジ Miri  依存関係 no_std
        │     │    │    │      │      │
        │     └────┴────┘      │      ▼
        │          │           │    ch10
        ▼          ▼           ▼   Windows
       ch02      ch07        ch07    │
     クロス   プロファイル プロファイル │
        │          │           │     │
        │          ▼           │     │
        │        ch08          │     │
        │     ビルド時間       │     │
        └──────────┴───────────┴─────┘
                   │
                   ▼
                 ch11
             CI/CD パイプライン
                   │
                   ▼
                ch12 ─── ch13
              Tips集   リファレンス
```

- **任意の順序で学習可能**: ch01, ch03, ch04, ch05, ch06, ch09 は互いに独立しています。
- **前提知識の後に学習**: ch02（ch01の知識が必要）、ch07–ch08（ch03–ch06を先に読むと効果的）、ch10（ch09を先に読むと効果的）。
- **最後に学習**: ch11（全体を統合）、ch12（現場のTips）、ch13（クイックリファレンス）。

## 章の概要一覧

### 第I部 — ビルド＆シップ

| # | 章 | 難易度 | 説明 |
|---|---------|:----------:|-------------|
| 1 | [ビルドスクリプト — `build.rs` 徹底解説](ch01-build-scripts-buildrs-in-depth.md) | 🟢 | コンパイル時定数、Cコードのコンパイル、protobuf生成、システムライブラリのリンク、アンチパターン |
| 2 | [クロスコンパイル — 1つのソースから複数のターゲットへ](ch02-cross-compilation-one-source-many-target.md) | 🟡 | ターゲットトリプル、musl静的バイナリ、ARMクロスコンパイル、`cross` ツール、`cargo-zigbuild`、GitHub Actions |

### 第II部 — 計測＆検証

| # | 章 | 難易度 | 説明 |
|---|---------|:----------:|-------------|
| 3 | [ベンチマーク — 本質的な性能を測る](ch03-benchmarking-measuring-what-matters.md) | 🟡 | Criterion.rs、Divan、`perf` フレームグラフ、PGO（プロファイル誘導最適化）、CIでの継続的ベンチマーク |
| 4 | [コードカバレッジ — テストが見逃した箇所の可視化](ch04-code-coverage-seeing-what-tests-miss.md) | 🟢 | `cargo-llvm-cov`、`cargo-tarpaulin`、`grcov`、Codecov/CoverallsによるCI統合 |
| 5 | [Miri・Valgrind・サニタイザ](ch05-miri-valgrind-and-sanitizers-verifying-u.md) | 🔴 | MIRインタープリタ、Valgrind memcheck/Helgrind、ASan/MSan/TSan、cargo-fuzz、loom |

### 第III部 — 堅牢化＆最適化

| # | 章 | 難易度 | 説明 |
|---|---------|:----------:|-------------|
| 6 | [依存関係管理とサプライチェーンセキュリティ](ch06-dependency-management-and-supply-chain-s.md) | 🟢 | `cargo-audit`、`cargo-deny`、`cargo-vet`、`cargo-outdated`、`cargo-semver-checks` |
| 7 | [リリースプロファイルとバイナリサイズ](ch07-release-profiles-and-binary-size.md) | 🟡 | リリースプロファイルの構造、LTOのトレードオフ、`cargo-bloat`、`cargo-udeps` |
| 8 | [コンパイル時間短縮と開発者ツール](ch08-compile-time-and-developer-tools.md) | 🟡 | `sccache`、`mold`、`cargo-nextest`、`cargo-expand`、`cargo-geiger`、ワークスペースlint、MSRV |
| 9 | [`no_std` と機能フラグの検証](ch09-no-std-and-feature-verification.md) | 🔴 | `cargo-hack`、`core`/`alloc`/`std` レイヤー、カスタムパニックハンドラ、`no_std` コードのテスト |
| 10 | [Windows環境と条件付きコンパイル](ch10-windows-and-conditional-compilation.md) | 🟡 | `#[cfg]` パターン、`windows-sys`/`windows` クレート、`cargo-xwin`、プラットフォーム抽象化 |

### 第IV部 — 統合

| # | 章 | 難易度 | 説明 |
|---|---------|:----------:|-------------|
| 11 | [すべてを統合する — 本番向けCI/CDパイプライン](ch11-putting-it-all-together-a-production-cic.md) | 🟡 | GitHub Actionsワークフロー、`cargo-make`、pre-commitフック、`cargo-dist`、総合演習 |
| 12 | [現場のプラクティス・実践テクニック](ch12-tricks-from-the-trenches.md) | 🟡 | 実戦で検証された10のパターン：`deny(warnings)` の罠、キャッシュ調整、依存関係の重複排除、RUSTFLAGS など |
| 13 | [クイックリファレンスカード](ch13-quick-reference-card.md) | — | コマンド一覧、60以上の決定テーブル、参考資料リンク |
