## 境界チェックなしのインデックスアクセスの回避

> **学習目標:** なぜRustで `vec[i]` が危険なのか（範囲外アクセスによるパニック）、そして `.get()` やイテレータ、`HashMap` の `entry()` API などの安全な代替手段について学びます。C++の未定義動作を明示的な処理へと置き換えます。

- C++において、`vec[i]` や `map[key]` は未定義動作を引き起こすか、キーが存在しない場合に自動挿入を行います。Rustの `[]` は境界外アクセスでパニックします。
- **ルール**: インデックスが有効であることを*証明*できる場合を除き、`[]` ではなく `.get()` を使用してください。

### C++ と Rust の比較
```cpp
// C++ — 暗黙の未定義動作（UB）または自動挿入
std::vector<int> v = {1, 2, 3};
int x = v[10];        // 未定義動作（UB）！operator[] による境界チェックなし

std::map<std::string, int> m;
int y = m["missing"]; // 値 0 でキーを暗黙的に挿入してしまう！
```

```rust
// Rust — 安全な代替手段
let v = vec![1, 2, 3];

// バッドプラクティス: インデックスが範囲外の場合にパニックする
// let x = v[10];

// グッドプラクティス: Option<&i32> を返す
let x = v.get(10);              // None — パニックしない
let x = v.get(1).copied().unwrap_or(0);  // 2、存在しない場合は 0
```

### 実例: 本番Rustコードでの安全なバイト列パース
```rust
// 例: diagnostics.rs
// バイナリSELレコードのパース — バッファが想定より短い可能性がある
let sensor_num = bytes.get(7).copied().unwrap_or(0);
let ppin = cpu_ppin.get(i).map(|s| s.as_str()).unwrap_or("");
```

### 実例: `.and_then()` による連鎖的な安全な検索
```rust
// 例: profile.rs — 二重検索: HashMap → Vec
pub fn get_processor(&self, location: &str) -> Option<&Processor> {
    self.processor_by_location
        .get(location)                              // HashMap → Option<&usize>
        .and_then(|&idx| self.processors.get(idx))   // Vec → Option<&Processor>
}
// どちらの検索も Option を返す — パニックも未定義動作もなし
```

### 実例: 安全なJSONナビゲーション
```rust
// 例: framework.rs — すべてのJSONキー検索が Option を返す
let manufacturer = product_fru
    .get("Manufacturer")            // Option<&Value>
    .and_then(|v| v.as_str())       // Option<&str>
    .unwrap_or(UNKNOWN_VALUE)       // &str (安全なフォールバック)
    .to_string();
```
C++のパターン `json["SystemInfo"]["ProductFru"]["Manufacturer"]` と比較してください。いずれかのキーが欠落していると `nlohmann::json::out_of_range` 例外がスローされます。

### `[]` が許容されるケース
- **境界チェック後**: `if i < v.len() { v[i] }`
- **テストコード内**: パニックすることが期待される動作である場合
- **定数・保証されたインデックス**: `assert!(!v.is_empty());` の直後の `let first = v[0];` など

----

## unwrap_or による安全な値の取り出し

- `unwrap()` は `None` や `Err` の場合にパニックします。本番環境のコードでは、安全な代替手段を優先して使用してください。

### unwrap ファミリ
| **メソッド** | **None/Err 時の挙動** | **使用する場面** |
|-----------|------------------------|-------------|
| `.unwrap()` | **パニック** | テストのみ、または絶対に失敗しないことが証明できる場合 |
| `.expect("msg")` | メッセージ付きでパニック | パニックが正当化される場合（理由を説明する） |
| `.unwrap_or(default)` | `default` を返す | 低コストな定数のフォールバックがある場合 |
| `.unwrap_or_else(|| expr)` | クロージャを呼び出す | フォールバック値の計算コストが高い場合 |
| `.unwrap_or_default()` | `Default::default()` を返す | 型が `Default` を実装している場合 |

### 実例: 安全なデフォルト値を用いたパース
```rust
// 例: peripherals.rs
// 正規表現のキャプチャグループが一致しない可能性があるため、安全なフォールバックを提供
let bus_hex = caps.get(1).map(|m| m.as_str()).unwrap_or("00");
let fw_status = caps.get(5).map(|m| m.as_str()).unwrap_or("0x0");
let bus = u8::from_str_radix(bus_hex, 16).unwrap_or(0);
```

### 実例: フォールバック構造体を用いた `unwrap_or_else`
```rust
// 例: framework.rs
// 関数全体で Option を返すクロージャ内にロジックをラップし、
// 何かが失敗した場合はデフォルトの構造体を返す:
(|| -> Option<BaseboardFru> {
    let content = std::fs::read_to_string(path).ok()?;
    let json: serde_json::Value = serde_json::from_str(&content).ok()?;
    // ... .get()? のチェーンでフィールドを抽出
    Some(baseboard_fru)
})()
.unwrap_or_else(|| BaseboardFru {
    manufacturer: String::new(),
    model: String::new(),
    product_part_number: String::new(),
    serial_number: String::new(),
    asset_tag: String::new(),
})
```

### 実例: 設定のデシリアライズにおける `unwrap_or_default`
```rust
// 例: framework.rs
// JSON設定のパースに失敗した場合、Default にフォールバック — クラッシュしない
Ok(json) => serde_json::from_str(&json).unwrap_or_default(),
```
C++での同等の処理は、`nlohmann::json::parse()` を `try/catch` で囲み、catch ブロック内で手動でデフォルト構築を行うことになります。

----

## 関数型変換: map, map_err, find_map

- `Option` や `Result` に用意されているこれらのメソッドを使用すると、値をアンラップすることなく変換でき、ネストした `if/else` を線形なメソッドチェーンに置き換えることができます。

### クイックリファレンス
| **メソッド** | **対象** | **動作** | **C++ での同等の処理** |
|-----------|-------|---------|-------------------|
| `.map(\|v\| ...)` | `Option` / `Result` | `Some` / `Ok` の値を変換する | `if (opt) { *opt = transform(*opt); }` |
| `.map_err(\|e\| ...)` | `Result` | `Err` の値を変換する | catch ブロックでコンテキストを追加する |
| `.and_then(\|v\| ...)` | `Option` / `Result` | `Option` / `Result` を返す操作を連鎖させる | ネストした if チェック |
| `.find_map(\|v\| ...)` | イテレータ | 1回のパスで `find` と `map` を実行する | `if + break` を含むループ |
| `.filter(\|v\| ...)` | `Option` / イテレータ | 述語（条件）に一致する値のみを保持する | `if (!predicate) return nullopt;` |
| `.ok()?` | `Result` | `Result → Option` に変換し、`None` を伝播する | `if (result.has_error()) return nullopt;` |

### 実例: JSONフィールド抽出のための `.and_then()` チェーン
```rust
// 例: framework.rs — フォールバック付きのシリアル番号検索
let sys_info = json.get("SystemInfo")?;

// 最初に BaseboardFru.BoardSerialNumber を試行
if let Some(serial) = sys_info
    .get("BaseboardFru")
    .and_then(|b| b.get("BoardSerialNumber"))
    .and_then(|v| v.as_str())
    .filter(valid_serial)     // 空でなく有効なシリアル番号のみを受け入れる
{
    return Some(serial.to_string());
}

// BoardFru.SerialNumber にフォールバック
sys_info
    .get("BoardFru")
    .and_then(|b| b.get("SerialNumber"))
    .and_then(|v| v.as_str())
    .filter(valid_serial)
    .map(|s| s.to_string())   // Some の場合のみ &str → String に変換
```
C++では、これは `if (json.contains("BaseboardFru")) { if (json["BaseboardFru"].contains("BoardSerialNumber")) { ... } }` というネストのピラミッドになってしまいます。

### 実例: `find_map` — 1回のパスで検索と変換を同時に実行
```rust
// 例: context.rs — センサーと所有者に一致するSDRレコードを検索
pub fn find_for_event(&self, sensor_number: u8, owner_id: u8) -> Option<&SdrRecord> {
    self.by_sensor.get(&sensor_number).and_then(|indices| {
        indices.iter().find_map(|&i| {
            let record = &self.records[i];
            if record.sensor_owner_id() == Some(owner_id) {
                Some(record)
            } else {
                None
            }
        })
    })
}
```
`find_map` は `find` と `map` を融合させたものです。最初に一致した時点で停止し、その値を変換します。C++での同等の処理は、`if` + `break` を持った `for` ループです。

### 実例: エラーコンテキストを付加する `map_err`
```rust
// 例: main.rs — 伝播する前にエラーにコンテキストを追加
let json_str = serde_json::to_string_pretty(&config)
    .map_err(|e| format!("Failed to serialize config: {}", e))?;
```
`serde_json::Error` を、*何が*失敗したかに関するコンテキストを含む説明的な `String` エラーに変換します。

----

## JSONの取り扱い: nlohmann::json から serde へ

- C++開発チームでは通常、JSONパースに `nlohmann::json` を使用します。Rustでは **serde** + **serde_json** を使用します。JSONスキーマが*型システム*に組み込まれるため、より強力です。

### C++（nlohmann）と Rust（serde）の比較

```cpp
// nlohmann::json を使用した C++ — 実行時のフィールドアクセス
#include <nlohmann/json.hpp>
using json = nlohmann::json;

struct Fan {
    std::string logical_id;
    std::vector<std::string> sensor_ids;
};

Fan parse_fan(const json& j) {
    Fan f;
    f.logical_id = j.at("LogicalID").get<std::string>();    // 存在しない場合は例外をスロー
    if (j.contains("SDRSensorIdHexes")) {                   // 手動でのデフォルト値処理
        f.sensor_ids = j["SDRSensorIdHexes"].get<std::vector<std::string>>();
    }
    return f;
}
```

```rust
// serde を使用した Rust — コンパイル時スキーマと自動フィールドマッピング
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Fan {
    pub logical_id: String,
    #[serde(rename = "SDRSensorIdHexes", default)]  // JSONキー → Rustフィールド
    pub sensor_ids: Vec<String>,                     // 欠落している場合 → 空の Vec
    #[serde(default)]
    pub sensor_names: Vec<String>,                   // 欠落している場合 → 空の Vec
}

// パース関数全体が1行で置き換わります:
let fan: Fan = serde_json::from_str(json_str)?;
```

### 主要な serde 属性（本番Rustコードの実例）

| **属性** | **用途** | **C++ での同等の処理** |
|--------------|------------|--------------------|
| `#[serde(default)]` | フィールドが存在しない場合に `Default::default()` を使用 | `if (j.contains(key)) { ... } else { default; }` |
| `#[serde(rename = "Key")]` | JSONキー名をRustフィールド名にマッピング | 手動での `j.at("Key")` アクセス |
| `#[serde(flatten)]` | 未知のキーを `HashMap` に吸収 | `for (auto& [k,v] : j.items()) { ... }` |
| `#[serde(skip)]` | このフィールドをシリアライズ/デシリアライズしない | JSONに格納しない |
| `#[serde(tag = "type")]` | 内部タグ付き列挙型（判別子フィールド） | `if (j["type"] == "gpu") { ... }` |

### 実例: 完全な設定構造体
```rust
// 例: diag.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DiagConfig {
    pub sku: SkuConfig,
    #[serde(default)]
    pub level: DiagLevel,            // 欠落している場合 → DiagLevel::default()
    #[serde(default)]
    pub modules: ModuleConfig,       // 欠落している場合 → ModuleConfig::default()
    #[serde(default)]
    pub output_dir: String,          // 欠落している場合 → ""
    #[serde(default, flatten)]
    pub options: HashMap<String, serde_json::Value>,  // 未知のキーを吸収
}

// ロード処理はわずか3行（C++ の nlohmann では20行以上）:
let content = std::fs::read_to_string(path)?;
let config: DiagConfig = serde_json::from_str(&content)?;
Ok(config)
```

### `#[serde(tag = "type")]` による列挙型のデシリアライズ
```rust
// 例: components.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]                   // JSON: {"type": "Gpu", "product": ...}
pub enum PcieDeviceKind {
    Gpu { product: GpuProduct, manufacturer: GpuManufacturer },
    Nic { product: NicProduct, manufacturer: NicManufacturer },
    NvmeDrive { drive_type: StorageDriveType, capacity_gb: u32 },
    // ... さらに9つのバリアント
}
// serde が "type" フィールドに基づいて自動的にディスパッチ — 手動の if/else チェーンは不要
```
C++での同等の処理は次のようになります: `if (j["type"] == "Gpu") { parse_gpu(j); } else if (j["type"] == "Nic") { parse_nic(j); } ...`

# 演習: serde によるJSONデシリアライズ

- 以下のJSONからデシリアライズ可能な `ServerConfig` 構造体を定義してください:
```json
{
    "hostname": "diag-node-01",
    "port": 8080,
    "debug": true,
    "modules": ["accel_diag", "nic_diag", "cpu_diag"]
}
```
- `#[derive(Deserialize)]` と `serde_json::from_str()` を使用してパースしてください
- `debug` に `#[serde(default)]` を追加し、欠落している場合はデフォルトで `false` になるようにしてください
- **ボーナス課題**: デフォルトで `Quick` になる `#[serde(default)]` を付与した `enum DiagLevel { Quick, Full, Extended }` フィールドを追加してください

**スターターコード**（`cargo add serde --features derive` および `cargo add serde_json` が必要です）:
```rust
use serde::Deserialize;

// TODO: Default を実装した DiagLevel 列挙型を定義する

// TODO: serde 属性を付けた ServerConfig 構造体を定義する

fn main() {
    let json_input = r#"{
        "hostname": "diag-node-01",
        "port": 8080,
        "debug": true,
        "modules": ["accel_diag", "nic_diag", "cpu_diag"]
    }"#;

    // TODO: 設定をデシリアライズして出力する
    // TODO: "debug" フィールドがないJSONのパースを試み、デフォルトで false になることを確認する
}
```

<details><summary>解答例（クリックして展開）</summary>

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize, Default)]
enum DiagLevel {
    #[default]
    Quick,
    Full,
    Extended,
}

#[derive(Debug, Deserialize)]
struct ServerConfig {
    hostname: String,
    port: u16,
    #[serde(default)]       // 欠落している場合はデフォルトで false
    debug: bool,
    modules: Vec<String>,
    #[serde(default)]       // 欠落している場合はデフォルトで DiagLevel::Quick
    level: DiagLevel,
}

fn main() {
    let json_input = r#"{
        "hostname": "diag-node-01",
        "port": 8080,
        "debug": true,
        "modules": ["accel_diag", "nic_diag", "cpu_diag"]
    }"#;

    let config: ServerConfig = serde_json::from_str(json_input)
        .expect("Failed to parse JSON");
    println!("{config:#?}");

    // オプションフィールドが欠落しているケースをテスト
    let minimal = r#"{
        "hostname": "node-02",
        "port": 9090,
        "modules": []
    }"#;
    let config2: ServerConfig = serde_json::from_str(minimal)
        .expect("Failed to parse minimal JSON");
    println!("debug (default): {}", config2.debug);    // false
    println!("level (default): {:?}", config2.level);  // Quick
}
// 出力結果:
// ServerConfig {
//     hostname: "diag-node-01",
//     port: 8080,
//     debug: true,
//     modules: ["accel_diag", "nic_diag", "cpu_diag"],
//     level: Quick,
// }
// debug (default): false
// level (default): Quick
```

</details>

----
