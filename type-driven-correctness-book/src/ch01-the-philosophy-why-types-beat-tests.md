# 哲学 — なぜ型はテストに勝るのか 🟢

> **学べること:** コンパイル時における正当性の3つのレベル（値、状態、プロトコル）、ジェネリック関数のシグネチャがいかにコンパイラ検証済みの保証として機能するか、そして構築による正当性（Correct-by-Construction）パターンが投資に見合う場合と見合わない場合。
>
> **関連章:** [第2章](ch02-typed-command-interfaces-request-determi.md)（型付きコマンド）、[第5章](ch05-protocol-state-machines-type-state-for-r.md)（型状態）、[第13章](ch13-reference-card.md)（リファレンスカード）

## 実行時チェックのコスト

診断コードベースでよく見られる、実行時のガード処理を考えてみましょう。

```rust,ignore
fn read_sensor(sensor_type: &str, raw: &[u8]) -> f64 {
    match sensor_type {
        "temperature" => raw[0] as i8 as f64,          // 符号付きバイト
        "fan_speed"   => u16::from_le_bytes([raw[0], raw[1]]) as f64,
        "voltage"     => u16::from_le_bytes([raw[0], raw[1]]) as f64 / 1000.0,
        _             => panic!("unknown sensor type: {sensor_type}"),
    }
}
```

この関数には、コンパイラが検出できない**4つの障害モード（Failure Mode）**が存在します。

1. タイプミス: `"temperture"` → 実行時パニック
2. `raw` の長さ不正: 1バイトしかない `fan_speed` → 実行時パニック
3. 呼び出し側が実際には °C である戻り値の `f64` を RPM として扱ってしまう → 検出されないサイレントな論理バグ
4. 新しいセンサー型が追加されたがこの `match` が更新されていない → 実行時パニック

これらの障害モードはすべて**デプロイ後**に発覚します。テストは助けにはなりますが、誰かが書くことを思いついたケースしかカバーできません。一方、型システムは誰も想像していなかったケースを含め、**すべての**ケースを網羅します。

## 正当性の3つのレベル

### レベル1 — 値の正当性（Value Correctness）
**不正な値を表現不可能にする。**

```rust,ignore
// ❌ 任意の u16 が「ポート」になり得る — 0 は不正だがコンパイルが通ってしまう
fn connect(port: u16) { /* ... */ }

// ✅ 検証済みのポートのみが存在可能
pub struct Port(u16);  // プライベートフィールド

impl TryFrom<u16> for Port {
    type Error = &'static str;
    fn try_from(v: u16) -> Result<Self, Self::Error> {
        if v > 0 { Ok(Port(v)) } else { Err("port must be > 0") }
    }
}

fn connect(port: Port) { /* ... */ }
// Port(0) は構築不可能 — 不変条件があらゆる場所で維持される
```

**ハードウェアの例:** `SensorId(u8)` — 生のセンサー番号をラップし、SDR（Sensor Data Record）の範囲内にあることを検証する。

### レベル2 — 状態の正当性（State Correctness）
**不正な遷移を表現不可能にする。**

```rust,ignore
use std::marker::PhantomData;

struct Disconnected;
struct Connected;

struct Socket<State> {
    fd: i32,
    _state: PhantomData<State>,
}

impl Socket<Disconnected> {
    fn connect(self, addr: &str) -> Socket<Connected> {
        // ... 接続ロジック ...
        Socket { fd: self.fd, _state: PhantomData }
    }
}

impl Socket<Connected> {
    fn send(&mut self, data: &[u8]) { /* ... */ }
    fn disconnect(self) -> Socket<Disconnected> {
        Socket { fd: self.fd, _state: PhantomData }
    }
}

// Socket<Disconnected> には send() メソッドが存在しない — 呼び出そうとするとコンパイルエラーになる
```

**ハードウェアの例:** GPIO ピンモード — `Pin<Input>` には `read()` はあるが `write()` はない。

### レベル3 — プロトコルの正当性（Protocol Correctness）
**不正な相互作用を表現不可能にする。**

```rust,ignore
use std::io;

trait IpmiCmd {
    type Response;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Self::Response>;
}

// 説明のために簡略化 — net_fn(), cmd_byte(), payload(), parse_response() を
// 含む完全なトレイトについては第2章を参照してください。

struct ReadTemp { sensor_id: u8 }
impl IpmiCmd for ReadTemp {
    type Response = Celsius;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Celsius> {
        Ok(Celsius(raw[0] as i8 as f64))
    }
}

# #[derive(Debug)] struct Celsius(f64);

fn execute<C: IpmiCmd>(cmd: &C, raw: &[u8]) -> io::Result<C::Response> {
    cmd.parse_response(raw)
}
// ReadTemp は常に Celsius を返す — 誤って Rpm を取得することはあり得ない
```

**ハードウェアの例:** IPMI、Redfish、NVMe Admin コマンド — リクエスト型がレスポンス型を一意に決定する。

## コンパイラ検証済みの保証としての型

次のように記述したとします。

```rust,ignore
fn execute<C: IpmiCmd>(cmd: &C) -> io::Result<C::Response>
```

これは単に関数を書いているのではなく、**保証**を宣言しています。「`IpmiCmd` を実装する任意のコマンド型 `C` について、それを実行すると厳密に `C::Response` が得られる」という保証です。コンパイラはコードをビルドするたびに、この保証を**検証**します。型が一致しなければ、プログラムはコンパイルすら通りません。

これこそがRustの型システムが極めて強力である理由です。単にミスを検出するだけでなく、**コンパイル時に正当性を強制**しているのです。

## これらのパターンを使うべきでない場合

構築による正当性（Correct-by-construction）が常に適切な選択肢とは限りません。

| 状況 | 推奨事項 |
|-----------|---------------|
| 安全性に関わるクリティカルな境界（電源シーケンス、暗号処理） | ✅ 常に適用 — ここでのバグはハードウェアの破損や機密漏洩に直結する |
| モジュール間の公開API | ✅ 基本的に適用 — 誤用はコンパイルエラーにすべき |
| 3つ以上の状態を持つ状態機械 | ✅ 基本的に適用 — 型状態により不正な状態遷移を防止できる |
| 50行程度の関数内の内部ヘルパー | ❌ 過剰設計 — 単純な `assert!` で十分 |
| プロトタイピング / 未知のハードウェアの調査 | ❌ まずは素の型を使用 — 挙動を把握した後に洗練させる |
| ユーザー向けCLIのパース | ⚠️ 境界で `clap` + `TryFrom` を使い、内部では素の型で扱ってもよい |

判断の鍵となる問いは、**「このバグが本番環境で発生した場合、被害はどれほど深刻か？」**です。

- ファンが停止 → GPUが溶損 → **型を使用する**
- DER レコードの誤り → 顧客に不正なデータが届く → **型を使用する**
- デバッグログのメッセージが少し間違っている → **`assert!` を使用する**

## 主なまとめ

1. **3つの正当性レベル** — 値（newtype）、状態（型状態）、プロトコル（関連型） — 各レベルがより広範なバグのカテゴリを排除します。
2. **保証としての型** — すべてのジェネリック関数のシグネチャは、ビルドのたびにコンパイラによって検証される契約です。
3. **コストの問い** — 「このバグが出荷されたらどれほど深刻か？」によって、型とテストのどちらが適切なツールであるかが決まります。
4. **型はテストを補完する** — 型はバグの*カテゴリ全体*を排除し、テストは特定の*値*やエッジケースを検証します。
5. **引き際を知る** — 内部ヘルパーや使い捨てのプロトタイプに型レベルの強制が必要になることは稀です。

---
