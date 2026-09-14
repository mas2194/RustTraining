# ケーススタディ3: フレームワーク間通信 → ライフタイム借用

> **学習内容:** C++における生ポインタを用いたフレームワーク間通信パターンを、Rustのライフタイムに基づく借用システムへと変換する方法。ゼロコスト抽象化を維持しながら、ダングリングポインタのリスクを排除します。

## C++のパターン: フレームワークへの生ポインタ
```cpp
// C++の原型: すべての診断モジュールがフレームワークへの生ポインタを保持
class DiagBase {
protected:
    DiagFramework* m_pFramework;  // 生ポインタ — 所有者は誰か？
public:
    DiagBase(DiagFramework* fw) : m_pFramework(fw) {}
    
    void LogEvent(uint32_t code, const std::string& msg) {
        m_pFramework->GetEventLog()->Record(code, msg);  // 生存していることを祈るしかない！
    }
};
// 問題点: m_pFramework はライフタイム保証のない生ポインタ
// モジュールが参照している間にフレームワークが破棄されると未定義動作（UB）になる
```

## Rustの解決策: ライフタイム借用を用いた DiagContext
```rust
// 実装例: module.rs — 保持するのではなく借用する

/// 実行中に診断モジュールへと渡されるコンテキスト。
/// ライフタイム 'a により、フレームワークがコンテキストよりも長く生存することが保証される。
pub struct DiagContext<'a> {
    pub der_log: &'a mut EventLogManager,
    pub config: &'a ModuleConfig,
    pub framework_opts: &'a HashMap<String, String>,
}

/// モジュールはパラメータとしてコンテキストを受け取る — フレームワークのポインタを保持することは決してない
pub trait DiagModule {
    fn id(&self) -> &str;
    fn execute(&mut self, ctx: &mut DiagContext) -> DiagResult<()>;
    fn pre_execute(&mut self, _ctx: &mut DiagContext) -> DiagResult<()> {
        Ok(())
    }
    fn post_execute(&mut self, _ctx: &mut DiagContext) -> DiagResult<()> {
        Ok(())
    }
}
```

### 重要な知見
- C++のモジュールはフレームワークへのポインタを**保持（store）**します（危険性: フレームワークが先に破棄されたらどうなるか？）
- Rustのモジュールは関数の引数としてコンテキストを**受け取り（receive）**ます — 借用チェッカが呼び出し中のフレームワークの生存を保証します
- 生ポインタも、ライフタイムの曖昧さも、「生存していることを祈る」必要もありません

----

# ケーススタディ4: 神オブジェクト（God object） → コンポーザブルな状態管理

## C++のパターン: モノリシックなフレームワーククラス
```cpp
// C++の原型: フレームワークが神オブジェクト化している
class DiagFramework {
    // ヘルスモニタのトラップ処理
    std::vector<AlertTriggerInfo> m_alertTriggers;
    std::vector<WarnTriggerInfo> m_warnTriggers;
    bool m_healthMonHasBootTimeError;
    uint32_t m_healthMonActionCounter;
    
    // GPU診断
    std::map<uint32_t, GpuPcieInfo> m_gpuPcieMap;
    bool m_isRecoveryContext;
    bool m_healthcheckDetectedDevices;
    // ... 他に30個以上のGPU関連フィールド
    
    // PCIeツリー
    std::shared_ptr<CPcieTreeLinux> m_pPcieTree;
    
    // イベントログ
    CEventLogMgr* m_pEventLogMgr;
    
    // ... その他複数のメソッド
    void HandleGpuEvents();
    void HandleNicEvents();
    void RunGpuDiag();
    // すべてがすべてに依存している
};
```

## Rustの解決策: コンポーザブルな状態構造体
```rust
// 実装例: main.rs — 関心事ごとに特化した構造体に状態を分解

#[derive(Default)]
struct HealthMonitorState {
    alert_triggers: Vec<AlertTriggerInfo>,
    warn_triggers: Vec<WarnTriggerInfo>,
    health_monitor_action_counter: u32,
    health_monitor_has_boot_time_error: bool,
    // ヘルスモニタ関連のフィールドのみ
}

#[derive(Default)]
struct GpuDiagState {
    gpu_pcie_map: HashMap<u32, GpuPcieInfo>,
    is_recovery_context: bool,
    healthcheck_detected_devices: bool,
    // GPU関連のフィールドのみ
}

/// フレームワークはすべてをフラットに所有するのではなく、これらの状態を合成（コンポーズ）する
struct DiagFramework {
    ctx: DiagContext,             // 実行コンテキスト
    args: Args,                   // CLI引数
    pcie_tree: Option<DeviceTree>,  // shared_ptr は不要
    event_log_mgr: EventLogManager,   // 生ポインタではなく所有
    fc_manager: FcManager,        // フォールトコード管理
    health: HealthMonitorState,   // ヘルスモニタの状態 — 独立した構造体
    gpu: GpuDiagState,           // GPUの状態 — 独立した構造体
}
```

### 重要な知見
- **テスト容易性**: 各状態構造体を個別に単体テスト可能
- **可読性**: `self.health.alert_triggers` 対 `m_alertTriggers` — 所有関係が明確
- **安心できるリファクタリング**: `GpuDiagState` を変更しても、ヘルスモニタの処理に誤って影響を与えることがない
- **巨大メソッド群（メソッドスープ）の解消**: ヘルスモニタの状態のみを必要とする関数は、フレームワーク全体ではなく `&mut HealthMonitorState` のみを受け取る

----

# ケーススタディ5: トレイトオブジェクト — それが「真に適切な」場面

- すべてを enum にすべきというわけではありません！ **診断モジュールのプラグインシステム**は、トレイトオブジェクトの真価が発揮されるユースケースです
- なぜでしょうか？ それは診断モジュールが**拡張に対して開かれている（Open for Extension）**ためです — フレームワークを変更することなく、新しいモジュールを追加できます

```rust
// 実装例: framework.rs — ここでは Vec<Box<dyn DiagModule>> の使用が適切
pub struct DiagFramework {
    modules: Vec<Box<dyn DiagModule>>,        // 実行時ポリモーフィズム
    pre_diag_modules: Vec<Box<dyn DiagModule>>,
    event_log_mgr: EventLogManager,
    // ...
}

impl DiagFramework {
    /// 診断モジュールを登録する — DiagModule を実装する任意の型
    pub fn register_module(&mut self, module: Box<dyn DiagModule>) {
        info!("モジュールを登録中: {}", module.id());
        self.modules.push(module);
    }
}
```

### 各パターンの使い分け

| **ユースケース** | **パターン** | **理由** |
|-------------|-----------|--------|
| コンパイル時に既知の固定バリアント群 | `enum` + `match` | 網羅性チェック、vtableなし |
| ハードウェアイベント型（Degrade, Fatal, Bootなど） | `enum GpuEventKind` | すべてのバリアントが既知、パフォーマンスが重視される |
| PCIeデバイス型（GPU, NIC, Switchなど） | `enum PcieDeviceKind` | 固定されたバリアント群、各バリアントが異なるデータを保持 |
| プラグイン/モジュールシステム（拡張に対してオープン） | `Box<dyn Trait>` | フレームワークを変更せずに新しいモジュールを追加可能 |
| テスト用のモック | `Box<dyn Trait>` | テストダブル（代役オブジェクト）の注入 |

### 演習: 移行する前に考えてみよう
以下のC++コードがあるとします:
```cpp
class Shape { public: virtual double area() = 0; };
class Circle : public Shape { double r; double area() override { return 3.14*r*r; } };
class Rect : public Shape { double w, h; double area() override { return w*h; } };
std::vector<std::unique_ptr<Shape>> shapes;
```
**問題**: Rustへの移行において、`enum Shape` と `Vec<Box<dyn Shape>>` のどちらを使うべきでしょうか？

<details><summary>解答（クリックして展開）</summary>

**解答**: `enum Shape` — 図形の種類が**閉じている**（コンパイル時にすべて判明している）ためです。`Box<dyn Shape>` を使うべきなのは、実行時にユーザーが新しい図形型を追加できるようにしたい場合のみです。

```rust
// 正しいRustへの移行例:
enum Shape {
    Circle { r: f64 },
    Rect { w: f64, h: f64 },
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { r } => std::f64::consts::PI * r * r,
            Shape::Rect { w, h } => w * h,
        }
    }
}

fn main() {
    let shapes: Vec<Shape> = vec![
        Shape::Circle { r: 5.0 },
        Shape::Rect { w: 3.0, h: 4.0 },
    ];
    for shape in &shapes {
        println!("面積: {:.2}", shape.area());
    }
}
// 出力:
// 面積: 78.54
// 面積: 12.00
```

</details>

----

# 移行メトリクスと得られた教訓

## 得られた教訓
1. **Enumディスパッチをデフォルトにする** — 約10万行のC++の中で、`Box<dyn Trait>` が真に必要だったのは約25箇所（プラグインシステムやテストモック）のみでした。他の約900個の仮想メソッドはすべて enum と match に置き換えられました。
2. **アリーナパターンによる循環参照の排除** — `shared_ptr` や `enable_shared_from_this` は所有権が曖昧であることの兆候です。まず誰がデータを**所有**しているのかを考えてください。
3. **ポインタを保持せず、コンテキストを渡す** — ライフタイム境界を持つ `DiagContext<'a>` を渡す設計は、すべてのモジュールに `Framework*` を保持させるよりも安全で明確です。
4. **神オブジェクトの分解** — ある構造体に30以上のフィールドがあるなら、それはおそらく3〜4個の構造体が1つにまとまってしまっている状態です。
5. **コンパイラはペアプログラマ** — 約400箇所の `dynamic_cast` 呼び出しは、約400箇所の潜在的な実行時エラーを意味していました。Rustにおいて `dynamic_cast` に相当するものがゼロになったことで、実行時の型エラーもゼロになりました。

## 最も苦労した点
- **ライフタイム注釈**: 生ポインタに慣れていると借用を正しく記述するのに時間がかかります — しかし、一度コンパイルが通れば、それは正しさが保証されたことになります。
- **借用チェッカとの戦い**: `&mut self` を2箇所で同時に使いたくなるケース。解決策：状態を別々の構造体に分解すること。
- **直訳の誘惑を抑えること**: いたる所に `Vec<Box<dyn Base>>` と書きたくなる衝動。問いかけるべきこと: 「このバリアントの集合は閉じているか？」→ Yesであれば enum を使用します。

## C++開発チームへの推奨事項
1. 小さく自己完結したモジュールから始める（神オブジェクトから始めない）
2. 振る舞い（ロジック）の前に、まずデータ構造を移行する
3. コンパイラに導いてもらう — Rustのエラーメッセージは極めて優秀です
4. `dyn Trait` に手を伸ばす前に、まず `enum` を検討する
5. 統合前に [Rust Playground](https://play.rust-lang.org/) を使ってパターンをプロトタイピングする

----
