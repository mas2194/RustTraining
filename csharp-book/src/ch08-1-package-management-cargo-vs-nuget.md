## パッケージ管理：Cargo と NuGet

> **学べること:** `Cargo.toml` と `.csproj` の比較、バージョン指定子、`Cargo.lock`、
> 条件付きコンパイルのためのフィーチャーフラグ、および NuGet/dotnet に対応する一般的な Cargo コマンド。
>
> **難易度:** 🟢 初級

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
serde = { version = "1.0", features = ["derive"] }  # フィーチャーを指定
log = "0.4"
tokio = { version = "1.0", features = ["full"] }

# ローカル依存関係（ProjectReference に相当）
my_library = { path = "../my_library" }

# Git リポジトリからの依存関係
my_git_crate = { git = "https://github.com/user/repo" }

# 開発用依存関係（テストパッケージ等に相当）
[dev-dependencies]
criterion = "0.5"               # ベンチマーク用
proptest = "1.0"               # プロパティベーステスト用
```

### バージョン管理

#### C# のパッケージバージョニング
```xml
<!-- 一元管理パッケージ管理（Directory.Packages.props） -->
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
serde = "1.0"        # 1.x.x と互換（>=1.0.0, <2.0.0）
log = "0.4.17"       # 0.4.x と互換（>=0.4.17, <0.5.0）
regex = "=1.5.4"     # 正確なバージョン
chrono = "^0.4"      # キャレット要件（デフォルト）
uuid = "~1.3.0"      # チルダ要件（>=1.3.0, <1.4.0）

# Cargo.lock - 再現可能なビルドのための厳密なバージョン（自動生成）
[[package]]
name = "serde"
version = "1.0.163"
# ... 厳密な依存関係ツリー
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

# Cargo.toml 内
[dependencies]
my_crate = { version = "1.0", registry = "my-registry" }
```

### よく使われるコマンドの比較

| タスク | C# コマンド | Rust コマンド |
|------|------------|-------------|
| パッケージの復元 | `dotnet restore` | `cargo fetch` |
| パッケージの追加 | `dotnet add package Newtonsoft.Json` | `cargo add serde_json` |
| パッケージの削除 | `dotnet remove package Newtonsoft.Json` | `cargo remove serde_json` |
| パッケージの更新 | `dotnet update` | `cargo update` |
| パッケージ一覧の表示 | `dotnet list package` | `cargo tree` |
| セキュリティ監査 | `dotnet list package --vulnerable` | `cargo audit` |
| ビルド成果物のクリーンアップ | `dotnet clean` | `cargo clean` |

### フィーチャー：条件付きコンパイル

#### C# の条件付きコンパイル
```csharp
#if DEBUG
    Console.WriteLine("Debug mode");
#elif RELEASE
    Console.WriteLine("Release mode");
#endif

// プロジェクトファイルのフィーチャー設定
<PropertyGroup Condition="'$(Configuration)'=='Debug'">
    <DefineConstants>DEBUG;TRACE</DefineConstants>
</PropertyGroup>
```

#### Rust のフィーチャーゲート
```toml
# Cargo.toml
[features]
default = ["json"]              # デフォルトのフィーチャー
json = ["serde_json"]          # serde_json を有効化するフィーチャー
xml = ["serde_xml"]            # 代替のシリアライズ
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
    return "有効なシリアライズフィーチャーがありません".to_string();
}
```

### 外部クレートの利用

#### C# 開発者向けの主要クレート

| C# ライブラリ | Rust クレート | 用途 |
|------------|------------|---------|
| System.Text.Json / Newtonsoft.Json | `serde_json` | JSON シリアライズ |
| HttpClient | `reqwest` | HTTP クライアント |
| Entity Framework | `diesel` / `sqlx` | ORM / SQL ツールキット |
| NLog/Serilog | `log` + `env_logger` | ロギング |
| xUnit/NUnit | Built-in `#[test]` | 単体テスト |
| Moq | `mockall` | モック |
| Flurl | `url` | URL 操作 |
| Polly | `tower` | レジリエンスパターン |

#### 例：HTTP クライアントの移行
```csharp
// C# HttpClient の使用例
public class ApiClient
{
    private readonly HttpClient _httpClient;
    
    public async Task<User> GetUserAsync(int id)
    {
        var response = await _httpClient.GetAsync($"/users/{id}");
        var json = await response.Content.ReadAsStringAsync();
        return System.Text.Json.JsonSerializer.Deserialize<User>(json);
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

---
