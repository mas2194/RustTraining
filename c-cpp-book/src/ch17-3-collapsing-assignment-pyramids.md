## クロージャによる代入ピラミッドの解消

> **学習目標:** Rustの式指向構文とクロージャを活用して、深くネストされたC++の `if/else` バリデーションチェーンを、クリーンで線形なコードへとフラット化する方法を学びます。

- C++では、特にバリデーションやフォールバック処理を伴う場合、変数を代入するために複数の `if/else` ブロックチェーンが必要になることがよくあります。Rustの式指向（expression-based）構文とクロージャを使用すると、これらをフラットで線形なコードに集約できます。

### パターン 1: if 式を用いたタプル代入
```cpp
// C++ — 複数の if/else ブロックチェーンにまたがって3つの変数を設定
uint32_t fault_code;
const char* der_marker;
const char* action;
if (is_c44ad) {
    fault_code = 32709; der_marker = "CSI_WARN"; action = "No action";
} else if (error.is_hardware_error()) {
    fault_code = 67956; der_marker = "CSI_ERR"; action = "Replace GPU";
} else {
    fault_code = 32709; der_marker = "CSI_WARN"; action = "No action";
}
```

```rust
// Rustでの同等コード: accel_fieldiag.rs
// 単一の式で3つの変数すべてを一度に代入:
let (fault_code, der_marker, recommended_action) = if is_c44ad {
    (32709u32, "CSI_WARN", "No action")
} else if error.is_hardware_error() {
    (67956u32, "CSI_ERR", "Replace GPU")
} else {
    (32709u32, "CSI_WARN", "No action")
};
```

### パターン 2: 失敗する可能性のある処理チェーンのための IIFE（即時実行関数式）
```cpp
// C++ — JSONナビゲーションにおける「破滅のピラミッド（Pyramid of Doom）」
std::string get_part_number(const nlohmann::json& root) {
    if (root.contains("SystemInfo")) {
        auto& sys = root["SystemInfo"];
        if (sys.contains("BaseboardFru")) {
            auto& bb = sys["BaseboardFru"];
            if (bb.contains("ProductPartNumber")) {
                return bb["ProductPartNumber"].get<std::string>();
            }
        }
    }
    return "UNKNOWN";
}
```

```rust
// Rustでの同等コード: framework.rs
// クロージャと ? 演算子によってピラミッドを線形なコードへと解消:
let part_number = (|| -> Option<String> {
    let path = self.args.sysinfo.as_ref()?;
    let content = std::fs::read_to_string(path).ok()?;
    let json: serde_json::Value = serde_json::from_str(&content).ok()?;
    let ppn = json
        .get("SystemInfo")?
        .get("BaseboardFru")?
        .get("ProductPartNumber")?
        .as_str()?;
    Some(ppn.to_string())
})()
.unwrap_or_else(|| "UNKNOWN".to_string());
```
クロージャによって `Option<String>` のスコープが作成され、どのステップでも `?` によって早期リターンできます。末尾の `.unwrap_or_else()` で一度だけフォールバックを提供します。

### パターン 3: 手動ループ + push_back を置き換えるイテレータチェーン
```cpp
// C++ — 中間変数を用いた手動ループ
std::vector<std::tuple<std::vector<std::string>, std::string, std::string>> gpu_info;
for (const auto& [key, info] : gpu_pcie_map) {
    std::vector<std::string> bdfs;
    // ... bdf_path をパースして bdfs に格納
    std::string serial = info.serial_number.value_or("UNKNOWN");
    std::string model = info.model_number.value_or(model_name);
    gpu_info.push_back({bdfs, serial, model});
}
```

```rust
// Rustでの同等コード: peripherals.rs
// 単一のチェーン: values() → map → collect
let gpu_info: Vec<(Vec<String>, String, String, String)> = self
    .gpu_pcie_map
    .values()
    .map(|info| {
        let bdfs: Vec<String> = info.bdf_path
            .split(')')
            .filter(|s| !s.is_empty())
            .map(|s| s.trim_start_matches('(').to_string())
            .collect();
        let serial = info.serial_number.clone()
            .unwrap_or_else(|| "UNKNOWN".to_string());
        let model = info.model_number.clone()
            .unwrap_or_else(|| model_name.to_string());
        let gpu_bdf = format!("{}:{}:{}.{}",
            info.bdf.segment, info.bdf.bus, info.bdf.device, info.bdf.function);
        (bdfs, serial, model, gpu_bdf)
    })
    .collect();
```

### パターン 4: ループ + if (condition) continue を置き換える `.filter().collect()`
```cpp
// C++
std::vector<TestResult*> failures;
for (auto& t : test_results) {
    if (!t.is_pass()) {
        failures.push_back(&t);
    }
}
```

```rust
// Rust — accel_diag/src/healthcheck.rs より
pub fn failed_tests(&self) -> Vec<&TestResult> {
    self.test_results.iter().filter(|t| !t.is_pass()).collect()
}
```

### まとめ: 各パターンを使用する場面
| **C++ のパターン** | **Rust での代替** | **主なメリット** |
|----------------|---------------------|-----------------|
| 複数ブロックでの変数代入 | `let (a, b) = if ... { } else { };` | すべての変数がアトミックに束縛される |
| ネストした `if (contains)` のピラミッド | `?` 演算子を用いた IIFE クロージャ | 線形でフラット、早期リターンが可能 |
| `for` ループ + `push_back` | `.iter().map(\|\|).collect()` | 中間の一時的な mut Vec が不要 |
| `for` + `if (cond) continue` | `.iter().filter(\|\|).collect()` | 宣言的な意図の表現 |
| `for` + `if + break`（最初の要素を検索） | `.iter().find_map(\|\|)` | 1回のパスで検索と変換を実行 |

----

# 総合演習: 診断イベントパイプライン

🔴 **チャレンジ課題** — 列挙型、トレイト、イテレータ、エラー処理、ジェネリクスを組み合わせた総合演習

この総合演習では、列挙型、トレイト、イテレータ、エラー処理、ジェネリクスを組み合わせます。本番のRustコードで使用されているパターンと同様の、簡略化された診断イベント処理パイプラインを構築します。

**要件:**
1. `Display` を実装した `enum Severity { Info, Warning, Critical }` と、`source: String`, `severity: Severity`, `message: String`, `fault_code: u32` を持つ `struct DiagEvent` を定義する
2. メソッド `fn should_include(&self, event: &DiagEvent) -> bool` を持つ `trait EventFilter` を定義する
3. 2つのフィルタを実装する: `SeverityFilter`（指定した重大度以上のイベントのみ）と `SourceFilter`（特定のソース文字列からのイベントのみ）
4. **すべての** フィルタを通過したイベントに対してフォーマット済みのレポート行を返す関数 `fn process_events(events: &[DiagEvent], filters: &[&dyn EventFilter]) -> Vec<String>` を作成する
5. `"source:severity:fault_code:message"` 形式の行をパースする `fn parse_event(line: &str) -> Result<DiagEvent, String>` を作成する（不正な入力に対しては `Err` を返す）

**スターターコード:**
```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
enum Severity {
    Info,
    Warning,
    Critical,
}

impl fmt::Display for Severity {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        todo!()
    }
}

#[derive(Debug, Clone)]
struct DiagEvent {
    source: String,
    severity: Severity,
    message: String,
    fault_code: u32,
}

trait EventFilter {
    fn should_include(&self, event: &DiagEvent) -> bool;
}

struct SeverityFilter {
    min_severity: Severity,
}
// TODO: SeverityFilter に EventFilter を実装する

struct SourceFilter {
    source: String,
}
// TODO: SourceFilter に EventFilter を実装する

fn process_events(events: &[DiagEvent], filters: &[&dyn EventFilter]) -> Vec<String> {
    // TODO: すべてのフィルタを通過したイベントを抽出し、
    // "[SEVERITY] source (FC:fault_code): message" の形式にフォーマットする
    todo!()
}

fn parse_event(line: &str) -> Result<DiagEvent, String> {
    // "source:severity:fault_code:message" をパースする
    // 不正な入力に対しては Err を返す
    todo!()
}

fn main() {
    let raw_lines = vec![
        "accel_diag:Critical:67956:ECC uncorrectable error detected",
        "nic_diag:Warning:32709:Link speed degraded",
        "accel_diag:Info:10001:Self-test passed",
        "cpu_diag:Critical:55012:Thermal throttling active",
        "accel_diag:Warning:32710:PCIe link width reduced",
    ];

    // すべての行をパースし、成功したものを収集してエラーを報告する
    let events: Vec<DiagEvent> = raw_lines.iter()
        .filter_map(|line| match parse_event(line) {
            Ok(e) => Some(e),
            Err(e) => { eprintln!("Parse error: {e}"); None }
        })
        .collect();

    // フィルタを適用: accel_diag からの Critical および Warning イベントのみ
    let sev_filter = SeverityFilter { min_severity: Severity::Warning };
    let src_filter = SourceFilter { source: "accel_diag".to_string() };
    let filters: Vec<&dyn EventFilter> = vec![&sev_filter, &src_filter];

    let report = process_events(&events, &filters);
    for line in &report {
        println!("{line}");
    }
    println!("--- {} event(s) matched ---", report.len());
}
```

<details><summary>解答例（クリックして展開）</summary>

```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
enum Severity {
    Info,
    Warning,
    Critical,
}

impl fmt::Display for Severity {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Severity::Info => write!(f, "INFO"),
            Severity::Warning => write!(f, "WARNING"),
            Severity::Critical => write!(f, "CRITICAL"),
        }
    }
}

impl Severity {
    fn from_str(s: &str) -> Result<Self, String> {
        match s {
            "Info" => Ok(Severity::Info),
            "Warning" => Ok(Severity::Warning),
            "Critical" => Ok(Severity::Critical),
            other => Err(format!("Unknown severity: {other}")),
        }
    }
}

#[derive(Debug, Clone)]
struct DiagEvent {
    source: String,
    severity: Severity,
    message: String,
    fault_code: u32,
}

trait EventFilter {
    fn should_include(&self, event: &DiagEvent) -> bool;
}

struct SeverityFilter {
    min_severity: Severity,
}

impl EventFilter for SeverityFilter {
    fn should_include(&self, event: &DiagEvent) -> bool {
        event.severity >= self.min_severity
    }
}

struct SourceFilter {
    source: String,
}

impl EventFilter for SourceFilter {
    fn should_include(&self, event: &DiagEvent) -> bool {
        event.source == self.source
    }
}

fn process_events(events: &[DiagEvent], filters: &[&dyn EventFilter]) -> Vec<String> {
    events.iter()
        .filter(|e| filters.iter().all(|f| f.should_include(e)))
        .map(|e| format!("[{}] {} (FC:{}): {}", e.severity, e.source, e.fault_code, e.message))
        .collect()
}

fn parse_event(line: &str) -> Result<DiagEvent, String> {
    let parts: Vec<&str> = line.splitn(4, ':').collect();
    if parts.len() != 4 {
        return Err(format!("Expected 4 colon-separated fields, got {}", parts.len()));
    }
    let fault_code = parts[2].parse::<u32>()
        .map_err(|e| format!("Invalid fault code '{}': {e}", parts[2]))?;
    Ok(DiagEvent {
        source: parts[0].to_string(),
        severity: Severity::from_str(parts[1])?,
        fault_code,
        message: parts[3].to_string(),
    })
}

fn main() {
    let raw_lines = vec![
        "accel_diag:Critical:67956:ECC uncorrectable error detected",
        "nic_diag:Warning:32709:Link speed degraded",
        "accel_diag:Info:10001:Self-test passed",
        "cpu_diag:Critical:55012:Thermal throttling active",
        "accel_diag:Warning:32710:PCIe link width reduced",
    ];

    let events: Vec<DiagEvent> = raw_lines.iter()
        .filter_map(|line| match parse_event(line) {
            Ok(e) => Some(e),
            Err(e) => { eprintln!("Parse error: {e}"); None }
        })
        .collect();

    let sev_filter = SeverityFilter { min_severity: Severity::Warning };
    let src_filter = SourceFilter { source: "accel_diag".to_string() };
    let filters: Vec<&dyn EventFilter> = vec![&sev_filter, &src_filter];

    let report = process_events(&events, &filters);
    for line in &report {
        println!("{line}");
    }
    println!("--- {} event(s) matched ---", report.len());
}
// 出力結果:
// [CRITICAL] accel_diag (FC:67956): ECC uncorrectable error detected
// [WARNING] accel_diag (FC:32710): PCIe link width reduced
// --- 2 event(s) matched ---
```

</details>

----
