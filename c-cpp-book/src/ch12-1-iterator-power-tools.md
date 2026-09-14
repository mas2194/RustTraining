## イテレータ便利ツール（Power Tools）リファレンス

> **学習目標:** `filter`/`map`/`collect` を超える高度なイテレータコンビネータ（`enumerate`、`zip`、`chain`、`flat_map`、`scan`、`windows`、`chunks`）について学びます。Cスタイルのインデックス付き `for` ループを、安全で表現力豊かなRustのイテレータへ置き換えるために不可欠な知識です。

基本的な `filter`/`map`/`collect` のチェーンでも多くのユースケースをカバーできますが、Rustのイテレータライブラリは遥かに強力です。本節では、日常的によく利用するツール群を紹介します。特に、インデックスを手動追跡したり、結果を累積したり、固定サイズのチャンク単位でデータを処理するようなC言語のループをRustに移行する際に役立ちます。

### クイックリファレンステーブル

| メソッド | C言語での相当処理 | 動作内容 | 戻り値の型 |
|--------|-------------|-------------|---------|
| `enumerate()` | `for (int i=0; ...)` | 各要素とインデックスのペアを生成 | `(usize, T)` |
| `zip(other)` | 同じインデックスを持つ並列配列 | 2つのイテレータの要素をペア化 | `(A, B)` |
| `chain(other)` | array1 を処理した後に array2 を処理 | 2つのイテレータを連結 | `T` |
| `flat_map(f)` | ネストしたループ | マッピング後に1階層フラット化 | `U` |
| `windows(n)` | `for (int i=0; i<len-n+1; i++) &arr[i..i+n]` | サイズ `n` の重複ありスライディングスライス | `&[T]` |
| `chunks(n)` | 一度に `n` 要素ずつ処理 | サイズ `n` の重複なしスライス | `&[T]` |
| `fold(init, f)` | `int acc = init; for (...) acc = f(acc, x);` | 単一の値へ畳み込み（縮約） | `Acc` |
| `scan(init, f)` | 出力付きの実行累積器 | `fold` に似ているが中間結果を順次 yield する | `Option<B>` |
| `take(n)` / `skip(n)` | ループの開始オフセット / 上限指定 | 先頭 `n` 要素を取得 / スキップ | `T` |
| `take_while(f)` / `skip_while(f)` | `while (pred) {...}` | 述語が真の間、取得 / スキップ | `T` |
| `peekable()` | `arr[i+1]` による先読み | 要素を消費せずに `.peek()` 可能にする | `T` |
| `step_by(n)` | `for (i=0; i<len; i+=n)` | `n` 要素ごとに取得 | `T` |
| `unzip()` | 並列配列の分割 | ペアを2つのコレクションに分離して collect | `(A, B)` |
| `sum()` / `product()` | 合計 / 積の累積 | `+` または `*` による畳み込み | `T` |
| `min()` / `max()` | 極値の探索 | `Option<T>` を返す | `Option<T>` |
| `any(f)` / `all(f)` | `bool found = false; for (...) ...` | 短絡評価（ショートサーキット）による真偽値探索 | `bool` |
| `position(f)` | `for (i=0; ...) if (pred) return i;` | 最初にマッチした要素のインデックス | `Option<usize>` |

### `enumerate` — インデックス + 値（Cスタイルのインデックスループを代替）

```rust
fn main() {
    let sensors = ["GPU_TEMP", "CPU_TEMP", "FAN_RPM", "PSU_WATT"];

    // Cスタイル: for (int i = 0; i < 4; i++) printf("[%d] %s\n", i, sensors[i]);
    for (i, name) in sensors.iter().enumerate() {
        println!("[{i}] {name}");
    }

    // 特定のセンサーのインデックスを検索
    let gpu_idx = sensors.iter().position(|&s| s == "GPU_TEMP");
    println!("GPU sensor at index: {gpu_idx:?}");  // Some(0)
}
```

### `zip` — 並行イテレーション（並行配列ループを代替）

```rust
fn main() {
    let names = ["accel_diag", "nic_diag", "cpu_diag"];
    let statuses = [true, false, true];
    let durations_ms = [1200, 850, 3400];

    // Cスタイル: for (int i=0; i<3; i++) printf("%s: %s (%d ms)\n", names[i], ...);
    for ((name, passed), ms) in names.iter().zip(&statuses).zip(&durations_ms) {
        let status = if *passed { "PASS" } else { "FAIL" };
        println!("{name}: {status} ({ms} ms)");
    }
}
```

### `chain` — イテレータの連結

```rust
fn main() {
    let critical = vec!["ECC error", "Thermal shutdown"];
    let warnings = vec!["Link degraded", "Fan slow"];

    // すべてのイベントを優先度順に処理
    let all_events: Vec<_> = critical.iter().chain(warnings.iter()).collect();
    println!("{all_events:?}");
    // ["ECC error", "Thermal shutdown", "Link degraded", "Fan slow"]
}
```

### `flat_map` — ネストした結果のフラット化

```rust
fn main() {
    let lines = vec!["gpu:42:ok", "nic:99:fail", "cpu:7:ok"];

    // コロン区切りの行からすべての数値を抽出
    let numbers: Vec<u32> = lines.iter()
        .flat_map(|line| line.split(':'))
        .filter_map(|token| token.parse::<u32>().ok())
        .collect();
    println!("{numbers:?}");  // [42, 99, 7]
}
```

### `windows` と `chunks` — スライディングおよび固定サイズグループ

```rust
fn main() {
    let temps = [65, 68, 72, 71, 75, 80, 78, 76];

    // windows(3): 重複する3要素グループ（移動平均などに便利）
    // Cスタイル: for (int i = 0; i <= len-3; i++) avg(arr[i], arr[i+1], arr[i+2]);
    let moving_avg: Vec<f64> = temps.windows(3)
        .map(|w| w.iter().sum::<i32>() as f64 / 3.0)
        .collect();
    println!("Moving avg: {moving_avg:.1?}");

    // chunks(2): 重複しない2要素グループ
    // Cスタイル: for (int i = 0; i < len; i += 2) process(arr[i], arr[i+1]);
    for pair in temps.chunks(2) {
        println!("Chunk: {pair:?}");
    }

    // chunks_exact(2): 同様だが余りが出た場合を厳格に扱う
    // また、.remainder() で余りの要素を取得可能
}
```

### `fold` と `scan` — 累積計算

```rust
fn main() {
    let values = [10, 20, 30, 40, 50];

    // fold: 単一の最終結果（C言語の累積アキュムレータループと同様）
    let sum = values.iter().fold(0, |acc, &x| acc + x);
    println!("Sum: {sum}");  // 150

    // fold を使って文字列を構築
    let csv = values.iter()
        .fold(String::new(), |acc, x| {
            if acc.is_empty() { format!("{x}") }
            else { format!("{acc},{x}") }
        });
    println!("CSV: {csv}");  // "10,20,30,40,50"

    // scan: fold に似ているが中間結果を順次 yield する
    let running_sum: Vec<i32> = values.iter()
        .scan(0, |state, &x| {
            *state += x;
            Some(*state)
        })
        .collect();
    println!("Running sum: {running_sum:?}");  // [10, 30, 60, 100, 150]
}
```

### 演習: センサーデータパイプライン

生のセンサー測定値（1行につき1件、フォーマット `"sensor_name:value:unit"`）が与えられたとき、以下を行うイテレータパイプラインを作成してください：
1. 各行を `(name, f64, unit)` にパースする
2. 閾値（threshold）未満の測定値を除外する
3. `fold` を使ってセンサー名ごとに `HashMap` へグループ化する
4. センサーごとの平均値を表示する

```rust
// スターターコード
fn main() {
    let raw_data = vec![
        "gpu_temp:72.5:C",
        "cpu_temp:65.0:C",
        "gpu_temp:74.2:C",
        "fan_rpm:1200.0:RPM",
        "cpu_temp:63.8:C",
        "gpu_temp:80.1:C",
        "fan_rpm:1150.0:RPM",
    ];
    let threshold = 70.0;
    // TODO: パース、threshold 以上の値をフィルタ、名前でグループ化、平均値を計算
}
```

<details><summary>解答例（クリックして展開）</summary>

```rust
use std::collections::HashMap;

fn main() {
    let raw_data = vec![
        "gpu_temp:72.5:C",
        "cpu_temp:65.0:C",
        "gpu_temp:74.2:C",
        "fan_rpm:1200.0:RPM",
        "cpu_temp:63.8:C",
        "gpu_temp:80.1:C",
        "fan_rpm:1150.0:RPM",
    ];
    let threshold = 70.0;

    // パース → フィルタ → グループ化 → 平均値計算
    let grouped = raw_data.iter()
        .filter_map(|line| {
            let parts: Vec<&str> = line.splitn(3, ':').collect();
            if parts.len() == 3 {
                let value: f64 = parts[1].parse().ok()?;
                Some((parts[0], value, parts[2]))
            } else {
                None
            }
        })
        .filter(|(_, value, _)| *value >= threshold)
        .fold(HashMap::<&str, Vec<f64>>::new(), |mut acc, (name, value, _)| {
            acc.entry(name).or_default().push(value);
            acc
        });

    for (name, values) in &grouped {
        let avg = values.iter().sum::<f64>() / values.len() as f64;
        println!("{name}: avg={avg:.1} ({} readings)", values.len());
    }
}
// 出力例（順序は異なる場合があります）:
// gpu_temp: avg=75.6 (3 readings)
// fan_rpm: avg=1175.0 (2 readings)
```

</details>


# Rustのイテレータ
- `Iterator` トレイトは、ユーザー定義型に対するイテレーションを実装するために使用されます (https://doc.rust-lang.org/std/iter/trait.IntoIterator.html)
    - この例では、1, 1, 2, ... と始まり後続の項が直前の2項の和となるフィボナッチ数列のイテレータを実装します
    - `Iterator` 内の `関連型（associated type）`（`type Item = u32;`）は、イテレータが出力する型（`u32`）を定義します
    - `next()` メソッドには、イテレータの進捗ロジックを記述します。今回のケースでは、すべての状態情報が `Fibonacci` 構造体内に保持されます
    - より特殊化されたイテレータ向けに `into_iter()` メソッドを実装するために、`IntoIterator` という別のトレイトを実装することも可能です
    - https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=ab367dc2611e1b5a0bf98f1185b38f3f
