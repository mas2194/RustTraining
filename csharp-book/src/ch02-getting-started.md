## インストールとセットアップ

> **学習内容:** Rust のインストールと IDE のセットアップ、Cargo ビルドシステムと MSBuild/NuGet の比較、
> C# と対比した最初の Rust プログラム、およびコマンドライン入力の読み取り方法について学びます。
>
> **難易度:** 🟢 初級

### Rust のインストール
```bash
# Rust のインストール（Windows、macOS、Linux で動作）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Windows では、https://rustup.rs/ からインストーラをダウンロードすることも可能です
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
| Visual Studio | VS Code + rust-analyzer | IDE |

### IDE のセットアップ
1. **VS Code**（初心者に推奨）
   - "rust-analyzer" 拡張機能をインストール
   - デバッグ用に "CodeLLDB" 拡張機能をインストール

2. **Visual Studio**（Windows）
   - Rust サポート拡張機能をインストール

3. **JetBrains RustRover**（統合開発環境）
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

### C# 開発者向けの主な相違点
1. **クラスが不要** - 関数をトップレベルに定義できます
2. **名前空間が不要** - 代わりにモジュールシステムを使用します
3. **`println!` はマクロ** - 末尾の `!` に注目してください
4. **セミコロンに意味がある** - 末尾のセミコロンを省略すると、文（Statement）から戻り値を表す式（Expression）になります
5. **明示的な戻り値型の指定が不要** - `main` は `()`（ユニット型）を返します

### 最初のプロジェクトの作成
```bash
# 新規プロジェクトの作成（'dotnet new console' に相当）
cargo new hello_rust
cd hello_rust

# 作成されるプロジェクト構造:
# hello_rust/
# ├── Cargo.toml      （.csproj ファイルに相当）
# └── src/
#     └── main.rs     （Program.cs に相当）

# プロジェクトの実行（'dotnet run' に相当）
cargo run
```

***

## Cargo vs NuGet / MSBuild

### プロジェクト構成

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

### よく使われる Cargo コマンド
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
```text
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

## 入力と CLI 引数の読み取り

C# 開発者なら誰もが `Console.ReadLine()` を知っています。ここでは Rust におけるユーザー入力、環境変数、コマンドライン引数の扱い方を解説します。

### コンソール入力
```csharp
// C# — ユーザー入力の読み取り
Console.Write("Enter your name: ");
string? name = Console.ReadLine();  // .NET 6+ では string? を返す
Console.WriteLine($"Hello, {name}!");

// 入力のパース
Console.Write("Enter a number: ");
if (int.TryParse(Console.ReadLine(), out int number))
{
    Console.WriteLine($"You entered: {number}");
}
else
{
    Console.WriteLine("That's not a valid number.");
}
```

```rust
use std::io::{self, Write};

fn main() {
    // 1行の入力を読み取る
    print!("Enter your name: ");
    io::stdout().flush().unwrap(); // print! は自動フラッシュしないため明示的に flush する

    let mut name = String::new();
    io::stdin().read_line(&mut name).expect("Failed to read line");
    let name = name.trim(); // 末尾の改行文字を削除
    println!("Hello, {name}!");

    // 入力のパース
    print!("Enter a number: ");
    io::stdout().flush().unwrap();

    let mut input = String::new();
    io::stdin().read_line(&mut input).expect("Failed to read");
    match input.trim().parse::<i32>() {
        Ok(number) => println!("You entered: {number}"),
        Err(_)     => println!("That's not a valid number."),
    }
}
```

### コマンドライン引数
```csharp
// C# — CLI 引数の読み取り
static void Main(string[] args)
{
    if (args.Length < 1)
    {
        Console.WriteLine("Usage: program <filename>");
        return;
    }
    string filename = args[0];
    Console.WriteLine($"Processing {filename}");
}
```

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    //  args[0] = プログラム名（C# のアセンブリ名に相当）
    //  args[1..] = 実際の引数

    if args.len() < 2 {
        eprintln!("Usage: {} <filename>", args[0]); // eprintln! → 標準エラー出力（stderr）
        std::process::exit(1);
    }
    let filename = &args[1];
    println!("Processing {filename}");
}
```

### 環境変数
```csharp
// C#
string dbUrl = Environment.GetEnvironmentVariable("DATABASE_URL") ?? "localhost";
```

```rust
use std::env;

let db_url = env::var("DATABASE_URL").unwrap_or_else(|_| "localhost".to_string());
// env::var は Result<String, VarError> を返す — null は一切なし！
```

### clap を使った本番向け CLI アプリケーション

単純な引数パースを超える用途には、**`clap`** クレートを使用します — これは C# における `System.CommandLine` や `CommandLineParser` などのライブラリに相当します。

```toml
# Cargo.toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

```rust
use clap::Parser;

/// シンプルなファイル処理ツール — このドキュメントコメントがヘルプメッセージになります
#[derive(Parser, Debug)]
#[command(name = "processor", version, about)]
struct Args {
    /// 処理対象の入力ファイル
    #[arg(short, long)]
    input: String,

    /// 出力先ファイル（省略時は標準出力）
    #[arg(short, long)]
    output: Option<String>,

    /// 詳細ログ出力を有効化
    #[arg(short, long, default_value_t = false)]
    verbose: bool,

    /// ワーカースレッド数
    #[arg(short = 'j', long, default_value_t = 4)]
    threads: usize,
}

fn main() {
    let args = Args::parse(); // 自動でパース、検証、--help の生成を行う

    if args.verbose {
        println!("Input:   {}", args.input);
        println!("Output:  {:?}", args.output);
        println!("Threads: {}", args.threads);
    }

    // args.input, args.output などを利用
}
```

```bash
# 自動生成されたヘルプ:
$ processor --help
A simple file processor

Usage: processor [OPTIONS] --input <INPUT>

Options:
  -i, --input <INPUT>      Input file to process
  -o, --output <OUTPUT>    Output file (defaults to stdout)
  -v, --verbose            Enable verbose logging
  -j, --threads <THREADS>  Number of worker threads [default: 4]
  -h, --help               Print help
  -V, --version            Print version
```

```csharp
// System.CommandLine を使った C# の同等コード（ボイラープレートが多い）:
var inputOption = new Option<string>("--input", "Input file") { IsRequired = true };
var verboseOption = new Option<bool>("--verbose", "Enable verbose logging");
var rootCommand = new RootCommand("A simple file processor");
rootCommand.AddOption(inputOption);
rootCommand.AddOption(verboseOption);
rootCommand.SetHandler((input, verbose) => { /* ... */ }, inputOption, verboseOption);
await rootCommand.InvokeAsync(args);
// clap の derive マクロによるアプローチのほうが、より簡潔で型安全です
```

| C# | Rust | 備考 |
|----|------|-------|
| `Console.ReadLine()` | `io::stdin().read_line(&mut buf)` | バッファの指定が必要、`Result` を返す |
| `int.TryParse(s, out n)` | `s.parse::<i32>()` | `Result<i32, ParseIntError>` を返す |
| `args[0]` | `env::args().nth(1)` | Rust の args[0] はプログラム名 |
| `Environment.GetEnvironmentVariable` | `env::var("KEY")` | `Result` を返す（Null 許容型ではない） |
| `System.CommandLine` | `clap` | derive ベース、ヘルプを自動生成 |

***
