# リファレンスカード

> **14以上の「構造的に正しくする（correct-by-construction）」パターンのクイックリファレンス**。選定フローチャート、パターンカタログ、組み合わせルール、クレートマッピング、そして保証としての型のチートシートを提供します。
>
> **相互参照:** すべての章 — 本書全体のルックアップテーブルです。

## クイックリファレンス: 構造的に正しくするパターン

### パターン選定ガイド

```text
見落とした場合のバグは壊滅的か？
├── はい → 型にエンコード可能か？
│          ├── はい → 「構造的に正しくする（CORRECT-BY-CONSTRUCTION）」を採用
│          └── いいえ → ランタイムチェック ＋ 入念なテスト
└── いいえ → ランタイムチェックで十分
```

### パターンカタログ

| # | パターン | 主要なトレイト/型 | 防止対象 | ランタイムコスト | 該当章 |
|---|---------|---------------|----------|:------:|---------|
| 1 | 型付きコマンド | `trait IpmiCmd { type Response; }` | 誤ったレスポンス型 | ゼロ | 第2章 |
| 2 | 単一使用型 | `struct Nonce`（Clone/Copy なし） | ノンスや鍵の再利用 | ゼロ | 第3章 |
| 3 | ケイパビリティトークン | `struct AdminToken { _private: () }` | 未認可のアクセス | ゼロ | 第4章 |
| 4 | 型状態（タイプステート） | `Session<Active>` | プロトコル違反 | ゼロ | 第5章 |
| 5 | 次元の型 | `struct Celsius(f64)` | 単位の混同 | ゼロ | 第6章 |
| 6 | 境界でのバリデーション | `struct ValidFru`（TryFrom 経由） | 未検証データの使用 | パースの1回のみ | 第7章 |
| 7 | ケイパビリティミックスイン | `trait FanDiagMixin: HasSpi + HasI2c` | 必要なバスアクセスの欠落 | ゼロ | 第8章 |
| 8 | 幽霊型（Phantom Types） | `Register<Width16>` | レジスタ幅やアクセスの不一致 | ゼロ | 第9章 |
| 9 | 番兵値 → Option | `Option<u8>`（`0xFF` ではない） | 番兵値を通常値として扱うバグ | ゼロ | 第11章 |
| 10 | シールドトレイト | `trait Cmd: private::Sealed` | 健全でない外部実装 | ゼロ | 第11章 |
| 11 | 非網羅的な列挙型 | `#[non_exhaustive] enum Sku` | match のサイレントなフォールスルー | ゼロ | 第11章 |
| 12 | 型状態ビルダー | `DerBuilder<Set, Missing>` | 不完全なオブジェクト構築 | ゼロ | 第11章 |
| 13 | FromStr によるバリデーション | `impl FromStr for DiagLevel` | 未検証の文字列入力 | パースの1回のみ | 第11章 |
| 14 | const ジェネリクスサイズ | `RegisterBank<const N: usize>` | バッファサイズの不一致 | ゼロ | 第11章 |
| 15 | 安全な unsafe ラッパー | `MmioRegion::read_u32()` | チェックされていない MMIO / FFI | ゼロ | 第11章 |
| 16 | 非同期型状態 | `AsyncSession<Active>` | 非同期プロトコル違反 | ゼロ | 第11章 |
| 17 | const アサーション | `SdrSensorId<const N: u8>` | 無効なコンパイル時 ID | ゼロ | 第11章 |
| 18 | セッション型 | `Chan<SendRequest>` | 順序違いのチャネル操作 | ゼロ | 第11章 |
| 19 | 自己参照用の Pin | `Pin<Box<StreamParser>>` | 構造体内部を指すダングリングポインタ | ゼロ | 第11章 |
| 20 | RAII / Drop | `impl Drop for Session` | 任意の終了パスでのリソースリーク | ゼロ | 第11章 |
| 21 | エラー型の階層構造 | `#[derive(Error)] enum DiagError` | サイレントなエラーの握りつぶし | ゼロ | 第11章 |
| 22 | `#[must_use]` | `#[must_use] struct Token` | 値のサイレントなドロップ | ゼロ | 第11章 |

### 組み合わせルール

```text
ケイパビリティトークン + 型状態 = 認可された状態遷移
型付きコマンド + 次元の型 = 物理的に型付けされたレスポンス
境界でのバリデーション + 幽霊型 = バリデーション済み設定に基づく型付きレジスタアクセス
ケイパビリティミックスイン + 型付きコマンド = バスを意識した型付き操作
単一使用型 + 型状態 = 遷移時に消費されるプロトコル
シールドトレイト + 型付きコマンド = 閉じた健全なコマンドセット
番兵値 → Option + 境界でのバリデーション = クリーンな1回パースパイプライン
型状態ビルダー + ケイパビリティトークン = 完全な構築の証明
FromStr + #[non_exhaustive] = 拡張可能でフェイルファストな列挙型パース
const ジェネリクスサイズ + 境界でのバリデーション = サイズ保証され検証されたプロトコルバッファ
安全な unsafe ラッパー + 幽霊型 = 型付けされた安全な MMIO アクセス
非同期型状態 + ケイパビリティトークン = 認可された非同期遷移
セッション型 + 型付きコマンド = 完全に型付けされたリクエスト・レスポンスチャネル
Pin + 型状態 = ムーブ不能な自己参照状態機械
RAII (Drop) + 型状態 = 状態に応じたクリーンアップ保証
エラー階層 + 境界でのバリデーション = 網羅的な処理を伴う型付きパースエラー
#[must_use] + 単一使用型 = 無視できず、再利用もできないトークン
```

### 避けるべきアンチパターン

| アンチパターン | なぜ誤りなのか | 正しい代替案 |
|-------------|---------------|-------------------|
| `fn read_sensor() -> f64` | 単位がない — °C、°F、RPM のどれか不明 | `fn read_sensor() -> Celsius` |
| `fn encrypt(nonce: &[u8; 12])` | ノンスが再利用可能（借用） | `fn encrypt(nonce: Nonce)`（ムーブ） |
| `fn admin_op(is_admin: bool)` | 呼び出し元が嘘をつける（`true`） | `fn admin_op(_: &AdminToken)` |
| `fn send(session: &Session)` | 状態の保証がない | `fn send(session: &Session<Active>)` |
| `fn process(data: &[u8])` | 検証されていない | `fn process(data: &ValidFru)` |
| エフェメラル鍵に対する `Clone` | 単一使用の保証を破綻させる | Clone を derive しない |
| `let vendor_id: u16 = 0xFFFF` | 番兵値が内部に持ち越される | `let vendor_id: Option<u16> = None` |
| フォールバック付きの `fn route(level: &str)` | タイポがサイレントにデフォルト値になる | `let level: DiagLevel = s.parse()?` |
| フィールド未設定の `Builder::new().finish()` | 不完全なオブジェクトが構築される | 型状態ビルダー: `finish()` を `Set` でゲート |
| 固定長ハードウェアバッファに対する `let buf: Vec<u8>` | サイズが実行時にしかチェックされない | `RegisterBank<4096>`（const ジェネリクス） |
| 散乱した生の `unsafe { ptr::read(...) }` | 未定義動作（UB）のリスク、監査不能 | `MmioRegion::read_u32()` 安全なラッパー |
| `async fn transition(&mut self)` | 可変借用では状態を強制できない | `async fn transition(self) -> NextState` |
| 手動で呼び出される `fn cleanup()` | 早期リターンやパニック時に呼び出しを忘れる | `impl Drop` — コンパイラが呼び出しを挿入 |
| `fn op() -> Result<T, String>` | 不透明なエラー、バリアントのマッチが不可 | `fn op() -> Result<T, DiagError>` 列挙型 |

### 診断コードベースへのマッピング

| モジュール | 適用可能なパターン |
|---------------------|----------------------|
| `protocol_lib` | 型付きコマンド、型状態セッション |
| `thermal_diag` | ケイパビリティミックスイン、次元の型 |
| `accel_diag` | 境界でのバリデーション、幽霊型レジスタ |
| `network_diag` | 型状態（リンクトレーニング）、ケイパビリティトークン |
| `pci_topology` | 幽霊型（レジスタ幅）、バリデーション済み設定、番兵値 → Option |
| `event_handler` | 単一使用の監査トークン、ケイパビリティトークン、FromStr (Component) |
| `event_log` | 境界でのバリデーション（SEL レコードパース） |
| `compute_diag` | 次元の型（温度、周波数） |
| `memory_diag` | 境界でのバリデーション（SPD データ）、次元の型 |
| `switch_diag` | 型状態（ポート列挙）、幽霊型 |
| `config_loader` | FromStr (DiagLevel, FaultStatus, DiagAction) |
| `log_analyzer` | 境界でのバリデーション（CompiledPatterns） |
| `diag_framework` | 型状態ビルダー (DerBuilder)、セッション型 (orchestrator↔worker) |
| `topology_lib` | const ジェネリクスレジスタバンク、安全な MMIO ラッパー |

### 保証としての型 — クイックマッピング

| 保証内容 | Rustでの表現 | 例 |
|-----------|----------------|---------|
| 「この証明が存在する」 | 型 | `AdminToken` |
| 「私は証明を所持している」 | その型の値 | `let tok = authenticate()?;` |
| 「A は B を含意する」 | 関数 `fn(A) -> B` | `fn activate(AdminToken) -> Session<Active>` |
| 「A と B の両方」 | タプル `(A, B)` または複数パラメータ | `fn op(a: &AdminToken, b: &LinkTrained)` |
| 「A または B のいずれか」 | `enum { A(A), B(B) }` または `Result<A, B>` | `Result<Session<Active>, Error>` |
| 「常に真」 | `()`（ユニット型） | 常に構築可能 |
| 「不可能」 | `!`（never型）または `enum Void {}` | 決して構築できない |

---
