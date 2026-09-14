# すべてを組み合わせる — 完全な診断プラットフォーム 🟡

> **学べること:** 7つのコアパターンすべて（第2章〜第9章）を、単一の診断ワークフロー — 認証、セッション、型付きコマンド、監査トークン、次元付き結果、バリデーション済みデータ、幽霊型レジスタ — へと組み合わせ、トータルのランタイムオーバーヘッドを完全にゼロにする方法。
>
> **相互参照:** すべてのコアパターンの章（第2章〜第9章）、[第14章](ch14-testing-type-level-guarantees.md)（これらの保証のテスト）

## 目標

本章では、第2章から第9章で学んだ**7つのパターン**を、単一の実践的な診断ワークフローへと組み合わせます。以下の処理を行うサーバー健全性チェックを構築します：

1. **認証**（ケイパビリティトークン — 第4章）
2. **IPMIセッションの開始**（型状態 — 第5章）
3. **型付きコマンドの送信**（型付きコマンド — 第2章）
4. **監査ログ用の使い捨てトークンの使用**（単一使用型 — 第3章）
5. **次元付き結果の返却**（次元解析 — 第6章）
6. **FRUデータのバリデーション**（境界でのバリデーション — 第7章）
7. **型付きレジスタの読み出し**（幽霊型 — 第9章）

```rust,ignore
use std::marker::PhantomData;
use std::io;
// ──── パターン1: 次元の型（第6章） ────

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Celsius(pub f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Rpm(pub f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Volts(pub f64);

// ──── パターン2: 型付きコマンド（第2章） ────

/// 一貫性のため、関連定数ではなくメソッドを使用した第2章と同じトレイト形状。
/// 値が型ごとに完全に固定されている場合は、関連定数（`const NETFN: u8`）も
/// 同等に有効な選択肢です。
pub trait IpmiCmd {
    type Response;
    fn net_fn(&self) -> u8;
    fn cmd_byte(&self) -> u8;
    fn payload(&self) -> Vec<u8>;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Self::Response>;
}

pub struct ReadTemp { pub sensor_id: u8 }
impl IpmiCmd for ReadTemp {
    type Response = Celsius;   // ← 次元の型！
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.sensor_id] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Celsius> {
        if raw.is_empty() {
            return Err(io::Error::new(io::ErrorKind::InvalidData, "empty"));
        }
        Ok(Celsius(raw[0] as f64))
    }
}

pub struct ReadFanSpeed { pub fan_id: u8 }
impl IpmiCmd for ReadFanSpeed {
    type Response = Rpm;
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.fan_id] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Rpm> {
        if raw.len() < 2 {
            return Err(io::Error::new(io::ErrorKind::InvalidData, "need 2 bytes"));
        }
        Ok(Rpm(u16::from_le_bytes([raw[0], raw[1]]) as f64))
    }
}

// ──── パターン3: ケイパビリティトークン（第4章） ────

pub struct AdminToken { _private: () }

pub fn authenticate(user: &str, pass: &str) -> Result<AdminToken, &'static str> {
    if user == "admin" && pass == "secret" {
        Ok(AdminToken { _private: () })
    } else {
        Err("authentication failed")
    }
}

// ──── パターン4: 型状態セッション（第5章） ────

pub struct Idle;
pub struct Active;

pub struct Session<State> {
    host: String,
    _state: PhantomData<State>,
}

impl Session<Idle> {
    pub fn connect(host: &str) -> Self {
        Session { host: host.to_string(), _state: PhantomData }
    }

    pub fn activate(
        self,
        _admin: &AdminToken,  // ← ケイパビリティトークンが必要
    ) -> Result<Session<Active>, String> {
        println!("Session activated on {}", self.host);
        Ok(Session { host: self.host, _state: PhantomData })
    }
}

impl Session<Active> {
    /// 型付きコマンドを実行 — Active セッションでのみ利用可能。
    /// トランスポートエラーを伝播するため io::Result を返す（第2章と整合）。
    pub fn execute<C: IpmiCmd>(&mut self, cmd: &C) -> io::Result<C::Response> {
        let raw_response = self.raw_send(cmd.net_fn(), cmd.cmd_byte(), &cmd.payload())?;
        cmd.parse_response(&raw_response)
    }

    fn raw_send(&self, _nf: u8, _cmd: u8, _data: &[u8]) -> io::Result<Vec<u8>> {
        Ok(vec![42, 0x1E]) // スタブ: 生のIPMIレスポンス
    }

    pub fn close(self) { println!("Session closed"); }
}

// ──── パターン5: 単一使用の監査トークン（第3章） ────

/// 診断の実行ごとに一意な監査トークンが発行される。
/// Clone も Copy も実装しない — 各監査エントリが一意であることを保証。
pub struct AuditToken {
    run_id: u64,
}

impl AuditToken {
    pub fn issue(run_id: u64) -> Self {
        AuditToken { run_id }
    }

    /// トークンを消費して監査ログエントリを書き込む。
    pub fn log(self, message: &str) {
        println!("[AUDIT run_id={}] {}", self.run_id, message);
        // トークンは消費された — 同じ run_id を2回ログに記録することはできない
    }
}

// ──── パターン6: 境界でのバリデーション（第7章） ────
// 本組み合わせ例に必要なフィールドのみを抽出した、第7章の完全な ValidFru の簡略版。
// 完全な TryFrom<RawFruData> バージョンについては第7章を参照。

pub struct ValidFru {
    pub board_serial: String,
    pub product_name: String,
}

impl ValidFru {
    pub fn parse(raw: &[u8]) -> Result<Self, &'static str> {
        if raw.len() < 8 { return Err("FRU too short"); }
        if raw[0] != 0x01 { return Err("bad FRU version"); }
        Ok(ValidFru {
            board_serial: "SN12345".to_string(),  // スタブ
            product_name: "ServerX".to_string(),
        })
    }
}

// ──── パターン7: 幽霊型レジスタ（第9章） ────

pub struct Width16;
pub struct Reg<W> { offset: u16, _w: PhantomData<W> }

impl Reg<Width16> {
    pub fn read(&self) -> u16 { 0x8086 } // スタブ
}

pub struct PcieDev {
    pub vendor_id: Reg<Width16>,
    pub device_id: Reg<Width16>,
}

impl PcieDev {
    pub fn new() -> Self {
        PcieDev {
            vendor_id: Reg { offset: 0x00, _w: PhantomData },
            device_id: Reg { offset: 0x02, _w: PhantomData },
        }
    }
}

// ──── 複合ワークフロー ────

fn full_diagnostic() -> Result<(), String> {
    // 1. 認証 → ケイパビリティトークンを取得
    let admin = authenticate("admin", "secret")
        .map_err(|e| e.to_string())?;

    // 2. 接続してセッションをアクティブ化（型状態: Idle → Active）
    let session = Session::connect("192.168.1.100");
    let mut session = session.activate(&admin)?;  // AdminToken が必要

    // 3. 型付きコマンドを送信（レスポンス型はコマンドと一致）
    let temp: Celsius = session.execute(&ReadTemp { sensor_id: 0 })
        .map_err(|e| e.to_string())?;
    let fan: Rpm = session.execute(&ReadFanSpeed { fan_id: 1 })
        .map_err(|e| e.to_string())?;

    // 型の不一致はコンパイル時に捕捉される:
    // let wrong: Volts = session.execute(&ReadTemp { sensor_id: 0 })?;
    //  ❌ エラー: expected Celsius, found Volts

    // 4. 幽霊型で型付けされたPCIeレジスタを読み出し
    let pcie = PcieDev::new();
    let vid: u16 = pcie.vendor_id.read();  // u16 であることが保証される

    // 5. 境界でFRUデータを検証
    let raw_fru = vec![0x01, 0x00, 0x00, 0x01, 0x01, 0x00, 0x00, 0xFD];
    let fru = ValidFru::parse(&raw_fru)
        .map_err(|e| e.to_string())?;

    // 6. 単一使用の監査トークンを発行
    let audit = AuditToken::issue(1001);

    // 7. レポートを生成（すべてのデータは型付けされ検証済み）
    let report = format!(
        "Server: {} (SN: {}), VID: 0x{:04X}, CPU: {:?}, Fan: {:?}",
        fru.product_name, fru.board_serial, vid, temp, fan,
    );

    // 8. 監査トークンを消費 — 2回ログを記録することはできない
    audit.log(&report);
    // audit.log("oops");  // ❌ ムーブされた値の使用

    // 9. セッションを閉じる（型状態: Active → ドロップ）
    session.close();

    Ok(())
}
```

### コンパイラが証明すること

| バグの種類 | 防止方法 | パターン |
|-----------|-------------------|---------|
| 未認証アクセス | `activate()` に `&AdminToken` が必要 | ケイパビリティトークン |
| 不正なセッション状態でのコマンド実行 | `execute()` は `Session<Active>` にのみ存在 | 型状態（タイプステート） |
| 誤ったレスポンス型 | `ReadTemp::Response = Celsius` がトレイトで固定 | 型付きコマンド |
| 単位の混同（°C vs RPM） | `Celsius` ≠ `Rpm` ≠ `Volts` | 次元の型 |
| レジスタ幅の不一致 | `Reg<Width16>` は `u16` を返す | 幽霊型（Phantom Types） |
| 未検証データの処理 | 事前に `ValidFru::parse()` を呼び出す必要がある | 境界でのバリデーション |
| 監査エントリの重複 | `AuditToken` はログ記録時に消費される | 単一使用型 |
| 電源投入シーケンスの順序違い | 各ステップで直前のトークンが必要 | ケイパビリティトークン（第4章） |

**これらの保証すべてによる合計ランタイムオーバーヘッド: ゼロ。**

すべてのチェックはコンパイル時に行われます。生成されるアセンブリは、チェックを一切含まない手書きのC言語コードと同一です — しかし**C言語にはバグが潜み得ますが、このコードにはそれがありません**。

## 重要ポイント

1. **7つのパターンはシームレスに組み合わさる** — ケイパビリティトークン、型状態、型付きコマンド、単一使用型、次元の型、境界でのバリデーション、そして幽霊型がすべて連動して機能します。
2. **コンパイラが8つのバグ分類を根本的に排除する** — 上記の「コンパイラが証明すること」の表を参照してください。
3. **トータルのランタイムオーバーヘッドはゼロ** — 生成されるアセンブリは未検証のC言語コードと同一です。
4. **各パターンは単体でも有用** — 7つすべてを一度に導入する必要はなく、段階的に導入できます。
5. **この統合の章は設計テンプレート** — 独自の型付き診断ワークフローを構築する出発点として活用してください。
6. **IPMIから大規模なRedfishへ** — 第17章および第18章では、これら7つのパターン（および第8章のケイパビリティミックスイン）を完全なRedfishクライアントとサーバーに適用します。ここでのIPMIワークフローはその基盤であり、Redfishのチュートリアルでは、複数のデータソースやスキーマバージョンの制約を抱える本番システムにこの構成がどうスケールするかを示します。

---
