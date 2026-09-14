### Unsafe Rust（安全でないRust）

> **学習目標:** `unsafe` をいつ、どのように使うべきか — 生ポインタの参照外し、RustからC言語（およびその逆）を呼び出すFFI（外部関数インターフェース）、文字列相互運用のための `CString`/`CStr`、そして unsafe コードを安全なラッパーで包む方法を学びます。

- `unsafe` は、Rustコンパイラによって通常は禁止されている機能へのアクセスを解放します
    - 生ポインタの参照外し
    - *可変*な静的（static）変数へのアクセス
    - https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html
- 大いなる力には大いなる責任が伴います
    - `unsafe` はコンパイラに対して「通常はコンパイラが保証する不変条件を維持する責任を、プログラマである私自身が引き受ける」と宣言するものです
    - 可変参照と不変参照のエイリアスが存在しないこと、ダングリングポインタが存在しないこと、無効な参照が存在しないことなどをプログラマが保証しなければなりません
    - `unsafe` の使用は可能な限り最小限のスコープに留めるべきです
    - `unsafe` を使用するすべてのコードには、その前提条件を説明する「安全性（Safety）」コメントを付与すべきです

### Unsafe Rust の例
```rust
unsafe fn harmless() {}
fn main() {
    // Safety: 害のない unsafe 関数を呼び出しています
    unsafe {
        harmless();
    }
    let a = 42u32;
    let p = &a as *const u32;
    // Safety: p はスコープ内に留まる変数への有効なポインタです
    unsafe {
        println!("{}", *p);
    }
    // Safety: 安全ではありません（説明目的のみのコードです）
    let dangerous_buffer = 0xb8000 as *mut u32;
    unsafe {
        println!("爆発寸前です!!!");
        *dangerous_buffer = 0; // これはほとんどの最新マシンで SEGV（セグメンテーション違反）を起こします
    }
}
```

### 単純なFFIの例（C言語から利用されるRustライブラリ関数）

## FFI文字列: CString と CStr

FFIは *Foreign Function Interface*（外部関数インターフェース）の略称であり、Rustが他言語（C言語など）で書かれた関数を呼び出したり、逆に呼び出されたりするための仕組みです。

C言語コードとインターフェースを取る際、Rustの `String` や `&str` 型（null終端文字を持たないUTF-8）は、C言語の文字列（null終端されたバイト配列）と直接の互換性がありません。この目的のために、Rustは `std::ffi` から `CString`（所有権あり）と `CStr`（借用）を提供しています：

| 型 | 対応するRust型 | 使用場面 |
|------|-------------|----------|
| `CString` | `String`（所有権あり） | RustのデータからC言語互換の文字列を作成する場合 |
| `&CStr` | `&str`（借用） | 外部コードからC言語文字列を受け取る場合 |

```rust
use std::ffi::{CString, CStr};
use std::os::raw::c_char;

fn demo_ffi_strings() {
    // C互換の文字列を作成（null終端文字を追加）
    let c_string = CString::new("Hello from Rust").expect("CString::new に失敗しました");
    let ptr: *const c_char = c_string.as_ptr();

    // C文字列をRustへ逆変換（ポインタを信用するため unsafe）
    // Safety: ptr は有効かつnull終端されています（直前で作成済み）
    let back_to_rust: &CStr = unsafe { CStr::from_ptr(ptr) };
    let rust_str: &str = back_to_rust.to_str().expect("不正なUTF-8です");
    println!("{}", rust_str);
}
```

> **警告**: `CString::new()` は、入力の途中にnullバイト（`\0`）が含まれている場合、エラーを返します。常に `Result` を適切に処理してください。以下のFFIの例でも `CStr` が頻繁に使用されます。

- `FFI` メソッドには、コンパイラが関数名をマングル（難読化・改名）しないように `#[no_mangle]` を付与する必要があります
- ここではクレートを静的ライブラリ（static library）としてコンパイルします
    ```rust
    #[no_mangle] 
    pub extern "C" fn add(left: u64, right: u64) -> u64 {
        left + right
    }
    ```
- 以下のCコードをコンパイルし、作成した静的ライブラリとリンクします。
    ```c
    #include <stdio.h>
    #include <stdint.h>
    extern uint64_t add(uint64_t, uint64_t);
    int main() {
        printf("add の戻り値: %llu\n", add(21, 21));
    }
    ``` 

### 発展的なFFIの例
- 以下の例では、Rustでロギングインターフェースを作成し、それをPythonおよび `C` に公開します
    - 同じインターフェースがRustとCの両方からネイティブに利用できる様子を確認します
    - `cbindgen` などのツールを使用して `C` 向けヘッダーファイルを自動生成する方法を探ります
    - `unsafe` ラッパーが安全なRustコードへの架け橋としてどのように機能するかを学びます

## ロガーのヘルパー関数
```rust
fn create_or_open_log_file(log_file: &str, overwrite: bool) -> Result<File, String> {
    if overwrite {
        File::create(log_file).map_err(|e| e.to_string())
    } else {
        OpenOptions::new()
            .write(true)
            .append(true)
            .open(log_file)
            .map_err(|e| e.to_string())
    }
}

fn log_to_file(file_handle: &mut File, message: &str) -> Result<(), String> {
    file_handle
        .write_all(message.as_bytes())
        .map_err(|e| e.to_string())
}
```

## Logger構造体
```rust
struct SimpleLogger {
    log_level: LogLevel,
    file_handle: File,
}

impl SimpleLogger {
    fn new(log_file: &str, overwrite: bool, log_level: LogLevel) -> Result<Self, String> {
        let file_handle = create_or_open_log_file(log_file, overwrite)?;
        Ok(Self {
            file_handle,
            log_level,
        })
    }

    fn log_message(&mut self, log_level: LogLevel, message: &str) -> Result<(), String> {
        if log_level as u32 <= self.log_level as u32 {
            let timestamp = Local::now().format("%Y-%m-%d %H:%M:%S").to_string();
            let message = format!("Simple: {timestamp} {log_level} {message}\n");
            log_to_file(&mut self.file_handle, &message)
        } else {
            Ok(())
        }
    }
}
```

## テスト
- Rustでの機能テストは非常に簡単です
    - テストメソッドには `#[test]` が付与され、通常のコンパイル済みバイナリには含まれません
    - テスト目的のモックメソッドも容易に作成できます
```rust
#[test]
fn testfunc() -> Result<(), String> {
    let mut logger = SimpleLogger::new("test.log", false, LogLevel::INFO)?;
    logger.log_message(LogLevel::TRACELEVEL1, "Hello world")?;
    logger.log_message(LogLevel::CRITICAL, "Critical message")?;
    Ok(()) // コンパイラはここで自動的に logger をドロップします
}
```
```bash
cargo test
```

## C言語 - Rust FFI
- cbindgen は、エクスポートされたRust関数用のCヘッダーファイルを生成するための優れたツールです
    - cargo を使用してインストールできます
```bash
cargo install cbindgen
cbindgen 
```
- 関数や構造体は `#[no_mangle]` や `#[repr(C)]` を使用してエクスポートできます
    - ここでは、実際の実装への `**`（ポインタのポインタ）を渡し、成功時に 0、エラー時に非ゼロを返す一般的なインターフェースパターンを採用します
    - **不透明（Opaque）構造体 vs 透過的（Transparent）構造体**: 今回の `SimpleLogger` は*不透明ポインタ*（`*mut SimpleLogger`）として渡されます — C言語側はそのフィールドに直接アクセスしないため、`#[repr(C)]` は**不要**です。Cコード側で構造体のフィールドを直接読み書きする必要がある場合にのみ `#[repr(C)]` を使用します:

```rust
// 不透明（Opaque） — C側はポインタを保持するだけでフィールドを検査しない。#[repr(C)] は不要。
struct SimpleLogger { /* Rust専用のフィールド */ }

// 透過的（Transparent） — C側がフィールドを直接読み書きする。必ず #[repr(C)] が必要。
#[repr(C)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}
```
```c
typedef struct SimpleLogger SimpleLogger;
uint32_t create_simple_logger(const char *file_name, struct SimpleLogger **out_logger);
uint32_t log_entry(struct SimpleLogger *logger, const char *message);
uint32_t drop_logger(struct SimpleLogger *logger);
```

- 多数の妥当性検査（サニティチェック）が必要になることに留意してください
- Rustによる自動メモリ解放を防ぐために、明示的にメモリをリーク（解放抑止）させる必要があります
```rust
#[no_mangle] 
pub extern "C" fn create_simple_logger(file_name: *const std::os::raw::c_char, out_logger: *mut *mut SimpleLogger) -> u32 {
    use std::ffi::CStr;
    // ポインタが NULL でないことを確認
    if file_name.is_null() || out_logger.is_null() {
        return 1;
    }
    // Safety: 契約上、渡されたポインタは NULL か null終端文字列のいずれかである
    let file_name = unsafe {
        CStr::from_ptr(file_name)
    };
    let file_name = file_name.to_str();
    // file_name に不正な文字が含まれていないことを確認
    if file_name.is_err() {
        return 1;
    }
    let file_name = file_name.unwrap();
    // デフォルト値を仮定（実際の実装では引数として渡します）
    let new_logger = SimpleLogger::new(file_name, true, LogLevel::CRITICAL);
    // ロガーが正常に構築できたか確認
    if new_logger.is_err() {
        return 1;
    }
    let new_logger = Box::new(new_logger.unwrap());
    // これにより、スコープを抜けた際に Box がドロップ（解放）されるのを防ぐ
    let logger_ptr: *mut SimpleLogger = Box::leak(new_logger);
    // Safety: logger は非NULLであり、logger_ptr は有効
    unsafe {
        *out_logger = logger_ptr;
    }
    return 0;
}
```

- `log_entry()` でも同様のエラーチェックを行います
```rust
#[no_mangle]
pub extern "C" fn log_entry(logger: *mut SimpleLogger, message: *const std::os::raw::c_char) -> u32 {
    use std::ffi::CStr;
    if message.is_null() || logger.is_null() {
        return 1;
    }
    // Safety: message は非NULL
    let message = unsafe {
        CStr::from_ptr(message)
    };
    let message = message.to_str();
    // file_name に不正な文字が含まれていないことを確認
    if message.is_err() {
        return 1;
    }
    // Safety: logger は以前 create_simple_logger() で構築された有効なポインタ
    unsafe {
        (*logger).log_message(LogLevel::CRITICAL, message.unwrap()).is_err() as u32
    }
}

#[no_mangle]
pub extern "C" fn drop_logger(logger: *mut SimpleLogger) -> u32 {
    if logger.is_null() {
        return 1;
    }
    // Safety: logger は以前 create_simple_logger() で構築された有効なポインタ
    unsafe {
        // これにより Box<SimpleLogger> が再構築され、スコープを抜ける際にドロップ（解放）される
        let _ = Box::from_raw(logger);
    }
    0
}
```

- この (C)-FFI は、Rustコードからテストすることも、(C)プログラムを作成してテストすることも可能です
```rust
#[test]
fn test_c_logger() {
    // c".." は NULL 終端文字列リテラルを生成します
    let file_name = c"test.log".as_ptr() as *const std::os::raw::c_char;
    let mut c_logger: *mut SimpleLogger = std::ptr::null_mut();
    assert_eq!(create_simple_logger(file_name, &mut c_logger), 0);
    // こちらは手動で c"..." 相当の文字列を作成する方法です
    let message = b"message from C\0".as_ptr() as *const std::os::raw::c_char;
    assert_eq!(log_entry(c_logger, message), 0);
    drop_logger(c_logger);
}
```
```c
#include "logger.h"
...
int main() {
    SimpleLogger *logger = NULL;
    if (create_simple_logger("test.log", &logger) == 0) {
        log_entry(logger, "Hello from C");
        drop_logger(logger); /* ハンドルのクローズ等のために必要 */
    } 
    ...
}
```

## unsafe コードの正しさを保証する
- 要約すると（TL;DR）、`unsafe` の使用には極めて慎重な判断が必要です
    - コードが前提としている安全性の仮定を必ず文書化し、専門家とレビューを行ってください
    - 正当性の検証を支援する cbindgen、Miri、Valgrind などのツールを活用してください
    - **パニックをFFI境界を越えて巻き戻させない（アンワインドさせない）こと** — これは未定義動作（UB）になります。FFIのエントリポイントでは `std::panic::catch_unwind` を使用するか、ビルドプロファイルで `panic = "abort"` を設定してください
    - 構造体をFFI間で共有する場合は、C互換のメモリレイアウトを保証するために `#[repr(C)]` を付与してください
    - https://doc.rust-lang.org/nomicon/intro.html （"The Rustonomicon" — unsafe Rust の奥義書）を参照してください
    - 社内の専門家にも相談してください

### 検証ツール: Miri と Valgrind の比較

C++開発者には Valgrind や各種サニタイザ（Sanitizers）がお馴染みでしょう。Rustではそれらに**加えて**、Rust固有の未定義動作（UB）をより精密に検出できる Miri が利用可能です：

| | **Miri** | **Valgrind** | **C++ サニタイザ（ASan/MSan/UBSan）** |
|---|---------|-------------|--------------------------------------|
| **検知対象** | Rust特有のUB: stacked borrows、不正な `enum` 判別子、未初期化メモリの読み取り、エイリアス違反 | メモリリーク、Use-After-Free、不正な読み書き、未初期化メモリ | バッファオーバーフロー、Use-After-Free、データ競合、UB |
| **動作原理** | MIR（Rustの中間表現）を解釈実行 — ネイティブ実行ではない | 実行時にコンパイル済みバイナリを計装（インストルメント） | コンパイル時のコード計装 |
| **FFIサポート** | ❌ FFI境界を越えられない（C言語の呼び出しはスキップ） | ✅ FFIを含むあらゆるコンパイル済みバイナリで動作 | ✅ C言語側もサニタイザ付きでコンパイルすれば動作 |
| **実行速度** | ネイティブ比で約100倍遅い | 約10〜50倍遅い | 約2〜5倍遅い |
| **推奨ユースケース** | 純粋なRustの `unsafe` コード、データ構造の不変条件 | FFIコード、完全なバイナリ結合テスト | FFIのC/C++側、パフォーマンス重視のテスト |
| **エイリアス違反の検知** | ✅ Stacked Borrows モデルにより可能 | ❌ 不可 | 部分的（TSanによるデータ競合検知など） |

**推奨方針**: **両方**併用してください — 純粋なRustの unsafe コードには Miri、FFI結合部には Valgrind を使用します:

- **Miri** — Valgrindでは検出できないRust固有のUB（エイリアス違反、不正なenum値、stacked borrowsの違反など）を検出:
    ```bash
    rustup +nightly component add miri
    cargo +nightly miri test                    # Miri上で全テストを実行
    cargo +nightly miri test -- test_name       # 特定のテストを実行
    ```
    > ⚠️ Miriは nightly ツールチェーンが必要であり、FFI呼び出しを実行できません。unsafe なRustロジックはテスト可能な単位に切り離してください。

- **Valgrind** — すでに使い慣れたツールであり、FFIを含むコンパイル済みバイナリ全体に対して機能します:
    ```bash
    sudo apt install valgrind
    cargo install cargo-valgrind
    cargo valgrind test                         # Valgrind上で全テストを実行
    ```
    > FFIコードでよく見られる `Box::leak` / `Box::from_raw` パターンのリークを検出できます。

- **cargo-careful** — 追加の実行時チェックを有効にしてテストを実行（通常のテストとMiriの中間に位置するツール）:
    ```bash
    cargo install cargo-careful
    cargo +nightly careful test
    ```

## Unsafe Rust のまとめ
- `cbindgen` は (C) FFI を Rust 向けに生成するための優れたツールです
    - 逆方向のFFIインターフェース生成には `bindgen` を使用してください（豊富なドキュメントが用意されています）
    - **自分の unsafe コードが正しいと思い込んだり、安全なRustから安全に呼び出せると安易に仮定してはいけません。ミスは極めて起こりやすく、一見正常に動作しているコードでも微妙な理由で誤っていることがあります**
    - 正当性を検証するツールを活用してください
    - 確信が持てない場合は、専門家のアドバイスを仰いでください
- `unsafe` コードには、前提条件やなぜ安全なのかについての明示的なドキュメントコメントを必ず記述してください
    - `unsafe` コードを呼び出す側も、同様に対応する安全性に関するコメントを記述し、制約を遵守してください

# 演習: 安全なFFIラッパーの作成

🔴 **チャレンジ課題** — unsafe ブロック、生ポインタ、安全なAPI設計の理解が必要です

- `unsafe` なFFIスタイルの関数を包む安全なRustラッパーを作成してください。この演習では、呼び出し側から提供されたバッファにフォーマットされた文字列を書き込むC言語関数をシミュレートします。
- **ステップ1**: 生の `*mut u8` バッファに挨拶文を書き込む unsafe 関数 `unsafe_greet` を実装する
- **ステップ2**: `Vec<u8>` を割り当て、unsafe 関数を呼び出し、`String` を返す安全なラッパー `safe_greet` を作成する
- **ステップ3**: すべての unsafe ブロックに適切な `// Safety:` コメントを付与する

**スターターコード:**
```rust
use std::fmt::Write as _;

/// C言語関数をシミュレート: バッファに "Hello, <name>!" を書き込む。
/// 書き込んだバイト数を返す（null終端文字は除く）。
/// # Safety
/// - `buf` は少なくとも `buf_len` バイトの書き込み可能な領域を指していなければならない
/// - `name` は null終端されたC文字列への有効なポインタでなければならない
unsafe fn unsafe_greet(buf: *mut u8, buf_len: usize, name: *const u8) -> isize {
    // TODO: 挨拶文を構築し、buf にバイト列をコピーして長さを返す
    // ヒント: std::ffi::CStr::from_ptr を使用するか、手動でバイトをイテレートする
    todo!()
}

/// 安全なラッパー — 公開APIに unsafe は現れない
fn safe_greet(name: &str) -> Result<String, String> {
    // TODO: Vec<u8> バッファを割り当て、null終端された name を作成し、
    // Safety コメント付きの unsafe ブロック内で unsafe_greet を呼び出し、
    // 結果を String に変換して返す
    todo!()
}

fn main() {
    match safe_greet("Rustacean") {
        Ok(msg) => println!("{msg}"),
        Err(e) => eprintln!("エラー: {e}"),
    }
    // 期待される出力: Hello, Rustacean!
}
```

<details><summary>解答例（クリックして展開）</summary>

```rust
use std::ffi::CStr;

/// C言語関数をシミュレート: バッファに "Hello, <name>!" を書き込む。
/// 書き込んだバイト数を返す。バッファが小さすぎる場合は -1 を返す。
/// # Safety
/// - `buf` は少なくとも `buf_len` バイトの書き込み可能な領域を指していなければならない
/// - `name` は null終端されたC文字列への有効なポインタでなければならない
unsafe fn unsafe_greet(buf: *mut u8, buf_len: usize, name: *const u8) -> isize {
    // Safety: 呼び出し側が name は有効なnull終端文字列であることを保証している
    let name_cstr = unsafe { CStr::from_ptr(name as *const std::os::raw::c_char) };
    let name_str = match name_cstr.to_str() {
        Ok(s) => s,
        Err(_) => return -1,
    };
    let greeting = format!("Hello, {}!", name_str);
    if greeting.len() > buf_len {
        return -1;
    }
    // Safety: buf は少なくとも buf_len バイトの書き込み可能領域を指している（呼び出し側の保証）
    unsafe {
        std::ptr::copy_nonoverlapping(greeting.as_ptr(), buf, greeting.len());
    }
    greeting.len() as isize
}

/// 安全なラッパー — 公開APIに unsafe は現れない
fn safe_greet(name: &str) -> Result<String, String> {
    let mut buffer = vec![0u8; 256];
    // C API向けに null終端された name を作成
    let name_with_null: Vec<u8> = name.bytes().chain(std::iter::once(0)).collect();

    // Safety: buffer は 256 バイトの書き込み可能領域を持ち、name_with_null は null終端されている
    let bytes_written = unsafe {
        unsafe_greet(buffer.as_mut_ptr(), buffer.len(), name_with_null.as_ptr())
    };

    if bytes_written < 0 {
        return Err("バッファが小さすぎるか、名前が無効です".to_string());
    }

    String::from_utf8(buffer[..bytes_written as usize].to_vec())
        .map_err(|e| format!("無効なUTF-8です: {e}"))
}

fn main() {
    match safe_greet("Rustacean") {
        Ok(msg) => println!("{msg}"),
        Err(e) => eprintln!("エラー: {e}"),
    }
}
// 出力:
// Hello, Rustacean!
```

</details>

----
