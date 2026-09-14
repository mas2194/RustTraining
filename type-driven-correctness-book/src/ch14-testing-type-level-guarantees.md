# 型レベル保証のテスト 🟡

> **学習内容:** 不正なコードが*コンパイルに失敗する*ことをテストする方法（trybuild）、検証済み境界のファジング（proptest）、RAII 不変条件の検証、および `cargo-show-asm` によるゼロコスト抽象化の証明。
>
> **関連章:** [第3章](ch03-single-use-types-cryptographic-guarantee.md)（ノンスのコンパイル失敗テスト）、[第7章](ch07-validated-boundaries-parse-dont-validate.md)（境界の proptest）、[第5章](ch05-protocol-state-machines-type-state-for-r.md)（セッションの RAII）

## 型レベル保証のテスト

「正しさを構造的に保証する（Correct-by-construction）」パターンは、バグを実行時からコンパイル時にシフトさせます。しかし、不正なコードが実際にコンパイルに失敗することをどのように**テスト**すればよいのでしょうか？また、検証済み境界がファジングの下でも維持されることをどうやって保証するのでしょうか？本章では、型レベルの正しさを補完するテストツールについて解説します。

### `trybuild` によるコンパイル失敗テスト

[`trybuild`](https://crates.io/crates/trybuild) クレートを使用すると、特定のコードが**コンパイルできないこと**をアサートできます。これは、リファクタリング全体を通じて型レベルの不変条件を維持するために不可欠です。たとえば、誰かが誤って使い捨ての `Nonce` に `Clone` を追加してしまった場合でも、コンパイル失敗テストがそれを捕捉します。

**セットアップ:**

```toml
# Cargo.toml
[dev-dependencies]
trybuild = "1"
```

**テストファイル (`tests/compile_fail.rs`):**

```rust,ignore
#[test]
fn type_safety_tests() {
    let t = trybuild::TestCases::new();
    t.compile_fail("tests/ui/*.rs");
}
```

**テストケース: Nonce の再利用はコンパイルされてはならない (`tests/ui/nonce_reuse.rs`):**

```rust,ignore
// tests/ui/nonce_reuse.rs
use my_crate::Nonce;

fn main() {
    let nonce = Nonce::new();
    encrypt(nonce);
    encrypt(nonce); // 失敗すべき: ムーブされた値の使用
}

fn encrypt(_n: Nonce) {}
```

**期待されるエラー (`tests/ui/nonce_reuse.stderr`):**

```text
error[E0382]: use of moved value: `nonce`
 --> tests/ui/nonce_reuse.rs:6:13
  |
4 |     let nonce = Nonce::new();
  |         ----- move occurs because `nonce` has type `Nonce`, which does not implement the `Copy` trait
5 |     encrypt(nonce);
  |             ----- value moved here
6 |     encrypt(nonce); // should fail: use of moved value
  |             ^^^^^ value used here after move
```

**各章に応じたさらなるコンパイル失敗テストケース:**

| パターン（章） | テストのアサーション | ファイル |
|-------------------|---------------|------|
| 使い捨て Nonce（第3章） | Nonce を2回使用できない | `nonce_reuse.rs` |
| ケーパビリティトークン（第4章） | トークンなしで `admin_op()` を呼び出せない | `missing_token.rs` |
| 型状態（第5章） | `Session<Idle>` で `send_command()` を呼び出せない | `wrong_state.rs` |
| 次元（第6章） | `Celsius + Rpm` を加算できない | `unit_mismatch.rs` |
| シールドトレイト（裏技2） | 外部クレートはシールドトレイトを実装できない | `unseal_attempt.rs` |
| Non-Exhaustive（裏技3） | ワイルドカードなしの外部 match は失敗する | `missing_wildcard.rs` |

**CI への統合:**

```yaml
# .github/workflows/ci.yml
- name: コンパイル失敗テストの実行
  run: cargo test --test compile_fail
```

### 検証済み境界のプロパティベーステスト

検証済み境界（第7章）はデータを一度パースし、不正な入力を拒否します。しかし、自分の実装した検証が**すべて**の不正な入力を確実に捕捉しているとどうしてわかるでしょうか？[`proptest`](https://crates.io/crates/proptest) を用いたプロパティベーステストは、何千ものランダムな入力を生成して境界に負荷をかけます：

```toml
# Cargo.toml
[dev-dependencies]
proptest = "1"
```

```rust,ignore
use proptest::prelude::*;

/// 第7章より: ValidFru は仕様に準拠した FRU ペイロードをラップします。
/// これらのテストでは、board_area()、product_area()、format_version() メソッドを持つ
/// 第7章の完全な ValidFru を使用します。
/// 注: 第7章では TryFrom<RawFruData> を定義しているため、まず生バイトをラップします。

proptest! {
    /// 検証を通過した任意のバイト列は、パニックすることなく使用できなければならない。
    #[test]
    fn valid_fru_never_panics(data in proptest::collection::vec(any::<u8>(), 0..1024)) {
        if let Ok(fru) = ValidFru::try_from(RawFruData(data)) {
            // これらは検証済み FRU において決してパニックしてはならない
            // （第7章の ValidFru 実装のメソッド）:
            let _ = fru.format_version();
            let _ = fru.board_area();
            let _ = fru.product_area();
        }
    }

    /// ラウンドトリップ: format_version は再パースしても保持される。
    #[test]
    fn fru_round_trip(data in valid_fru_strategy()) {
        let raw = RawFruData(data.clone());
        let fru = ValidFru::try_from(raw).unwrap();
        let version = fru.format_version();
        // 同じバイト列を再パースする — バージョンは同一でなければならない
        let reparsed = ValidFru::try_from(RawFruData(data)).unwrap();
        prop_assert_eq!(version, reparsed.format_version());
    }
}

/// カスタムストラテジ: FRU 仕様のヘッダーを満たすバイトベクタを生成する。
/// ヘッダーフォーマットは第7章の `TryFrom<RawFruData>` 検証に準拠:
///   - バイト 0: version = 0x01
///   - バイト 1-6: エリアオフセット（×8 = 実際のバイトオフセット）
///   - バイト 7: チェックサム（バイト 0〜7 の合計 = 0 mod 256）
/// ボディはランダムだが、オフセットが範囲内に収まる十分な大きさを持つ。
fn valid_fru_strategy() -> impl Strategy<Value = Vec<u8>> {
    let header = vec![0x01, 0x00, 0x01, 0x02, 0x00, 0x00, 0x00];
    proptest::collection::vec(any::<u8>(), 64..256)
        .prop_map(move |body| {
            let mut fru = header.clone();
            let sum: u8 = fru.iter().fold(0u8, |a, &b| a.wrapping_add(b));
            fru.push(0u8.wrapping_sub(sum));
            fru.extend_from_slice(&body);
            fru
        })
}
```

**正しさを構造的に保証するコードのためのテストピラミッド:**

```text
┌───────────────────────────────────┐
│  コンパイル失敗テスト (trybuild)   │ ← 「不正なコードはコンパイルされてはならない」
├───────────────────────────────────┤
│ プロパティテスト (proptest/quickcheck) │ ← 「有効な入力が決してパニックしない」
├───────────────────────────────────┤
│    ユニットテスト (#[test])       │ ← 「特定の入力が期待される出力を生成する」
├───────────────────────────────────┤
│    型システム (第2〜13章のパターン) │ ← 「バグのクラス全体が存在し得ない」
└───────────────────────────────────┘
```

### RAII の検証

RAII（裏技12）はクリーンアップを保証します。これをテストするには、`Drop` の実装が実際に実行されることを検証します：

```rust,ignore
use std::sync::atomic::{AtomicBool, Ordering};

// 注: これらのテストはグローバルの AtomicBool を使用するため、互いに並列実行
// してはなりません。`#[serial_test::serial]` を使用するか、`cargo test -- --test-threads=1`
// で実行してください。あるいは、グローバルを完全に回避するためにクロージャ経由で
// テストごとの `Arc<AtomicBool>` を渡してください。
static DROPPED: AtomicBool = AtomicBool::new(false);

struct TestSession;
impl Drop for TestSession {
    fn drop(&mut self) {
        DROPPED.store(true, Ordering::SeqCst);
    }
}

#[test]
fn session_drops_on_early_return() {
    DROPPED.store(false, Ordering::SeqCst);
    let result: Result<(), &str> = (|| {
        let _session = TestSession;
        Err("simulated failure")?;
        Ok(())
    })();
    assert!(result.is_err());
    assert!(DROPPED.load(Ordering::SeqCst), "Drop must fire on early return");
}

#[test]
fn session_drops_on_panic() {
    DROPPED.store(false, Ordering::SeqCst);
    let result = std::panic::catch_unwind(|| {
        let _session = TestSession;
        panic!("simulated panic");
    });
    assert!(result.is_err());
    assert!(DROPPED.load(Ordering::SeqCst), "Drop must fire on panic");
}
```

### 実際のコードベースへの適用

ワークスペースに型レベルテストを追加するための優先順位付けされた計画は以下のとおりです：

| クレート | テスト種別 | テスト対象 |
|-------|-----------|-------------|
| `protocol_lib` | コンパイル失敗 | `Session<Idle>` は `send_command()` を呼び出せない |
| `protocol_lib` | プロパティ | 任意のバイト列 → `TryFrom` が成功するか Err を返す（パニックなし） |
| `thermal_diag` | コンパイル失敗 | `HasSpi` ミックスインなしでは `FanReading` を構築できない |
| `accel_diag` | プロパティ | GPU センサーのパース: ランダムバイト → 検証成功または拒否 |
| `config_loader` | プロパティ | ランダム文字列 → `DiagLevel` の `FromStr` が決してパニックしない |
| `pci_topology` | コンパイル失敗 | `Width32` が期待される場所に `Register<Width16>` を渡せない |
| `event_handler` | コンパイル失敗 | 監査トークンはクローンできない |
| `diag_framework` | コンパイル失敗 | `DerBuilder<Missing, _>` は `finish()` を呼び出せない |

### ゼロコスト抽象化: アセンブリによる証明

よくある懸念事項として、「ニュータイプや幽霊型（ファントム型）は実行時のオーバーヘッドを追加するのではないか？」というものがあります。
答えは **No** です — これらはプリミティブそのままの場合と完全に同一のアセンブリにコンパイルされます。
検証方法は以下のとおりです：

**セットアップ:**

```bash
cargo install cargo-show-asm
```

**例: ニュータイプ vs 生の u32:**

```rust,ignore
// src/lib.rs
#[derive(Clone, Copy)]
pub struct Rpm(pub u32);

#[derive(Clone, Copy)]
pub struct Celsius(pub f64);

// ニュータイプの算術
#[inline(never)]
pub fn add_rpm(a: Rpm, b: Rpm) -> Rpm {
    Rpm(a.0 + b.0)
}

// 生の算術（比較用）
#[inline(never)]
pub fn add_raw(a: u32, b: u32) -> u32 {
    a + b
}
```

**実行:**

```bash
cargo asm my_crate::add_rpm
cargo asm my_crate::add_raw
```

**結果 — 同一のアセンブリ:**

```asm
; add_rpm (ニュータイプ)      ; add_raw (生の u32)
my_crate::add_rpm:            my_crate::add_raw:
  lea eax, [rdi + rsi]         lea eax, [rdi + rsi]
  ret                          ret
```

`Rpm` ラッパーはコンパイル時に完全に消去されます。同じことが `PhantomData<S>`（0バイト）、`ZST`（ゼロサイズ型）トークン（0バイト）、および本ガイド全体で使用されているその他すべての型レベルマーカーにも当てはまります。

**独自の型について検証する:**

```bash
# 特定の関数のアセンブリを表示
cargo asm --lib ipmi_lib::session::execute

# PhantomData が 0 バイトを追加することを示す
cargo asm --lib --rust ipmi_lib::session::IpmiSession
```

> **重要なポイント:** 本ガイドで紹介したすべてのパターンは**実行時コストがゼロ**です。
> 型システムがすべての作業を行い、コンパイル中に完全に消去されます。
> Haskell の安全性と C のパフォーマンスを同時に手に入れることができます。

## 主なポイント

1. **trybuild は不正なコードがコンパイルできないことをテストする** — リファクタリング全体を通じて型レベルの不変条件を維持するために不可欠。
2. **proptest は検証境界をファジングする** — 何千ものランダム入力を生成して `TryFrom` 実装に負荷をかける。
3. **RAII 検証は Drop が実行されることをテストする** — Arc カウンタやモックフラグによってクリーンアップが行われたことを証明する。
4. **cargo-show-asm はゼロコストを証明する** — 幽霊型、ZST、ニュータイプは生の C と同じアセンブリを生成する。
5. **すべての「あり得ない」状態に対してコンパイル失敗テストを追加する** — 使い捨て型に対して誤って `Clone` を導出してしまった場合でも、テストがそれを捕捉する。

---

*『Rust における型駆動の正しさ（Type-Driven Correctness in Rust）』 完*
