# 実践から生まれた14のテクニック 🟡

> **学べること:** 番兵値の排除やシールドトレイトから、セッション型、`Pin`、RAII、`#[must_use]` に至るまで、構造的に正しくする14の小さなテクニック。それぞれがほぼゼロの手間で特定のバグ分類を排除します。
>
> **相互参照:** [第2章](ch02-typed-command-interfaces-request-determi.md)（シールドトレイトは第2章を拡張）、[第5章](ch05-protocol-state-machines-type-state-for-r.md)（型状態ビルダーは第5章を拡張）、[第7章](ch07-validated-boundaries-parse-dont-validate.md)（FromStr は第7章を拡張）

## 実践から生まれた14のテクニック

第2章から第9章で扱った8つのコアパターンは、主要な「構造的に正しい（correct-by-construction）」設計手法を網羅しています。本章では、本番のRustコードで頻繁に登場する、**小規模ながら価値の高い14のテクニック**を集めました。これらはそれぞれ、ゼロまたはほぼゼロの手間で特定のバグの分類を排除します。

### トリック1 — 境界で番兵値（センチネル）を `Option` に変換する

ハードウェアプロトコルには番兵値（sentinel values）が溢れています。IPMIでは「センサーが存在しない」ことを表すために `0xFF` を、PCIでは「デバイスが存在しない」ことを表すために `0xFFFF` を、SMBIOSでは「不明」を表すために `0x00` を使用します。これらの番兵値を通常の整数としてコード内で引き回すと、利用側のすべての箇所でマジックナンバーのチェックを忘れずに行う必要があります。もし1箇所でも比較を忘れると、幻の255 °Cという読み取り値が得られたり、誤ったベンダーIDの一致が発生したりします。

**原則:** 最初のパース境界で直ちに番兵値を `Option` に変換し、番兵値への逆変換はシリアライズ境界でのみ行います。

#### アンチパターン（`pcie_tree/src/lspci.rs` より）

```rust,ignore
// 番兵値が内部に持ち越されている — すべての比較でチェックを意識する必要がある
let mut current_vendor_id: u16 = 0xFFFF;
let mut current_device_id: u16 = 0xFFFF;

// ... その後、パースがサイレントに失敗する ...
current_vendor_id = u16::from_str_radix(hex, 16)
    .unwrap_or(0xFFFF);  // 番兵値がエラーを隠蔽してしまう
```

`current_vendor_id` を受け取るすべての関数は、`0xFFFF` が特別であることを知っていなければなりません。もし誰かが `0xFFFF` を事前にチェックせずに `if vendor_id == target_id` と書いてしまうと、ターゲット側も不正な入力から `0xFFFF` としてパースされていた場合に、存在しないデバイスがサイレントに一致してしまいます。

#### 正しいパターン（`nic_sel/src/events.rs` より）

```rust,ignore
pub struct ThermalEvent {
    pub record_id: u16,
    pub temperature: Option<u8>,  // センサーが 0xFF を報告した場合は None
}

impl ThermalEvent {
    pub fn from_raw(record_id: u16, raw_temp: u8) -> Self {
        ThermalEvent {
            record_id,
            temperature: if raw_temp != 0xFF {
                Some(raw_temp)
            } else {
                None
            },
        }
    }
}
```

これで、利用側は必ず `None` のケースを処理しなければならなくなります — コンパイラがそれを強制します：

```rust,ignore
// 安全 — 欠損温度を処理することがコンパイラによって保証される
fn is_overtemp(temp: Option<u8>, threshold: u8) -> bool {
    temp.map_or(false, |t| t > threshold)
}

// None の処理を忘れるとコンパイルエラーになる:
// fn bad_check(temp: Option<u8>, threshold: u8) -> bool {
//     temp > threshold  // エラー: Option<u8> と u8 は比較できない
// }
```

#### 実世界への影響

`inventory/src/events.rs` でも、GPU温度アラートに同じパターンを使用しています：
```rust,ignore
temperature: if data[1] != 0xFF {
    Some(data[1] as i8)
} else {
    None
},
```

`pcie_tree/src/lspci.rs` のリファクタリングは非常にシンプルです：`current_vendor_id: u16` を `current_vendor_id: Option<u16>` に変更し、`0xFFFF` を `None` に置き換え、更新が必要なすべての箇所をコンパイラに検出させます。

| 変更前 | 変更後 |
|--------|-------|
| `let mut vendor_id: u16 = 0xFFFF` | `let mut vendor_id: Option<u16> = None` |
| `.unwrap_or(0xFFFF)` | `.ok()`（すでに `Option` を返す） |
| `if vendor_id != 0xFFFF { ... }` | `if let Some(vid) = vendor_id { ... }` |
| シリアライズ: `vendor_id` | `vendor_id.unwrap_or(0xFFFF)` |

***

### トリック2 — シールドトレイト（Sealed Traits）

第2章では、各コマンドをレスポンスにバインドする関連型を持つ `IpmiCmd` を紹介しました。しかし、ここには抜け穴があります。もし*任意の*コードが `IpmiCmd` を実装できるとしたら、`parse_response` が誤った型を返したりパニックを起こしたりするような `MaliciousCmd` を誰かが書けてしまいます。システム全体の型安全性は、すべての実装が正しいことに依存しています。

**シールドトレイト（Sealed Trait）** はこの抜け穴を塞ぎます。アイデアはシンプルです。自分たちのクレートだけが実装できる*プライベート*なスーパートレイトを、パブリックなトレイトの必須要件にするのです。

```rust,ignore
// — プライベートモジュール: クレート外にはエクスポートされない —
mod private {
    pub trait Sealed {}
}

// — パブリックトレイト: 外部からは実装できない Sealed を要求 —
pub trait IpmiCmd: private::Sealed {
    type Response;
    fn net_fn(&self) -> u8;
    fn cmd_byte(&self) -> u8;
    fn payload(&self) -> Vec<u8>;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Self::Response>;
}
```

クレート内では、承認された各コマンド型に対して `Sealed` を実装します：

```rust,ignore
pub struct ReadTemp { pub sensor_id: u8 }
impl private::Sealed for ReadTemp {}

impl IpmiCmd for ReadTemp {
    type Response = Celsius;
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.sensor_id] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Celsius> {
        if raw.is_empty() { return Err(io::Error::new(io::ErrorKind::InvalidData, "empty")); }
        Ok(Celsius(raw[0] as f64))
    }
}
```

外部コードからは `IpmiCmd` が見え、`execute()` を呼び出すことはできますが、それを実装することはできません：

```rust,ignore
// 別のクレート内:
struct EvilCmd;
// impl private::Sealed for EvilCmd {}  // エラー: モジュール `private` は非公開
// impl IpmiCmd for EvilCmd { ... }     // エラー: `Sealed` が満たされていない
```

#### シールドすべき場合とすべきでない場合

| シールドすべき場合… | シールドすべきでない場合… |
|-----------|-----------------|
| 安全性が正しい実装に依存している場合（`IpmiCmd`, `DiagModule`） | ユーザーによるシステムの拡張を許容する場合（カスタムレポートフォーマッタ） |
| 関連型が不変条件を満たす必要がある場合 | 単純なケイパビリティマーカートレイトである場合（`HasIpmi`） |
| 正統な実装の集合を自身が管理している場合 | サードパーティ製プラグインの提供が設計目標である場合 |

#### 実世界での適用候補

- `IpmiCmd` — 不正なパースにより型付きレスポンスが破損する可能性がある
- `DiagModule` — フレームワークが `run()` の返す有効なDERレコードを前提としている
- `SelEventFilter` — 壊れたフィルタが重要なSELイベントを握りつぶす可能性がある

***

### トリック3 — 将来拡張される列挙型のための `#[non_exhaustive]`

現在、`inventory/src/types.rs` の `SkuVariant` には5つのバリアントがあります：

```rust,ignore
pub enum SkuVariant {
    S1001, S2001, S2002, S2003, S3001,
}
```

次世代製品が出荷され、`S4001` を追加したとします。このとき、`SkuVariant` を `match` しておりワイルドカード（`_`）アームを持たない外部コードは、**警告なしにコンパイルエラー**になります — これは意図通りの動作です。しかし、内部コードはどうでしょうか？`#[non_exhaustive]` がなければ、*同一クレート*内の `match` もワイルドカードなしでコンパイルでき、新しいバリアントを追加した際に自分自身のビルドが壊れてしまいます。

列挙型に `#[non_exhaustive]` を付けると、その列挙型を `match` する**外部クレート**に対してワイルドカードアームの記述を強制できます。定義元のクレート内では、`#[non_exhaustive]` は影響を持たず、これまで通り網羅的な（exhaustive）パターンマッチを書くことができます。

**なぜこれが役立つのか:** ライブラリクレート（またはワークスペース内の共有サブクレート）から `SkuVariant` を公開する場合、ダウンストリームのコードに未知の将来のバリアントを処理させることができます。次の世代で `S4001` を追加した際も、ダウンストリームのコードには既にワイルドカードアームがあるため、そのままコンパイルが通ります。

```rust,ignore
// gpu_sel クレート内（定義元クレート）:
#[non_exhaustive]
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum SkuVariant {
    S1001,
    S2001,
    S2002,
    S2003,
    S3001,
    // 次のSKUが出荷されたらここに追加する。
    // 外部の利用側にはすでにワイルドカードがあるため、そちらのコードが壊れることはない。
}

// gpu_sel 内部 — 網羅的マッチが許可される（ワイルドカードは不要）:
fn diag_path_internal(sku: SkuVariant) -> &'static str {
    match sku {
        SkuVariant::S1001 => "legacy_gen1",
        SkuVariant::S2001 => "gen2_accel_diag",
        SkuVariant::S2002 => "gen2_alt_diag",
        SkuVariant::S2003 => "gen2_alt_hf_diag",
        SkuVariant::S3001 => "gen3_accel_diag",
        // 定義元クレート内部ではワイルドカードは不要。
        // ここに S4001 を追加するとこの match でコンパイルエラーが発生するが、
        // それは更新漏れを防ぐためのまさに望ましい挙動である。
    }
}
```

```rust,ignore
// バイナリクレート内（inventory に依存するダウンストリームクレート）:
fn diag_path_external(sku: inventory::SkuVariant) -> &'static str {
    match sku {
        inventory::SkuVariant::S1001 => "legacy_gen1",
        inventory::SkuVariant::S2001 => "gen2_accel_diag",
        inventory::SkuVariant::S2002 => "gen2_alt_diag",
        inventory::SkuVariant::S2003 => "gen2_alt_hf_diag",
        inventory::SkuVariant::S3001 => "gen3_accel_diag",
        _ => "generic_diag",  // 外部クレートでは #[non_exhaustive] により必須
    }
}
```

> **ワークスペースでのヒント:** すべてのコードが単一のクレート内にある場合、`#[non_exhaustive]` は効果を発揮しません — クレート境界を越える場合にのみ作用します。プロジェクトの大規模なワークスペースでは、将来変化する列挙型を共有クレート（`core_lib` や `inventory` など）に配置することで、他のワークスペースクレートの利用側を保護できます。

#### 適用候補

| 列挙型 | モジュール | 理由 |
|------|--------|-----|
| `SkuVariant` | `inventory`, `net_inventory` | 世代ごとに新しいSKUが追加される |
| `SensorType` | `protocol_lib` | IPMI仕様で 0xC0〜0xFF がOEM用に予約されている |
| `CompletionCode` | `protocol_lib` | カスタムBMCベンダーがコードを追加する |
| `Component` | `event_handler` | 新しいハードウェアカテゴリ（最近 NewSoC が追加された） |

***

### トリック4 — 型状態ビルダー（Typestate Builder）

第5章では、*プロトコル*（セッションのライフサイクル、リンクトレーニング）に対する型状態を示しました。同じ考え方は*ビルダー*にも適用できます — 必須フィールドがすべて設定されたときのみ `build()` / `finish()` を呼び出せるようにする構造体です。

#### 流れるようなビルダー（Fluent Builder）の問題点

現在、`diag_framework/src/der.rs` の `DerBuilder` は以下のようになっています（簡略版）：

```rust,ignore
// 現在の流れるようなビルダー — finish() が常に呼び出し可能
pub struct DerBuilder {
    der: Der,
}

impl DerBuilder {
    pub fn new(marker: &str, fault_code: u32) -> Self { ... }
    pub fn mnemonic(mut self, m: &str) -> Self { ... }
    pub fn fault_class(mut self, fc: &str) -> Self { ... }
    pub fn finish(self) -> Der { self.der }  // ← 常に呼び出し可能！
}
```

これはエラーなくコンパイルされますが、不完全なDERレコードを生成してしまいます：

```rust,ignore
let bad = DerBuilder::new("CSI_ERR", 62691)
    .finish();  // 不正 — mnemonic も fault_class も設定されていない
```

#### 型状態ビルダー: `finish()` は両方のフィールドを要求する

```rust,ignore
pub struct Missing;
pub struct Set<T>(T);

pub struct DerBuilder<Mnemonic, FaultClass> {
    marker: String,
    fault_code: u32,
    mnemonic: Mnemonic,
    fault_class: FaultClass,
    description: Option<String>,
}

// コンストラクタ: 両方の必須フィールドが Missing の状態で開始
impl DerBuilder<Missing, Missing> {
    pub fn new(marker: &str, fault_code: u32) -> Self {
        DerBuilder {
            marker: marker.to_string(),
            fault_code,
            mnemonic: Missing,
            fault_class: Missing,
            description: None,
        }
    }
}

// mnemonic の設定（fault_class の状態に関係なく動作）
impl<FC> DerBuilder<Missing, FC> {
    pub fn mnemonic(self, m: &str) -> DerBuilder<Set<String>, FC> {
        DerBuilder {
            marker: self.marker, fault_code: self.fault_code,
            mnemonic: Set(m.to_string()),
            fault_class: self.fault_class,
            description: self.description,
        }
    }
}

// fault_class の設定（mnemonic の状態に関係なく動作）
impl<MN> DerBuilder<MN, Missing> {
    pub fn fault_class(self, fc: &str) -> DerBuilder<MN, Set<String>> {
        DerBuilder {
            marker: self.marker, fault_code: self.fault_code,
            mnemonic: self.mnemonic,
            fault_class: Set(fc.to_string()),
            description: self.description,
        }
    }
}

// オプショナルフィールド — どの状態でも利用可能
impl<MN, FC> DerBuilder<MN, FC> {
    pub fn description(mut self, desc: &str) -> Self {
        self.description = Some(desc.to_string());
        self
    }
}

/// 完全に構築された DER レコード。
pub struct Der {
    pub marker: String,
    pub fault_code: u32,
    pub mnemonic: String,
    pub fault_class: String,
    pub description: Option<String>,
}

// finish() は両方の必須フィールドが Set のときのみ利用可能
impl DerBuilder<Set<String>, Set<String>> {
    pub fn finish(self) -> Der {
        Der {
            marker: self.marker,
            fault_code: self.fault_code,
            mnemonic: self.mnemonic.0,
            fault_class: self.fault_class.0,
            description: self.description,
        }
    }
}
```

これで、バグのある呼び出しはコンパイルエラーになります：

```rust,ignore
// ✅ コンパイル成功 — 両方の必須フィールドが設定されている（順序は任意）
let der = DerBuilder::new("CSI_ERR", 62691)
    .fault_class("GPU Module")   // 呼び出し順序は問わない
    .mnemonic("ACCEL_CARD_ER691")
    .description("Thermal throttle")
    .finish();

// ❌ コンパイルエラー — DerBuilder<Set<String>, Missing> に finish() は存在しない
let bad = DerBuilder::new("CSI_ERR", 62691)
    .mnemonic("ACCEL_CARD_ER691")
    .finish();  // エラー: メソッド `finish` が見つからない
```

#### 型状態ビルダーを使用すべき場合

| 使用すべき場合… | 使用しなくてよい場合… |
|-----------|-------------------|
| フィールドの欠落がサイレントなバグになる場合（DERの mnemonic 欠落など） | すべてのフィールドに適切なデフォルト値がある場合 |
| ビルダーがパブリックAPIの一部である場合 | ビルダーがテスト専用の足場である場合 |
| 必須フィールドが2〜3個以上ある場合 | 必須フィールドが1つだけの場合（`new()` で受け取ればよい） |

***

### トリック5 — バリデーション境界としての `FromStr`

第7章ではバイナリデータ（FRUレコード、SELエントリ）に対する `TryFrom<&[u8]>` を扱いました。**文字列**入力（設定ファイル、CLI引数、JSONフィールド）に対して、これに相当する境界が `FromStr` です。

#### 問題点

```rust,ignore
// C++ / 検証されていないRust: サイレントにデフォルト値へフォールスルーする
fn route_diag(level: &str) -> DiagMode {
    if level == "quick" { ... }
    else if level == "standard" { ... }
    else { QuickMode }  // 設定ファイルにタイポがあっても気づかない
}
```

設定ファイルに `"diag_level": "extendedd"`（タイポ）とあっても、サイレントに `QuickMode` が選択されてしまいます。

#### パターン（`config_loader/src/diag.rs` より）

```rust,ignore
use std::str::FromStr;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum DiagLevel {
    Quick,
    Standard,
    Extended,
    Stress,
}

impl FromStr for DiagLevel {
    type Err = String;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        match s.to_lowercase().as_str() {
            "quick"    | "1" => Ok(DiagLevel::Quick),
            "standard" | "2" => Ok(DiagLevel::Standard),
            "extended" | "3" => Ok(DiagLevel::Extended),
            "stress"   | "4" => Ok(DiagLevel::Stress),
            other => Err(format!("unknown diag level: '{other}'")),
        }
    }
}
```

これで、タイポは即座に検出されます：

```rust,ignore
let level: DiagLevel = "extendedd".parse()?;
// Err("unknown diag level: 'extendedd'")
```

#### 3つのメリット

1. **フェイルファスト（早期失敗）:** 不正な入力は、診断ロジックの3層奥深くではなく、パース境界で即座に捕捉されます。
2. **エイリアスが明示的:** `"MEM"`、`"DIMM"`、`"MEMORY"` はすべて `Component::Memory` にマップされ、match アームがそのマッピング仕様として機能します。
3. **`.parse()` が人間工学的:** `FromStr` は `str::parse()` と統合されているため、`let level: DiagLevel = config["level"].parse()?;` のように簡潔な1行で書けます。

#### 実コードベースでの使用例

プロジェクトにはすでに8つの `FromStr` 実装があります：

| 型 | モジュール | 主なエイリアス |
|------|--------|----------------|
| `DiagLevel` | `config_loader` | `"1"` = Quick, `"4"` = Stress |
| `Component` | `event_handler` | `"MEM"` / `"DIMM"` = Memory, `"SSD"` / `"NVME"` = Disk |
| `SkuVariant` | `net_inventory` | `"Accel-X1"` = S2001, `"Accel-M1"` = S2002, `"Accel-Z1"` = S3001 |
| `SkuVariant` | `inventory` | 同様のエイリアス（別モジュール、同一パターン） |
| `FaultStatus` | `config_loader` | 障害のライフサイクル状態 |
| `DiagAction` | `config_loader` | 復旧アクションの型 |
| `ActionType` | `config_loader` | アクションのカテゴリ |
| `DiagMode` | `cluster_diag` | マルチノードテストモード |

`TryFrom` との対比：

| | `TryFrom<&[u8]>` | `FromStr` |
|---|---|---|
| 入力 | 生バイト列（バイナリプロトコル） | 文字列（設定、CLI、JSON） |
| 典型的なソース | IPMI、PCIeコンフィグ空間、FRU | JSONフィールド、環境変数、ユーザー入力 |
| 該当章 | 第7章 | 第11章 |
| 共通点 | 呼び出し元に不正な入力の処理を強制する `Result` を使用 |

***

### トリック6 — コンパイル時のサイズ検証のための const ジェネリクス

ハードウェアバッファ、レジスタバンク、プロトコルフレームが固定サイズを持つ場合、const ジェネリクスを使用することでコンパイラにそのサイズを強制させることができます：

```rust,ignore
/// 固定サイズのレジスタバンク。サイズは型の一部となる。
/// `RegisterBank<256>` と `RegisterBank<4096>` は異なる型である。
pub struct RegisterBank<const N: usize> {
    data: [u8; N],
}

impl<const N: usize> RegisterBank<N> {
    /// 指定されたオフセットのレジスタを読み出す。
    /// コンパイル時: N が既知であるため配列サイズは固定。
    /// ランタイム時: オフセットのみがチェックされる。
    pub fn read(&self, offset: usize) -> Option<u8> {
        self.data.get(offset).copied()
    }
}

// PCIe 従来型コンフィグ空間: 256 バイト
type PciConfigSpace = RegisterBank<256>;

// PCIe 拡張コンフィグ空間: 4096 バイト
type PcieExtConfigSpace = RegisterBank<4096>;

// これらは異なる型であるため、誤って一方を他方に渡すことはできない:
fn read_extended_cap(config: &PcieExtConfigSpace, offset: usize) -> Option<u8> {
    config.read(offset)
}
// read_extended_cap(&pci_config, 0x100);
//                   ^^^^^^^^^^^ expected RegisterBank<4096>, found RegisterBank<256> ❌
```

**const ジェネリクスによるコンパイル時アサーション:**

```rust,ignore
/// NVMe 管理コマンドは 4096 バイトのバッファを使用する。コンパイル時に強制する。
pub struct NvmeBuffer<const N: usize> {
    data: Box<[u8; N]>,
}

impl<const N: usize> NvmeBuffer<N> {
    pub fn new() -> Self {
        // ランタイムアサーション: 512 または 4096 のみ許可
        assert!(N == 4096 || N == 512, "NVMe buffers must be 512 or 4096 bytes");
        NvmeBuffer { data: Box::new([0u8; N]) }
    }
}
// NvmeBuffer::<1024>::new();  // この形式では実行時にパニック
// 真のコンパイル時強制については、トリック9（const アサーション）を参照。
```

> **使い所:** 固定サイズのプロトコルバッファ（NVMe、PCIeコンフィグ空間）、DMAディスクリプタ、ハードウェアFIFOの深さなど。サイズがハードウェア定数であり、実行時に決して変動すべきでないすべての場所。

***

### トリック7 — `unsafe` に対する安全なラッパー

プロジェクトには現在 `unsafe` ブロックがまったく存在しません。しかし、MMIOレジスタアクセス、DMA、または accel-mgmt/accel-query へのFFIを追加する際には、`unsafe` が必要になります。構造的に正しいアプローチとは、**すべての `unsafe` ブロックを安全な抽象化の中にカプセル化する**ことで、危険性を閉じ込め、監査可能に保つことです。

```rust,ignore
/// MMIOマップされたレジスタ。ポインタはこのマッピングのライフタイムの間有効である。
/// すべての unsafe はこのモジュール内に封入されており、呼び出し元は安全なメソッドを使用する。
pub struct MmioRegion {
    base: *mut u8,
    len: usize,
}

impl MmioRegion {
    /// # Safety
    /// - `base` は MMIO マップされた領域への有効なポインタでなければならない
    /// - この領域は、この構造体のライフタイムの間マップされたままでなければならない
    /// - 他のコードがこの領域にエイリアスしてはならない
    pub unsafe fn new(base: *mut u8, len: usize) -> Self {
        MmioRegion { base, len }
    }

    /// 安全な読み出し — 境界チェックにより範囲外 MMIO アクセスを防止。
    pub fn read_u32(&self, offset: usize) -> Option<u32> {
        if offset + 4 > self.len { return None; }
        // SAFETY: offset は上記で境界チェックされており、base は new() の契約により有効
        Some(unsafe {
            core::ptr::read_volatile(self.base.add(offset) as *const u32)
        })
    }

    /// 安全な書き込み — 境界チェックにより範囲外 MMIO アクセスを防止。
    pub fn write_u32(&self, offset: usize, value: u32) -> bool {
        if offset + 4 > self.len { return false; }
        // SAFETY: offset は上記で境界チェックされており、base は new() の契約により有効
        unsafe {
            core::ptr::write_volatile(self.base.add(offset) as *mut u32, value);
        }
        true
    }
}
```

**型付きMMIOのための幽霊型（第9章）との組み合わせ:**

```rust,ignore
use std::marker::PhantomData;

pub struct ReadOnly;
pub struct ReadWrite;

pub struct TypedMmio<Perm> {
    region: MmioRegion,
    _perm: PhantomData<Perm>,
}

impl TypedMmio<ReadOnly> {
    pub fn read_u32(&self, offset: usize) -> Option<u32> {
        self.region.read_u32(offset)
    }
    // write メソッドは存在しない — ReadOnly な領域に書き込もうとするとコンパイルエラー
}

impl TypedMmio<ReadWrite> {
    pub fn read_u32(&self, offset: usize) -> Option<u32> {
        self.region.read_u32(offset)
    }
    pub fn write_u32(&self, offset: usize, value: u32) -> bool {
        self.region.write_u32(offset, value)
    }
}
```

> **`unsafe` ラッパーのガイドライン:**
>
> | ルール | 理由 |
> |------|-----|
> | ドキュメント化された `# Safety` 不変条件を持つ単一の `unsafe fn new()` | 呼び出し元が一度だけ責任を負う |
> | 他のすべてのメソッドは safe にする | 呼び出し元が未定義動作（UB）を引き起こせないようにする |
> | すべての `unsafe` ブロックに `# SAFETY:` コメントを記載 | 監査者が局所的に検証できるようにする |
> | `#[deny(unsafe_op_in_unsafe_fn)]` を持つモジュールで包む | `unsafe fn` の内部であっても、個々の操作に `unsafe` ブロックを明示させる |
> | ラッパーに対して `cargo +nightly miri test` を実行する | メモリモデルへの準拠を検証する |

---

### ✅ チェックポイント: トリック1〜7

日常的に使える7つのテクニックが揃いました。簡単なスコアカードです：

| トリック | 排除されるバグの分類 | 導入コスト |
|:-----:|----------------------|:---------------:|
| 1 | 番兵値の混同（0xFF） | 低 — 境界での単一の `match` |
| 2 | 不正なトレイト実装 | 低 — `Sealed` スーパートレイトを追加 |
| 3 | enum の拡張による利用側の破損 | 低 — 1行のアトリビュート |
| 4 | ビルダーのフィールド設定漏れ | 中 — 型パラメータの追加 |
| 5 | 文字列設定のタイポ | 低 — `impl FromStr` |
| 6 | バッファサイズの誤り | 低 — const ジェネリクスパラメータ |
| 7 | コードベース全体への unsafe の散乱 | 中 — ラッパーモジュール |

トリック8〜14は**より高度な内容**です — 非同期（async）、const 評価、セッション型、`Pin`、そして `Drop` を扱います。必要に応じてここで一息入れてください。上記のテクニックだけでも、明日からすぐに導入できる価値の高い成果が得られます。

***

### トリック8 — 非同期の型状態機械（Async Type-State Machines）

ハードウェアドライバが `async` を使用する場合（例: 非同期BMC通信、非同期NVMe I/O）、型状態パターンは引き続き機能します — ただし、`.await` ポイントをまたぐ所有権の扱いに注意が必要です：

```rust,ignore
use std::marker::PhantomData;

pub struct Idle;
pub struct Authenticating;
pub struct Active;

pub struct AsyncSession<S> {
    host: String,
    _state: PhantomData<S>,
}

impl AsyncSession<Idle> {
    pub fn new(host: &str) -> Self {
        AsyncSession { host: host.to_string(), _state: PhantomData }
    }

    /// Idle → Authenticating → Active の遷移。
    /// Session は .await をまたいで消費（Future にムーブ）される。
    pub async fn authenticate(self, user: &str, pass: &str)
        -> Result<AsyncSession<Active>, String>
    {
        // フェーズ1: 認証情報を送信（Idle セッションを消費）
        let pending: AsyncSession<Authenticating> = AsyncSession {
            host: self.host,
            _state: PhantomData,
        };

        // 非同期 BMC 認証をシミュレート
        // tokio::time::sleep(Duration::from_secs(1)).await;

        // フェーズ2: Active セッションを返却
        Ok(AsyncSession {
            host: pending.host,
            _state: PhantomData,
        })
    }
}

impl AsyncSession<Active> {
    pub async fn send_command(&mut self, cmd: &[u8]) -> Vec<u8> {
        // 非同期 I/O を実行...
        vec![0x00]
    }
}

// 使い方:
// let session = AsyncSession::new("192.168.1.100");
// let mut session = session.authenticate("admin", "pass").await?;
// let resp = session.send_command(&[0x04, 0x2D]).await;
```

**非同期型状態の重要ルール:**

| ルール | 理由 |
|------|-----|
| 状態遷移メソッドは `&mut self` ではなく `self`（値渡し）を受け取る | `.await` をまたいで所有権の移転が機能するようにする |
| 回復可能なエラーには `Result<NextState, (Error, PrevState)>` を返す | 呼び出し元が以前の状態からリトライできるようにする |
| 状態を複数の Future に分割しない | 1つの Future が1つのセッションを所有する |
| `tokio::spawn` を使用する場合は `Send + 'static` 境界を設ける | セッションがスレッド間を移動できるようにする |

> **注意点:** エラー時にリトライするために*前の*状態を取り戻す必要がある場合は、呼び出し元が所有権を取り戻せるように `Result<AsyncSession<Active>, (Error, AsyncSession<Idle>)>` を返してください。これを行わないと、失敗した `.await` によってセッションが完全に破棄されてしまいます。

***

### トリック9 — const アサーションによる篩（リファインメント）型

数値の制約がランタイムデータではなくコンパイル時の不変条件である場合、それを強制するために `const` 評価を使用します。これはトリック6（型レベルのサイズ区別を提供）とは異なり、ここではコンパイル時に*無効な値を拒絶*します：

```rust,ignore
/// IPMI SDR の範囲（0x01..=0xFE）内でなければならないセンサー ID。
/// `N` が const である場合、制約はコンパイル時にチェックされる。
pub struct SdrSensorId<const N: u8>;

impl<const N: u8> SdrSensorId<N> {
    /// コンパイル時バリデーション: N が範囲外の場合、コンパイル中にパニックする。
    pub const fn validate() {
        assert!(N >= 0x01, "Sensor ID must be >= 0x01");
        assert!(N <= 0xFE, "Sensor ID must be <= 0xFE (0xFF is reserved)");
    }

    pub const VALIDATED: () = Self::validate();

    pub const fn value() -> u8 { N }
}

// 使い方:
fn read_sensor_const<const N: u8>() -> f64 {
    let _ = SdrSensorId::<N>::VALIDATED;  // コンパイル時チェック
    // センサー N を読み出す...
    42.0
}

// read_sensor_const::<0x20>();   // ✅ コンパイル成功 — 0x20 は有効
// read_sensor_const::<0x00>();   // ❌ コンパイルエラー — "Sensor ID must be >= 0x01"
// read_sensor_const::<0xFF>();   // ❌ コンパイルエラー — 0xFF は予約済み
```

**よりシンプルな形式 — 範囲制限付きファンID:**

```rust,ignore
pub struct BoundedFanId<const N: u8>;

impl<const N: u8> BoundedFanId<N> {
    pub const VALIDATED: () = assert!(N < 8, "Server has at most 8 fans (0..7)");

    pub const fn id() -> u8 {
        let _ = Self::VALIDATED;
        N
    }
}

// BoundedFanId::<3>::id();   // ✅
// BoundedFanId::<10>::id();  // ❌ コンパイルエラー
```

> **使い所:** コンパイル時に判明しているハードウェア定義の固定ID（センサーID、ファンスロット、PCIeスロット番号など）。値がランタイムデータ（設定ファイル、ユーザー入力）から来る場合は、代わりに `TryFrom` / `FromStr`（第7章、トリック5）を使用してください。

***

### トリック10 — チャネル通信のためのセッション型（Session Types）

2つのコンポーネントがチャネルを介して通信する場合（例: 診断オーケストレータ ↔ ワーカースレッド）、**セッション型（Session Types）** はプロトコルを型システムにエンコードします：

```rust,ignore
use std::marker::PhantomData;

// プロトコル: クライアントが Request を送信し、サーバーが Response を返し、完了する。
pub struct SendRequest;
pub struct RecvResponse;
pub struct Done;

/// 型付きチャネルエンドポイント。`S` は現在のプロトコル状態。
pub struct Chan<S> {
    // 実際のコード: mpsc::Sender/Receiver ペアをラップ
    _state: PhantomData<S>,
}

impl Chan<SendRequest> {
    /// リクエストを送信 — RecvResponse 状態に遷移。
    pub fn send(self, request: DiagRequest) -> Chan<RecvResponse> {
        // ... チャネルに送信 ...
        Chan { _state: PhantomData }
    }
}

impl Chan<RecvResponse> {
    /// レスポンスを受信 — Done 状態に遷移。
    pub fn recv(self) -> (DiagResponse, Chan<Done>) {
        // ... チャネルから受信 ...
        (DiagResponse { passed: true }, Chan { _state: PhantomData })
    }
}

impl Chan<Done> {
    /// チャネルを閉じる — プロトコルが完了したときにのみ可能。
    pub fn close(self) { /* ドロップ */ }
}

pub struct DiagRequest { pub test_name: String }
pub struct DiagResponse { pub passed: bool }

// プロトコルは必ず順番通りに従わなければならない:
fn orchestrator(chan: Chan<SendRequest>) {
    let chan = chan.send(DiagRequest { test_name: "gpu_stress".into() });
    let (response, chan) = chan.recv();
    chan.close();
    println!("Result: {}", if response.passed { "PASS" } else { "FAIL" });
}

// send の前に recv することはできない:
// fn wrong_order(chan: Chan<SendRequest>) {
//     chan.recv();  // ❌ Chan<SendRequest> に `recv` メソッドは存在しない
// }
```

> **使い所:** スレッド間の診断プロトコル、BMCコマンドシーケンス、順序が重要となるあらゆるリクエスト・レスポンスパターン。複雑なマルチメッセージプロトコルには、[`session-types`](https://crates.io/crates/session-types) や [`rumpsteak`](https://crates.io/crates/rumpsteak) クレートの利用も検討してください。

***

### トリック11 — 自己参照状態機械のための `Pin`

一部の型状態機械は、自身のデータへの参照を保持する必要があります（例: 所有するバッファ内の位置を追跡するパーサー）。構造体をムーブすると内部ポインタが無効化されるため、Rustは通常これを禁止します。`Pin<T>` は、値が**ムーブされない**ことを保証することでこれを解決します：

```rust,ignore
use std::pin::Pin;
use std::marker::PhantomPinned;

/// 自身のバッファへの参照を保持するストリーミングパーサー。
/// 一度ピン留めされるとムーブできなくなり、内部参照の有効性が保たれる。
pub struct StreamParser {
    buffer: Vec<u8>,
    /// `buffer` の内部を指す。ピン留めされている間のみ有効。
    cursor: *const u8,
    _pin: PhantomPinned,  // Unpin をオプトアウト — 意図しないピン留め解除を防ぐ
}

impl StreamParser {
    pub fn new(data: Vec<u8>) -> Pin<Box<Self>> {
        let parser = StreamParser {
            buffer: data,
            cursor: std::ptr::null(),
            _pin: PhantomPinned,
        };
        let mut boxed = Box::pin(parser);

        // cursor がピン留めされたバッファを指すように設定
        let cursor = boxed.buffer.as_ptr();
        // SAFETY: 排他的アクセス権があり、パーサーはピン留めされている
        unsafe {
            let mut_ref = Pin::as_mut(&mut boxed);
            Pin::get_unchecked_mut(mut_ref).cursor = cursor;
        }

        boxed
    }

    /// 次のバイトを読み出す — Pin<&mut Self> 経由でのみ呼び出し可能。
    pub fn next_byte(self: Pin<&mut Self>) -> Option<u8> {
        // パーサーはムーブできないため、cursor は有効なまま
        if self.cursor.is_null() { return None; }
        // ... バッファ内でカーソルを進める ...
        Some(42) // スタブ
    }
}

// 使い方:
// let mut parser = StreamParser::new(vec![0x01, 0x02, 0x03]);
// let byte = parser.as_mut().next_byte();
```

**重要な洞察:** `Pin` は、自己参照構造体の問題に対する構造的に正しい解決策です。これがない場合、`unsafe` と手動のライフタイム追跡が必要になります。これがあれば、コンパイラがムーブを防止し、内部ポインタの不変条件が維持されます。

| `Pin` を使うべき場合… | `Pin` を使わない場合… |
|-----------------|----------------------|
| 状態機械が構造体内部への参照を保持する場合 | すべてのフィールドが独立して所有されている場合 |
| `.await` をまたいで借用する非同期 Future | 自己参照が不要な場合 |
| メモリ上で再配置されてはならない DMA ディスクリプタ | データを自由にムーブできる場合 |
| 内部カーソルを持つハードウェアリングバッファ | 単純なインデックスベースの反復で十分な場合 |

***

### トリック12 — 正しさの保証としての RAII / `Drop`

Rustの `Drop` トレイトは構造的に正しいメカニズムです。コンパイラがクリーンアップコードを自動的に挿入するため、クリーンアップが**忘れられることはありません**。これは、厳密に1回だけ解放しなければならないハードウェアリソースにとって特に価値があります。

```rust,ignore
use std::io;

/// 終了時に必ず閉じなければならない IPMI セッション。
/// `Drop` 実装により、パニック時や `?` による早期リターン時でもクリーンアップが保証される。
pub struct IpmiSession {
    handle: u32,
}

impl IpmiSession {
    pub fn open(host: &str) -> io::Result<Self> {
        // ... IPMI セッションをネゴシエート ...
        Ok(IpmiSession { handle: 42 })
    }

    pub fn send_raw(&self, _data: &[u8]) -> io::Result<Vec<u8>> {
        Ok(vec![0x00])
    }
}

impl Drop for IpmiSession {
    fn drop(&mut self) {
        // Close Session コマンド: パニックや早期リターン時でも常に実行される。
        // C言語では、CloseSession() を忘れると BMC のセッションスロットがリークする。
        let _ = self.send_raw(&[0x06, 0x3C]);
        eprintln!("[RAII] session {} closed", self.handle);
    }
}
// 使い方:
fn diagnose(host: &str) -> io::Result<()> {
    let session = IpmiSession::open(host)?;
    session.send_raw(&[0x04, 0x2D, 0x20])?;
    // 明示的な close は不要 — ここで自動的に Drop が実行される
    Ok(())
    // send_raw が Err(...) を返した場合でも、セッションは確実に閉じられる。
}
```

**RAII が排除する C/C++ の失敗パターン:**

```text
C:     session = ipmi_open(host);
       ipmi_send(session, data);
       if (error) return -1;        // 🐛 セッションのリーク — close() を忘れた
       ipmi_close(session);

Rust:  let session = IpmiSession::open(host)?;
       session.send_raw(data)?;     // ✅ ? によるリターン時に Drop が実行される
       // Drop は常に実行される — リークは起こり得ない
```

**順序付けられたクリーンアップのための RAII と型状態（第5章）の組み合わせ:**

ジェネリックパラメータに対して `Drop` を特殊化することはできません（Rust エラー E0366）。代わりに、状態ごとに**個別のラッパー型**を使用します：

```rust,ignore
use std::marker::PhantomData;

pub struct Open;
pub struct Locked;

pub struct GpuContext<S> {
    device_id: u32,
    _state: PhantomData<S>,
}

impl GpuContext<Open> {
    pub fn lock_clocks(self) -> LockedGpu {
        // ... 安定したベンチマークのために GPU クロックをロック ...
        LockedGpu { device_id: self.device_id }
    }
}

/// ロック状態専用の個別の型 — 独自の Drop を持つ。
/// `impl Drop for GpuContext<Locked>` は記述できないため（E0366）、
/// ロックされたリソースを所有する個別のラッパーを使用する。
pub struct LockedGpu {
    device_id: u32,
}

impl LockedGpu {
    pub fn run_benchmark(&self) -> f64 {
        // ... ロックされたクロックでベンチマークを実行 ...
        42.0
    }
}

impl Drop for LockedGpu {
    fn drop(&mut self) {
        // ドロップ時にクロックのロックを解除 — ロック状態のラッパーに対してのみ発火する。
        eprintln!("[RAII] GPU {} clocks unlocked", self.device_id);
    }
}

// GpuContext<Open> には特別な Drop はない — 解除すべきクロックがないため。
// LockedGpu は、パニックや早期リターン時であっても、ドロップ時に必ずロックを解除する。
```

> **なぜ `impl Drop for GpuContext<Locked>` ができないのか？** Rust では、`Drop` 実装がジェネリック型の*すべての*インスタンス化に適用される必要があります。状態に応じたクリーンアップを実現するには、以下のいずれかのアプローチを使用します：
>
> | アプローチ | メリット | デメリット |
> |----------|------|------|
> | 個別のラッパー型（上記） | 明快、ゼロコスト | 型名が増える |
> | ジェネリックな `Drop` + 実行時の `TypeId` チェック | 単一の型 | `'static` が必要、ランタイムコスト |
> | `enum` 状態と `Drop` 内での網羅的マッチ | 単一のジェネリック型 | ランタイムディスパッチ、型安全性が低下 |

> **使い所:** BMCセッション、GPUクロックロック、DMAバッファマッピング、ファイルハンドル、ミューテックスガードなど、解放ステップが必須であるあらゆるリソース。もし `fn close(&mut self)` や `fn cleanup()` といったメソッドを書いている自分に気づいたら、ほぼ間違いなく代わりに `Drop` を使うべきです。

***

### トリック13 — 正しさを担保するエラー型の階層構造

適切に設計されたエラー型は、エラーのサイレントな握りつぶしを防ぎ、呼び出し元がそれぞれの失敗モードを適切に処理できるようにします。構造化されたエラーに `thiserror` を使用することは、構造的に正しいパターンです。コンパイラが網羅的なパターンマッチを強制するためです。

```toml
# Cargo.toml
[dependencies]
thiserror = "1"
# アプリケーションレベルのエラー処理用（任意）:
# anyhow = "1"
```

```rust,ignore
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DiagError {
    #[error("IPMI communication failed: {0}")]
    Ipmi(#[from] IpmiError),

    #[error("sensor {sensor_id:#04x} reading out of range: {value}")]
    SensorRange { sensor_id: u8, value: f64 },

    #[error("GPU {gpu_id} not responding")]
    GpuTimeout { gpu_id: u32 },

    #[error("configuration invalid: {0}")]
    Config(String),
}

#[derive(Debug, Error)]
pub enum IpmiError {
    #[error("session authentication failed")]
    AuthFailed,

    #[error("command {net_fn:#04x}/{cmd:#04x} timed out")]
    Timeout { net_fn: u8, cmd: u8 },

    #[error("completion code {0:#04x}")]
    CompletionCode(u8),
}

// 呼び出し元は各バリアントを必ず処理しなければならない — サイレントな握りつぶしは不可:
fn run_thermal_check() -> Result<(), DiagError> {
    // これが IpmiError を返した場合、#[from] アトリビュートにより
    // 自動的に DiagError::Ipmi に変換される。
    let temp = read_cpu_temp()?;
    if temp > 105.0 {
        return Err(DiagError::SensorRange {
            sensor_id: 0x20,
            value: temp,
        });
    }
    Ok(())
}

# fn read_cpu_temp() -> Result<f64, DiagError> { Ok(42.0) }
```

**なぜこれが構造的に正しいのか:**

| 構造化されていないエラー | `thiserror` の列挙型 |
|--------------------------|----------------------|
| `fn op() -> Result<T, String>` | `fn op() -> Result<T, DiagError>` |
| 呼び出し元は不透明な文字列を受け取る | 呼び出し元は特定のバリアントでマッチできる |
| 認証失敗とタイムアウトを区別できない | `DiagError::Ipmi(IpmiError::AuthFailed)` vs `Timeout` |
| ログ出力でエラーが握りつぶされる | `match` により各ケースの処理が強制される |
| 新しいエラーバリアントの追加に誰も気づかない | 新しいバリアントの追加時に未処理のアームをコンパイラが警告する |

**`anyhow` と `thiserror` の使い分けの判断:**

| `thiserror` を使う場合… | `anyhow` を使う場合… |
|-----------------------|-------------------|
| ライブラリ / クレートの開発 | バイナリ / CLI の開発 |
| 呼び出し元がエラーバリアントで分岐する必要がある場合 | 呼び出し元が単にログを出力して終了する場合 |
| エラー型がパブリックAPIの一部である場合 | 内部的なエラーの配管処理 |
| `protocol_lib`, `accel_diag`, `thermal_diag` | `diag_tool` の main バイナリ |

> **使い所:** ワークスペース内のすべてのクレートは、`thiserror` を使用して独自のエラー列挙型を定義すべきです。最上位のバイナリクレートは、それらを統合するために `anyhow` を使用できます。これにより、ライブラリの呼び出し元にはコンパイル時のエラー処理の保証が与えられ、バイナリ側は簡潔に保たれます。

***

### トリック14 — 消費を強制する `#[must_use]`

`#[must_use]` アトリビュートは、戻り値の無視をコンパイラの警告に変えます。これは、本ガイドのすべてのパターンと組み合わせることができる、軽量で構造的に正しいツールです：

```rust,ignore
/// 必ず使用しなければならないキャリブレーション（校正）トークン — サイレントにドロップするのはバグ。
#[must_use = "calibration token must be passed to calibrate(), not dropped"]
pub struct CalibrationToken {
    _private: (),
}

/// 必ずチェックしなければならない診断結果 — 失敗の無視はバグ。
#[must_use = "diagnostic result must be inspected for failures"]
pub struct DiagResult {
    pub passed: bool,
    pub details: String,
}

/// 重要な値を返す関数にもアノテーションを付けるべきである:
#[must_use = "the authenticated session must be used or explicitly closed"]
pub fn authenticate(user: &str, pass: &str) -> Result<Session, AuthError> {
    // ...
#   unimplemented!()
}
#
# pub struct Session;
# pub struct AuthError;
```

**コンパイラが伝える警告:**

```text
warning: unused `CalibrationToken` that must be used
  --> src/main.rs:5:5
   |
5  |     CalibrationToken { _private: () };
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: calibration token must be passed to calibrate(), not dropped
```

**以下のパターンに `#[must_use]` を適用する:**

| パターン | アノテーションを付与する対象 | 理由 |
|---------|-----------------|-----|
| 単一使用トークン（第3章） | `CalibrationToken`, `FusePayload` | 使用せずにドロップするのはロジックバグ |
| ケイパビリティトークン（第4章） | `AdminToken` | 認証したのにトークンを無視している |
| 型状態の遷移 | `authenticate()`, `activate()` の戻り値の型 | セッションを作成したのに使用していない |
| 実行結果 | `DiagResult`, `SensorReading` | サイレントな障害の見落とし |
| RAII ハンドル（トリック12） | `IpmiSession`, `LockedGpu` | リソースを開いたのに使用していない |

> **経験則:** 値を使用せずにドロップすることが常にバグである場合は、`#[must_use]` を追加してください。時として意図的なドロップがあり得る場合（例: `Vec`）は追加しないでください。`_` プレフィックス（`let _ = foo()`）は警告を明示的に認識して抑制します — これは意図的なドロップである場合には問題ありません。

## 重要ポイント

1. **境界で番兵値を Option に変換する** — パース時にマジックナンバーを `Option` に変換することで、コンパイラが呼び出し元に `None` の処理を強制します。
2. **シールドトレイトで実装の抜け穴を塞ぐ** — プライベートなスーパートレイトにより、自分たちのクレートだけがトレイトを実装できるようになります。
3. **`#[non_exhaustive]` と `#[must_use]` は1行で高価値なアトリビュート** — 将来拡張される列挙型や消費されるべきトークンに追加してください。
4. **型状態ビルダーで必須フィールドを強制する** — 必須の型パラメータがすべて `Set` されたときにのみ `finish()` が存在するようにします。
5. **各トリックが特定のバグ分類を狙い撃ちする** — アーキテクチャを書き直す必要はなく、段階的に導入できます。

---
