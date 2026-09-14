## クイックリファレンスカード

### パターン決定ガイド

```text
プリミティブ型に型安全性が必要？
└── ニュータイプ（Newtype）パターン（第3章）

コンパイル時に状態遷移を強制したい？
└── 型状態（タイプステート）パターン（第3章）

実行時データを持たない「タグ」が必要？
└── PhantomData（第4章）

Rc/Arc の循環参照を解消したい？
└── Weak<T> / sync::Weak<T>（第8章）

ビジーループなしで条件を満たすまで待機したい？
└── Condvar + Mutex（第6章）

「N 個の型のうちの 1 つ」を扱いたい？
├── 既知の閉じたセット → Enum
├── オープンなセット、ホットパス → ジェネリクス
├── オープンなセット、コールドパス → dyn Trait
└── 完全に未知の型 → Any + TypeId（第2章）

スレッド間で状態を共有したい？
├── 単純なカウンタ / フラグ → アトミック（Atomics）
├── 短いクリティカルセクション → Mutex
├── 読み取り多頻度 → RwLock
├── 遅延ワンタイム初期化 → OnceLock / LazyLock（第6章）
└── 複雑な状態 → アクター + チャンネル

計算を並列化したい？
├── コレクションの処理 → rayon::par_iter
├── バックグラウンドタスク → thread::spawn
└── ローカルデータの借用 → thread::scope

非同期 I/O や並行ネットワーク処理が必要？
├── 基本 → tokio + async/await（第16章）
└── 高度（ストリーム、ミドルウェア） → Async Rust Training を参照

エラー処理が必要？
├── ライブラリ → thiserror (#[derive(Error)])
└── アプリケーション → anyhow (Result<T>)

値がムーブされるのを防ぎたい？
└── Pin<T>（第8章） — Future や自己参照型に必須
```

### トレイト境界チートシート

| 境界 | 意味 |
|-------|---------|
| `T: Clone` | 複製可能 |
| `T: Send` | 別のスレッドにムーブ可能 |
| `T: Sync` | `&T` をスレッド間で共有可能 |
| `T: 'static` | 非 static な参照を含まない |
| `T: Sized` | コンパイル時にサイズが既知（デフォルト） |
| `T: ?Sized` | サイズが未確定の可能性がある（`[T]`, `dyn Trait`） |
| `T: Unpin` | ピン留め後も安全にムーブ可能 |
| `T: Default` | デフォルト値を持つ |
| `T: Into<U>` | `U` に変換可能 |
| `T: AsRef<U>` | `&U` として借用可能 |
| `T: Deref<Target = U>` | `&U` へ自動参照外し（auto-deref）可能 |
| `F: Fn(A) -> B` | 呼び出し可能、状態を不変借用する |
| `F: FnMut(A) -> B` | 呼び出し可能、状態を可変化できる |
| `F: FnOnce(A) -> B` | 1回のみ呼び出し可能、状態を消費（ムーブ）する可能性がある |

### ライフタイム省略ルール

コンパイラは以下の3つのケースにおいて、ライフタイムを自動的に補完します（そのため手動で記述する必要がありません）:

```rust
// ルール 1: 各参照パラメータはそれぞれ固有のライフタイムを持つ
// fn foo(x: &str, y: &str)  →  fn foo<'a, 'b>(x: &'a str, y: &'b str)

// ルール 2: 入力ライフタイムがちょうど 1 つだけ存在する場合、それがすべての出力に適用される
// fn foo(x: &str) -> &str   →  fn foo<'a>(x: &'a str) -> &'a str

// ルール 3: パラメータの 1 つが &self または &mut self である場合、そのライフタイムが出力に適用される
// fn foo(&self, x: &str) -> &str  →  fn foo<'a>(&'a self, x: &str) -> &'a str
```

**明示的なライフタイムの記述が必須となるケース**:
- 複数の入力参照があり、かつ戻り値が参照である場合（どの入力のライフタイムを引き継ぐかをコンパイラが判断できない）
- 参照を保持する構造体のフィールド: `struct Ref<'a> { data: &'a str }`
- 借用参照を持たないデータを要求する際の `'static` 境界

### よく使われる Derive トレイト

```rust
#[derive(
    Debug,          // {:?} フォーマット出力
    Clone,          // .clone()
    Copy,           // 暗黙のコピー（単純な型のみ）
    PartialEq, Eq,  // == 比較
    PartialOrd, Ord, // < > 比較 + ソート
    Hash,           // HashMap/HashSet のキー
    Default,        // Type::default()
)]
struct MyType { /* ... */ }
```

### モジュール可視性クイックリファレンス

```text
pub           → すべての場所から可視
pub(crate)    → クレート内でのみ可視
pub(super)    → 親モジュールから可視
pub(in path)  → 指定されたパス内から可視
(なし)        → 現在のモジュールとその子モジュールにのみ非公開（private）
```

### 参考資料・推薦図書

| リソース | 推薦理由 |
|----------|-----|
| [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) | 慣用的なパターンとアンチパターンのカタログ |
| [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) | 洗練された公開 API のための公式チェックリスト |
| [Rust Atomics and Locks](https://marabos.nl/atomics/) | Mara Bos 氏による並行性プリミティブの徹底解説 |
| [The Rustonomicon](https://doc.rust-lang.org/nomicon/) | Unsafe Rust とダークサイドに関する公式ガイド |
| [Error Handling in Rust](https://blog.burntsushi.net/rust-error-handling/) | Andrew Gallant 氏による包括的なエラー処理ガイド |
| [Jon Gjengset — Crust of Rust series](https://www.youtube.com/playlist?list=PLqbS7AVVErFiWDOAVrPt7aYmnuuOLYvOa) | イテレータ、ライフタイム、チャンネルなどの深掘り動画シリーズ |
| [Effective Rust](https://www.lurklurk.org/effective-rust/) | Rust コードを改善するための35の具体的な実践手法 |

***

*Rust パターン & エンジニアリング How-To 完*
