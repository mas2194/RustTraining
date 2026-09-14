## ロギングとトレーシング: syslog/printf から `log` + `tracing` へ

> **学習目標:** Rustの2層ロギングアーキテクチャ（ファサード + バックエンド）、`log` および `tracing` クレート、スパンを用いた構造化ロギング、そしてこれらがどのように `printf`/`syslog` によるデバッグを置き換えるかについて学びます。

C++の診断コードでは通常、`printf`、`syslog`、あるいは独自のロギングフレームワークが使用されます。
Rustには標準化された2層のロギングアーキテクチャがあります。すなわち、**ファサード（facade）**クレート（`log` または `tracing`）と、**バックエンド（backend）**（実際のロガー実装）です。

### `log` ファサード — Rustの汎用ロギングAPI

`log` クレートは、syslogの重大度レベルを反映したマクロを提供します。ライブラリは `log` マクロを使用し、バイナリ側でバックエンドを選択します:

```rust
// Cargo.toml
// [dependencies]
// log = "0.4"
// env_logger = "0.11"    # 数多くあるバックエンドの1つ

use log::{info, warn, error, debug, trace};

fn check_sensor(id: u32, temp: f64) {
    trace!("センサー {id} を読み取り中");           // 最も細かい粒度
    debug!("センサー {id} の生値: {temp}");       // 開発時の詳細情報

    if temp > 85.0 {
        warn!("センサー {id} が高温です: {temp}°C");
    }
    if temp > 95.0 {
        error!("センサー {id} が重大（CRITICAL）レベルです: {temp}°C — シャットダウンを開始します");
    }
    info!("センサー {id} のチェック完了");         // 通常の動作
}

fn main() {
    // バックエンドの初期化 — 通常は main() で1度だけ実行
    env_logger::init();  // RUST_LOG 環境変数によって制御される

    check_sensor(0, 72.5);
    check_sensor(1, 91.0);
}
```

```bash
# 環境変数経由でログレベルを制御
RUST_LOG=debug cargo run          # debug 以上を表示
RUST_LOG=warn cargo run           # warn と error のみを表示
RUST_LOG=my_crate=trace cargo run # モジュールごとのフィルタリング
RUST_LOG=my_crate::gpu=debug,warn cargo run  # 複数レベルの混在指定
```

### C++ との比較

| C++ | Rust (`log`) | 備考 |
|-----|-------------|-------|
| `printf("DEBUG: %s\n", msg)` | `debug!("{msg}")` | コンパイル時にフォーマットを検証 |
| `syslog(LOG_ERR, "...")` | `error!("...")` | 出力先はバックエンドが決定 |
| ログ呼び出しを囲む `#ifdef DEBUG` | max_level に応じて `trace!` / `debug!` はコンパイル時に除外 | 無効化時はゼロコスト |
| 独自の `Logger::log(level, msg)` | `log::info!("...")` — すべてのクレートが同一のAPIを使用 | 共通のファサード、差し替え可能なバックエンド |
| ファイルごとのログ詳細度制御 | `RUST_LOG=crate::module=level` | 環境変数ベース、再コンパイル不要 |

### `tracing` クレート — スパンを用いた構造化ロギング

`tracing` は `log` を拡張し、**構造化フィールド**と**スパン**（時間計測スコープ）を提供します。
これは、コンテキストを追跡したい診断コードで特に役立ちます:

```rust
// Cargo.toml
// [dependencies]
// tracing = "0.1"
// tracing-subscriber = { version = "0.3", features = ["env-filter"] }

use tracing::{info, warn, error, instrument, info_span};

#[instrument(skip(data), fields(gpu_id = gpu_id, data_len = data.len()))]
fn run_gpu_test(gpu_id: u32, data: &[u8]) -> Result<(), String> {
    info!("GPUテストを開始");

    let span = info_span!("ecc_check", gpu_id);
    let _guard = span.enter();  // このスコープ内のすべてのログに gpu_id が自動的に付与される

    if data.is_empty() {
        error!(gpu_id, "テストデータが提供されていません");
        return Err("空のデータ".to_string());
    }

    // 構造化フィールド — 単なる文字列展開ではなく、機械可読な形式
    info!(
        gpu_id,
        temp_celsius = 72.5,
        ecc_errors = 0,
        "ECCチェック合格"
    );

    Ok(())
}

fn main() {
    // tracing サブスクライバの初期化
    tracing_subscriber::fmt()
        .with_env_filter("debug")  // または RUST_LOG 環境変数を使用
        .with_target(true)          // モジュールパスを表示
        .with_thread_ids(true)      // スレッドIDを表示
        .init();

    let _ = run_gpu_test(0, &[1, 2, 3]);
}
```

`tracing-subscriber` による出力例:
```rust
2026-02-15T10:30:00.123Z DEBUG ThreadId(01) run_gpu_test{gpu_id=0 data_len=3}: my_crate: GPUテストを開始
2026-02-15T10:30:00.124Z  INFO ThreadId(01) run_gpu_test{gpu_id=0 data_len=3}:ecc_check{gpu_id=0}: my_crate: ECCチェック合格 gpu_id=0 temp_celsius=72.5 ecc_errors=0
```

### `#[instrument]` — 自動的なスパンの生成

`#[instrument]` 属性は、関数名とその引数を含むスパンを自動的に生成します:

```rust
use tracing::instrument;

#[instrument]
fn parse_sel_record(record_id: u16, sensor_type: u8, data: &[u8]) -> Result<(), String> {
    // この関数内のすべてのログには、自動的に以下が含まれます:
    // record_id, sensor_type, data（Debug実装がある場合）
    tracing::debug!("SELレコードをパース中");
    Ok(())
}

// skip: サイズの大きい引数や機密データをスパンから除外
// fields: 計算されたフィールドを追加
#[instrument(skip(raw_buffer), fields(buf_len = raw_buffer.len()))]
fn decode_ipmi_response(raw_buffer: &[u8]) -> Result<Vec<u8>, String> {
    tracing::trace!("{} バイトをデコード中", raw_buffer.len());
    Ok(raw_buffer.to_vec())
}
```

### `log` vs `tracing` — どちらを使うべきか

| 観点 | `log` | `tracing` |
|--------|-------|-----------|
| **複雑さ** | シンプル — 5つのマクロ | より高機能 — スパン、フィールド、計装（instrument） |
| **構造化データ** | 文字列展開のみ | キー・バリュー形式のフィールド: `info!(gpu_id = 0, "msg")` |
| **タイミング / スパン** | なし | あり — `#[instrument]`, `span.enter()` |
| **非同期サポート** | 基本的なもののみ | 第一級サポート — スパンが `.await` を越えて伝播 |
| **互換性** | 汎用ファサード | `log` との互換性あり（`log` ブリッジを同梱） |
| **使用すべき場面** | シンプルなアプリケーション、ライブラリ | 診断ツール、非同期コード、オブザーバビリティ |

> **推奨事項**: 本番環境の診断系プロジェクト（構造化出力を伴う診断ツールなど）には `tracing` を使用してください。依存関係を最小限に抑えたいシンプルなライブラリには `log` を使用してください。`tracing` には互換レイヤーが含まれているため、`log` マクロを使用しているライブラリも `tracing` サブスクライバでそのまま機能します。

### バックエンドの選択肢

| バックエンドクレート | 出力先 | ユースケース |
|--------------|--------|----------|
| `env_logger` | 標準エラー出力（stderr）、色付き | 開発、シンプルなCLIツール |
| `tracing-subscriber` | 標準エラー出力（stderr）、フォーマット済み | `tracing` を用いた本番環境 |
| `syslog` | システムの syslog | Linuxシステムサービス |
| `tracing-journald` | systemd ジャーナル | systemd管理のサービス |
| `tracing-appender` | ローテーション付きログファイル | 長時間実行されるデーモン |
| `tracing-opentelemetry` | OpenTelemetry コレクター | 分散トレーシング |

----
