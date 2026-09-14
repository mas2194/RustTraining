> [!IMPORTANT] 
> 本プロジェクトはこの1行を除いてMicrosoftのRustTrainingをGeminiに日本語化させた自分用のプロジェクトです。

<div style="background-color: #d9d9d9; padding: 16px; border-radius: 6px; color: #000000;">

**ライセンス** 本プロジェクトは、[MIT License](LICENSE) および [Creative Commons Attribution 4.0 International (CC-BY-4.0)](LICENSE-DOCS) のデュアルライセンスの下で提供されています。

</div>

<div style="background-color: #d9d9d9; padding: 16px; border-radius: 6px; color: #000000;">

**商標** 本プロジェクトには、プロジェクト、製品、またはサービスの商標やロゴが含まれている場合があります。Microsoftの商標またはロゴの許可された使用は、[Microsoftの商標およびブランドガイドライン](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general)に準拠する必要があります。改変された本プロジェクトのバージョンにおけるMicrosoftの商標またはロゴの使用は、混乱を招いたり、Microsoftによる後援を示唆したりしてはなりません。サードパーティの商標またはロゴの使用は、各サードパーティのポリシーに準拠します。

</div>

# Rust トレーニングブック

さまざまなプログラミング言語のバックグラウンドに応じたRustの学習コースに加え、非同期処理、高度なパターン、エンジニアリングの実践プラクティスを掘り下げる7つのトレーニング教材です。

本教材は、独自のコンテンツに加えて、Rustエコシステムにおける優れたリソースから得られた知見や実例を組み合わせて作成されています。書籍、ブログ、カンファレンストーク、動画シリーズなどに散らばる知識を結集し、詳細かつ技術的に正確で、教育的に体系化された学習体験を提供することを目指しています。

> **免責事項:** これらのブックはトレーニング教材であり、公式なリファレンスではありません。正確性には万全を期していますが、重要な詳細については必ず[公式のRustドキュメント](https://doc.rust-lang.org/)および[Rustリファレンス](https://doc.rust-lang.org/reference/)をご確認ください。

### インスピレーションと謝辞

- [**The Rust Programming Language**](https://doc.rust-lang.org/book/) — すべての基礎となる公式ガイドブック
- [**Jon Gjengset**](https://www.youtube.com/c/JonGjengset) — 高度なRustの内部構造に関するディープダイブ配信、`Crust of Rust` シリーズ
- [**withoutboats**](https://without.boats/blog/) — 非同期設計、`Pin`、Futureモデルに関する知見
- [**fasterthanlime (Amos)**](https://fasterthanli.me/) — 第一原理からのシステムプログラミング探究、魅力的な長編解説記事
- [**Mara Bos**](https://marabos.nl/) — 『*Rust Atomics and Locks*』、並行性プリミティブの解説
- [**Aleksey Kladov (matklad)**](https://matklad.github.io/) — rust-analyzer の洞察、API設計、エラー処理パターン
- [**Niko Matsakis**](https://smallcultfollowing.com/babysteps/) — 言語設計、借用チェッカーの内部構造、Polonius
- [**Rust by Example**](https://doc.rust-lang.org/rust-by-example/) および [**Rustonomicon**](https://doc.rust-lang.org/nomicon/) — 実践的なパターンとUnsafeのディープダイブ
- [**This Week in Rust**](https://this-week-in-rust.org/) — 多くの実例に影響を与えたコミュニティの発見や知見
- [**Binary Musings - Tag(Rust)**](https://binarymusings.org/posts/category/rust/) — Rustの内部構造に関する徹底解説
- …その他、ブログ記事、カンファレンストーク、RFC、フォーラムでの議論を通じて本教材に知見を提供してくださった**Rustコミュニティ全体の皆様**（個別に挙げきれないほど多数ですが、心より感謝いたします）

## 📖 読み始める

ご自身のバックグラウンドに合ったブックを選択してください。ブックは難易度・テーマ別に分類されており、学習パスを計画しやすくなっています。

| レベル | 説明 |
|-------|-------------|
| 🟢 **Bridge** | 他の言語からRustを学ぶ — まずはこちらから |
| 🔵 **Deep Dive** | Rustの主要サブシステムの集中的な探求 |
| 🟡 **Advanced** | 経験豊富なRustacean向けの実践パターンとテクニック |
| 🟣 **Expert** | 最先端の型レベルプログラミングと正当性担保テクニック |
| 🟤 **Practices** | エンジニアリング、ツールチェイン、本番運用のベストプラクティス |

| ブック | レベル | 対象者 |
|------|-------|-------------|
| [**C/C++ プログラマーのための Rust**](https://microsoft.github.io/RustTraining/c-cpp-book/) | 🟢 Bridge | ムーブセマンティクス、RAII、FFI、組み込み、no_std |
| [**C# プログラマーのための Rust**](https://microsoft.github.io/RustTraining/csharp-book/) | 🟢 Bridge | Swift / C# / Java → 所有権と型システム |
| [**Python プログラマーのための Rust**](https://microsoft.github.io/RustTraining/python-book/) | 🟢 Bridge | 動的型付け → 静的型付け、GIL不要の並行性 |
| [**非同期 Rust**](https://microsoft.github.io/RustTraining/async-book/) | 🔵 Deep Dive | Tokio、ストリーム、キャンセル安全性 |
| [**Rust パターン**](https://microsoft.github.io/RustTraining/rust-patterns-book/) | 🟡 Advanced | Pin、アロケータ、ロックフリーデータ構造、Unsafe |
| [**型主導の正当性**](https://microsoft.github.io/RustTraining/type-driven-correctness-book/) | 🟣 Expert | 型状態（タイプステート）、幽霊型（Phantom types）、ケーパビリティトークン |
| [**Rust エンジニアリング実践**](https://microsoft.github.io/RustTraining/engineering-book/) | 🟤 Practices | ビルドスクリプト、クロスコンパイル、CI/CD、Miri |

各ブックには、Mermaidダイアグラム、編集可能なRust Playground、演習問題、全文検索機能を備えた15〜16の章が含まれています。

> **ヒント:** サイドバーナビゲーションや検索機能を備えたレンダリング済みのブックは、[GitHub Pages サイト](https://microsoft.github.io/RustTraining/)で閲覧できます。
>
> **ローカルプレビュー:** オフラインでの閲覧や教材への貢献時には、以下を実行してください（事前に[Rustのインストール](https://rustup.rs/)が必要です）:
> ```
> git clone https://github.com/microsoft/RustTraining.git
> cd RustTraining
> cargo install mdbook mdbook-mermaid
> cargo xtask serve    # http://localhost:3000
> ```

---

## 🔧 メンテナー向け情報

<details>
<summary>ブックのローカルでのビルド、配信、および編集</summary>

### 前提条件

まだインストールしていない場合は、[**rustup** 経由で Rust をインストール](https://rustup.rs/)し、続いて以下を実行します:

```bash
cargo install mdbook@0.4.52 mdbook-mermaid@0.14.0
```

### リポジトリのクローン

```bash
git clone https://github.com/microsoft/RustTraining.git
cd RustTraining
```

### ビルドとローカル配信

```bash
cargo xtask build               # すべてのブックを site/ にビルド（ローカルプレビュー用）
cargo xtask serve               # ビルドして http://localhost:3000 でローカルサーバーを起動
cargo xtask deploy              # すべてのブックを docs/ にビルド（GitHub Pages用）
cargo xtask clean               # site/ および docs/ ディレクトリを削除
```

単一のブックのみをビルドまたはプレビューする場合:

```bash
cd c-cpp-book && mdbook serve --open    # http://localhost:3000
```

### デプロイ

本サイトは、`.github/workflows/pages.yml` により `main` ブランチへのプッシュ時に GitHub Pages へ自動デプロイされます。手動での操作は不要です。

</details>
