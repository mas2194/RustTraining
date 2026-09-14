# クイックリファレンスカード

### チートシート: コマンド早見表

```bash
# ─── ビルドスクリプト ───
cargo build                          # 最初に build.rs をコンパイルし、次にクレート本体をコンパイル
cargo build -vv                      # 詳細出力 — build.rs の標準出力を表示

# ─── クロスコンパイル ───
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl
cargo zigbuild --release --target x86_64-unknown-linux-gnu.2.17
cross build --release --target aarch64-unknown-linux-gnu

# ─── ベンチマーク ───
cargo bench                          # すべてのベンチマークを実行
cargo bench -- parse                 # "parse" に一致するベンチマークを実行
cargo flamegraph -- --args           # バイナリからフレームグラフを生成
perf record -g ./target/release/bin  # perf データを記録
perf report                          # perf データを対話的に表示

# ─── カバレッジ ───
cargo llvm-cov --html                # HTML レポートを生成
cargo llvm-cov --lcov --output-path lcov.info
cargo llvm-cov --workspace --fail-under-lines 80
cargo tarpaulin --out Html           # 代替ツール

# ─── 安全性の検証 ───
cargo +nightly miri test             # Miri 環境下でテストを実行
MIRIFLAGS="-Zmiri-disable-isolation" cargo +nightly miri test
valgrind --leak-check=full ./target/debug/binary
RUSTFLAGS="-Zsanitizer=address" cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu

# ─── 監査とサプライチェーンセキュリティ ───
cargo audit                          # 既知の脆弱性をスキャン
cargo audit --deny warnings          # アドバイザリが検出された場合に CI を失敗させる
cargo deny check                     # ライセンス + アドバイザリ + 禁止クレート + ソース元のチェック
cargo deny list                      # 依存関係ツリー内のすべてのライセンスを一覧表示
cargo vet                            # サプライチェーンの信頼性検証
cargo outdated --workspace           # 古くなった依存関係を検出
cargo semver-checks                  # 破壊的 API 変更を検出
cargo geiger                         # 依存関係ツリー内の unsafe の数を集計

# ─── バイナリの最適化 ───
cargo bloat --release --crates       # クレートごとのバイナリサイズ寄与度を表示
cargo bloat --release -n 20          # サイズの大きい上位 20 関数を表示
cargo +nightly udeps --workspace     # 未使用の依存関係を検出
cargo machete                        # 未使用の依存関係を高速に検出
cargo expand --lib module::name      # マクロの展開結果を表示
cargo msrv find                      # サポートする最小 Rust バージョン（MSRV）を調査
cargo clippy --fix --workspace --allow-dirty  # リント警告を自動修正

# ─── コンパイル時間の最適化 ───
export RUSTC_WRAPPER=sccache         # 共有コンパイルキャッシュを有効化
sccache --show-stats                 # キャッシュヒット統計を表示
cargo nextest run                    # 高速なテストランナーで実行
cargo nextest run --retries 2        # 不安定なテストをリトライ実行

# ─── プラットフォームエンジニアリング ───
cargo check --target thumbv7em-none-eabihf   # no_std ビルドを検証
cargo build --target x86_64-pc-windows-gnu   # Windows 向けにクロスコンパイル
cargo xwin build --target x86_64-pc-windows-msvc  # MSVC ABI 向けクロスコンパイル
cfg!(target_os = "linux")                    # コンパイル時 cfg (bool 値として評価)

# ─── リリース ───
cargo release patch --dry-run        # リリースのプレビュー
cargo release patch --execute        # バージョン引き上げ、コミット、タグ付け、公開
cargo dist plan                      # 配布成果物のプレビュー
```

### 決定表: 目的別ツールの使い分け

| 目的 | ツール | 使い分け・利用シーン |
|------|--------|----------------------|
| git ハッシュやビルド情報の埋め込み | `build.rs` | バイナリにトレーサビリティを持たせたい場合 |
| Rust と一緒に C コードをコンパイル | `build.rs` 内の `cc` クレート | 小さな C ライブラリへの FFI |
| スキーマ定義からのコード生成 | `prost-build` / `tonic-build` | Protobuf, gRPC, FlatBuffers |
| システムライブラリのリンク | `build.rs` 内の `pkg-config` | OpenSSL, libpci, systemd |
| 静的リンクされた Linux バイナリ | `--target x86_64-unknown-linux-musl` | コンテナ / クラウドデプロイ |
| 古い glibc を対象にする | `cargo-zigbuild` | RHEL 7, CentOS 7 互換性 |
| ARM サーバー用バイナリ | `cross` または `cargo-zigbuild` | Graviton / Ampere へのデプロイ |
| 統計的ベンチマーク | Criterion.rs | パフォーマンス低下（リグレッション）の検知 |
| 手軽なパフォーマンス確認 | Divan | 開発中のプロファイリング |
| ホットスポットの特定 | `cargo flamegraph` / `perf` | ベンチマークでボトルネックを特定した後 |
| 行・分岐カバレッジ測定 | `cargo-llvm-cov` | CI のカバレッジゲート、未テスト箇所の分析 |
| 手軽なカバレッジ確認 | `cargo-tarpaulin` | ローカル開発時 |
| Rust の未定義動作（UB）検出 | Miri | 純粋な Rust の `unsafe` コード |
| C FFI のメモリ安全性検証 | Valgrind memcheck | Rust と C が混在するコードベース |
| データ競合の検出 | TSan または Miri | 並行処理を行う `unsafe` コード |
| バッファオーバーフローの検出 | ASan | `unsafe` なポインタ演算 |
| メモリリークの検出 | Valgrind または LSan | 長時間稼働するサービス |
| ローカルでの CI 同等環境 | `cargo-make` | 開発者ワークフローの自動化 |
| プッシュ前のチェック | `cargo-husky` または git フック | プッシュ前に問題を検出 |
| 自動リリース | `cargo-release` + `cargo-dist` | バージョン管理 + パッケージ配布 |
| 依存関係の脆弱性監査 | `cargo-audit` / `cargo-deny` | サプライチェーンセキュリティ |
| ライセンスコンプライアンス | `cargo-deny` (licenses) | 商用 / エンタープライズプロジェクト |
| サプライチェーンの信頼性 | `cargo-vet` | 高セキュリティ環境 |
| 古くなった依存関係の特定 | `cargo-outdated` | 定期メンテナンス |
| 破壊的変更の検出 | `cargo-semver-checks` | ライブラリクレートの公開時 |
| 依存関係ツリーの分析 | `cargo tree --duplicates` | 依存グラフの重複排除とスリム化 |
| バイナリサイズの分析 | `cargo-bloat` | 容量制約のある環境へのデプロイ |
| 未使用の依存関係の検出 | `cargo-udeps` / `cargo-machete` | コンパイル時間とバイナリサイズの削減 |
| LTO のチューニング | `lto = true` または `"thin"` | リリースバイナリの最適化 |
| サイズ優先のバイナリ最適化 | `opt-level = "z"` + `strip = true` | 組み込み / WASM / コンテナ |
| unsafe 使用箇所の監査 | `cargo-geiger` | セキュリティポリシーの適用 |
| マクロのデバッグ | `cargo-expand` | derive マクロや `macro_rules` のデバッグ |
| リンクの高速化 | `mold` リンカ | 開発時のインナーループ高速化 |
| コンパイルキャッシュ | `sccache` | CI およびローカルビルドの高速化 |
| テストの高速化 | `cargo-nextest` | CI およびローカルテストの高速化 |
| MSRV 準拠の確認 | `cargo-msrv` | ライブラリの公開前確認 |
| `no_std` ライブラリ | `#![no_std]` + `default-features = false` | 組み込み、UEFI、WASM |
| Windows 向けクロスコンパイル | `cargo-xwin` / MinGW | Linux から Windows 向けビルド |
| プラットフォームの抽象化 | `#[cfg]` + トレイトパターン | マルチ OS 対応コードベース |
| Windows API の直接呼び出し | `windows-sys` / `windows` クレート | Windows ネイティブ機能の利用 |
| エンドツーエンドの実行時間計測 | `hyperfine` | バイナリ全体のベンチマーク、適用前後の比較 |
| プロパティベーステスト | `proptest` | エッジケースの発見、パーサーの堅牢性検証 |
| スナップショットテスト | `insta` | 大規模な構造化出力の検証 |
| カバレッジ誘導ファジング | `cargo-fuzz` | パーサーにおけるクラッシュの発見 |
| 並行性モデル検査 | `loom` | ロックフリーデータ構造、アトミック操作の順序付け |
| フィーチャ組み合わせのテスト | `cargo-hack` | 複数の `#[cfg]` フィーチャを持つクレート |
| 高速な UB チェック（ネイティブ並み） | `cargo-careful` | CI 安全性ゲート、Miri より軽量 |
| 保存時の自動リビルド | `cargo-watch` | 開発インナーループ、迅速なフィードバック |
| ワークスペースのドキュメント生成 | `cargo doc` + rustdoc | API の探索、オンボーディング、ドキュメントリンク CI |
| 再現可能なビルド | `--locked` + `SOURCE_DATE_EPOCH` | リリース成果物の完全性検証 |
| CI キャッシュのチューニング | `Swatinem/rust-cache@v2` | ビルド時間短縮（コールド → キャッシュあり） |
| ワークスペースのリント方針統一 | Cargo.toml 内の `[workspace.lints]` | すべてのクレートで一貫した Clippy/コンパイラリント |
| リント警告の自動修正 | `cargo clippy --fix` | 単純な問題の自動クリーンアップ |

### 参考資料

| トピック | リソース |
|----------|----------|
| Cargo ビルドスクリプト | [Cargo Book — Build Scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html) |
| クロスコンパイル | [Rust Cross-Compilation](https://rust-lang.github.io/rustup/cross-compilation.html) |
| `cross` ツール | [cross-rs/cross](https://github.com/cross-rs/cross) |
| `cargo-zigbuild` | [cargo-zigbuild docs](https://github.com/rust-cross/cargo-zigbuild) |
| Criterion.rs | [Criterion User Guide](https://bheisler.github.io/criterion.rs/book/) |
| Divan | [Divan docs](https://github.com/nvzqz/divan) |
| `cargo-llvm-cov` | [cargo-llvm-cov](https://github.com/taiki-e/cargo-llvm-cov) |
| `cargo-tarpaulin` | [tarpaulin docs](https://github.com/xd009642/tarpaulin) |
| Miri | [Miri GitHub](https://github.com/rust-lang/miri) |
| Rust におけるサニタイザ | [rustc Sanitizer docs](https://doc.rust-lang.org/nightly/unstable-book/compiler-flags/sanitizer.html) |
| `cargo-make` | [cargo-make book](https://sagiegurari.github.io/cargo-make/) |
| `cargo-release` | [cargo-release docs](https://github.com/crate-ci/cargo-release) |
| `cargo-dist` | [cargo-dist docs](https://axodotdev.github.io/cargo-dist/book/) |
| プロファイルガイド最適化 (PGO) | [Rust PGO guide](https://doc.rust-lang.org/rustc/profile-guided-optimization.html) |
| フレームグラフ | [cargo-flamegraph](https://github.com/flamegraph-rs/flamegraph) |
| `cargo-deny` | [cargo-deny docs](https://embarkstudios.github.io/cargo-deny/) |
| `cargo-vet` | [cargo-vet docs](https://mozilla.github.io/cargo-vet/) |
| `cargo-audit` | [cargo-audit](https://github.com/rustsec/rustsec/tree/main/cargo-audit) |
| `cargo-bloat` | [cargo-bloat](https://github.com/RazrFalcon/cargo-bloat) |
| `cargo-udeps` | [cargo-udeps](https://github.com/est31/cargo-udeps) |
| `cargo-geiger` | [cargo-geiger](https://github.com/geiger-rs/cargo-geiger) |
| `cargo-semver-checks` | [cargo-semver-checks](https://github.com/obi1kenobi/cargo-semver-checks) |
| `cargo-nextest` | [nextest docs](https://nexte.st/) |
| `sccache` | [sccache](https://github.com/mozilla/sccache) |
| `mold` リンカ | [mold](https://github.com/rui314/mold) |
| `cargo-msrv` | [cargo-msrv](https://github.com/foresterre/cargo-msrv) |
| LTO | [rustc Codegen Options](https://doc.rust-lang.org/rustc/codegen-options/index.html) |
| Cargo プロファイル | [Cargo Book — Profiles](https://doc.rust-lang.org/cargo/reference/profiles.html) |
| `no_std` | [Rust Embedded Book](https://docs.rust-embedded.org/book/) |
| `windows-sys` クレート | [windows-rs](https://github.com/microsoft/windows-rs) |
| `cargo-xwin` | [cargo-xwin docs](https://github.com/rust-cross/cargo-xwin) |
| `cargo-hack` | [cargo-hack](https://github.com/taiki-e/cargo-hack) |
| `cargo-careful` | [cargo-careful](https://github.com/RalfJung/cargo-careful) |
| `cargo-watch` | [cargo-watch](https://github.com/watchexec/cargo-watch) |
| Rust CI キャッシュ | [Swatinem/rust-cache](https://github.com/Swatinem/rust-cache) |
| Rustdoc ブック | [Rustdoc Book](https://doc.rust-lang.org/rustdoc/) |
| 条件付きコンパイル | [Rust Reference — cfg](https://doc.rust-lang.org/reference/conditional-compilation.html) |
| 組み込み Rust | [Awesome Embedded Rust](https://github.com/rust-embedded/awesome-embedded-rust) |
| `hyperfine` | [hyperfine](https://github.com/sharkdp/hyperfine) |
| `proptest` | [proptest](https://github.com/proptest-rs/proptest) |
| `insta` | [insta snapshot testing](https://insta.rs/) |
| `cargo-fuzz` | [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz) |
| `loom` | [loom concurrency testing](https://github.com/tokio-rs/loom) |

---

*副読本リファレンスとして生成 — 『Rust のパターンと型駆動による正確性』のコンパニオンガイド。*

*バージョン 1.3 — 網羅性を高めるため、cargo-hack、cargo-careful、cargo-watch、cargo doc、再現可能なビルド、CI キャッシュ戦略、総合演習問題、および章依存関係図を追加。*
