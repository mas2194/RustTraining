# 実戦現場のノウハウ・裏技集 🟡

> **学べること:**
> - 単一の章には収まりきらない、実践で鍛え抜かれたパターン集
> - よくある落とし穴とその解決策 — CI のフレーキネスからバイナリ肥大化まで
> - 今すぐあらゆる Rust プロジェクトに適用できる即効性の高いテクニック
>
> **関連リンク:** 本書のすべての章 — ここに挙げるテクニックはすべてのトピックを横断します

本章では、本番環境の Rust コードベースで繰り返し遭遇するエンジニアリングパターンを集約しました。各 Tips は独立しているため、どの順番から読んでも構いません。

---

### 1. `deny(warnings)` の罠

**問題**: ソースコード内に `#![deny(warnings)]` を記述していると、Clippy が新しいリントを追加した際にビルドが壊れます — 昨日までコンパイルできていたコードが突然失敗するようになります。

**解決策**: ソースレベルの属性ではなく、CI で `CARGO_ENCODED_RUSTFLAGS` を使用します：

```yaml
# CI: ソースコードに手を加えずに警告をエラーとして扱う
env:
  CARGO_ENCODED_RUSTFLAGS: "-Dwarnings"
```

あるいは、より細かな制御のために `[workspace.lints]` を使用します：

```toml
# Cargo.toml
[workspace.lints.rust]
unsafe_code = "deny"

[workspace.lints.clippy]
all = { level = "deny", priority = -1 }
pedantic = { level = "warn", priority = -1 }
```

> 完全なパターンについては [コンパイル時ツール、ワークスペースのリント](ch08-compile-time-and-developer-tools.md) を参照してください。

---

### 2. 1 回のコンパイルで、すべてのテストを実行

**問題**: `cargo test` は `--lib`, `--doc`, `--test` の間で切り替えるたびに異なるプロファイルを使用するため、再コンパイルが発生します。

**解決策**: 単体テスト・統合テストには `cargo nextest` を使用し、ドキュメントテストは個別に実行します：

```bash
cargo nextest run --workspace        # 高速: 並列実行、キャッシュ
cargo test --workspace --doc         # ドキュメントテスト (nextest では実行不可)
```

> `cargo-nextest` のセットアップについては [コンパイル時ツール](ch08-compile-time-and-developer-tools.md) を参照してください。

---

### 3. フィーチャフラグの衛生管理

**問題**: ライブラリクレートで `default = ["std"]` としているのに、誰も `--no-default-features` をテストしていません。ある日組み込み環境のユーザーからコンパイルできないと報告されます。

**解決策**: CI に `cargo-hack` を追加します：

```yaml
- name: Feature matrix
  run: |
    cargo hack check --each-feature --no-dev-deps
    cargo check --no-default-features
    cargo check --all-features
```

> 完全なパターンについては [`no_std` とフィーチャの検証](ch09-no-std-and-feature-verification.md) を参照してください。

---

### 4. ロックファイルの議論 — コミットすべきか無視すべきか？

**目安:**

| クレートの種類 | `Cargo.lock` をコミットする？ | 理由 |
|----------------|------------------------------|------|
| バイナリ / アプリケーション | **はい** | ビルドの再現性を担保するため |
| ライブラリ | **いいえ** (`.gitignore` に追加) | 下流の利用者にバージョン選択を委ねるため |
| 両方を含むワークスペース | **はい** | バイナリ側の要求を優先するため |

ロックファイルが最新状態に保たれているかを確認する CI チェックを追加します：

```yaml
- name: Check lock file
  run: cargo update --locked  # Cargo.lock が古い場合に失敗する
```

---

### 5. 依存関係のみを最適化したデバッグビルド

**問題**: 依存関係（特に `serde` や `regex` など）が最適化されていないため、デバッグビルド時の実行速度が耐えられないほど遅い。

**解決策**: 迅速な再コンパイルのために自分のコードは未最適化のままにしつつ、dev プロファイルで依存関係のみを最適化します：

```toml
# Cargo.toml
[profile.dev.package."*"]
opt-level = 2  # dev モードですべての依存関係を最適化
```

これにより初回ビルドはわずかに遅くなりますが、開発中の実行速度は劇的に向上します。データベースを扱うサービスやパーサーにおいて特に効果的です。

> クレートごとのプロファイル上書きについては [リリースプロファイル](ch07-release-profiles-and-binary-size.md) を参照してください。

---

### 6. CI キャッシュのスラッシング

**問題**: `Swatinem/rust-cache@v2` が PR ごとに新しいキャッシュを保存するため、ストレージが圧迫され、リストア時間も遅くなる。

**解決策**: キャッシュの保存は `main` ブランチからのみ行い、リストアはすべてのブランチで利用可能にします：

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    save-if: ${{ github.ref == 'refs/heads/main' }}
```

複数のバイナリを持つワークスペースでは、`shared-key` を追加します：

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    shared-key: "ci-${{ matrix.target }}"
    save-if: ${{ github.ref == 'refs/heads/main' }}
```

> 完全なワークフローについては [CI/CD パイプライン](ch11-putting-it-all-together-a-production-cic.md) を参照してください。

---

### 7. `RUSTFLAGS` vs `CARGO_ENCODED_RUSTFLAGS`

**問題**: `RUSTFLAGS="-Dwarnings"` はビルドスクリプトやプロシージャルマクロを含む*すべて*に適用されます。`serde_derive` の build.rs 内に警告があると、CI が落ちてしまいます。

**解決策**: トップレベルのクレートにのみ適用される `CARGO_ENCODED_RUSTFLAGS` を使用します：

```bash
# BAD — サードパーティ製ビルドスクリプトの警告で失敗する
RUSTFLAGS="-Dwarnings" cargo build

# GOOD — 自身のクレートにのみ影響
CARGO_ENCODED_RUSTFLAGS="-Dwarnings" cargo build

# こちらも GOOD — ワークスペースのリント設定 (Cargo.toml)
[workspace.lints.rust]
warnings = "deny"
```

---

### 8. `SOURCE_DATE_EPOCH` による再現可能なビルド

**問題**: `build.rs` に `chrono::Utc::now()` を埋め込むとビルドの再現性が失われ、ビルドするたびに異なるバイナリハッシュが生成されてしまいます。

**解決策**: `SOURCE_DATE_EPOCH` に従います：

```rust
// build.rs
let timestamp = std::env::var("SOURCE_DATE_EPOCH")
    .ok()
    .and_then(|s| s.parse::<i64>().ok())
    .unwrap_or_else(|| chrono::Utc::now().timestamp());
println!("cargo:rustc-env=BUILD_TIMESTAMP={timestamp}");
```

> 完全な build.rs パターンについては [ビルドスクリプト](ch01-build-scripts-buildrs-in-depth.md) を参照してください。

---

### 9. `cargo tree` による重複排除ワークフロー

**問題**: `cargo tree --duplicates` を実行すると、5 種類のバージョンの `syn` や 3 種類の `tokio-util` が表示される。コンパイル時間が長大化してしまう。

**解決策**: 体系的な重複排除を実施します：

```bash
# ステップ 1: 重複を見つける
cargo tree --duplicates

# ステップ 2: 古いバージョンを引き込んでいるクレートを特定する
cargo tree --invert --package syn@1.0.109

# ステップ 3: 原因となっているクレートを更新する
cargo update -p serde_derive  # syn 2.x を引き込む可能性がある

# ステップ 4: アップデートが存在しない場合は [patch] で固定する
# [patch.crates-io]
# old-crate = { git = "...", branch = "syn2-migration" }

# ステップ 5: 検証する
cargo tree --duplicates  # 出力が短くなっているはず
```

> `cargo-deny` とサプライチェーンセキュリティについては [依存関係管理](ch06-dependency-management-and-supply-chain-s.md) を参照してください。

---

### 10. プッシュ前のスモークテスト

**問題**: コードをプッシュして CI に 10 分待たされた挙句、フォーマットの問題で失敗する。

**解決策**: プッシュ前にローカルで高速なチェックを実行します：

```toml
# Makefile.toml (cargo-make)
[tasks.pre-push]
description = "プッシュ前のローカルスモークテスト"
script = '''
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace --lib
'''
```

```bash
cargo make pre-push  # 30秒未満
git push
```

あるいは git pre-push フックを使用します：

```bash
#!/bin/sh
# .git/hooks/pre-push
cargo fmt --all -- --check && cargo clippy --workspace -- -D warnings
```

> `Makefile.toml` のパターンについては [CI/CD パイプライン](ch11-putting-it-all-together-a-production-cic.md) を参照してください。

---

### 🏋️ 演習問題

#### 🟢 演習 1: 3 つのノウハウを適用する

本章から 3 つのノウハウを選び、既存の Rust プロジェクトに適用してください。どのノウハウが最も大きな効果をもたらしましたか？

<details>
<summary>解答例</summary>

一般的に効果の高い組み合わせ：

1. **`[profile.dev.package."*"] opt-level = 2`** — 開発モードにおける実行速度が即座に改善（パース処理が多いコードでは 2〜10 倍高速化）

2. **`CARGO_ENCODED_RUSTFLAGS`** — サードパーティの警告による CI の誤検知エラーを根絶

3. **`cargo-hack --each-feature`** — 3 つ以上のフィーチャを持つプロジェクトでは、通常少なくとも 1 つの壊れた組み合わせが発見される

```bash
# ノウハウ 5 を適用:
echo '[profile.dev.package."*"]' >> Cargo.toml
echo 'opt-level = 2' >> Cargo.toml

# CI でノウハウ 7 を適用:
# RUSTFLAGS を CARGO_ENCODED_RUSTFLAGS に置き換える

# ノウハウ 3 を適用:
cargo install cargo-hack
cargo hack check --each-feature --no-dev-deps
```
</details>

#### 🟡 演習 2: 依存関係ツリーの重複を排除する

実際のプロジェクトで `cargo tree --duplicates` を実行してください。少なくとも 1 つの重複を解消します。適用前後のコンパイル時間を測定してください。

<details>
<summary>解答例</summary>

```bash
# 適用前
time cargo build --release 2>&1 | tail -1
cargo tree --duplicates | wc -l  # 重複行数をカウント

# 1 つの重複を見つけて修正
cargo tree --duplicates
cargo tree --invert --package <duplicate-crate>@<old-version>
cargo update -p <parent-crate>

# 適用後
time cargo build --release 2>&1 | tail -1
cargo tree --duplicates | wc -l  # 行数が減っているはず

# 一般的な結果: 重複を 1 つ解消するごとにコンパイル時間が 5〜15% 短縮
# （特に syn や tokio などの重量級クレートで顕著）
```
</details>

### 重要ポイント

- サードパーティのビルドスクリプトを壊さないよう、`RUSTFLAGS` ではなく `CARGO_ENCODED_RUSTFLAGS` を使用する。
- `[profile.dev.package."*"] opt-level = 2` は、開発者体験を向上させる最も効果的な単一の設定。
- キャッシュのチューニング（main でのみ `save-if`）により、アクティブなリポジトリでの CI キャッシュ肥大化を防ぐ。
- `cargo tree --duplicates` + `cargo update` はコストゼロでコンパイル時間を短縮できる — 毎月実行しましょう。
- `cargo make pre-push` でローカルで高速なチェックを実行し、CI の無駄な往復を回避する。

---
