# C# 開発者のための Rust ブートストラップ

C# の開発経験を持つエンジニアのための、体系的な Rust 入門ガイドです。本ガイドは実績ある教育的アプローチに従い、Rust が「どのように動作するか」だけでなく、「なぜそのように設計されたのか」を段階的に理解できるように構成されています。

## コース概要
- **Rust を学ぶ意義** — C# 開発者にとってなぜ Rust が重要なのか
- **導入手順** — インストール、ツールチェーン、最初のプログラム
- **基本構成要素** — 型、変数、制御フロー
- **データ構造** — 配列、タプル、構造体
- **パターンマッチングと列挙型** — Rust の中核概念
- **モジュールとクレート** — コードの構成と依存関係（.NET アセンブリとの比較）
- **トレイトとジェネリクス** — 高度な型システム
- **エラー処理** — 安全性に対する Rust のアプローチ
- **メモリ管理** — 所有権、借用、ライフタイム
- **実践的な移行** — 実践的な移行例
- **ベストプラクティス** — C# 開発者のためのイディオマティックな Rust

## 目次

### 1. 導入と動機
- [クイックリファレンス: Rust vs C#](#quick-reference-rust-vs-c)
- [C# 開発者にとっての Rust の意義](#the-case-for-rust-for-c-developers)
- [Rust が解決する C# の代表的な課題](#common-c-pain-points-that-rust-addresses)
- [C# ではなく Rust を選択すべき場面](#when-to-choose-rust-over-c)

### 2. 導入手順
- [インストールとセットアップ](#installation-and-setup)
- [最初の Rust プログラム](#your-first-rust-program)
- [Cargo vs NuGet/MSBuild](#cargo-vs-nugetmsbuild)
- [C# 開発者のための IDE セットアップ](#ide-setup-for-c-developers)

### 3. 基本的な型と変数
- [組み込み型の比較](#built-in-types-comparison)
- [変数と可変性](#variables-and-mutability)
- [文字列型: String vs &str](#string-types-string-vs-str)
- [コメントとドキュメント](#comments-and-documentation)

### 4. 制御フロー
- [条件分岐](#conditional-statements)
- [ループと反復処理](#loops-and-iteration)
- [式ブロック](#expression-blocks)
- [関数 vs メソッド](#functions-vs-methods)

### 5. データ構造
- [配列とスライス](#arrays-and-slices)
- [タプル](#tuples)
- [構造体 vs クラス](#structs-vs-classes)
- [参照と借用の基礎](#references-and-borrowing-basics)

### 6. パターンマッチングと列挙型
- [列挙型 vs C# enum](#enums-vs-c-enums)
- [match 式](#match-expressions)
- [null 安全性のための Option<T>](#optiont-for-null-safety)
- [エラー処理のための Result<T, E>](#resultt-e-for-error-handling)

### 7. モジュールとクレート
- [Rust のモジュール vs C# の名前空間](#rust-modules-vs-c-namespaces)
- [クレート vs .NET アセンブリ](#crates-vs-net-assemblies)
- [パッケージ管理: Cargo vs NuGet](#package-management-cargo-vs-nuget)
- [可視性とアクセス制御](#visibility-and-access-control)

### 8. トレイトとジェネリクス
- [トレイト vs インターフェイス](#traits-vs-interfaces)
- [ジェネリック型と関数](#generic-types-and-functions)
- [トレイト境界と制約](#trait-bounds-and-constraints)
- [標準ライブラリの代表的なトレイト](#common-standard-library-traits)

### 9. コレクションとエラー処理
- [Vec<T> vs List<T>](#vect-vs-listt)
- [HashMap vs Dictionary](#hashmap-vs-dictionary)
- [イテレータパターン](#iterator-patterns)
- [包括的なエラー処理](#comprehensive-error-handling)

### 10. メモリ管理
- [所有権の理解](#understanding-ownership)
- [ムーブセマンティクス vs 参照セマンティクス](#move-semantics-vs-reference-semantics)
- [借用とライフタイム](#borrowing-and-lifetimes)
- [スマートポインタ](#smart-pointers)

### 11. 実践的な移行例
- [設定管理](#configuration-management)
- [データ処理パイプライン](#data-processing-pipelines)
- [HTTP クライアントと API](#http-clients-and-apis)
- [ファイル I/O とシリアライズ](#file-io-and-serialization)

### 12. 次のステップとベストプラクティス
- [Rust と C# のテスト比較](#testing-in-rust-vs-c)
- [C# 開発者が陥りやすい落とし穴](#common-pitfalls-for-c-developers)
- [学習ロードマップとリソース](#learning-path-and-resources)
- [高度なトピックへのステップアップ](#moving-to-advanced-topics)

***

## クイックリファレンス: Rust vs C#

| **概念** | **C#** | **Rust** | **主な相違点** |
|-------------|--------|----------|-------------------|
| メモリ管理 | ガベージコレクタ (GC) | 所有権システム | ゼロコストで確定的なクリーンアップ |
| null 参照 | 至る所に `null` が存在可能 | `Option<T>` | コンパイル時の null 安全性 |
| エラー処理 | 例外 (Exception) | `Result<T, E>` | 明示的で隠れた制御フローがない |
| 可変性 | デフォルトで可変 (mutable) | デフォルトで不変 (immutable) | 可変にするには明示的な宣言が必要 |
| 型システム | 参照型 / 値型 | 所有権型 | ムーブセマンティクス、借用 |
| アセンブリ | GAC、AppDomain（.NET Framework）; Side-by-side（.NET 5+） | クレート (Crate) | 静的リンク、ランタイム不要 |
| 名前空間 | `using System.IO;` | `use std::fs;` | モジュールシステム |
| インターフェイス | `interface IFoo` | `trait Foo` | デフォルト実装 |
| ジェネリクス | `List<T>`（`where` による制約は任意） | `Vec<T>`（`T: Clone` などのトレイト境界） | ゼロコスト抽象化 |
| スレッド処理 | lock、async/await | 所有権 + Send/Sync | データ競合の防止 |
| パフォーマンス | JIT コンパイル | AOT コンパイル | 予測可能、GC による一時停止がない |

***

## C# 開発者にとっての Rust の意義

### ランタイムのオーバーヘッドを伴わないパフォーマンス
```csharp
// C# - 高い生産性、ランタイムのオーバーヘッド
public class DataProcessor
{
    private List<int> data = new List<int>();
    
    public void ProcessLargeDataset()
    {
        // アロケーションにより GC がトリガーされる
        for (int i = 0; i < 10_000_000; i++)
        {
            data.Add(i * 2); // GC への負荷
        }
        // 処理中に予測不可能な GC 一時停止（GC ポーズ）が発生する可能性がある
    }
}
// 実行時間: 変動あり（GC により 50〜200ms）
// メモリ使用量: 約 80MB（GC オーバーヘッドを含む）
// 予測可能性: 低（GC 一時停止のため）
```

```rust
// Rust - 同等の表現力、ランタイムオーバーヘッドはゼロ
struct DataProcessor {
    data: Vec<i32>,
}

impl DataProcessor {
    fn process_large_dataset(&mut self) {
        // ゼロコスト抽象化
        for i in 0..10_000_000 {
            self.data.push(i * 2); // GC への負荷なし
        }
        // 確定的なパフォーマンス
    }
}
// 実行時間: 一貫（約 30ms）
// メモリ使用量: 約 40MB（正確なアロケーション）
// 予測可能性: 高（GC なし）
```

### ランタイムチェックに頼らないメモリ安全性
```csharp
// C# - オーバーヘッドを伴うランタイム安全性
public class UnsafeOperations
{
    public string ProcessArray(int[] array)
    {
        // ランタイムでの境界チェック
        if (array.Length > 0)
        {
            return array[0].ToString(); // NullReferenceException が発生する可能性あり
        }
        return null; // null の伝播
    }
    
    public void ProcessConcurrently()
    {
        var list = new List<int>();
        
        // データ競合の可能性があり、慎重なロックが必要
        Parallel.For(0, 1000, i =>
        {
            lock (list) // ランタイムのオーバーヘッド
            {
                list.Add(i);
            }
        });
    }
}
```

```rust
// Rust - ランタイムコストゼロのコンパイル時安全性
struct SafeOperations;

impl SafeOperations {
    // コンパイル時の null 安全性、ランタイムチェックなし
    fn process_array(array: &[i32]) -> Option<String> {
        array.first().map(|x| x.to_string())
        // null 参照が発生することはあり得ない
        // 安全性が証明可能な場合は境界チェックが最適化により除去される
    }
    
    fn process_concurrently() {
        use std::sync::Mutex;
        use std::thread;
        
        let data = Mutex::new(Vec::new());
        
        // データ競合はコンパイル時に防止される
        let handles: Vec<_> = (0..1000).map(|i| {
            let data = &data;
            thread::spawn(move || {
                data.lock().unwrap().push(i);
                // シングルスレッド時にはロックのオーバーヘッドなし
            })
        }).collect();
        
        for handle in handles {
            handle.join().unwrap();
        }
    }
}
```

***

## Rust が解決する C# の代表的な課題

### 1. 10億ドルの誤り: Null 参照
```csharp
// C# - NullReferenceException は実行時の時限爆弾
public class UserService
{
    public string GetUserDisplayName(User user)
    {
        // これらのいずれも NullReferenceException をスローする可能性がある
        return user.Profile.DisplayName.ToUpper();
        //     ^^^^^ ^^^^^^^ ^^^^^^^^^^^ ^^^^^^^
        //     実行時に null の可能性がある
    }
    
    // null 許容参照型 (C# 8+) は役立つが、null がすり抜ける可能性は依然として残る
    public string GetDisplayName(User? user)
    {
        return user?.Profile?.DisplayName?.ToUpper() ?? "Unknown";
        // この行自体は ?. と ?? のおかげで null 安全だが、
        // null 許容参照型は警告に過ぎず、`!` でコンパイラを上書きできてしまう
    }
}
```

```rust
// Rust - コンパイル時に null 安全性を保証
struct UserService;

impl UserService {
    fn get_user_display_name(user: &User) -> Option<String> {
        user.profile.as_ref()?
            .display_name.as_ref()
            .map(|name| name.to_uppercase())
        // コンパイラにより None ケースの処理が強制される
        // ヌルポインタ例外が発生することはあり得ない
    }
    
    fn get_display_name_safe(user: Option<&User>) -> String {
        user.and_then(|u| u.profile.as_ref())
            .and_then(|p| p.display_name.as_ref())
            .map(|name| name.to_uppercase())
            .unwrap_or_else(|| "Unknown".to_string())
        // 明示的な処理、予期せぬ挙動なし
    }
}
```

### 2. 隠れた例外と制御フロー
```csharp
// C# - 例外はどこからでもスローされる可能性がある
public async Task<UserData> GetUserDataAsync(int userId)
{
    // これらはそれぞれ異なる例外をスローする可能性がある
    var user = await userRepository.GetAsync(userId);        // SqlException
    var permissions = await permissionService.GetAsync(user); // HttpRequestException  
    var preferences = await preferenceService.GetAsync(user); // TimeoutException
    
    return new UserData(user, permissions, preferences);
    // 呼び出し元はどんな例外を予期すべきか分からない
}
```

```rust
// Rust - すべてのエラーが関数シグネチャに明示される
#[derive(Debug)]
enum UserDataError {
    DatabaseError(String),
    NetworkError(String),
    Timeout,
    UserNotFound(i32),
}

async fn get_user_data(user_id: i32) -> Result<UserData, UserDataError> {
    // すべてのエラーが明示的かつ処理済み
    let user = user_repository.get(user_id).await
        .map_err(UserDataError::DatabaseError)?;
    
    let permissions = permission_service.get(&user).await
        .map_err(UserDataError::NetworkError)?;
    
    let preferences = preference_service.get(&user).await
        .map_err(|_| UserDataError::Timeout)?;
    
    Ok(UserData::new(user, permissions, preferences))
    // 呼び出し元は発生し得るエラーを完全に把握できる
}
```

### 3. GC による予測不可能なパフォーマンス
```csharp
// C# - GC はいつでも一時停止する可能性がある
public class HighFrequencyTrader
{
    private List<Trade> trades = new List<Trade>();
    
    public void ProcessMarketData(MarketTick tick)
    {
        // アロケーションが最悪のタイミングで GC をトリガーする可能性がある
        var analysis = new MarketAnalysis(tick);
        trades.Add(new Trade(analysis.Signal, tick.Price));
        
        // 重要な取引の瞬間にここで GC 一時停止が発生するかもしれない
        // 一時停止時間: ヒープサイズに応じて 1〜100ms
    }
}
```

```rust
// Rust - 予測可能で確定的なパフォーマンス
struct HighFrequencyTrader {
    trades: Vec<Trade>,
}

impl HighFrequencyTrader {
    fn process_market_data(&mut self, tick: MarketTick) {
        // アロケーション不要、予測可能なパフォーマンス
        let analysis = MarketAnalysis::from(tick);
        self.trades.push(Trade::new(analysis.signal(), tick.price));
        
        // GC 一時停止なし、1マイクロ秒未満の一貫したレイテンシ
        // パフォーマンスは型システムによって保証される
    }
}
```

***

## C# ではなく Rust を選択すべき場面

### ✅ Rust を選択すべき場合:
- **パフォーマンスが極めて重要な場合**: リアルタイムシステム、高頻度取引 (HFT)、ゲームエンジン
- **メモリ使用量が重要な場合**: 組込みシステム、クラウドコストの削減、モバイルアプリ
- **予測可能性が求められる場合**: 医療機器、車載システム、金融システム
- **セキュリティが最重要の場合**: 暗号処理、ネットワークセキュリティ、システムレベルのコード
- **長時間稼働サービス**: GC の一時停止が問題を引き起こす環境
- **リソース制約のある環境**: IoT、エッジコンピューティング
- **システムプログラミング**: CLI ツール、データベース、Web サーバー、オペレーティングシステム

### ✅ C# を維持すべき場合:
- **迅速なアプリケーション開発 (RAD)**: 一般的なビジネスアプリ、CRUD アプリケーション
- **大規模な既存コードベース**: 移行コストがメリットを上回る場合
- **チームの専門知識**: Rust の学習コストがもたらすメリットに見合わない場合
- **エンタープライズ統合**: .NET Framework や Windows への依存度が高い場合
- **GUI アプリケーション**: WPF、WinUI、Blazor などの充実したエコシステム
- **市場投入スピード重視**: 開発速度がパフォーマンスよりも優先される場合

### 🔄 両者の併用（ハイブリッドアプローチ）を検討する場合:
- **パフォーマンスが極めて重要なコンポーネントを Rust で記述**: P/Invoke 経由で C# から呼び出す
- **ビジネスロジックは C# で記述**: 慣れ親しんだ生産性の高い開発環境を活用
- **段階的な移行**: 新規マイクロサービスから Rust を採用していく

***

## 実世界でのインパクト: 企業が Rust を選ぶ理由

### Dropbox: ストレージインフラ
- **以前 (Python)**: 高い CPU 使用率、メモリオーバーヘッド
- **移行後 (Rust)**: パフォーマンス 10 倍向上、メモリ消費量 50% 削減
- **成果**: インフラコストを数百万ドル規模で削減

### Discord: 音声/動画バックエンド
- **以前 (Go)**: GC 一時停止による音声途切れの発生
- **移行後 (Rust)**: 一貫した低レイテンシのパフォーマンスを実現
- **成果**: ユーザー体験の大幅な向上、サーバー費用の削減

### Microsoft: Windows コンポーネント
- **Windows における Rust**: ファイルシステム、ネットワークスタックなどのコンポーネント
- **メリット**: パフォーマンスを損なうことなくメモリ安全性を確保
- **インパクト**: パフォーマンスを維持しつつ、セキュリティ脆弱性を大幅に低減

### これらが C# 開発者にとって重要な理由:
1. **補完的なスキル**: Rust と C# はそれぞれ異なる課題を解決する
2. **キャリアの成長**: システムプログラミングの専門知識は市場価値が非常に高い
3. **パフォーマンスの深い理解**: ゼロコスト抽象化の仕組みを深く理解できる
4. **安全性のマインドセット**: 所有権の考え方はどのプログラミング言語にも応用可能
5. **クラウドコストの削減**: パフォーマンスの向上はインフラ費用削減に直結する

***

## インストールとセットアップ

### Rust のインストール
```bash
# Rust のインストール（Windows、macOS、Linux で動作）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Windows の場合は、以下からインストーラをダウンロードすることも可能です: https://rustup.rs/
```

### Rust ツール vs C# ツール
| C# ツール | Rust の対応ツール | 用途 |
|---------|----------------|---------|
| `dotnet new` | `cargo new` | 新規プロジェクトの作成 |
| `dotnet build` | `cargo build` | プロジェクトのコンパイル |
| `dotnet run` | `cargo run` | プロジェクトの実行 |
| `dotnet test` | `cargo test` | テストの実行 |
| NuGet | Crates.io | パッケージリポジトリ |
| MSBuild | Cargo | ビルドシステム |
| Visual Studio | VS Code + rust-analyzer | 統合開発環境 (IDE) |

### IDE のセットアップ
1. **VS Code**（初心者におすすめ）
   - "rust-analyzer" 拡張機能をインストール
   - デバッグ用に "CodeLLDB" 拡張機能をインストール

2. **Visual Studio**（Windows）
   - Rust サポート拡張機能をインストール

3. **JetBrains RustRover**（フル機能 IDE）
   - C# における Rider に相当

***

## 最初の Rust プログラム

### C# の Hello World
```csharp
// Program.cs
using System;

namespace HelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello, World!");
        }
    }
}
```

### Rust の Hello World
```rust
// main.rs
fn main() {
    println!("Hello, World!");
}
```

### C# 開発者のための主な相違点
1. **クラスが不要** — 関数はトップレベルに直接定義できます
2. **名前空間（namespace）がない** — 代わりにモジュールシステムを使用します
3. **`println!` はマクロ** — 末尾の `!` に注目してください
4. **セミコロンの意味** — 末尾のセミコロンを省略すると、文ではなく「戻り値の式」になります
5. **明示的な戻り値の型が不要** — `main` は `()`（ユニット型）を返します

### 最初のプロジェクトの作成
```bash
# 新規プロジェクトの作成（'dotnet new console' に相当）
cargo new hello_rust
cd hello_rust

# 作成されるプロジェクト構造:
# hello_rust/
# ├── Cargo.toml      (.csproj ファイルに相当)
# └── src/
#     └── main.rs     (Program.cs に相当)

# プロジェクトの実行（'dotnet run' に相当）
cargo run
```

***

## Cargo vs NuGet/MSBuild

### プロジェクト設定

**C# (.csproj)**
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  <PackageReference Include="Serilog" Version="3.0.1" />
</Project>
```

**Rust (Cargo.toml)**
```toml
[package]
name = "hello_rust"
version = "0.1.0"
edition = "2021"

[dependencies]
serde_json = "1.0"    # Newtonsoft.Json に相当
log = "0.4"           # Serilog に相当
```

### 一般的な Cargo コマンド
```bash
# 新規プロジェクトの作成
cargo new my_project
cargo new my_project --lib  # ライブラリプロジェクトの作成

# ビルドと実行
cargo build          # 'dotnet build' に相当
cargo run            # 'dotnet run' に相当
cargo test           # 'dotnet test' に相当

# パッケージ管理
cargo add serde      # 依存関係の追加（'dotnet add package' に相当）
cargo update         # 依存関係の更新

# リリースビルド
cargo build --release  # 最適化ビルド
cargo run --release    # 最適化バージョンの実行

# ドキュメント
cargo doc --open     # ドキュメントを生成してブラウザで開く
```

### ワークスペース vs ソリューション

**C# ソリューション (.sln)**
```
MySolution/
├── MySolution.sln
├── WebApi/
│   └── WebApi.csproj
├── Business/
│   └── Business.csproj
└── Tests/
    └── Tests.csproj
```

**Rust ワークスペース (Cargo.toml)**
```toml
[workspace]
members = [
    "web_api",
    "business", 
    "tests"
]
```

***

## 変数と可変性

### C# の変数宣言
```csharp
// C# - 変数はデフォルトで可変 (mutable)
int count = 0;           // 可変
count = 5;               // ✅ 動作する

readonly int maxSize = 100;  // 初期化後は不変
// maxSize = 200;        // ❌ コンパイルエラー

const int BUFFER_SIZE = 1024; // コンパイル時定数
```

### Rust の変数宣言
```rust
// Rust - 変数はデフォルトで不変 (immutable)
let count = 0;           // デフォルトで不変
// count = 5;            // ❌ コンパイルエラー: 不変変数への再代入は不可

let mut count = 0;       // 明示的に可変
count = 5;               // ✅ 動作する

const BUFFER_SIZE: usize = 1024; // コンパイル時定数
```

### C# 開発者のための重要な発想の転換
```rust
// 'let' はデフォルトで 'readonly' であると考えます
let name = "John";       // C# の readonly string name = "John"; に相当
let mut age = 30;        // C# の int age = 30; に相当

// 変数のシャドーイング (Rust 固有の機能)
let spaces = "   ";      // 文字列型
let spaces = spaces.len(); // これ以降は数値型 (usize)
// これは可変（ミューテーション）とは異なり、新しい変数を再宣言しています
```

### 実践例: カウンタ
```csharp
// C# バージョン
public class Counter
{
    private int value = 0;
    
    public void Increment()
    {
        value++;  // ミューテーション
    }
    
    public int GetValue() => value;
}
```

```rust
// Rust バージョン
pub struct Counter {
    value: i32,  // デフォルトで非公開 (private)
}

impl Counter {
    pub fn new() -> Counter {
        Counter { value: 0 }
    }
    
    pub fn increment(&mut self) {  // 変更には &mut が必要
        self.value += 1;
    }
    
    pub fn get_value(&self) -> i32 {
        self.value
    }
}
```

***

## データ型の比較

### プリミティブ型

| C# の型 | Rust の型 | サイズ | 範囲 |
|---------|-----------|------|-------|
| `byte` | `u8` | 8 ビット | 0 〜 255 |
| `sbyte` | `i8` | 8 ビット | -128 〜 127 |
| `short` | `i16` | 16 ビット | -32,768 〜 32,767 |
| `ushort` | `u16` | 16 ビット | 0 〜 65,535 |
| `int` | `i32` | 32 ビット | -2³¹ 〜 2³¹-1 |
| `uint` | `u32` | 32 ビット | 0 〜 2³²-1 |
| `long` | `i64` | 64 ビット | -2⁶³ 〜 2⁶³-1 |
| `ulong` | `u64` | 64 ビット | 0 〜 2⁶⁴-1 |
| `float` | `f32` | 32 ビット | IEEE 754 |
| `double` | `f64` | 64 ビット | IEEE 754 |
| `bool` | `bool` | 1 ビット | true / false |
| `char` | `char` | 32 ビット | Unicode スカラー値 |

### サイズ型（重要！）
```csharp
// C# - int は常に 32 ビット
int arrayIndex = 0;
long fileSize = file.Length;
```

```rust
// Rust - サイズ型はポインタサイズ（32 ビットまたは 64 ビット）に一致
let array_index: usize = 0;    // C 言語の size_t に相当
let file_size: u64 = file.len(); // 明示的な 64 ビット
```

### 型推論
```csharp
// C# - var キーワード
var name = "John";        // string
var count = 42;           // int
var price = 29.99;        // double
```

```rust
// Rust - 自動型推論
let name = "John";        // &str (文字列スライス)
let count = 42;           // i32 (デフォルトの整数型)
let price = 29.99;        // f64 (デフォルトの浮動小数点型)

// 明示的な型注釈
let count: u32 = 42;
let price: f32 = 29.99;
```

### 配列とコレクションの概要
```csharp
// C# - 参照型、ヒープ割り当て
int[] numbers = new int[5];        // 固定長
List<int> list = new List<int>();  // 可変長
```

```rust
// Rust - 複数の選択肢
let numbers: [i32; 5] = [1, 2, 3, 4, 5];  // スタック配列、固定長
let mut list: Vec<i32> = Vec::new();       // ヒープベクタ、可変長
```

***

## 文字列型: String vs &str

これは C# 開発者が最も混乱しやすい概念の 1 つですので、丁寧に分解して解説します。

### C# の文字列処理
```csharp
// C# - シンプルな文字列モデル
string name = "John";           // 文字列リテラル
string greeting = "Hello, " + name;  // 文字列連結
string upper = name.ToUpper();  // メソッド呼び出し
```

### Rust の文字列型
```rust
// Rust - 主に 2 つの文字列型が存在する

// 1. &str (文字列スライス) - C# の ReadOnlySpan<char> に近い
let name: &str = "John";        // 文字列リテラル (不変、借用)

// 2. String - StringBuilder や変更可能な文字列に近い
let mut greeting = String::new();       // 空の文字列
greeting.push_str("Hello, ");          // 末尾追加
greeting.push_str(name);               // 末尾追加

// または直接作成
let greeting = String::from("Hello, John");
let greeting = "Hello, John".to_string();  // &str を String に変換
```

### どちらを使うべきか？

| 場面 | 使用する型 | C# における対応概念 |
|----------|-----|---------------|
| 文字列リテラル | `&str` | `string` リテラル |
| 関数の引数（読み取り専用） | `&str` | `string` または `ReadOnlySpan<char>` |
| 所有権を持つ可変文字列 | `String` | `StringBuilder` |
| 所有権を持つ文字列を返す場合 | `String` | `string` |

### 実践例
```rust
// 任意の文字列型を受け入れ可能な関数
fn greet(name: &str) {  // String と &str の両方を受け取れる
    println!("Hello, {}!", name);
}

fn main() {
    let literal = "John";                    // &str
    let owned = String::from("Jane");        // String
    
    greet(literal);                          // 動作する
    greet(&owned);                           // 動作する (String を &str として借用)
    greet("Bob");                            // 動作する
}

// 所有権を持つ文字列を返す関数
fn create_greeting(name: &str) -> String {
    format!("Hello, {}!", name)  // format! マクロは String を返す
}
```

### C# 開発者のための捉え方
```rust
// &str は ReadOnlySpan<char> のようなもの — 文字列データへのビュー（参照）
// String は自身が所有し変更可能な char[] のようなもの

let borrowed: &str = "I don't own this data";
let owned: String = String::from("I own this data");

// 相互変換
let owned_copy: String = borrowed.to_string();  // 所有する String へコピー
let borrowed_view: &str = &owned;               // 所有する String から借用
```

***

## コメントとドキュメント

### 通常のコメント
```csharp
// C# のコメント
// 単一行コメント
/* 複数行
   コメント */

/// <summary>
/// XML ドキュメントコメント
/// </summary>
/// <param name="name">ユーザー名</param>
/// <returns>挨拶文字列</returns>
public string Greet(string name)
{
    return $"Hello, {name}!";
}
```

```rust
// Rust のコメント
// 単一行コメント
/* 複数行
   コメント */

/// ドキュメントコメント (C# の /// に相当)
/// ユーザー名を受け取って挨拶を返します。
/// 
/// # 引数
/// 
/// * `name` - 文字列スライスとしてのユーザー名
/// 
/// # 戻り値
/// 
/// 挨拶を含む `String`
/// 
/// # 例
/// 
/// ```
/// let greeting = greet("Alice");
/// assert_eq!(greeting, "Hello, Alice!");
/// ```
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

### ドキュメントの生成
```bash
# ドキュメントを生成してブラウザで開く (C# の XML ドキュメント生成に相当)
cargo doc --open

# ドキュメント内のテストコード（doctest）を実行
cargo test --doc
```

***

## C# 開発者のための重要な Rust キーワード

Rust のキーワードとその役割を理解することで、C# 開発者はよりスムーズに言語の仕様を把握できるようになります。

### 可視性とアクセス制御キーワード

#### C# のアクセス修飾子
```csharp
public class Example
{
    public int PublicField;           // どこからでもアクセス可能
    private int privateField;        // このクラス内からのみアクセス可能
    protected int protectedField;    // このクラスおよび派生クラスからアクセス可能
    internal int internalField;      // このアセンブリ内からアクセス可能
    protected internal int protectedInternalField; // 両者の組み合わせ
}
```

#### Rust の可視性キーワード
```rust
// pub - アイテムを公開にする（C# の public に相当）
pub struct PublicStruct {
    pub public_field: i32,           // 公開フィールド
    private_field: i32,              // デフォルトで非公開（キーワード不要）
}

pub mod my_module {
    pub(crate) fn crate_public() {}     // 現在のクレート内でのみ公開（C# の internal に相当）
    pub(super) fn parent_public() {}    // 親モジュールに対して公開
    pub(self) fn self_public() {}       // 現在のモジュール内でのみ公開（private と同等）
    
    pub use super::PublicStruct;        // 再エクスポート（C# の using エイリアスに相当）
}

// C# の protected に直接相当するものはありません — 代わりにコンポジションを使用します
```

### メモリと所有権に関するキーワード

#### C# のメモリ関連キーワード
```csharp
// ref - 参照渡し
public void Method(ref int value) { value = 10; }

// out - 出力引数
public bool TryParse(string input, out int result) { /* */ }

// in - 読み取り専用参照 (C# 7.2+)
public void ReadOnly(in LargeStruct data) { /* データを変更不可 */ }
```

#### Rust の所有権キーワード
```rust
// & - 不変参照（C# の in 引数に類似）
fn read_only(data: &Vec<i32>) {
    println!("Length: {}", data.len()); // 読み取り可能、変更不可
}

// &mut - 可変参照（C# の ref 引数に類似）
fn modify(data: &mut Vec<i32>) {
    data.push(42); // 変更可能
}

// move - クロージャで所有権の強制ムーブキャプチャを行う
let data = vec![1, 2, 3];
let closure = move || {
    println!("{:?}", data); // data の所有権がクロージャ内にムーブされる
};
// ここでは data にアクセスできなくなる

// Box - ヒープ割り当て（参照型に対する C# の new に相当）
let boxed_data = Box::new(42); // ヒープ上に割り当て
```

### 制御フローキーワード

#### C# の制御フロー
```csharp
// return - 関数の終了と値の返却
public int GetValue() { return 42; }

// yield return - イテレータパターン
public IEnumerable<int> GetNumbers()
{
    yield return 1;
    yield return 2;
}

// break/continue - ループ制御
foreach (var item in items)
{
    if (item == null) continue;
    if (item.Stop) break;
}
```

#### Rust の制御フローキーワード
```rust
// return - 明示的なリターン（通常は省略可能）
fn get_value() -> i32 {
    return 42; // 明示的な return
    // または単に: 42 (暗黙的な戻り値)
}

// break/continue - 値を伴うループ制御
fn find_value() -> Option<i32> {
    loop {
        let value = get_next();
        if value < 0 { continue; }
        if value > 100 { break None; }      // 値を返して break
        if value == 42 { break Some(value); } // 成功値を返して break
    }
}

// loop - 無限ループ（while(true) に相当）
loop {
    if condition { break; }
}

// while - 条件付きループ
while condition {
    // 処理
}

// for - イテレータループ
for item in collection {
    // 処理
}
```

### 型定義キーワード

#### C# の型キーワード
```csharp
// class - 参照型
public class MyClass { }

// struct - 値型
public struct MyStruct { }

// interface - 契約の定義
public interface IMyInterface { }

// enum - 列挙型
public enum MyEnum { Value1, Value2 }

// delegate - 関数ポインタ
public delegate void MyDelegate(int value);
```

#### Rust の型キーワード
```rust
// struct - データ構造（C# の class と struct を統合したような存在）
struct MyStruct {
    field: i32,
}

// enum - 代数的データ型（C# の enum よりはるかに強力）
enum MyEnum {
    Variant1,
    Variant2(i32),              // データを保持可能
    Variant3 { x: i32, y: i32 }, // 構造体スタイルのバリアント
}

// trait - インターフェイスの定義（C# の interface に類似するがより強力）
trait MyTrait {
    fn method(&self);
    
    // デフォルト実装（C# 8+ のインターフェイスのデフォルト実装に相当）
    fn default_method(&self) {
        println!("Default implementation");
    }
}

// type - 型エイリアス（C# の using エイリアスに相当）
type UserId = u32;
type Result<T> = std::result::Result<T, MyError>;

// impl - 実装ブロック（C# には直接の対応なし — メソッドを構造体定義とは別に定義）
impl MyStruct {
    fn new() -> MyStruct {
        MyStruct { field: 0 }
    }
}

impl MyTrait for MyStruct {
    fn method(&self) {
        println!("Implementation");
    }
}
```

### 関数定義キーワード

#### C# の関数キーワード
```csharp
// static - クラスメソッド
public static void StaticMethod() { }

// virtual - オーバーライド可能
public virtual void VirtualMethod() { }

// override - 基底メソッドのオーバーライド
public override void VirtualMethod() { }

// abstract - 実装必須
public abstract void AbstractMethod();

// async - 非同期メソッド
public async Task<int> AsyncMethod() { return await SomeTask(); }
```

#### Rust の関数キーワード
```rust
// fn - 関数定義（C# のメソッドに類似するがスタンドアロンでも存在可能）
fn regular_function() {
    println!("Hello");
}

// const fn - コンパイル時評価関数（関数の const 化）
const fn compile_time_function() -> i32 {
    42 // コンパイル時に評価可能
}

// async fn - 非同期関数（C# の async に相当）
async fn async_function() -> i32 {
    some_async_operation().await
}

// unsafe fn - メモリ安全性を破る可能性のある関数
unsafe fn unsafe_function() {
    // アンセーフな操作を実行可能
}

// extern fn - 外部関数インターフェイス (FFI)
extern "C" fn c_compatible_function() {
    // C 言語から呼び出し可能
}
```

### 変数宣言キーワード

#### C# の変数キーワード
```csharp
// var - 型推論
var name = "John"; // string と推論される

// const - コンパイル時定数
const int MaxSize = 100;

// readonly - 実行時定数
readonly DateTime createdAt = DateTime.Now;

// static - クラスレベル変数
static int instanceCount = 0;
```

#### Rust の変数キーワード
```rust
// let - 変数束縛（C# の var に相当）
let name = "John"; // デフォルトで不変

// let mut - 可変変数束縛
let mut count = 0; // 変更可能
count += 1;

// const - コンパイル時定数（C# の const に相当）
const MAX_SIZE: usize = 100;

// static - グローバル変数（C# の static フィールドに相当）
static INSTANCE_COUNT: std::sync::atomic::AtomicUsize = 
    std::sync::atomic::AtomicUsize::new(0);
```

### パターンマッチングキーワード

#### C# のパターンマッチング (C# 8+)
```csharp
// switch 式
string result = value switch
{
    1 => "One",
    2 => "Two",
    _ => "Other"
};

// is パターン
if (obj is string str)
{
    Console.WriteLine(str.Length);
}
```

#### Rust のパターンマッチングキーワード
```rust
// match - パターンマッチング（C# の switch に似ているがはるかに強力）
let result = match value {
    1 => "One",
    2 => "Two",
    3..=10 => "Between 3 and 10", // 範囲パターン
    _ => "Other", // ワイルドカード（C# の _ と同じ）
};

// if let - 条件付きパターンマッチング
if let Some(value) = optional {
    println!("Got value: {}", value);
}

// while let - パターンマッチングを伴うループ
while let Some(item) = iterator.next() {
    println!("Item: {}", item);
}

// let による分配束縛（パターン分解）
let (x, y) = point; // タプルの分解
let Some(value) = optional else {
    return; // パターンに一致しない場合は早期リターン
};
```

### メモリ安全性キーワード

#### C# のメモリキーワード
```csharp
// unsafe - 安全性チェックを無効化
unsafe
{
    int* ptr = &variable;
    *ptr = 42;
}

// fixed - マネージドメモリをピン留め
unsafe
{
    fixed (byte* ptr = array)
    {
        // ptr を使用
    }
}
```

#### Rust の安全性キーワード
```rust
// unsafe - ボローチェッカを局所的に無効化（使用は最小限に留める）
unsafe {
    let ptr = &variable as *const i32;
    let value = *ptr; // 生ポインタの逆参照
}

// 生ポインタ型（C# には直接の対応なし — 通常は不要）
let ptr: *const i32 = &42;  // 不変生ポインタ
let ptr: *mut i32 = &mut 42; // 可変生ポインタ
```

### C# には存在しない代表的な Rust キーワード

```rust
// where - ジェネリクス制約（C# の where より柔軟）
fn generic_function<T>() 
where 
    T: Clone + Send + Sync,
{
    // T は Clone、Send、Sync トレイトを実装している必要がある
}

// dyn - 動的トレイトオブジェクト（C# の object 型に近いが型安全）
let drawable: Box<dyn Draw> = Box::new(Circle::new());

// Self - 実装対象の型を参照（C# の this に似ているが型そのものを指す）
impl MyStruct {
    fn new() -> Self { // Self = MyStruct
        Self { field: 0 }
    }
}

// self - メソッドレシーバ（インスタンス参照）
impl MyStruct {
    fn method(&self) { }        // 不変借用
    fn method_mut(&mut self) { } // 可変借用  
    fn consume(self) { }        // 所有権の消費（ムーブ）
}

// crate - 現在のクレートのルートを参照
use crate::models::User; // クレートルートからの絶対パス

// super - 親モジュールを参照
use super::utils; // 親モジュールからのインポート
```

### C# 開発者のためのキーワードまとめ

| 用途 | C# | Rust | 主な相違点 |
|---------|----|----|----------------|
| 可視性 | `public`, `private`, `internal` | `pub`, デフォルトで非公開 | `pub(crate)` などより細かな粒度 |
| 変数 | `var`, `readonly`, `const` | `let`, `let mut`, `const` | デフォルトで不変 |
| 関数 | `method()` | `fn` | 単独の関数定義が可能 |
| 型 | `class`, `struct`, `interface` | `struct`, `enum`, `trait` | enum は代数的データ型 |
| ジェネリクス | `<T> where T : IFoo` | `<T> where T: Foo` | より柔軟な制約の記述が可能 |
| 参照 | `ref`, `out`, `in` | `&`, `&mut` | コンパイル時ボローチェッカによる検証 |
| パターン | `switch`, `is` | `match`, `if let` | 網羅性チェック（全パターン網羅）が必須 |

***

## 所有権の理解

所有権（Ownership）は Rust の最も特徴的な機能であり、C# 開発者にとって最大の認知的パラダイムシフトとなる概念です。段階を追って理解していきましょう。

### C# のメモリモデル（復習）
```csharp
// C# - 自動メモリ管理
public void ProcessData()
{
    var data = new List<int> { 1, 2, 3, 4, 5 };
    ProcessList(data);
    // ここでも data にアクセス可能
    Console.WriteLine(data.Count);  // 正常に動作する
    
    // 参照がすべてなくなった時点で GC が回収する
}

public void ProcessList(List<int> list)
{
    list.Add(6);  // 元のリストを変更する
}
```

### Rust の所有権規則
1. **各値には、必ず 1 つの「所有者（変数）」が存在する**
2. **所有者がスコープを外れると、値は破棄（ドロップ）される**
3. **所有権は別の変数に譲渡（ムーブ）できる**

```rust
// Rust - 明示的な所有権管理
fn process_data() {
    let data = vec![1, 2, 3, 4, 5];  // data がベクタの所有者となる
    process_list(data);              // 所有権が関数にムーブされる
    // println!("{:?}", data);       // ❌ エラー: data はもはやこの場所で所有されていない
}

fn process_list(mut list: Vec<i32>) {  // list がベクタの新たな所有者となる
    list.push(6);
    // 関数が終了すると、list はここでドロップされる
}
```

### C# 開発者のための「ムーブ」の理解
```csharp
// C# - 参照がコピーされ、オブジェクト自体はその場に留まる
var original = new List<int> { 1, 2, 3 };
var reference = original;  // 両方の変数が同じオブジェクトを指す
original.Add(4);
Console.WriteLine(reference.Count);  // 4 - 同じオブジェクト
```

```rust
// Rust - 所有権が譲渡（ムーブ）される
let original = vec![1, 2, 3];
let moved = original;       // 所有権が譲渡される
// println!("{:?}", original);  // ❌ エラー: original はもはやデータを所有していない
println!("{:?}", moved);    // ✅ 動作する: moved がデータを所有している
```

### Copy 型 vs Move 型
```rust
// Copy 型（C# の値型に近い）— ムーブされず、ビット単位でコピーされる
let x = 5;        // i32 は Copy を実装している
let y = x;        // x が y にコピーされる
println!("{}", x); // ✅ 動作する: x は依然として有効

// Move 型（C# の参照型に近い）— コピーされず、ムーブされる  
let s1 = String::from("hello");  // String は Copy を実装していない
let s2 = s1;                     // s1 から s2 に所有権がムーブされる
// println!("{}", s1);           // ❌ エラー: s1 はもはや有効ではない
```

### 実践例: 値の交換
```csharp
// C# - 単純な参照の交換
public void SwapLists(ref List<int> a, ref List<int> b)
{
    var temp = a;
    a = b;
    b = temp;
}
```

```rust
// Rust - 所有権を考慮した交換
fn swap_vectors(a: &mut Vec<i32>, b: &mut Vec<i32>) {
    std::mem::swap(a, b);  // 組み込みの swap 関数
}

// または手動でのアプローチ
fn manual_swap() {
    let mut a = vec![1, 2, 3];
    let mut b = vec![4, 5, 6];
    
    let temp = a;  // a を temp にムーブ
    a = b;         // b を a にムーブ
    b = temp;      // temp を b にムーブ
    
    println!("a: {:?}, b: {:?}", a, b);
}
```

***

## 借用の基礎

借用（Borrowing）は、C# で参照を受け取ることに似ていますが、コンパイル時に安全性が厳密に保証される点が決定的に異なります。

### C# の参照引数
```csharp
// C# - ref および out 引数
public void ModifyValue(ref int value)
{
    value += 10;
}

public void ReadValue(in int value)  // 読み取り専用参照
{
    Console.WriteLine(value);
}

public bool TryParse(string input, out int result)
{
    return int.TryParse(input, out result);
}
```

### Rust の借用
```rust
// Rust - & と &mut による借用
fn modify_value(value: &mut i32) {  // 可変借用
    *value += 10;
}

fn read_value(value: &i32) {        // 不変借用
    println!("{}", value);
}

fn main() {
    let mut x = 5;
    
    read_value(&x);      // 不変借用
    modify_value(&mut x); // 可変借用
    
    println!("{}", x);   // x は依然としてここで所有されている
}
```

### 借用規則（コンパイル時に強制！）
```rust
fn borrowing_rules() {
    let mut data = vec![1, 2, 3];
    
    // 規則 1: 複数の不変借用は可能
    let r1 = &data;
    let r2 = &data;
    println!("{:?} {:?}", r1, r2);  // ✅ 動作する
    
    // 規則 2: 可変借用は同時に 1 つのみ許可される
    let r3 = &mut data;
    // let r4 = &mut data;  // ❌ エラー: 2 つ目の可変借用は作成不可
    // let r5 = &data;      // ❌ エラー: 可変借用中に不変借用は作成不可
    
    r3.push(4);  // 可変借用を使用
    // r3 はここでスコープを抜ける
    
    // 規則 3: 以前の借用が終了した後は、再び借用可能
    let r6 = &data;  // ✅ 今度は動作する
    println!("{:?}", r6);
}
```

### C# vs Rust: 参照の安全性
```csharp
// C# - 実行時エラーの潜在的リスク
public class ReferenceSafety
{
    private List<int> data = new List<int>();
    
    public List<int> GetData() => data;  // 内部データの参照を返却
    
    public void UnsafeExample()
    {
        var reference = GetData();
        
        // 別のスレッドがここで data を変更する可能性がある！
        Thread.Sleep(1000);
        
        // reference が無効化されたり変更されている可能性がある
        reference.Add(42);  // 競合状態（レースコンディション）の可能性
    }
}
```

```rust
// Rust - コンパイル時の安全性
pub struct SafeContainer {
    data: Vec<i32>,
}

impl SafeContainer {
    // 不変借用を返す — 呼び出し元は変更できない
    pub fn get_data(&self) -> &Vec<i32> {
        &self.data
    }
    
    // 可変借用を返す — 排他アクセスが保証される
    pub fn get_data_mut(&mut self) -> &mut Vec<i32> {
        &mut self.data
    }
}

fn safe_example() {
    let mut container = SafeContainer { data: vec![1, 2, 3] };
    
    let reference = container.get_data();
    // container.get_data_mut();  // ❌ エラー: 不変借用中に可変借用は不可
    
    println!("{:?}", reference);  // 不変参照を使用
    // reference はここでスコープを抜ける
    
    let mut_reference = container.get_data_mut();  // ✅ ここなら OK
    mut_reference.push(4);
}
```

***

## 参照 vs ポインタ

### C# のポインタ（unsafe コンテキスト）
```csharp
// C# のアンセーフポインタ（滅多に使用されない）
unsafe void UnsafeExample()
{
    int value = 42;
    int* ptr = &value;  // 値へのポインタ
    *ptr = 100;         // 逆参照して変更
    Console.WriteLine(value);  // 100
}
```

### Rust の参照（デフォルトで安全）
```rust
// Rust の参照（常に安全）
fn safe_example() {
    let mut value = 42;
    let ptr = &mut value;  // 可変参照
    *ptr = 100;           // 逆参照して変更
    println!("{}", value); // 100
}

// "unsafe" キーワードは不要 — ボローチェッカが安全性を保証する
```

### C# 開発者のためのライフタイムの基礎
```csharp
// C# - 無効になり得る参照を返す危険性
public class LifetimeIssues
{
    public string GetFirstWord(string input)
    {
        return input.Split(' ')[0];  // 新しい文字列を生成して返却（安全）
    }
    
    public unsafe char* GetFirstChar(string input)
    {
        // これは危険 — マネージドメモリへのポインタを返却している
        fixed (char* ptr = input)
            return ptr;  // ❌ 危険: メソッド終了後に ptr が無効になる
    }
}
```

```rust
// Rust - ライフタイム検査によりダングリング参照を防止
fn get_first_word(input: &str) -> &str {
    input.split_whitespace().next().unwrap_or("")
    // ✅ 安全: 返される参照は input と同じライフタイムを持つ
}

fn invalid_reference() -> &str {
    let temp = String::from("hello");
    &temp  // ❌ コンパイルエラー: temp の生存期間が短すぎる
    // temp は関数終了時にドロップされてしまう
}

fn valid_reference() -> String {
    let temp = String::from("hello");
    temp  // ✅ 動作する: 所有権が呼び出し元に譲渡される
}
```

***

## ムーブセマンティクス

### C# の値型 vs 参照型
```csharp
// C# - 値型はコピーされる
struct Point
{
    public int X { get; set; }
    public int Y { get; set; }
}

var p1 = new Point { X = 1, Y = 2 };
var p2 = p1;  // コピー
p2.X = 10;
Console.WriteLine(p1.X);  // 依然として 1

// C# - 参照型はオブジェクトを共有する
var list1 = new List<int> { 1, 2, 3 };
var list2 = list1;  // 参照のコピー
list2.Add(4);
Console.WriteLine(list1.Count);  // 4 - 同一オブジェクト
```

### Rust のムーブセマンティクス
```rust
// Rust - Copy でない型はデフォルトでムーブされる
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn move_example() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1;  // ムーブ（コピーではない）
    // println!("{:?}", p1);  // ❌ エラー: p1 はムーブ済み
    println!("{:?}", p2);    // ✅ 動作する
}

// コピー可能にするには、Copy トレイトを実装する
#[derive(Debug, Copy, Clone)]
struct CopyablePoint {
    x: i32,
    y: i32,
}

fn copy_example() {
    let p1 = CopyablePoint { x: 1, y: 2 };
    let p2 = p1;  // コピー（Copy を実装しているため）
    println!("{:?}", p1);  // ✅ 動作する
    println!("{:?}", p2);  // ✅ 動作する
}
```

### 値がムーブされるタイミング
```rust
fn demonstrate_moves() {
    let s = String::from("hello");
    
    // 1. 代入によるムーブ
    let s2 = s;  // s が s2 にムーブされる
    
    // 2. 関数呼び出しによるムーブ
    take_ownership(s2);  // s2 が関数内にムーブされる
    
    // 3. 関数からのリターンによるムーブ
    let s3 = give_ownership();  // 戻り値が s3 にムーブされる
    
    println!("{}", s3);  // s3 は有効
}

fn take_ownership(s: String) {
    println!("{}", s);
    // s はここでドロップされる
}

fn give_ownership() -> String {
    String::from("yours")  // 所有権が呼び出し元にムーブされる
}
```

### 借用によるムーブの回避
```rust
fn demonstrate_borrowing() {
    let s = String::from("hello");
    
    // ムーブする代わりに借用する
    let len = calculate_length(&s);  // s を借用
    println!("'{}' has length {}", s, len);  // s は依然として有効
}

fn calculate_length(s: &String) -> usize {
    s.len()  // s の所有権は持たないため、ここではドロップされない
}
```

***

## 関数 vs メソッド

### C# の関数・メソッド宣言
```csharp
// C# - クラス内のメソッド
public class Calculator
{
    // インスタンスメソッド
    public int Add(int a, int b)
    {
        return a + b;
    }
    
    // 静的メソッド
    public static int Multiply(int a, int b)
    {
        return a * b;
    }
    
    // ref 引数を持つメソッド
    public void Increment(ref int value)
    {
        value++;
    }
}
```

### Rust の関数宣言
```rust
// Rust - スタンドアロンの関数
fn add(a: i32, b: i32) -> i32 {
    a + b  // 最後の式には 'return' を書く必要がない
}

fn multiply(a: i32, b: i32) -> i32 {
    return a * b;  // 明示的な return を書いても問題ない
}

// 可変参照を受け取る関数
fn increment(value: &mut i32) {
    *value += 1;
}

fn main() {
    let result = add(5, 3);
    println!("5 + 3 = {}", result);
    
    let mut x = 10;
    increment(&mut x);
    println!("After increment: {}", x);
}
```

### 式 vs 文（重要！）
```csharp
// C# - 文 (Statement) vs 式 (Expression)
public int GetValue()
{
    if (condition)
    {
        return 42;  // 文
    }
    return 0;       // 文
}
```

```rust
// Rust - あらゆるものが式になり得る
fn get_value(condition: bool) -> i32 {
    if condition {
        42  // 式（セミコロンなし）
    } else {
        0   // 式（セミコロンなし）
    }
    // if-else ブロック自体が値を返す式となる
}

// さらに簡潔に記述可能
fn get_value_ternary(condition: bool) -> i32 {
    if condition { 42 } else { 0 }
}
```

### 関数の引数と戻り値の型
```rust
// 引数なし、戻り値なし（ユニット型 () を返す）
fn say_hello() {
    println!("Hello!");
}

// 複数の引数
fn greet(name: &str, age: u32) {
    println!("{} is {} years old", name, age);
}

// タプルを使用した複数の戻り値
fn divide_and_remainder(dividend: i32, divisor: i32) -> (i32, i32) {
    (dividend / divisor, dividend % divisor)
}

fn main() {
    let (quotient, remainder) = divide_and_remainder(10, 3);
    println!("10 ÷ 3 = {} remainder {}", quotient, remainder);
}
```

***

## 制御フローの基礎

### 条件分岐
```csharp
// C# の if 文
int x = 5;
if (x > 10)
{
    Console.WriteLine("Big number");
}
else if (x > 5)
{
    Console.WriteLine("Medium number");
}
else
{
    Console.WriteLine("Small number");
}

// C# の三項演算子
string message = x > 10 ? "Big" : "Small";
```

```rust
// Rust の if 式
let x = 5;
if x > 10 {
    println!("Big number");
} else if x > 5 {
    println!("Medium number");
} else {
    println!("Small number");
}

// 式としての if（三項演算子に相当）
let message = if x > 10 { "Big" } else { "Small" };

// 複数の条件分岐
let message = if x > 10 {
    "Big"
} else if x > 5 {
    "Medium"
} else {
    "Small"
};
```

### ループ
```csharp
// C# のループ
// for ループ
for (int i = 0; i < 5; i++)
{
    Console.WriteLine(i);
}

// foreach ループ
var numbers = new[] { 1, 2, 3, 4, 5 };
foreach (var num in numbers)
{
    Console.WriteLine(num);
}

// while ループ
int count = 0;
while (count < 3)
{
    Console.WriteLine(count);
    count++;
}
```

```rust
// Rust のループ
// 範囲指定の for ループ
for i in 0..5 {  // 0 から 4（終端は含まない）
    println!("{}", i);
}

// コレクションの反復処理
let numbers = vec![1, 2, 3, 4, 5];
for num in numbers {  // 所有権を消費
    println!("{}", num);
}

// 参照による反復処理（こちらが一般的）
let numbers = vec![1, 2, 3, 4, 5];
for num in &numbers {  // 要素を借用
    println!("{}", num);
}

// while ループ
let mut count = 0;
while count < 3 {
    println!("{}", count);
    count += 1;
}

// break を伴う無限ループ
let mut counter = 0;
loop {
    if counter >= 3 {
        break;
    }
    println!("{}", counter);
    counter += 1;
}
```

### ループ制御
```csharp
// C# のループ制御
for (int i = 0; i < 10; i++)
{
    if (i == 3) continue;
    if (i == 7) break;
    Console.WriteLine(i);
}
```

```rust
// Rust のループ制御
for i in 0..10 {
    if i == 3 { continue; }
    if i == 7 { break; }
    println!("{}", i);
}

// ループラベル（ネストしたループ用）
'outer: for i in 0..3 {
    'inner: for j in 0..3 {
        if i == 1 && j == 1 {
            break 'outer;  // 外側のループを抜ける
        }
        println!("i: {}, j: {}", i, j);
    }
}
```

***

## パターンマッチング入門

Rust のパターンマッチングは、C# の switch 文よりもはるかに強力です。

### C# の switch 文
```csharp
// C# の従来の switch 文
int value = 2;
switch (value)
{
    case 1:
        Console.WriteLine("One");
        break;
    case 2:
        Console.WriteLine("Two");
        break;
    default:
        Console.WriteLine("Other");
        break;
}

// C# 8+ の switch 式
string result = value switch
{
    1 => "One",
    2 => "Two",
    _ => "Other"
};
```

### Rust の match 式
```rust
// Rust の match（網羅的でなければならない）
let value = 2;
match value {
    1 => println!("One"),
    2 => println!("Two"),
    _ => println!("Other"),  // _ はワイルドカード（default に相当）
}

// 式としての match（switch 式に相当）
let result = match value {
    1 => "One",
    2 => "Two",
    _ => "Other",
};

// 複数の値のマッチ
match value {
    1 | 2 => println!("One or Two"),  // 複数パターンの指定
    3..=5 => println!("Three to Five"), // 範囲パターン
    _ => println!("Other"),
}
```

### match による分解束縛
```csharp
// C# のタプル分解
var point = (3, 4);
var (x, y) = point;
Console.WriteLine($"x: {x}, y: {y}");

// C# のタプルを使用したパターンマッチング
string classify = point switch
{
    (0, 0) => "Origin",
    (var a, 0) => $"On X-axis at {a}",
    (0, var b) => $"On Y-axis at {b}",
    _ => "Somewhere else"
};
```

```rust
// Rust の match によるタプル分解
let point = (3, 4);
match point {
    (0, 0) => println!("Origin"),
    (x, 0) => println!("On X-axis at {}", x),
    (0, y) => println!("On Y-axis at {}", y),
    (x, y) => println!("Point at ({}, {})", x, y),
}

// マッチガード（条件式の付加）
match point {
    (x, y) if x == y => println!("On diagonal"),
    (x, y) if x > y => println!("Above diagonal"),
    _ => println!("Below diagonal"),
}
```

***

## エラー処理の基礎

これは C# の例外モデルから Rust の明示的エラー処理への根本的な転換点です。

### C# の例外処理
```csharp
// C# - 例外ベースのエラー処理
public class FileProcessor
{
    public string ReadConfig(string path)
    {
        try
        {
            return File.ReadAllText(path);
        }
        catch (FileNotFoundException)
        {
            throw new InvalidOperationException("Config file not found");
        }
        catch (UnauthorizedAccessException)
        {
            throw new InvalidOperationException("Cannot access config file");
        }
    }
    
    public int ParseNumber(string input)
    {
        if (int.TryParse(input, out int result))
        {
            return result;
        }
        throw new ArgumentException("Invalid number format");
    }
}
```

### Rust の Result に基づくエラー処理
```rust
use std::fs;
use std::num::ParseIntError;

// カスタムエラー型の定義
#[derive(Debug)]
enum ConfigError {
    FileNotFound,
    AccessDenied,
    InvalidFormat,
}

// Result を返す関数
fn read_config(path: &str) -> Result<String, ConfigError> {
    match fs::read_to_string(path) {
        Ok(content) => Ok(content),
        Err(_) => Err(ConfigError::FileNotFound),  // 例示のため簡略化
    }
}

// 失敗する可能性のある関数
fn parse_number(input: &str) -> Result<i32, ParseIntError> {
    input.parse::<i32>()  // Result<i32, ParseIntError> を返す
}

fn main() {
    // エラーを明示的に処理
    match read_config("config.txt") {
        Ok(content) => println!("Config: {}", content),
        Err(ConfigError::FileNotFound) => println!("Config file not found"),
        Err(error) => println!("Config error: {:?}", error),
    }
    
    // パースエラーを処理
    match parse_number("42") {
        Ok(num) => println!("Number: {}", num),
        Err(error) => println!("Parse error: {}", error),
    }
}
```

### ? 演算子（C# の await に似た伝播構造）
```csharp
// C# - 例外の伝播（暗黙的）
public async Task<string> ProcessFileAsync(string path)
{
    var content = await File.ReadAllTextAsync(path);  // エラー時は例外がスローされる
    var processed = ProcessContent(content);          // エラー時は例外がスローされる
    return processed;
}
```

```rust
// Rust - ? によるエラーの伝播
fn process_file(path: &str) -> Result<String, ConfigError> {
    let content = read_config(path)?;  // Err の場合はエラーを呼び出し元に伝播
    let processed = process_content(&content)?;  // Err の場合はエラーを呼び出し元に伝播
    Ok(processed)  // 成功値を Ok でラップ
}

fn process_content(content: &str) -> Result<String, ConfigError> {
    if content.is_empty() {
        Err(ConfigError::InvalidFormat)
    } else {
        Ok(content.to_uppercase())
    }
}
```

### null 許容値のための Option<T>
```csharp
// C# - null 許容参照型
public string? FindUserName(int userId)
{
    var user = database.FindUser(userId);
    return user?.Name;  // ユーザーが見つからない場合は null を返す
}

public void ProcessUser(int userId)
{
    string? name = FindUserName(userId);
    if (name != null)
    {
        Console.WriteLine($"User: {name}");
    }
    else
    {
        Console.WriteLine("User not found");
    }
}
```

```rust
// Rust - オプショナルな値のための Option<T>
fn find_user_name(user_id: u32) -> Option<String> {
    // データベース検索のシミュレーション
    if user_id == 1 {
        Some("Alice".to_string())
    } else {
        None
    }
}

fn process_user(user_id: u32) {
    match find_user_name(user_id) {
        Some(name) => println!("User: {}", name),
        None => println!("User not found"),
    }
    
    // または if let（パターンマッチングの簡略構文）を使用
    if let Some(name) = find_user_name(user_id) {
        println!("User: {}", name);
    } else {
        println!("User not found");
    }
}
```

### Option と Result の組み合わせ
```rust
fn safe_divide(a: f64, b: f64) -> Option<f64> {
    if b != 0.0 {
        Some(a / b)
    } else {
        None
    }
}

fn parse_and_divide(a_str: &str, b_str: &str) -> Result<Option<f64>, ParseFloatError> {
    let a: f64 = a_str.parse()?;  // パースに失敗した場合はエラーを返す
    let b: f64 = b_str.parse()?;  // パースに失敗した場合はエラーを返す
    Ok(safe_divide(a, b))         // Ok(Some(result)) または Ok(None) を返す
}

use std::num::ParseFloatError;

fn main() {
    match parse_and_divide("10.0", "2.0") {
        Ok(Some(result)) => println!("Result: {}", result),
        Ok(None) => println!("Division by zero"),
        Err(error) => println!("Parse error: {}", error),
    }
}
```

***

## Vec<T> vs List<T>

`Vec<T>` は C# の `List<T>` に相当する Rust の動的配列ですが、所有権セマンティクスが適用される点が異なります。

### C# の List<T>
```csharp
// C# List<T> - 参照型、ヒープ割り当て
var numbers = new List<int>();
numbers.Add(1);
numbers.Add(2);
numbers.Add(3);

// メソッドへの受け渡し — 参照がコピーされる
ProcessList(numbers);
Console.WriteLine(numbers.Count);  // 引き続きアクセス可能

void ProcessList(List<int> list)
{
    list.Add(4);  // 元のリストが変更される
    Console.WriteLine($"Count in method: {list.Count}");
}
```

### Rust の Vec<T>
```rust
// Rust Vec<T> - 所有権を持つ型、ヒープ割り当て
let mut numbers = Vec::new();
numbers.push(1);
numbers.push(2);
numbers.push(3);

// 所有権を奪うメソッド
process_vec(numbers);
// println!("{:?}", numbers);  // ❌ エラー: numbers はムーブ済み

// 借用するメソッド
let mut numbers = vec![1, 2, 3];  // 利便性の高い vec! マクロ
process_vec_borrowed(&mut numbers);
println!("{:?}", numbers);  // ✅ 引き続きアクセス可能

fn process_vec(mut vec: Vec<i32>) {  // 所有権を受け取る
    vec.push(4);
    println!("Count in method: {}", vec.len());
    // vec はここでドロップされる
}

fn process_vec_borrowed(vec: &mut Vec<i32>) {  // 可変借用する
    vec.push(4);
    println!("Count in method: {}", vec.len());
}
```

### ベクタの作成と初期化
```csharp
// C# の List 初期化
var numbers = new List<int> { 1, 2, 3, 4, 5 };
var empty = new List<int>();
var sized = new List<int>(10);  // 初期キャパシティ（容量）の指定

// 他のコレクションから作成
var fromArray = new List<int>(new[] { 1, 2, 3 });
```

```rust
// Rust の Vec 初期化
let numbers = vec![1, 2, 3, 4, 5];  // vec! マクロ
let empty: Vec<i32> = Vec::new();   // 空の場合は型注釈が必要
let sized = Vec::with_capacity(10); // キャパシティを事前に確保

// イテレータから作成
let from_range: Vec<i32> = (1..=5).collect();
let from_array = vec![1, 2, 3];
```

### 一般的な操作の比較
```csharp
// C# の List 操作
var list = new List<int> { 1, 2, 3 };

list.Add(4);                    // 要素の追加
list.Insert(0, 0);              // インデックスを指定して挿入
list.Remove(2);                 // 最初に見つかった要素を削除
list.RemoveAt(1);               // 指定インデックスの要素を削除
list.Clear();                   // すべて削除

int first = list[0];            // インデックスアクセス
int count = list.Count;         // 要素数の取得
bool contains = list.Contains(3); // 含有確認
```

```rust
// Rust の Vec 操作
let mut vec = vec![1, 2, 3];

vec.push(4);                    // 要素の追加
vec.insert(0, 0);               // インデックスを指定して挿入
vec.retain(|&x| x != 2);        // 条件に合う要素を残す（関数型スタイルでの削除）
vec.remove(1);                  // 指定インデックスの要素を削除
vec.clear();                    // すべて削除

let first = vec[0];             // インデックスアクセス（範囲外はパニック）
let safe_first = vec.get(0);    // 安全なアクセス、Option<&T> を返す
let count = vec.len();          // 要素数の取得
let contains = vec.contains(&3); // 含有確認
```

### 安全なアクセスパターン
```csharp
// C# - 例外ベースの境界チェック
public int SafeAccess(List<int> list, int index)
{
    try
    {
        return list[index];
    }
    catch (ArgumentOutOfRangeException)
    {
        return -1;  // デフォルト値
    }
}
```

```rust
// Rust - Option に基づく安全なアクセス
fn safe_access(vec: &Vec<i32>, index: usize) -> Option<i32> {
    vec.get(index).copied()  // Option<i32> を返す
}

fn main() {
    let vec = vec![1, 2, 3];
    
    // 安全なアクセスパターン
    match vec.get(10) {
        Some(value) => println!("Value: {}", value),
        None => println!("Index out of bounds"),
    }
    
    // または unwrap_or を使用
    let value = vec.get(10).copied().unwrap_or(-1);
    println!("Value: {}", value);
}
```

***

## HashMap vs Dictionary

`HashMap` は C# の `Dictionary<K,V>` に相当する Rust のコレクションです。

### C# の Dictionary
```csharp
// C# Dictionary<TKey, TValue>
var scores = new Dictionary<string, int>
{
    ["Alice"] = 100,
    ["Bob"] = 85,
    ["Charlie"] = 92
};

// 追加 / 更新
scores["Dave"] = 78;
scores["Alice"] = 105;  // 既存値の更新

// 安全なアクセス
if (scores.TryGetValue("Eve", out int score))
{
    Console.WriteLine($"Eve's score: {score}");
}
else
{
    Console.WriteLine("Eve not found");
}

// 反復処理
foreach (var kvp in scores)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}
```

### Rust の HashMap
```rust
use std::collections::HashMap;

// HashMap の作成と初期化
let mut scores = HashMap::new();
scores.insert("Alice".to_string(), 100);
scores.insert("Bob".to_string(), 85);
scores.insert("Charlie".to_string(), 92);

// またはイテレータから生成
let scores: HashMap<String, i32> = [
    ("Alice".to_string(), 100),
    ("Bob".to_string(), 85),
    ("Charlie".to_string(), 92),
].into_iter().collect();

// 追加 / 更新
let mut scores = scores;  // 可変にする
scores.insert("Dave".to_string(), 78);
scores.insert("Alice".to_string(), 105);  // 既存値の更新

// 安全なアクセス
match scores.get("Eve") {
    Some(score) => println!("Eve's score: {}", score),
    None => println!("Eve not found"),
}

// 反復処理
for (name, score) in &scores {
    println!("{}: {}", name, score);
}
```

### HashMap の操作
```csharp
// C# の Dictionary 操作
var dict = new Dictionary<string, int>();

dict["key"] = 42;                    // 挿入 / 更新
bool exists = dict.ContainsKey("key"); // 存在確認
bool removed = dict.Remove("key");    // 削除
dict.Clear();                        // 全削除

// デフォルト値付き取得
int value = dict.GetValueOrDefault("missing", 0);
```

```rust
use std::collections::HashMap;

// Rust の HashMap 操作
let mut map = HashMap::new();

map.insert("key".to_string(), 42);   // 挿入 / 更新
let exists = map.contains_key("key"); // 存在確認
let removed = map.remove("key");      // 削除、Option<V> を返す
map.clear();                         // 全削除

// 高度な操作のための Entry API
let mut map = HashMap::new();
map.entry("key".to_string()).or_insert(42);  // 存在しない場合のみ挿入
map.entry("key".to_string()).and_modify(|v| *v += 1); // 存在する場合に変更

// デフォルト値付き取得
let value = map.get("missing").copied().unwrap_or(0);
```

### HashMap のキーと値における所有権
```rust
// HashMap における所有権の理解
fn ownership_example() {
    let mut map = HashMap::new();
    
    // String のキーと値はマップ内にムーブされる
    let key = String::from("name");
    let value = String::from("Alice");
    
    map.insert(key, value);
    // println!("{}", key);   // ❌ エラー: key はムーブ済み
    // println!("{}", value); // ❌ エラー: value はムーブ済み
    
    // 参照経由でのアクセス
    if let Some(name) = map.get("name") {
        println!("Name: {}", name);  // 値を借用
    }
}

// &str キーの使用（所有権の譲渡なし）
fn string_slice_keys() {
    let mut map = HashMap::new();
    
    map.insert("name", "Alice");     // &str のキーと値
    map.insert("age", "30");
    
    // 文字列リテラルの場合は所有権の問題が発生しない
    println!("Name exists: {}", map.contains_key("name"));
}
```

***

## 配列とスライス

配列、スライス、ベクタの違いを理解することは極めて重要です。

### C# の配列
```csharp
// C# の配列
int[] numbers = new int[5];         // 固定長、ヒープ割り当て
int[] initialized = { 1, 2, 3, 4, 5 }; // 配列リテラル

// アクセス
numbers[0] = 10;
int first = numbers[0];

// 長さ
int length = numbers.Length;

// 引数としての配列（参照型）
void ProcessArray(int[] array)
{
    array[0] = 99;  // 元の配列を変更
}
```

### Rust の配列、スライス、ベクタ
```rust
// 1. 配列 — 固定長、スタック割り当て
let numbers: [i32; 5] = [1, 2, 3, 4, 5];  // 型: [i32; 5]
let zeros = [0; 10];                       // 10 個のゼロ

// アクセス
let first = numbers[0];
// numbers[0] = 10;  // ❌ エラー: 配列はデフォルトで不変

let mut mut_array = [1, 2, 3, 4, 5];
mut_array[0] = 10;  // ✅ mut を付ければ変更可能

// 2. スライス — 配列やベクタへのビュー（参照）
let slice: &[i32] = &numbers[1..4];  // 要素 1, 2, 3
let all_slice: &[i32] = &numbers;    // 配列全体をスライスとして参照

// 3. ベクタ — 可変長、ヒープ割り当て（前述）
let mut vec = vec![1, 2, 3, 4, 5];
vec.push(6);  // 要素を追加可能
```

### 関数の引数としてのスライス
```csharp
// C# - 配列を受け取るメソッド
public void ProcessNumbers(int[] numbers)
{
    for (int i = 0; i < numbers.Length; i++)
    {
        Console.WriteLine(numbers[i]);
    }
}

// 配列のみを受け入れ可能
ProcessNumbers(new int[] { 1, 2, 3 });
```

```rust
// Rust - 任意のシーケンスを受け入れ可能な関数
fn process_numbers(numbers: &[i32]) {  // スライス引数
    for (i, num) in numbers.iter().enumerate() {
        println!("Index {}: {}", i, num);
    }
}

fn main() {
    let array = [1, 2, 3, 4, 5];
    let vec = vec![1, 2, 3, 4, 5];
    
    // 同じ関数で両方を処理可能！
    process_numbers(&array);      // 配列をスライスとして渡す
    process_numbers(&vec);        // ベクタをスライスとして渡す
    process_numbers(&vec[1..4]);  // 部分スライスを渡す
}
```

### 文字列スライス (&str) の再訪
```rust
// String と &str の関係
fn string_slice_example() {
    let owned = String::from("Hello, World!");
    let slice: &str = &owned[0..5];      // "Hello"
    let slice2: &str = &owned[7..];      // "World!"
    
    println!("{}", slice);   // "Hello"
    println!("{}", slice2);  // "World!"
    
    // 任意の文字列型を受け入れる関数
    print_string("String literal");      // &str
    print_string(&owned);               // String を &str として渡す
    print_string(slice);                // &str スライス
}

fn print_string(s: &str) {
    println!("{}", s);
}
```

***

## コレクションの操作

### 反復処理（イテレーション）のパターン
```csharp
// C# の反復処理パターン
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// インデックス付き for ループ
for (int i = 0; i < numbers.Count; i++)
{
    Console.WriteLine($"Index {i}: {numbers[i]}");
}

// foreach ループ
foreach (int num in numbers)
{
    Console.WriteLine(num);
}

// LINQ メソッド
var doubled = numbers.Select(x => x * 2).ToList();
var evens = numbers.Where(x => x % 2 == 0).ToList();
```

```rust
// Rust の反復処理パターン
let numbers = vec![1, 2, 3, 4, 5];

// インデックス付き for ループ
for (i, num) in numbers.iter().enumerate() {
    println!("Index {}: {}", i, num);
}

// 値を順に巡回する for ループ
for num in &numbers {  // 各要素を借用
    println!("{}", num);
}

// イテレータメソッド（LINQ に相当）
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
let evens: Vec<i32> = numbers.iter().filter(|&x| x % 2 == 0).cloned().collect();

// またはより効率的に、所有権を消費するイテレータを使用
let doubled: Vec<i32> = numbers.into_iter().map(|x| x * 2).collect();
```

### Iterator vs IntoIterator vs Iter
```rust
// さまざまなイテレーションメソッドの理解
fn iteration_methods() {
    let vec = vec![1, 2, 3, 4, 5];
    
    // 1. iter() — 要素を借用 (&T)
    for item in vec.iter() {
        println!("{}", item);  // item は &i32
    }
    // ここでも vec は引き続き利用可能
    
    // 2. into_iter() — 所有権を消費 (T)
    for item in vec.into_iter() {
        println!("{}", item);  // item は i32
    }
    // ここ以降 vec は利用不可
    
    let mut vec = vec![1, 2, 3, 4, 5];
    
    // 3. iter_mut() — 可変借用 (&mut T)
    for item in vec.iter_mut() {
        *item *= 2;  // item は &mut i32
    }
    println!("{:?}", vec);  // [2, 4, 6, 8, 10]
}
```

### 結果の集約（Collect）
```csharp
// C# - エラーが発生し得るコレクションの処理
public List<int> ParseNumbers(List<string> inputs)
{
    var results = new List<int>();
    foreach (string input in inputs)
    {
        if (int.TryParse(input, out int result))
        {
            results.Add(result);
        }
        // 不正な入力は暗黙的にスキップ
    }
    return results;
}
```

```rust
// Rust - collect による明示的なエラー処理
fn parse_numbers(inputs: Vec<String>) -> Result<Vec<i32>, std::num::ParseIntError> {
    inputs.into_iter()
        .map(|s| s.parse::<i32>())  // Result<i32, ParseIntError> を返す
        .collect()                  // Result<Vec<i32>, ParseIntError> に集約
}

// 別解: エラーを除外して成功値のみを取得
fn parse_numbers_filter(inputs: Vec<String>) -> Vec<i32> {
    inputs.into_iter()
        .filter_map(|s| s.parse::<i32>().ok())  // Ok の値のみを残す
        .collect()
}

fn main() {
    let inputs = vec!["1".to_string(), "2".to_string(), "invalid".to_string(), "4".to_string()];
    
    // 最初のエラーで失敗するバージョン
    match parse_numbers(inputs.clone()) {
        Ok(numbers) => println!("All parsed: {:?}", numbers),
        Err(error) => println!("Parse error: {}", error),
    }
    
    // エラーをスキップするバージョン
    let numbers = parse_numbers_filter(inputs);
    println!("Successfully parsed: {:?}", numbers);  // [1, 2, 4]
}
```

***

## 構造体 vs クラス

Rust の構造体（struct）は C# のクラス（class）に似ていますが、所有権やメソッドの定義方法に関して重要な違いがあります。

### C# のクラス定義
```csharp
// プロパティとメソッドを持つ C# のクラス
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    public List<string> Hobbies { get; set; }
    
    public Person(string name, int age)
    {
        Name = name;
        Age = age;
        Hobbies = new List<string>();
    }
    
    public void AddHobby(string hobby)
    {
        Hobbies.Add(hobby);
    }
    
    public string GetInfo()
    {
        return $"{Name} is {Age} years old";
    }
}
```

### Rust の構造体定義
```rust
// 関連関数とメソッドを持つ Rust の構造体
#[derive(Debug)]  // Debug トレイトを自動実装
pub struct Person {
    pub name: String,    // 公開フィールド
    pub age: u32,        // 公開フィールド
    hobbies: Vec<String>, // 非公開フィールド（pub なし）
}

impl Person {
    // 関連関数（静的メソッドに相当）
    pub fn new(name: String, age: u32) -> Person {
        Person {
            name,
            age,
            hobbies: Vec::new(),
        }
    }
    
    // メソッド（&self、&mut self、または self を受け取る）
    pub fn add_hobby(&mut self, hobby: String) {
        self.hobbies.push(hobby);
    }
    
    // 不変借用するメソッド
    pub fn get_info(&self) -> String {
        format!("{} is {} years old", self.name, self.age)
    }
    
    // 非公開フィールドのゲッター
    pub fn hobbies(&self) -> &Vec<String> {
        &self.hobbies
    }
}
```

### インスタンスの生成と使用
```csharp
// C# のオブジェクト生成と使用
var person = new Person("Alice", 30);
person.AddHobby("Reading");
person.AddHobby("Swimming");

Console.WriteLine(person.GetInfo());
Console.WriteLine($"Hobbies: {string.Join(", ", person.Hobbies)}");

// プロパティを直接変更
person.Age = 31;
```

```rust
// Rust の構造体生成と使用
let mut person = Person::new("Alice".to_string(), 30);
person.add_hobby("Reading".to_string());
person.add_hobby("Swimming".to_string());

println!("{}", person.get_info());
println!("Hobbies: {:?}", person.hobbies());

// 公開フィールドを直接変更
person.age = 31;

// デバッグ出力で構造体全体を表示
println!("{:?}", person);
```

### 構造体の初期化パターン
```csharp
// C# のオブジェクト初期化子
var person = new Person("Bob", 25)
{
    Hobbies = new List<string> { "Gaming", "Coding" }
};

// 匿名型
var anonymous = new { Name = "Charlie", Age = 35 };
```

```rust
// Rust の構造体初期化
let person = Person {
    name: "Bob".to_string(),
    age: 25,
    hobbies: vec!["Gaming".to_string(), "Coding".to_string()],
};

// 構造体更新構文（オブジェクトスプレッドに類似）
let older_person = Person {
    age: 26,
    ..person  // person の残りのフィールドを使用（person からムーブされる！）
};

// タプル構造体（匿名型に類似）
#[derive(Debug)]
struct Point(i32, i32);

let point = Point(10, 20);
println!("Point: ({}, {})", point.0, point.1);
```

***

## メソッドと関連関数

メソッドと関連関数の違いを理解することが重要です。

### C# のメソッド種別
```csharp
public class Calculator
{
    private int memory = 0;
    
    // インスタンスメソッド
    public int Add(int a, int b)
    {
        return a + b;
    }
    
    // 状態を使用するインスタンスメソッド
    public void StoreInMemory(int value)
    {
        memory = value;
    }
    
    // 静的メソッド
    public static int Multiply(int a, int b)
    {
        return a * b;
    }
    
    // 静的ファクトリメソッド
    public static Calculator CreateWithMemory(int initialMemory)
    {
        var calc = new Calculator();
        calc.memory = initialMemory;
        return calc;
    }
}
```

### Rust のメソッド種別
```rust
#[derive(Debug)]
pub struct Calculator {
    memory: i32,
}

impl Calculator {
    // 関連関数（静的メソッドに相当）— self 引数を持たない
    pub fn new() -> Calculator {
        Calculator { memory: 0 }
    }
    
    // 引数付きの関連関数
    pub fn with_memory(initial_memory: i32) -> Calculator {
        Calculator { memory: initial_memory }
    }
    
    // 不変借用するメソッド (&self)
    pub fn add(&self, a: i32, b: i32) -> i32 {
        a + b
    }
    
    // 可変借用するメソッド (&mut self)
    pub fn store_in_memory(&mut self, value: i32) {
        self.memory = value;
    }
    
    // 所有権を奪うメソッド (self)
    pub fn into_memory(self) -> i32 {
        self.memory  // Calculator は消費される
    }
    
    // ゲッターメソッド
    pub fn memory(&self) -> i32 {
        self.memory
    }
}

fn main() {
    // 関連関数は :: で呼び出す
    let mut calc = Calculator::new();
    let calc2 = Calculator::with_memory(42);
    
    // メソッドは . で呼び出す
    let result = calc.add(5, 3);
    calc.store_in_memory(result);
    
    println!("Memory: {}", calc.memory());
    
    // インスタンスを消費するメソッド
    let memory_value = calc.into_memory();  // calc はこれ以降使用不可
    println!("Final memory: {}", memory_value);
}
```

### メソッドレシーバの型の詳細解説
```rust
impl Person {
    // &self — 不変借用（最も一般的）
    // データを読み取るだけでよい場合に使用
    pub fn get_name(&self) -> &str {
        &self.name
    }
    
    // &mut self — 可変借用
    // データを変更する必要がある場合に使用
    pub fn set_name(&mut self, name: String) {
        self.name = name;
    }
    
    // self — 所有権を消費（やや特殊）
    // 構造体を消費（変換・分解など）したい場合に使用
    pub fn consume(self) -> String {
        self.name  // Person はムーブされ、以降アクセス不可
    }
}

fn method_examples() {
    let mut person = Person::new("Alice".to_string(), 30);
    
    // 不変借用
    let name = person.get_name();  // person は引き続き使用可能
    println!("Name: {}", name);
    
    // 可変借用
    person.set_name("Alice Smith".to_string());  // person は引き続き使用可能
    
    // 所有権の消費
    let final_name = person.consume();  // person はこれ以降使用不可
    println!("Final name: {}", final_name);
}
```

***

## 振る舞いの実装

### C# のインターフェイス実装
```csharp
// C# のインターフェイス
public interface IDrawable
{
    void Draw();
    double GetArea();
}

public class Circle : IDrawable
{
    public double Radius { get; set; }
    
    public Circle(double radius)
    {
        Radius = radius;
    }
    
    public void Draw()
    {
        Console.WriteLine($"Drawing a circle with radius {Radius}");
    }
    
    public double GetArea()
    {
        return Math.PI * Radius * Radius;
    }
}
```

### Rust のトレイト実装（プレビュー）
```rust
// Rust のトレイト（インターフェイスに相当）
trait Drawable {
    fn draw(&self);
    fn get_area(&self) -> f64;
}

#[derive(Debug)]
struct Circle {
    radius: f64,
}

impl Circle {
    pub fn new(radius: f64) -> Circle {
        Circle { radius }
    }
}

// Circle に Drawable トレイトを実装
impl Drawable for Circle {
    fn draw(&self) {
        println!("Drawing a circle with radius {}", self.radius);
    }
    
    fn get_area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}

fn main() {
    let circle = Circle::new(5.0);
    circle.draw();
    println!("Area: {}", circle.get_area());
}
```

### 複数のインターフェイス / トレイトの実装
```csharp
// C# - 複数のインターフェイスを実装するクラス
public interface IComparable<T>
{
    int CompareTo(T other);
}

public class Person : IDrawable, IComparable<Person>
{
    public string Name { get; set; }
    public int Age { get; set; }
    
    public void Draw()
    {
        Console.WriteLine($"Drawing person: {Name}");
    }
    
    public double GetArea()
    {
        return 0.0; // 人間に面積はない
    }
    
    public int CompareTo(Person other)
    {
        return Age.CompareTo(other.Age);
    }
}
```

```rust
// Rust - 複数のトレイト実装
use std::cmp::Ordering;

impl Drawable for Person {
    fn draw(&self) {
        println!("Drawing person: {}", self.name);
    }
    
    fn get_area(&self) -> f64 {
        0.0  // 人間に面積はない
    }
}

impl PartialOrd for Person {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        self.age.partial_cmp(&other.age)
    }
}

impl PartialEq for Person {
    fn eq(&self, other: &Self) -> bool {
        self.age == other.age
    }
}

fn main() {
    let mut people = vec![
        Person::new("Alice".to_string(), 30),
        Person::new("Bob".to_string(), 25),
        Person::new("Charlie".to_string(), 35),
    ];
    
    people.sort_by(|a, b| a.partial_cmp(b).unwrap());
    
    for person in &people {
        person.draw();
    }
}
```

***

## コンストラクタパターン

### C# のコンストラクタパターン
```csharp
public class Configuration
{
    public string DatabaseUrl { get; set; }
    public int MaxConnections { get; set; }
    public bool EnableLogging { get; set; }
    
    // デフォルトコンストラクタ
    public Configuration()
    {
        DatabaseUrl = "localhost";
        MaxConnections = 10;
        EnableLogging = false;
    }
    
    // パラメータ付きコンストラクタ
    public Configuration(string databaseUrl, int maxConnections)
    {
        DatabaseUrl = databaseUrl;
        MaxConnections = maxConnections;
        EnableLogging = false;
    }
    
    // ファクトリメソッド
    public static Configuration ForProduction()
    {
        return new Configuration("prod.db.server", 100)
        {
            EnableLogging = true
        };
    }
}
```

### Rust のコンストラクタパターン
```rust
#[derive(Debug)]
pub struct Configuration {
    pub database_url: String,
    pub max_connections: u32,
    pub enable_logging: bool,
}

impl Configuration {
    // デフォルトコンストラクタ
    pub fn new() -> Configuration {
        Configuration {
            database_url: "localhost".to_string(),
            max_connections: 10,
            enable_logging: false,
        }
    }
    
    // パラメータ付きコンストラクタ
    pub fn with_database(database_url: String, max_connections: u32) -> Configuration {
        Configuration {
            database_url,
            max_connections,
            enable_logging: false,
        }
    }
    
    // ファクトリメソッド
    pub fn for_production() -> Configuration {
        Configuration {
            database_url: "prod.db.server".to_string(),
            max_connections: 100,
            enable_logging: true,
        }
    }
    
    // ビルダーパターンのメソッド
    pub fn enable_logging(mut self) -> Configuration {
        self.enable_logging = true;
        self  // メソッドチェーン用に self を返す
    }
    
    pub fn max_connections(mut self, count: u32) -> Configuration {
        self.max_connections = count;
        self
    }
}

// Default トレイトの実装
impl Default for Configuration {
    fn default() -> Self {
        Self::new()
    }
}

fn main() {
    // さまざまな生成パターン
    let config1 = Configuration::new();
    let config2 = Configuration::with_database("localhost:5432".to_string(), 20);
    let config3 = Configuration::for_production();
    
    // ビルダーパターン
    let config4 = Configuration::new()
        .enable_logging()
        .max_connections(50);
    
    // Default トレイトの使用
    let config5 = Configuration::default();
    
    println!("{:?}", config4);
}
```

### ビルダーパターンの実装
```rust
// より高度なビルダーパターン
#[derive(Debug)]
pub struct DatabaseConfig {
    host: String,
    port: u16,
    username: String,
    password: Option<String>,
    ssl_enabled: bool,
    timeout_seconds: u64,
}

pub struct DatabaseConfigBuilder {
    host: Option<String>,
    port: Option<u16>,
    username: Option<String>,
    password: Option<String>,
    ssl_enabled: bool,
    timeout_seconds: u64,
}

impl DatabaseConfigBuilder {
    pub fn new() -> Self {
        DatabaseConfigBuilder {
            host: None,
            port: None,
            username: None,
            password: None,
            ssl_enabled: false,
            timeout_seconds: 30,
        }
    }
    
    pub fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }
    
    pub fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }
    
    pub fn username(mut self, username: impl Into<String>) -> Self {
        self.username = Some(username.into());
        self
    }
    
    pub fn password(mut self, password: impl Into<String>) -> Self {
        self.password = Some(password.into());
        self
    }
    
    pub fn enable_ssl(mut self) -> Self {
        self.ssl_enabled = true;
        self
    }
    
    pub fn timeout(mut self, seconds: u64) -> Self {
        self.timeout_seconds = seconds;
        self
    }
    
    pub fn build(self) -> Result<DatabaseConfig, String> {
        let host = self.host.ok_or("Host is required")?;
        let port = self.port.ok_or("Port is required")?;
        let username = self.username.ok_or("Username is required")?;
        
        Ok(DatabaseConfig {
            host,
            port,
            username,
            password: self.password,
            ssl_enabled: self.ssl_enabled,
            timeout_seconds: self.timeout_seconds,
        })
    }
}

fn main() {
    let config = DatabaseConfigBuilder::new()
        .host("localhost")
        .port(5432)
        .username("admin")
        .password("secret123")
        .enable_ssl()
        .timeout(60)
        .build()
        .expect("Failed to build config");
    
    println!("{:?}", config);
}
```

***

## 列挙型とパターンマッチング

Rust の列挙型（enum）は C# の enum よりはるかに強力です。各バリアントにデータを持たせることができ、型安全なプログラミングの基礎となります。

### C# の enum の限界
```csharp
// C# の enum - 単なる名前付き定数
public enum Status
{
    Pending,
    Approved,
    Rejected
}

// 基底となる値を持つ C# の enum
public enum HttpStatusCode
{
    OK = 200,
    NotFound = 404,
    InternalServerError = 500
}

// 複雑なデータを保持するには個別のクラスが必要
public abstract class Result
{
    public abstract bool IsSuccess { get; }
}

public class Success : Result
{
    public string Value { get; }
    public override bool IsSuccess => true;
    
    public Success(string value)
    {
        Value = value;
    }
}

public class Error : Result
{
    public string Message { get; }
    public override bool IsSuccess => false;
    
    public Error(string message)
    {
        Message = message;
    }
}
```

### Rust の列挙型の真価
```rust
// 単純な列挙型（C# の enum に相当）
#[derive(Debug, PartialEq)]
enum Status {
    Pending,
    Approved,
    Rejected,
}

// データを保持する列挙型（Rust の真骨頂！）
#[derive(Debug)]
enum Result<T, E> {
    Ok(T),      // 型 T の値を保持する成功バリアント
    Err(E),     // 型 E のエラーを保持する失敗バリアント
}

// 異なるデータ型を持つ複雑な列挙型
#[derive(Debug)]
enum Message {
    Quit,                       // データなし
    Move { x: i32, y: i32 },   // 構造体スタイルのバリアント
    Write(String),             // タプルスタイルのバリアント
    ChangeColor(i32, i32, i32), // 複数の値を保持
}

// 実世界での例: HTTP レスポンス
#[derive(Debug)]
enum HttpResponse {
    Ok { body: String, headers: Vec<String> },
    NotFound { path: String },
    InternalError { message: String, code: u16 },
    Redirect { location: String },
}
```

### match によるパターンマッチング
```csharp
// C# の switch 文（機能が限定的）
public string HandleStatus(Status status)
{
    switch (status)
    {
        case Status.Pending:
            return "Waiting for approval";
        case Status.Approved:
            return "Request approved";
        case Status.Rejected:
            return "Request rejected";
        default:
            return "Unknown status"; // 常に default が必要
    }
}

// C# のパターンマッチング (C# 8+)
public string HandleResult(Result result)
{
    return result switch
    {
        Success success => $"Success: {success.Value}",
        Error error => $"Error: {error.Message}",
        _ => "Unknown result" // やはりフォールバックが必要
    };
}
```

```rust
// Rust の match — 網羅的かつ強力
fn handle_status(status: Status) -> String {
    match status {
        Status::Pending => "Waiting for approval".to_string(),
        Status::Approved => "Request approved".to_string(),
        Status::Rejected => "Request rejected".to_string(),
        // default は不要 — コンパイラが全ケースの網羅を保証する
    }
}

// データの抽出を伴うパターンマッチング
fn handle_result<T, E>(result: Result<T, E>) -> String 
where 
    T: std::fmt::Debug,
    E: std::fmt::Debug,
{
    match result {
        Result::Ok(value) => format!("Success: {:?}", value),
        Result::Err(error) => format!("Error: {:?}", error),
        // 網羅的 — default は不要
    }
}

// 複雑なパターンマッチング
fn handle_message(msg: Message) -> String {
    match msg {
        Message::Quit => "Goodbye!".to_string(),
        Message::Move { x, y } => format!("Move to ({}, {})", x, y),
        Message::Write(text) => format!("Write: {}", text),
        Message::ChangeColor(r, g, b) => format!("Change color to RGB({}, {}, {})", r, g, b),
    }
}

// HTTP レスポンスの処理
fn handle_http_response(response: HttpResponse) -> String {
    match response {
        HttpResponse::Ok { body, headers } => {
            format!("Success! Body: {}, Headers: {:?}", body, headers)
        },
        HttpResponse::NotFound { path } => {
            format!("404: Path '{}' not found", path)
        },
        HttpResponse::InternalError { message, code } => {
            format!("Error {}: {}", code, message)
        },
        HttpResponse::Redirect { location } => {
            format!("Redirect to: {}", location)
        },
    }
}
```

### ガードと高度なパターン
```rust
// ガード付きパターンマッチング
fn describe_number(x: i32) -> String {
    match x {
        n if n < 0 => "negative".to_string(),
        0 => "zero".to_string(),
        n if n < 10 => "single digit".to_string(),
        n if n < 100 => "double digit".to_string(),
        _ => "large number".to_string(),
    }
}

// 範囲のマッチング
fn describe_age(age: u32) -> String {
    match age {
        0..=12 => "child".to_string(),
        13..=19 => "teenager".to_string(),
        20..=64 => "adult".to_string(),
        65.. => "senior".to_string(),
    }
}

// 構造体とタプルの分解束縛
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn describe_point(point: Point) -> String {
    match point {
        Point { x: 0, y: 0 } => "origin".to_string(),
        Point { x: 0, y } => format!("on y-axis at y={}", y),
        Point { x, y: 0 } => format!("on x-axis at x={}", x),
        Point { x, y } if x == y => format!("on diagonal at ({}, {})", x, y),
        Point { x, y } => format!("point at ({}, {})", x, y),
    }
}
```

### Option 型と Result 型
```csharp
// C# の null 許容参照型 (C# 8+)
public class PersonService
{
    private Dictionary<int, string> people = new();
    
    public string? FindPerson(int id)
    {
        return people.TryGetValue(id, out string? name) ? name : null;
    }
    
    public string GetPersonOrDefault(int id)
    {
        return FindPerson(id) ?? "Unknown";
    }
    
    // 例外ベースのエラー処理
    public void SavePerson(int id, string name)
    {
        if (string.IsNullOrEmpty(name))
            throw new ArgumentException("Name cannot be empty");
        
        people[id] = name;
    }
}
```

```rust
use std::collections::HashMap;

// Rust では null の代わりに Option<T> を使用する
struct PersonService {
    people: HashMap<i32, String>,
}

impl PersonService {
    fn new() -> Self {
        PersonService {
            people: HashMap::new(),
        }
    }
    
    // Option<T> を返す — null は存在しない！
    fn find_person(&self, id: i32) -> Option<&String> {
        self.people.get(&id)
    }
    
    // Option に対するパターンマッチング
    fn get_person_or_default(&self, id: i32) -> String {
        match self.find_person(id) {
            Some(name) => name.clone(),
            None => "Unknown".to_string(),
        }
    }
    
    // Option のメソッドを使用（より関数型のスタイル）
    fn get_person_or_default_functional(&self, id: i32) -> String {
        self.find_person(id)
            .map(|name| name.clone())
            .unwrap_or_else(|| "Unknown".to_string())
    }
    
    // エラー処理のための Result<T, E>
    fn save_person(&mut self, id: i32, name: String) -> Result<(), String> {
        if name.is_empty() {
            return Err("Name cannot be empty".to_string());
        }
        
        self.people.insert(id, name);
        Ok(())
    }
    
    // 操作の連鎖
    fn get_person_length(&self, id: i32) -> Option<usize> {
        self.find_person(id).map(|name| name.len())
    }
}

fn main() {
    let mut service = PersonService::new();
    
    // Result の処理
    match service.save_person(1, "Alice".to_string()) {
        Ok(()) => println!("Person saved successfully"),
        Err(error) => println!("Error: {}", error),
    }
    
    // Option の処理
    match service.find_person(1) {
        Some(name) => println!("Found: {}", name),
        None => println!("Person not found"),
    }
    
    // Option を使った関数型スタイル
    let name_length = service.get_person_length(1)
        .unwrap_or(0);
    println!("Name length: {}", name_length);
    
    // 早期リターンのための ? 演算子
    fn try_operation(service: &mut PersonService) -> Result<String, String> {
        service.save_person(2, "Bob".to_string())?; // エラー時は早期リターン
        let name = service.find_person(2).ok_or("Person not found")?; // Option を Result に変換
        Ok(format!("Hello, {}", name))
    }
    
    match try_operation(&mut service) {
        Ok(message) => println!("{}", message),
        Err(error) => println!("Operation failed: {}", error),
    }
}
```

### カスタムエラー型
```rust
// カスタムエラー enum の定義
#[derive(Debug)]
enum PersonError {
    NotFound(i32),
    InvalidName(String),
    DatabaseError(String),
}

impl std::fmt::Display for PersonError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            PersonError::NotFound(id) => write!(f, "Person with ID {} not found", id),
            PersonError::InvalidName(name) => write!(f, "Invalid name: '{}'", name),
            PersonError::DatabaseError(msg) => write!(f, "Database error: {}", msg),
        }
    }
}

impl std::error::Error for PersonError {}

// カスタムエラーを用いた機能拡張版 PersonService
impl PersonService {
    fn save_person_enhanced(&mut self, id: i32, name: String) -> Result<(), PersonError> {
        if name.is_empty() || name.len() > 50 {
            return Err(PersonError::InvalidName(name));
        }
        
        // 失敗する可能性のあるデータベース操作のシミュレーション
        if id < 0 {
            return Err(PersonError::DatabaseError("Negative IDs not allowed".to_string()));
        }
        
        self.people.insert(id, name);
        Ok(())
    }
    
    fn find_person_enhanced(&self, id: i32) -> Result<&String, PersonError> {
        self.people.get(&id).ok_or(PersonError::NotFound(id))
    }
}

fn demo_error_handling() {
    let mut service = PersonService::new();
    
    // さまざまなエラー型を個別に処理
    match service.save_person_enhanced(-1, "Invalid".to_string()) {
        Ok(()) => println!("Success"),
        Err(PersonError::NotFound(id)) => println!("Not found: {}", id),
        Err(PersonError::InvalidName(name)) => println!("Invalid name: {}", name),
        Err(PersonError::DatabaseError(msg)) => println!("DB Error: {}", msg),
    }
}
```

***

## モジュールとクレート: コードの構成

Rust のモジュールシステムを理解することは、コードの整理や依存関係の管理において不可欠です。C# 開発者にとっては、名前空間（namespace）、アセンブリ（assembly）、NuGet パッケージの理解に相当します。

### Rust のモジュール vs C# の名前空間

#### C# の名前空間による構成
```csharp
// ファイル: Models/User.cs
namespace MyApp.Models
{
    public class User
    {
        public string Name { get; set; }
        public int Age { get; set; }
    }
}

// ファイル: Services/UserService.cs
using MyApp.Models;

namespace MyApp.Services
{
    public class UserService
    {
        public User CreateUser(string name, int age)
        {
            return new User { Name = name, Age = age };
        }
    }
}

// ファイル: Program.cs
using MyApp.Models;
using MyApp.Services;

namespace MyApp
{
    class Program
    {
        static void Main(string[] args)
        {
            var service = new UserService();
            var user = service.CreateUser("Alice", 30);
        }
    }
}
```

#### Rust のモジュールによる構成
```rust
// ファイル: src/models.rs
pub struct User {
    pub name: String,
    pub age: u32,
}

impl User {
    pub fn new(name: String, age: u32) -> User {
        User { name, age }
    }
}

// ファイル: src/services.rs
use crate::models::User;

pub struct UserService;

impl UserService {
    pub fn create_user(name: String, age: u32) -> User {
        User::new(name, age)
    }
}

// ファイル: src/lib.rs（または main.rs）
pub mod models;
pub mod services;

use models::User;
use services::UserService;

fn main() {
    let service = UserService;
    let user = UserService::create_user("Alice".to_string(), 30);
}
```

### モジュール階層と可視性

#### C# の可視性修飾子
```csharp
namespace MyApp.Data
{
    // public - どこからでもアクセス可能
    public class Repository
    {
        // private - このクラス内からのみアクセス可能
        private string connectionString;
        
        // internal - このアセンブリ内からアクセス可能
        internal void Connect() { }
        
        // protected - このクラスおよび派生クラスからアクセス可能
        protected virtual void Initialize() { }
        
        // public - どこからでもアクセス可能
        public void Save(object data) { }
    }
}
```

#### Rust の可視性規則
```rust
// Rust ではすべてがデフォルトで非公開（private）
mod data {
    struct Repository {  // 非公開構造体
        connection_string: String,  // 非公開フィールド
    }
    
    impl Repository {
        fn new() -> Repository {  // 非公開関数
            Repository {
                connection_string: "localhost".to_string(),
            }
        }
        
        pub fn connect(&self) {  // 公開メソッド
            // このモジュールおよびその子モジュール内からのみアクセス可能
        }
        
        pub(crate) fn initialize(&self) {  // クレートレベルで公開
            // このクレート内のどこからでもアクセス可能
        }
        
        pub(super) fn internal_method(&self) {  // 親モジュールに対して公開
            // 親モジュール内からアクセス可能
        }
    }
    
    // 公開構造体 — モジュール外からアクセス可能
    pub struct PublicRepository {
        pub data: String,  // 公開フィールド
        private_data: String,  // 非公開フィールド（pub なし）
    }
}

pub use data::PublicRepository;  // 外部利用のために再エクスポート
```

### モジュールのファイル構成

#### C# のプロジェクト構造
```
MyApp/
├── MyApp.csproj
├── Models/
│   ├── User.cs
│   └── Product.cs
├── Services/
│   ├── UserService.cs
│   └── ProductService.cs
├── Controllers/
│   └── ApiController.cs
└── Program.cs
```

#### Rust のモジュールファイル構造
```
my_app/
├── Cargo.toml
└── src/
    ├── main.rs (または lib.rs)
    ├── models/
    │   ├── mod.rs        // モジュール宣言
    │   ├── user.rs
    │   └── product.rs
    ├── services/
    │   ├── mod.rs        // モジュール宣言
    │   ├── user_service.rs
    │   └── product_service.rs
    └── controllers/
        ├── mod.rs
        └── api_controller.rs
```

#### モジュール宣言のパターン
```rust
// src/models/mod.rs
pub mod user;      // user.rs をサブモジュールとして宣言
pub mod product;   // product.rs をサブモジュールとして宣言

// よく使われる型を再エクスポート
pub use user::User;
pub use product::Product;

// src/main.rs
mod models;     // models/ をモジュールとして宣言
mod services;   // services/ をモジュールとして宣言

// 特定のアイテムをインポート
use models::{User, Product};
use services::UserService;

// またはモジュール全体をインポート
use models::user::*;  // user モジュール内のすべての公開アイテムをインポート
```

***

## クレート vs .NET アセンブリ

### クレートの理解
Rust における**クレート（crate）**は、コンパイルとコード配布の基本単位であり、.NET における**アセンブリ（assembly）**の役割に相当します。

#### C# のアセンブリモデル
```csharp
// MyLibrary.dll - コンパイルされたアセンブリ
namespace MyLibrary
{
    public class Calculator
    {
        public int Add(int a, int b) => a + b;
    }
}

// MyApp.exe - MyLibrary.dll を参照する実行可能アセンブリ
using MyLibrary;

class Program
{
    static void Main()
    {
        var calc = new Calculator();
        Console.WriteLine(calc.Add(2, 3));
    }
}
```

#### Rust のクレートモデル
```toml
# ライブラリクレートの Cargo.toml
[package]
name = "my_calculator"
version = "0.1.0"
edition = "2021"

[lib]
name = "my_calculator"
```

```rust
// src/lib.rs - ライブラリクレート
pub struct Calculator;

impl Calculator {
    pub fn add(&self, a: i32, b: i32) -> i32 {
        a + b
    }
}
```

```toml
# ライブラリを使用するバイナリクレートの Cargo.toml
[package]
name = "my_app"
version = "0.1.0"
edition = "2021"

[dependencies]
my_calculator = { path = "../my_calculator" }
```

```rust
// src/main.rs - バイナリクレート
use my_calculator::Calculator;

fn main() {
    let calc = Calculator;
    println!("{}", calc.add(2, 3));
}
```

### クレート種別の比較

| C# の概念 | Rust の対応概念 | 用途 |
|------------|----------------|---------|
| クラスライブラリ (.dll) | ライブラリクレート (lib crate) | 再利用可能なコード |
| コンソールアプリ (.exe) | バイナリクレート (bin crate) | 実行可能プログラム |
| NuGet パッケージ | 公開クレート (published crate) | 配布単位 |
| アセンブリ (.dll/.exe) | コンパイル済みクレート | コンパイル単位 |
| ソリューション (.sln) | ワークスペース (workspace) | 複数プロジェクトの統合管理 |

### ワークスペース vs ソリューション

#### C# のソリューション構造
```xml
<!-- MySolution.sln の構造概念 -->
<Solution>
    <Project Include="WebApi/WebApi.csproj" />
    <Project Include="Business/Business.csproj" />
    <Project Include="DataAccess/DataAccess.csproj" />
    <Project Include="Tests/Tests.csproj" />
</Solution>
```

#### Rust のワークスペース構造
```toml
# ワークスペースルートの Cargo.toml
[workspace]
members = [
    "web_api",
    "business",
    "data_access",
    "tests"
]

[workspace.dependencies]
serde = "1.0"           # 共有する依存関係のバージョン
tokio = "1.0"
```

```toml
# web_api/Cargo.toml
[package]
name = "web_api"
version = "0.1.0"
edition = "2021"

[dependencies]
business = { path = "../business" }
serde = { workspace = true }    # ワークスペースのバージョンを使用
tokio = { workspace = true }
```

***

## パッケージ管理: Cargo vs NuGet

### 依存関係の宣言

#### C# の NuGet 依存関係
```xml
<!-- MyApp.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  <PackageReference Include="Serilog" Version="3.0.1" />
  <PackageReference Include="Microsoft.AspNetCore.App" />
  
  <ProjectReference Include="../MyLibrary/MyLibrary.csproj" />
</Project>
```

#### Rust の Cargo 依存関係
```toml
# Cargo.toml
[package]
name = "my_app"
version = "0.1.0"
edition = "2021"

[dependencies]
serde_json = "1.0"               # crates.io から取得（NuGet に相当）
serde = { version = "1.0", features = ["derive"] }  # フィーチャー付き
log = "0.4"
tokio = { version = "1.0", features = ["full"] }

# ローカル依存（ProjectReference に相当）
my_library = { path = "../my_library" }

# Git リポジトリからの依存
my_git_crate = { git = "https://github.com/user/repo" }

# 開発用依存（テストパッケージ等に相当）
[dev-dependencies]
criterion = "0.5"               # ベンチマーク
proptest = "1.0"               # プロパティベーステスト
```

### バージョン管理

#### C# のパッケージバージョン管理
```xml
<!-- 中央パッケージ管理 (Directory.Packages.props) -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  
  <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
  <PackageVersion Include="Serilog" Version="3.0.1" />
</Project>

<!-- 再現可能なビルドのための packages.lock.json -->
```

#### Rust のバージョン管理
```toml
# Cargo.toml - セマンティックバージョニング
[dependencies]
serde = "1.0"        # 1.x.x と互換性あり (>=1.0.0, <2.0.0)
log = "0.4.17"       # 0.4.x と互換性あり (>=0.4.17, <0.5.0)
regex = "=1.5.4"     # 完全一致バージョン
chrono = "^0.4"      # キャレット要件（デフォルト動作）
uuid = "~1.3.0"      # チルダ要件 (>=1.3.0, <1.4.0)

# Cargo.lock - 再現可能なビルドのための完全一致バージョン（自動生成）
[[package]]
name = "serde"
version = "1.0.163"
# ... 正確な依存関係ツリー
```

### パッケージソース

#### C# のパッケージソース
```xml
<!-- nuget.config -->
<configuration>
  <packageSources>
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="MyCompanyFeed" value="https://pkgs.dev.azure.com/company/_packaging/feed/nuget/v3/index.json" />
  </packageSources>
</configuration>
```

#### Rust のパッケージソース
```toml
# .cargo/config.toml
[source.crates-io]
replace-with = "my-awesome-registry"

[source.my-awesome-registry]
registry = "https://my-intranet:8080/index"

# 代替レジストリ
[registries]
my-registry = { index = "https://my-intranet:8080/index" }

# Cargo.toml 内での指定
[dependencies]
my_crate = { version = "1.0", registry = "my-registry" }
```

### 一般的なコマンドの比較

| タスク | C# コマンド | Rust コマンド |
|------|------------|-------------|
| パッケージの復元 | `dotnet restore` | `cargo fetch` |
| パッケージの追加 | `dotnet add package Newtonsoft.Json` | `cargo add serde_json` |
| パッケージの削除 | `dotnet remove package Newtonsoft.Json` | `cargo remove serde_json` |
| パッケージの更新 | `dotnet update` | `cargo update` |
| パッケージ一覧表示 | `dotnet list package` | `cargo tree` |
| セキュリティ監査 | `dotnet list package --vulnerable` | `cargo audit` |
| ビルド成果物のクリーン | `dotnet clean` | `cargo clean` |

### フィーチャー（Features）: 条件付きコンパイル

#### C# の条件付きコンパイル
```csharp
#if DEBUG
    Console.WriteLine("Debug mode");
#elif RELEASE
    Console.WriteLine("Release mode");
#endif

// プロジェクトファイルの条件定義
<PropertyGroup Condition="'$(Configuration)'=='Debug'">
    <DefineConstants>DEBUG;TRACE</DefineConstants>
</PropertyGroup>
```

#### Rust のフィーチャーゲート
```toml
# Cargo.toml
[features]
default = ["json"]              # デフォルトで有効なフィーチャー
json = ["serde_json"]          # serde_json を有効化するフィーチャー
xml = ["serde_xml"]            # 代替シリアライズ形式
advanced = ["json", "xml"]     # 複合フィーチャー

[dependencies]
serde_json = { version = "1.0", optional = true }
serde_xml = { version = "0.4", optional = true }
```

```rust
// フィーチャーに基づく条件付きコンパイル
#[cfg(feature = "json")]
use serde_json;

#[cfg(feature = "xml")]
use serde_xml;

pub fn serialize_data(data: &MyStruct) -> String {
    #[cfg(feature = "json")]
    return serde_json::to_string(data).unwrap();
    
    #[cfg(feature = "xml")]
    return serde_xml::to_string(data).unwrap();
    
    #[cfg(not(any(feature = "json", feature = "xml")))]
    return "No serialization feature enabled".to_string();
}
```

### 外部クレートの活用

#### C# 開発者に親しみやすい代表的クレート

| C# ライブラリ | Rust クレート | 用途 |
|------------|------------|---------|
| Newtonsoft.Json | `serde_json` | JSON シリアライズ / デシリアライズ |
| HttpClient | `reqwest` | HTTP クライアント |
| Entity Framework | `diesel` / `sqlx` | ORM / SQL ツールキット |
| NLog / Serilog | `log` + `env_logger` | ロギング |
| xUnit / NUnit | 組み込みの `#[test]` | 単体テスト |
| Moq | `mockall` | モック作成 |
| Flurl | `url` | URL 操作 |
| Polly | `tower` | レジリエンスパターン |

#### 実装例: HTTP クライアントの移行
```csharp
// C# HttpClient の使用例
public class ApiClient
{
    private readonly HttpClient _httpClient;
    
    public async Task<User> GetUserAsync(int id)
    {
        var response = await _httpClient.GetAsync($"/users/{id}");
        var json = await response.Content.ReadAsStringAsync();
        return JsonConvert.DeserializeObject<User>(json);
    }
}
```

```rust
// Rust reqwest の使用例
use reqwest;
use serde::Deserialize;

#[derive(Deserialize)]
struct User {
    id: u32,
    name: String,
}

struct ApiClient {
    client: reqwest::Client,
}

impl ApiClient {
    async fn get_user(&self, id: u32) -> Result<User, reqwest::Error> {
        let user = self.client
            .get(&format!("https://api.example.com/users/{}", id))
            .send()
            .await?
            .json::<User>()
            .await?;
        
        Ok(user)
    }
}
```

***

## トレイト — Rust のインターフェイス

トレイト（Trait）は Rust において共通の振る舞いを定義するための仕組みであり、C# のインターフェイスに類似していますが、より柔軟で強力な表現力を持ちます。

### C# のインターフェイスとの比較
```csharp
// C# のインターフェイス定義
public interface IAnimal
{
    string Name { get; }
    void MakeSound();
    
    // デフォルト実装 (C# 8+)
    string Describe()
    {
        return $"{Name} makes a sound";
    }
}

// C# のインターフェイス実装
public class Dog : IAnimal
{
    public string Name { get; }
    
    public Dog(string name)
    {
        Name = name;
    }
    
    public void MakeSound()
    {
        Console.WriteLine("Woof!");
    }
    
    // デフォルト実装のオーバーライドが可能
    public string Describe()
    {
        return $"{Name} is a loyal dog";
    }
}

// ジェネリック制約
public void ProcessAnimal<T>(T animal) where T : IAnimal
{
    animal.MakeSound();
    Console.WriteLine(animal.Describe());
}
```

### Rust のトレイト定義と実装
```rust
// トレイトの定義
trait Animal {
    fn name(&self) -> &str;
    fn make_sound(&self);
    
    // デフォルト実装
    fn describe(&self) -> String {
        format!("{} makes a sound", self.name())
    }
    
    // 他のトレイトメソッドを呼び出すデフォルト実装
    fn introduce(&self) {
        println!("Hi, I'm {}", self.name());
        self.make_sound();
    }
}

// 構造体の定義
#[derive(Debug)]
struct Dog {
    name: String,
    breed: String,
}

impl Dog {
    fn new(name: String, breed: String) -> Dog {
        Dog { name, breed }
    }
}

// トレイトの実装
impl Animal for Dog {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn make_sound(&self) {
        println!("Woof!");
    }
    
    // デフォルト実装のオーバーライド
    fn describe(&self) -> String {
        format!("{} is a loyal {} dog", self.name, self.breed)
    }
}

// 別の構造体での実装
#[derive(Debug)]
struct Cat {
    name: String,
    indoor: bool,
}

impl Animal for Cat {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn make_sound(&self) {
        println!("Meow!");
    }
    
    // describe() はデフォルト実装をそのまま利用
}

// トレイト境界を持つジェネリック関数
fn process_animal<T: Animal>(animal: &T) {
    animal.make_sound();
    println!("{}", animal.describe());
    animal.introduce();
}

// 複数のトレイト境界
fn process_animal_debug<T: Animal + std::fmt::Debug>(animal: &T) {
    println!("Debug: {:?}", animal);
    process_animal(animal);
}

fn main() {
    let dog = Dog::new("Buddy".to_string(), "Golden Retriever".to_string());
    let cat = Cat { name: "Whiskers".to_string(), indoor: true };
    
    process_animal(&dog);
    process_animal(&cat);
    
    process_animal_debug(&dog);
}
```

### トレイトオブジェクトと動的ディスパッチ
```csharp
// C# の動的ポリモーフィズム
public void ProcessAnimals(List<IAnimal> animals)
{
    foreach (var animal in animals)
    {
        animal.MakeSound(); // 動的ディスパッチ（仮想関数テーブル経由）
        Console.WriteLine(animal.Describe());
    }
}

// 使用例
var animals = new List<IAnimal>
{
    new Dog("Buddy"),
    new Cat("Whiskers"),
    new Dog("Rex")
};

ProcessAnimals(animals);
```

```rust
// 動的ディスパッチのための Rust トレイトオブジェクト
fn process_animals(animals: &[Box<dyn Animal>]) {
    for animal in animals {
        animal.make_sound(); // 動的ディスパッチ
        println!("{}", animal.describe());
    }
}

// 別解: 参照を使用するパターン
fn process_animal_refs(animals: &[&dyn Animal]) {
    for animal in animals {
        animal.make_sound();
        println!("{}", animal.describe());
    }
}

fn main() {
    // Box<dyn Trait> の使用
    let animals: Vec<Box<dyn Animal>> = vec![
        Box::new(Dog::new("Buddy".to_string(), "Golden Retriever".to_string())),
        Box::new(Cat { name: "Whiskers".to_string(), indoor: true }),
        Box::new(Dog::new("Rex".to_string(), "German Shepherd".to_string())),
    ];
    
    process_animals(&animals);
    
    // 参照の使用
    let dog = Dog::new("Buddy".to_string(), "Golden Retriever".to_string());
    let cat = Cat { name: "Whiskers".to_string(), indoor: true };
    
    let animal_refs: Vec<&dyn Animal> = vec![&dog, &cat];
    process_animal_refs(&animal_refs);
}
```

### 導出（derive）可能なトレイト
```rust
// 一般的なトレイトの自動導出
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct Person {
    name: String,
    age: u32,
}

// 上記によって生成されるコード（簡略化表現）:
impl std::fmt::Debug for Person {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Person")
            .field("name", &self.name)
            .field("age", &self.age)
            .finish()
    }
}

impl Clone for Person {
    fn clone(&self) -> Self {
        Person {
            name: self.name.clone(),
            age: self.age,
        }
    }
}

impl PartialEq for Person {
    fn eq(&self, other: &Self) -> bool {
        self.name == other.name && self.age == other.age
    }
}

// 使用例
fn main() {
    let person1 = Person {
        name: "Alice".to_string(),
        age: 30,
    };
    
    let person2 = person1.clone(); // Clone トレイト
    
    println!("{:?}", person1); // Debug トレイト
    println!("Equal: {}", person1 == person2); // PartialEq トレイト
}
```

### 標準ライブラリの代表的なトレイト
```rust
use std::collections::HashMap;

// ユーザー向け表示用文字列のための Display トレイト
impl std::fmt::Display for Person {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{} (age {})", self.name, self.age)
    }
}

// 型変換のための From トレイト
impl From<(String, u32)> for Person {
    fn from((name, age): (String, u32)) -> Self {
        Person { name, age }
    }
}

// From を実装すると Into トレイトは自動的に実装される
fn create_person() {
    let person: Person = ("Alice".to_string(), 30).into();
    println!("{}", person);
}

// Iterator トレイトの実装
struct PersonIterator {
    people: Vec<Person>,
    index: usize,
}

impl Iterator for PersonIterator {
    type Item = Person;
    
    fn next(&mut self) -> Option<Self::Item> {
        if self.index < self.people.len() {
            let person = self.people[self.index].clone();
            self.index += 1;
            Some(person)
        } else {
            None
        }
    }
}

impl Person {
    fn iterator(people: Vec<Person>) -> PersonIterator {
        PersonIterator { people, index: 0 }
    }
}

fn main() {
    let people = vec![
        Person::from(("Alice".to_string(), 30)),
        Person::from(("Bob".to_string(), 25)),
        Person::from(("Charlie".to_string(), 35)),
    ];
    
    // 自作のイテレータを使用
    for person in Person::iterator(people.clone()) {
        println!("{}", person); // Display トレイトを利用
    }
}
```

***

## エラー処理の深掘り

### C# の例外モデル
```csharp
public class FileProcessor
{
    public string ProcessFile(string path)
    {
        try
        {
            var content = File.ReadAllText(path);
            
            if (string.IsNullOrEmpty(content))
                throw new InvalidOperationException("File is empty");
            
            return content.ToUpper();
        }
        catch (FileNotFoundException)
        {
            throw new ApplicationException($"File not found: {path}");
        }
        catch (UnauthorizedAccessException)
        {
            throw new ApplicationException($"Access denied: {path}");
        }
        catch (Exception ex)
        {
            throw new ApplicationException($"Unexpected error: {ex.Message}");
        }
    }
    
    public async Task<List<string>> ProcessMultipleFiles(List<string> paths)
    {
        var results = new List<string>();
        
        foreach (var path in paths)
        {
            try
            {
                var result = ProcessFile(path);
                results.Add(result);
            }
            catch (Exception ex)
            {
                // エラーをログに記録しつつ他のファイルの処理を継続
                Console.WriteLine($"Error processing {path}: {ex.Message}");
            }
        }
        
        return results;
    }
}
```

### Rust の Result に基づくエラー処理
```rust
use std::fs;
use std::io;

#[derive(Debug)]
enum ProcessingError {
    FileNotFound(String),
    AccessDenied(String),
    EmptyFile(String),
    IoError(io::Error),
}

impl std::fmt::Display for ProcessingError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ProcessingError::FileNotFound(path) => write!(f, "File not found: {}", path),
            ProcessingError::AccessDenied(path) => write!(f, "Access denied: {}", path),
            ProcessingError::EmptyFile(path) => write!(f, "File is empty: {}", path),
            ProcessingError::IoError(err) => write!(f, "IO error: {}", err),
        }
    }
}

impl std::error::Error for ProcessingError {}

impl From<io::Error> for ProcessingError {
    fn from(error: io::Error) -> Self {
        ProcessingError::IoError(error)
    }
}

struct FileProcessor;

impl FileProcessor {
    fn process_file(path: &str) -> Result<String, ProcessingError> {
        // 早期リターンのための ? 演算子の活用
        let content = fs::read_to_string(path)
            .map_err(|err| match err.kind() {
                io::ErrorKind::NotFound => ProcessingError::FileNotFound(path.to_string()),
                io::ErrorKind::PermissionDenied => ProcessingError::AccessDenied(path.to_string()),
                _ => ProcessingError::IoError(err),
            })?;
        
        if content.is_empty() {
            return Err(ProcessingError::EmptyFile(path.to_string()));
        }
        
        Ok(content.to_uppercase())
    }
    
    fn process_multiple_files(paths: &[&str]) -> Vec<Result<String, ProcessingError>> {
        paths.iter()
            .map(|&path| Self::process_file(path))
            .collect()
    }
    
    // 別解: 成功した結果のみを収集するパターン
    fn process_multiple_files_successful(paths: &[&str]) -> (Vec<String>, Vec<ProcessingError>) {
        let results: Vec<_> = Self::process_multiple_files(paths);
        
        let mut successes = Vec::new();
        let mut errors = Vec::new();
        
        for result in results {
            match result {
                Ok(content) => successes.push(content),
                Err(error) => {
                    eprintln!("Error: {}", error);
                    errors.push(error);
                }
            }
        }
        
        (successes, errors)
    }
}

fn main() {
    let paths = vec!["file1.txt", "file2.txt", "nonexistent.txt"];
    
    // 単一ファイルの処理
    match FileProcessor::process_file("example.txt") {
        Ok(content) => println!("Content: {}", content),
        Err(error) => eprintln!("Error: {}", error),
    }
    
    // 複数ファイルの処理 — すべての結果を保持
    let results = FileProcessor::process_multiple_files(&paths);
    for (i, result) in results.iter().enumerate() {
        match result {
            Ok(content) => println!("File {}: Success", i),
            Err(error) => println!("File {}: Error - {}", i, error),
        }
    }
    
    // 複数ファイルの処理 — 成功とエラーを分離
    let (successes, errors) = FileProcessor::process_multiple_files_successful(&paths);
    println!("Processed {} files successfully, {} errors", successes.len(), errors.len());
}
```

***

## 実践的な移行例

一般的な C# の設計パターンが Rust でどのように実装されるか、現実的なシナリオを通じて確認していきましょう。

### 設定管理
```csharp
// C# の設定クラス
public class AppConfig
{
    public string DatabaseUrl { get; set; } = "localhost";
    public int Port { get; set; } = 5432;
    public List<string> AllowedHosts { get; set; } = new();
    public Dictionary<string, string> FeatureFlags { get; set; } = new();
    
    public static AppConfig LoadFromFile(string path)
    {
        try
        {
            var json = File.ReadAllText(path);
            return JsonSerializer.Deserialize<AppConfig>(json) ?? new AppConfig();
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Failed to load config: {ex.Message}");
            return new AppConfig(); // デフォルト値にフォールバック
        }
    }
    
    public void Validate()
    {
        if (string.IsNullOrEmpty(DatabaseUrl))
            throw new InvalidOperationException("DatabaseUrl is required");
        
        if (Port <= 0 || Port > 65535)
            throw new InvalidOperationException("Port must be between 1 and 65535");
    }
}
```

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::fs;

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct AppConfig {
    pub database_url: String,
    pub port: u16,
    pub allowed_hosts: Vec<String>,
    pub feature_flags: HashMap<String, String>,
}

#[derive(Debug)]
pub enum ConfigError {
    FileNotFound(String),
    ParseError(String),
    ValidationError(String),
}

impl std::fmt::Display for ConfigError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ConfigError::FileNotFound(path) => write!(f, "Config file not found: {}", path),
            ConfigError::ParseError(msg) => write!(f, "Failed to parse config: {}", msg),
            ConfigError::ValidationError(msg) => write!(f, "Invalid config: {}", msg),
        }
    }
}

impl std::error::Error for ConfigError {}

impl Default for AppConfig {
    fn default() -> Self {
        AppConfig {
            database_url: "localhost".to_string(),
            port: 5432,
            allowed_hosts: Vec::new(),
            feature_flags: HashMap::new(),
        }
    }
}

impl AppConfig {
    pub fn load_from_file(path: &str) -> Result<AppConfig, ConfigError> {
        let contents = fs::read_to_string(path)
            .map_err(|_| ConfigError::FileNotFound(path.to_string()))?;
        
        let config: AppConfig = serde_json::from_str(&contents)
            .map_err(|e| ConfigError::ParseError(e.to_string()))?;
        
        config.validate()?;
        Ok(config)
    }
    
    pub fn load_or_default(path: &str) -> AppConfig {
        Self::load_from_file(path)
            .unwrap_or_else(|error| {
                eprintln!("Failed to load config: {}", error);
                AppConfig::default()
            })
    }
    
    pub fn validate(&self) -> Result<(), ConfigError> {
        if self.database_url.is_empty() {
            return Err(ConfigError::ValidationError("DatabaseUrl is required".to_string()));
        }
        
        if self.port == 0 {
            return Err(ConfigError::ValidationError("Port must be greater than 0".to_string()));
        }
        
        Ok(())
    }
    
    pub fn get_feature_flag(&self, key: &str) -> Option<&String> {
        self.feature_flags.get(key)
    }
    
    pub fn is_feature_enabled(&self, key: &str) -> bool {
        self.get_feature_flag(key)
            .map(|value| value.to_lowercase() == "true")
            .unwrap_or(false)
    }
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 設定の読み込みを試み、失敗時はデフォルト値にフォールバック
    let config = AppConfig::load_or_default("config.json");
    println!("Config: {:?}", config);
    
    // フィーチャーフラグの確認
    if config.is_feature_enabled("debug_mode") {
        println!("Debug mode is enabled");
    }
    
    Ok(())
}
```

### データ処理パイプライン
```csharp
// C# のデータ処理
public class DataProcessor
{
    public async Task<List<ProcessedData>> ProcessAsync(List<RawData> data)
    {
        var results = new List<ProcessedData>();
        
        foreach (var item in data)
        {
            try
            {
                if (IsValid(item))
                {
                    var processed = await TransformAsync(item);
                    results.Add(processed);
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error processing item {item.Id}: {ex.Message}");
            }
        }
        
        return results;
    }
    
    private bool IsValid(RawData data)
    {
        return !string.IsNullOrEmpty(data.Value) && data.Timestamp > DateTime.MinValue;
    }
    
    private async Task<ProcessedData> TransformAsync(RawData data)
    {
        // 非同期処理のシミュレーション
        await Task.Delay(10);
        
        return new ProcessedData
        {
            Id = data.Id,
            ProcessedValue = data.Value.ToUpper(),
            ProcessedAt = DateTime.UtcNow
        };
    }
}

public class RawData
{
    public int Id { get; set; }
    public string Value { get; set; } = "";
    public DateTime Timestamp { get; set; }
}

public class ProcessedData
{
    public int Id { get; set; }
    public string ProcessedValue { get; set; } = "";
    public DateTime ProcessedAt { get; set; }
}
```

```rust
use std::time::{SystemTime, UNIX_EPOCH};
use tokio;

#[derive(Debug, Clone)]
pub struct RawData {
    pub id: u32,
    pub value: String,
    pub timestamp: u64,
}

#[derive(Debug)]
pub struct ProcessedData {
    pub id: u32,
    pub processed_value: String,
    pub processed_at: u64,
}

#[derive(Debug)]
pub enum ProcessingError {
    InvalidData(String),
    TransformationFailed(String),
}

impl std::fmt::Display for ProcessingError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ProcessingError::InvalidData(msg) => write!(f, "Invalid data: {}", msg),
            ProcessingError::TransformationFailed(msg) => write!(f, "Transformation failed: {}", msg),
        }
    }
}

impl std::error::Error for ProcessingError {}

pub struct DataProcessor;

impl DataProcessor {
    pub async fn process(data: Vec<RawData>) -> Vec<Result<ProcessedData, ProcessingError>> {
        // 並行処理のために Future を使用
        let futures = data.into_iter().map(|item| async move {
            Self::validate(&item)?;
            Self::transform(item).await
        });
        
        // すべての Future を集約
        futures::future::join_all(futures).await
    }
    
    pub async fn process_successful_only(data: Vec<RawData>) -> Vec<ProcessedData> {
        let results = Self::process(data).await;
        
        results.into_iter()
            .filter_map(|result| match result {
                Ok(processed) => Some(processed),
                Err(error) => {
                    eprintln!("Processing error: {}", error);
                    None
                }
            })
            .collect()
    }
    
    fn validate(data: &RawData) -> Result<(), ProcessingError> {
        if data.value.is_empty() {
            return Err(ProcessingError::InvalidData("Value cannot be empty".to_string()));
        }
        
        if data.timestamp == 0 {
            return Err(ProcessingError::InvalidData("Invalid timestamp".to_string()));
        }
        
        Ok(())
    }
    
    async fn transform(data: RawData) -> Result<ProcessedData, ProcessingError> {
        // 非同期処理のシミュレーション
        tokio::time::sleep(tokio::time::Duration::from_millis(10)).await;
        
        let processed_value = data.value.to_uppercase();
        
        if processed_value.len() > 1000 {
            return Err(ProcessingError::TransformationFailed("Processed value too long".to_string()));
        }
        
        let processed_at = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs();
        
        Ok(ProcessedData {
            id: data.id,
            processed_value,
            processed_at,
        })
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let raw_data = vec![
        RawData { id: 1, value: "hello".to_string(), timestamp: 1234567890 },
        RawData { id: 2, value: "world".to_string(), timestamp: 1234567891 },
        RawData { id: 3, value: "".to_string(), timestamp: 1234567892 }, // 不正データ
    ];
    
    // 処理を実行しエラーを明示的に処理
    let results = DataProcessor::process(raw_data.clone()).await;
    for (i, result) in results.iter().enumerate() {
        match result {
            Ok(processed) => println!("Item {}: {:?}", i, processed),
            Err(error) => println!("Item {}: Error - {}", i, error),
        }
    }
    
    // 成功した結果のみを保持して処理
    let successful = DataProcessor::process_successful_only(raw_data).await;
    println!("Successfully processed {} items", successful.len());
    
    Ok(())
}
```

### HTTP クライアントの例
```csharp
// C# の HTTP クライアント
public class ApiClient
{
    private readonly HttpClient _httpClient;
    
    public ApiClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }
    
    public async Task<T?> GetAsync<T>(string endpoint) where T : class
    {
        try
        {
            var response = await _httpClient.GetAsync(endpoint);
            
            if (response.IsSuccessStatusCode)
            {
                var json = await response.Content.ReadAsStringAsync();
                return JsonSerializer.Deserialize<T>(json);
            }
            
            Console.WriteLine($"HTTP Error: {response.StatusCode}");
            return null;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Request failed: {ex.Message}");
            return null;
        }
    }
    
    public async Task<bool> PostAsync<T>(string endpoint, T data)
    {
        try
        {
            var json = JsonSerializer.Serialize(data);
            var content = new StringContent(json, Encoding.UTF8, "application/json");
            
            var response = await _httpClient.PostAsync(endpoint, content);
            return response.IsSuccessStatusCode;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"POST failed: {ex.Message}");
            return false;
        }
    }
}
```

```rust
use reqwest;
use serde::{Deserialize, Serialize};

#[derive(Debug)]
pub enum ApiError {
    NetworkError(reqwest::Error),
    HttpError(u16, String),
    ParseError(String),
}

impl std::fmt::Display for ApiError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ApiError::NetworkError(err) => write!(f, "Network error: {}", err),
            ApiError::HttpError(code, msg) => write!(f, "HTTP {} error: {}", code, msg),
            ApiError::ParseError(msg) => write!(f, "Parse error: {}", msg),
        }
    }
}

impl std::error::Error for ApiError {}

impl From<reqwest::Error> for ApiError {
    fn from(error: reqwest::Error) -> Self {
        ApiError::NetworkError(error)
    }
}

pub struct ApiClient {
    client: reqwest::Client,
    base_url: String,
}

impl ApiClient {
    pub fn new(base_url: String) -> Self {
        ApiClient {
            client: reqwest::Client::new(),
            base_url,
        }
    }
    
    pub async fn get<T>(&self, endpoint: &str) -> Result<T, ApiError>
    where
        T: for<'de> Deserialize<'de>,
    {
        let url = format!("{}/{}", self.base_url, endpoint);
        
        let response = self.client.get(&url).send().await?;
        
        if response.status().is_success() {
            let data = response.json::<T>().await
                .map_err(|e| ApiError::ParseError(e.to_string()))?;
            Ok(data)
        } else {
            let status = response.status().as_u16();
            let body = response.text().await.unwrap_or_default();
            Err(ApiError::HttpError(status, body))
        }
    }
    
    pub async fn post<T, R>(&self, endpoint: &str, data: &T) -> Result<R, ApiError>
    where
        T: Serialize,
        R: for<'de> Deserialize<'de>,
    {
        let url = format!("{}/{}", self.base_url, endpoint);
        
        let response = self.client
            .post(&url)
            .json(data)
            .send()
            .await?;
        
        if response.status().is_success() {
            let result = response.json::<R>().await
                .map_err(|e| ApiError::ParseError(e.to_string()))?;
            Ok(result)
        } else {
            let status = response.status().as_u16();
            let body = response.text().await.unwrap_or_default();
            Err(ApiError::HttpError(status, body))
        }
    }
}

#[derive(Serialize, Deserialize, Debug)]
struct User {
    id: u32,
    name: String,
    email: String,
}

#[derive(Serialize, Debug)]
struct CreateUserRequest {
    name: String,
    email: String,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = ApiClient::new("https://jsonplaceholder.typicode.com".to_string());
    
    // GET リクエスト
    match client.get::<User>("users/1").await {
        Ok(user) => println!("User: {:?}", user),
        Err(error) => eprintln!("Failed to get user: {}", error),
    }
    
    // POST リクエスト
    let new_user = CreateUserRequest {
        name: "John Doe".to_string(),
        email: "john@example.com".to_string(),
    };
    
    match client.post::<CreateUserRequest, User>("users", &new_user).await {
        Ok(created_user) => println!("Created user: {:?}", created_user),
        Err(error) => eprintln!("Failed to create user: {}", error),
    }
    
    Ok(())
}
```

***

## 学習ロードマップと次のステップ

### すぐに取り組むべきステップ（第 1〜2 週）
1. **開発環境のセットアップ**
   - [rustup.rs](https://rustup.rs/) から Rust をインストール
   - VS Code に rust-analyzer 拡張機能を設定
   - 最初の `cargo new hello_world` プロジェクトを作成

2. **基礎の習得**
   - シンプルな練習問題で所有権を体験する
   - さまざまな引数の型（`&str`, `String`, `&mut`）を持つ関数を書く
   - 基本的な構造体とメソッドを実装する

3. **エラー処理の練習**
   - C# の try-catch コードを Result ベースのパターンに書き換える
   - `?` 演算子と `match` 式の扱いに慣れる
   - カスタムエラー型を実装する

### 中期目標（第 1〜2 ヶ月）
1. **コレクションとイテレータ**
   - `Vec<T>`、`HashMap<K,V>`、`HashSet<T>` を自在に使いこなす
   - イテレータメソッド（`map`, `filter`, `collect`, `fold` など）を習得する
   - `for` ループとイテレータチェーンの使い分けを学ぶ

2. **トレイトとジェネリクス**
   - 一般的なトレイト（`Debug`, `Clone`, `PartialEq`）を実装する
   - ジェネリックな関数や構造体を記述する
   - トレイト境界と where 節を理解する

3. **プロジェクト構造の整理**
   - コードをモジュールに分割して整理する
   - `pub` による可視性の制御を理解する
   - crates.io から外部クレートを導入して活用する

### 高度なトピック（第 3 ヶ月以降）
1. **並行性（Concurrency）**
   - `Send` トレイトと `Sync` トレイトの役割を学ぶ
   - 基本的な並列処理に `std::thread` を使用する
   - 非同期プログラミングのための `tokio` を探求する

2. **メモリ管理**
   - 共有所有権のための `Rc<T>` と `Arc<T>` を理解する
   - ヒープ割り当てに `Box<T>` を使用するタイミングを学ぶ
   - 複雑なシナリオにおけるライフタイムをマスターする

3. **実践的なプロジェクト開発**
   - `clap` を使用した CLI ツールの構築
   - `axum` または `warp` による Web API の作成
   - ライブラリを作成して crates.io に公開する

### 推奨される学習リソース

#### 書籍
- **『The Rust Programming Language』**（オンライン無料、日本語訳あり）— 公式ガイド本
- **『Rust by Example』**（オンライン無料、日本語訳あり）— 実践的なサンプル集
- **『Programming Rust』**（Jim Blandy 著）— 深い技術的解説

#### オンラインリソース
- [Rust Playground](https://play.rust-lang.org/) — ブラウザ上で Rust コードを実行
- [Rustlings](https://github.com/rust-lang/rustlings) — インタラクティブな練習問題集
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) — 実践的なコード例

#### 練習用プロジェクト
1. **コマンドライン電卓** — 列挙型とパターンマッチングの練習
2. **ファイル整理ツール** — ファイルシステム操作とエラー処理の実践
3. **JSON 処理ツール** — serde とデータ変換の習得
4. **HTTP サーバー** — 非同期プログラミングとネットワーク通信の理解
5. **データベースライブラリ** — トレイト、ジェネリクス、エラー処理の総合的な実践

### C# 開発者が陥りやすい落とし穴

#### 所有権に関する混乱
```rust
// やってはいけない例: ムーブ済みの値を使用しようとする
fn wrong_way() {
    let s = String::from("hello");
    takes_ownership(s);
    // println!("{}", s); // エラー: s はムーブ済み
}

// 推奨される例: 参照を使用するか、必要に応じてクローンする
fn right_way() {
    let s = String::from("hello");
    borrows_string(&s);
    println!("{}", s); // OK: s の所有権は保持されている
}

fn takes_ownership(s: String) { /* s はここにムーブされる */ }
fn borrows_string(s: &str) { /* s はここで借用される */ }
```

#### ボローチェッカとの格闘
```rust
// やってはいけない例: 複数の可変参照を同時に作成する
fn wrong_borrowing() {
    let mut v = vec![1, 2, 3];
    let r1 = &mut v;
    // let r2 = &mut v; // エラー: 可変借用は同時に 1 つしか持てない
}

// 推奨される例: 可変借用のスコープを限定する
fn right_borrowing() {
    let mut v = vec![1, 2, 3];
    {
        let r1 = &mut v;
        r1.push(4);
    } // r1 はここでスコープを抜ける
    
    let r2 = &mut v; // OK: 他の可変借用は存在しない
    r2.push(5);
}
```

#### null 的な挙動の期待
```rust
// やってはいけない例: null の存在を期待する
fn no_null_in_rust() {
    // let s: String = null; // Rust に null は存在しない！
}

// 推奨される例: Option<T> を明示的に使用する
fn use_option_instead() {
    let maybe_string: Option<String> = None;
    
    match maybe_string {
        Some(s) => println!("Got string: {}", s),
        None => println!("No string available"),
    }
}
```

### 最後に

1. **コンパイラを受け入れる** — Rust のコンパイラエラーは敵ではなく、親切な家庭教師です
2. **小さく始める** — 単純なプログラムから始め、徐々に複雑さを増やしていきましょう
3. **他の人のコードを読む** — GitHub で人気のあるクレートの実装を研究してみましょう
4. **助けを求める** — Rust コミュニティは非常に親切で協力的です
5. **定期的に手を動かす** — Rust の概念はコードを書くことで自然と身につきます

覚えておいてください: Rust には一定の学習曲線がありますが、メモリ安全性、優れたパフォーマンス、恐れなき並行性（Fearless Concurrency）によってその苦労は必ず報われます。最初は制約のように感じられる所有権システムも、正しく効率的なプログラムを書くための強力無比な武器となるはずです。

---

**お疲れさまでした！** これで、C# から Rust へ移行するための確固たる基礎が身につきました。まずは小さなプロジェクトから始め、焦らずじっくりと学習を進め、徐々により高度なアプリケーションへと挑戦していってください。Rust がもたらす安全性とパフォーマンスの向上は、初期の学習投資に見合う十分な価値があります。

学習の次のステップとして、より洗練された設計パターン、パフォーマンスの最適化、実践的なアプリケーションアーキテクチャを解説している『[C# プログラマのための高度な Rust トレーニング](./RustTrainingForCSharp.md)』に進むことをおすすめします。
