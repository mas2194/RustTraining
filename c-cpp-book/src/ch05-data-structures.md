### Rustの配列型

> **学習目標:** Rustの主要なデータ構造 — 配列、タプル、スライス、文字列、構造体、`Vec`、`HashMap` を学びます。本章は内容が豊富です。特に `String` と `&str` の違い、および構造体の仕組みの理解に集中してください。参照と借用については第7章でさらに詳しく掘り下げます。

- 配列（array）は、同じ型の固定数の要素を格納します
    - 他のすべてのRustの型と同様に、配列はデフォルトで不変（immutable）です（`mut` を使用しない限り）
    - 配列は `[]` を使ってインデックスアクセスされ、境界チェック（bounds check）が行われます。配列の長さを取得するには `len()` メソッドを使用できます
```rust
    fn get_index(y : usize) -> usize {
        y+1        
    }
    
    fn main() {
        // 3要素の配列を初期化し、すべてを42に設定します
        let a : [u8; 3] = [42; 3];
        // 代替構文
        // let a = [42u8, 42u8, 42u8];
        for x in a {
            println!("{x}");
        }
        let y = get_index(a.len());
        // 以下のコメントを解除するとパニック（panic）が発生します
        //println!("{}", a[y]);
    }
```

----
### Rustの配列型（続き）
- 配列は入れ子（ネスト）にできます
    - Rustには出力用の組み込みフォーマッタがいくつか用意されています。以下において、`:?` は `debug` 出力フォーマッタです。`:#?` フォーマッタは `pretty print`（整形出力）に使用できます。これらのフォーマッタは型ごとにカスタマイズ可能です（詳細は後述）
```rust
    fn main() {
        let a = [
            [40, 0], // ネストされた配列を定義
            [41, 0],
            [42, 1],
        ];
        for x in a {
            println!("{x:?}");
        }
    }
```
----
### Rustのタプル
- タプル（tuple）は固定長であり、任意の型を1つの複合型にグループ化できます
    - 構成する要素には、相対位置（`.0`, `.1`, `.2`, ...）でインデックスアクセスできます。空のタプル、すなわち `()` はユニット値（unit value）と呼ばれ、voidの戻り値に相当します
    - Rustはタプルの分配束縛（destructuring）をサポートしており、個々の要素に変数を簡単に束縛できます
```rust
fn get_tuple() -> (u32, bool) {
    (42, true)        
}

fn main() {
   let t : (u8, bool) = (42, true);
   let u : (u32, bool) = (43, false);
   println!("{}, {}", t.0, t.1);
   println!("{}, {}", u.0, u.1);
   let (num, flag) = get_tuple(); // タプルの分配束縛
   println!("{num}, {flag}");
}
```

### Rustの参照
- Rustにおける参照（reference）は、いくつかの重要な違いを除けば、C言語のポインタにおおよそ相当します
    - ある時点で、1つの変数に対して任意の数の読み取り専用（不変）参照を持つことは正当です。参照はその変数のスコープより長く生存することはできません（これは**ライフタイム**と呼ばれる極めて重要な概念であり、後で詳しく説明します）
    - 可変な変数に対する書き込み可能（可変）な参照は1つだけ許可され、他の参照と重複してはなりません。
```rust
fn main() {
    let mut a = 42;
    {
        let b = &a;
        let c = b;
        println!("{} {}", *b, *c); // コンパイラは自動的に *c を逆参照します
        
        let d = &mut a;
        
        /*
         * 以下の行のコメントを解除するとプログラムはコンパイルエラーになります。
         * なぜなら、可変参照 `d` が現在のスコープで有効である間に `b` が使用されているためです。
         * 
         * 同じスコープ内で可変参照と不変参照を同時に使用することはできません！
         */
        // println!("{}", *b);
    }
    let d = &mut a; // OK: b と c はスコープ外です
    *d = 43;
}
```

----
# Rustのスライス
- Rustの参照を使用して、配列の部分集合（スライス）を作成できます
    - コンパイル時に静的な固定長が決まる配列とは異なり、スライスは任意のサイズにできます。内部的には、スライスはスライスの長さと元の配列の開始要素へのポインタを含む「ファットポインタ（fat pointer）」として実装されています
```rust
fn main() {
    let a = [40, 41, 42, 43];
    let b = &a[1..a.len()]; // 元の配列の2番目の要素から始まるスライス
    let c = &a[1..]; // 上記と同じ
    let d = &a[..]; // &a[0..] や &a[0..a.len()] と同じ
    println!("{b:?} {c:?} {d:?}");
}
```
----
# Rustの定数と静的変数
- `const` キーワードを使用して定数値を定義できます。定数値は**コンパイル時**に評価され、プログラム内にインライン展開されます
- `static` キーワードは、C/C++などの言語におけるグローバル変数に相当するものを定義するために使用されます。静的変数はアドレス指定可能なメモリ位置を持ち、一度だけ作成され、プログラム全体のライフタイムにわたって存続します
```rust
const SECRET_OF_LIFE: u32 = 42;
static GLOBAL_VARIABLE : u32 = 2;
fn main() {
    println!("人生の秘密は {}", SECRET_OF_LIFE);
    println!("グローバル変数の値は {GLOBAL_VARIABLE}")
}
```

----
# Rustの文字列: String vs &str

- Rustには、異なる目的を果たす**2つ**の文字列型があります
    - `String` — 所有権を持つ（owned）、ヒープ確保、伸長可能（C言語の `malloc` で確保されたバッファや、C++の `std::string` に類似）
    - `&str` — 借用された（borrowed）、軽量な参照（長さ情報を持つC言語の `const char*` や、C++の `std::string_view` に類似 — ただし `&str` は**ライフタイムがチェックされる**ため、ダングリングポインタになることはありません）
    - C言語のヌル終端文字列とは異なり、Rustの文字列は自身の長さを保持しており、有効なUTF-8であることが保証されています

> **C++開発者向け:** `String` ≈ `std::string`、`&str` ≈ `std::string_view` です。`std::string_view` とは異なり、`&str` はボローチェッカによってその生存期間全体で有効であることが保証されます。

## String vs &str: 所有（Owned）vs 借用（Borrowed）

> **実践パターン**: 実プロダクションコードにおいてserdeで文字列処理がどのように機能するかについては、[JSON処理: nlohmann::json → serde](ch17-2-avoiding-unchecked-indexing.md#json-handling-nlohmannjson--serde) を参照してください。

| **観点** | **C `char*`** | **C++ `std::string`** | **Rust `String`** | **Rust `&str`** |
|------------|--------------|----------------------|-------------------|----------------|
| **メモリ** | 手動（`malloc`/`free`） | ヒープ確保、バッファを所有 | ヒープ確保、自動解放 | 借用された参照（ライフタイムチェック付き） |
| **可変性** | ポインタ経由で常に可変 | 可変 | `mut` により可変 | 常に不変 |
| **サイズ情報** | なし（`'\0'` に依存） | 長さとキャパシティを追跡 | 長さとキャパシティを追跡 | 長さを追跡（ファットポインタ） |
| **エンコーディング** | 未規定（通常はASCII） | 未規定（通常はASCII） | 有効なUTF-8を保証 | 有効なUTF-8を保証 |
| **ヌル終端** | 必須 | 必須（`c_str()`） | 使用しない | 使用しない |

```rust
fn main() {
    // &str - 文字列スライス（借用、不変、通常は文字列リテラル）
    let greeting: &str = "Hello";  // 読み取り専用メモリを指す

    // String - 所有権を持つ、ヒープ確保、伸長可能
    let mut owned = String::from(greeting);  // データをヒープにコピー
    owned.push_str(", World!");        // 文字列を拡張
    owned.push('!');                   // 単一の文字を追加

    // String と &str 間の変換
    let slice: &str = &owned;          // String -> &str（ゼロコスト、単なる借用）
    let owned2: String = slice.to_string();  // &str -> String（メモリ確保が発生）
    let owned3: String = String::from(slice); // 上記と同じ

    // 文字列の連結（注: + は左側のオペランドを消費します）
    let hello = String::from("Hello");
    let world = String::from(", World!");
    let combined = hello + &world;  // hello はムーブ（消費）され、world は借用される
    // println!("{hello}");  // コンパイルエラー: hello はムーブされたため使用不可

    // ムーブの問題を避けるには format! を使用する
    let a = String::from("Hello");
    let b = String::from("World");
    let combined = format!("{a}, {b}!");  // a も b も消費されない

    println!("{combined}");
}
```

## なぜ文字列を `[]` でインデックス指定できないのか
```rust
fn main() {
    let s = String::from("hello");
    // let c = s[0];  // コンパイルエラー！Rustの文字列はバイト配列ではなくUTF-8です

    // 安全な代替手段:
    let first_char = s.chars().next();           // Option<char>: Some('h')
    let as_bytes = s.as_bytes();                 // &[u8]: 生のUTF-8バイト列
    let substring = &s[0..1];                    // &str: "h"（バイト範囲、有効なUTF-8境界である必要あり）

    println!("最初の文字: {:?}", first_char);
    println!("バイト列: {:?}", &as_bytes[..5]);
}
```

## 演習: 文字列操作

🟢 **初級課題**
- 文字列内の空白で区切られた単語数をカウントする関数 `fn count_words(text: &str) -> usize` を書いてください
- 最も長い単語を返す関数 `fn longest_word(text: &str) -> &str` を書いてください（ヒント: ライフタイムを考慮する必要があります — なぜ戻り値の型は `String` ではなく `&str` である必要があるのでしょうか？）

<details><summary>解答（クリックして展開）</summary>

```rust
fn count_words(text: &str) -> usize {
    text.split_whitespace().count()
}

fn longest_word(text: &str) -> &str {
    text.split_whitespace()
        .max_by_key(|word| word.len())
        .unwrap_or("")
}

fn main() {
    let text = "the quick brown fox jumps over the lazy dog";
    println!("単語数: {}", count_words(text));       // 9
    println!("最長単語: {}", longest_word(text));     // "jumps"
}
```

</details>

# Rustの構造体
- `struct` キーワードはユーザー定義の構造体型を宣言します
    - `struct` のメンバは名前付き、または無名（タプル構造体）のいずれかにできます
- C++などの言語とは異なり、Rustには「データの継承」という概念はありません
```rust
fn main() {
    struct MyStruct {
        num: u32,
        is_secret_of_life: bool,
    }
    let x = MyStruct {
        num: 42,
        is_secret_of_life: true,
    };
    let y = MyStruct {
        num: x.num,
        is_secret_of_life: x.is_secret_of_life,
    };
    let z = MyStruct { num: x.num, ..x }; // .. は残りのフィールドをコピー/ムーブすることを意味します
    println!("{} {} {}", x.num, y.is_secret_of_life, z.num);
}
```

# Rustのタプル構造体
- Rustのタプル構造体はタプルに似ており、個々のフィールドには名前がありません
    - タプルと同様に、個々の要素には `.0`, `.1`, `.2`, ... でアクセスします。タプル構造体の一般的なユースケースは、プリミティブ型をラップしてカスタム型を作成することです。**これは、同じ基本型を持つ異なる意味の値を混同するのを防ぐのに役立ちます**
```rust
struct WeightInGrams(u32);
struct WeightInMilligrams(u32);
fn to_weight_in_grams(kilograms: u32) -> WeightInGrams {
    WeightInGrams(kilograms * 1000)
}

fn to_weight_in_milligrams(w : WeightInGrams) -> WeightInMilligrams  {
    WeightInMilligrams(w.0 * 1000)
}

fn main() {
    let x = to_weight_in_grams(42);
    let y = to_weight_in_milligrams(x);
    // let z : WeightInGrams = x;  // コンパイルエラー: x は to_weight_in_milligrams() へムーブされました
    // let a : WeightInGrams = y;   // コンパイルエラー: 型の不一致 (WeightInMilligrams と WeightInGrams)
}
```


**注意**: `#[derive(...)]` 属性は、構造体や列挙型に対して一般的なトレイトの実装を自動的に生成します。これはコース全体を通じて頻繁に使用されます：
```rust
#[derive(Debug, Clone, PartialEq)]
struct Point { x: i32, y: i32 }

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);           // Debug: #[derive(Debug)] のおかげで動作
    let p2 = p.clone();           // Clone: #[derive(Clone)] のおかげで動作
    assert_eq!(p, p2);            // PartialEq: #[derive(PartialEq)] のおかげで動作
}
```
トレイトシステムについては後で詳しく説明しますが、`#[derive(Debug)]` は非常に便利なので、作成するほぼすべての `struct` や `enum` に追加することをお勧めします。

# Rustの Vec 型
- `Vec<T>` 型は、動的にヒープ確保されるバッファを実装しています（C言語における手動管理の `malloc`/`realloc` 配列や、C++の `std::vector` に類似）
    - 固定長の配列とは異なり、`Vec` は実行時に拡大・縮小できます
    - `Vec` はそのデータを所有し、メモリの確保と解放を自動的に管理します
- 一般的な操作: `push()`, `pop()`, `insert()`, `remove()`, `len()`, `capacity()`
```rust
fn main() {
    let mut v = Vec::new();    // 空のベクタ、型は使用状況から推論される
    v.push(42);                // 末尾に要素を追加 - Vec<i32>
    v.push(43);                
    
    // 安全な反復処理（推奨）
    for x in &v {              // ベクタを消費せず、要素を借用する
        println!("{x}");
    }
    
    // 初期化のショートカット
    let mut v2 = vec![1, 2, 3, 4, 5];           // 初期化用マクロ
    let v3 = vec![0; 10];                       // 10個のゼロ
    
    // 安全なアクセス方法（インデックス直接指定より推奨）
    match v2.get(0) {
        Some(first) => println!("先頭: {first}"),
        None => println!("空のベクタ"),
    }
    
    // 便利なメソッド
    println!("長さ: {}, キャパシティ: {}", v2.len(), v2.capacity());
    if let Some(last) = v2.pop() {             // 末尾の要素を取り出して削除
        println!("取り出した値: {last}");
    }
    
    // 危険: 直接のインデックス指定（パニックの可能性あり！）
    // println!("{}", v2[100]);  // 実行時にパニックを引き起こします
}
```
> **実践パターン**: プロダクションレベルのRustコードにおける安全な `.get()` パターンについては、[チェックなしのインデックスアクセスの回避](ch17-2-avoiding-unchecked-indexing.md#avoiding-unchecked-indexing) を参照してください。

# Rustの HashMap 型
- `HashMap` はジェネリックな `キー` -> `値` の検索（いわゆる `辞書（dictionary）` や `マップ（map）`）を実装しています
```rust
fn main() {
    use std::collections::HashMap;  // Vecとは異なり、明示的なインポートが必要
    let mut map = HashMap::new();       // 空のHashMapを確保
    map.insert(40, false);  // 型は int -> bool と推論される
    map.insert(41, false);
    map.insert(42, true);
    for (key, value) in map {
        println!("{key} {value}");
    }
    let map = HashMap::from([(40, false), (41, false), (42, true)]);
    if let Some(x) = map.get(&43) {
        println!("43は {x:?} にマップされました");
    } else {
        println!("43に対するマッピングは見つかりませんでした");
    }
    let x = map.get(&43).or(Some(&false));  // キーが見つからない場合のデフォルト値
    println!("{x:?}"); 
}
```

# 演習: Vec と HashMap

🟢 **初級課題**
- いくつかのエントリを持つ `HashMap<u32, bool>` を作成してください（一部の値が `true` で、他の値が `false` になるようにしてください）。HashMap内のすべての要素をループ処理し、キーを1つの `Vec` に、値を別の `Vec` に格納してください

<details><summary>解答（クリックして展開）</summary>

```rust
use std::collections::HashMap;

fn main() {
    let map = HashMap::from([(1, true), (2, false), (3, true), (4, false)]);
    let mut keys = Vec::new();
    let mut values = Vec::new();
    for (k, v) in &map {
        keys.push(*k);
        values.push(*v);
    }
    println!("キー:   {keys:?}");
    println!("値:     {values:?}");

    // 代替案: unzip() を使ったイテレータの利用
    let (keys2, values2): (Vec<u32>, Vec<bool>) = map.into_iter().unzip();
    println!("キー (unzip):   {keys2:?}");
    println!("値 (unzip):     {values2:?}");
}
```

</details>

---

## 詳細解説: C++の参照 vs Rustの参照

> **C++開発者向け:** C++プログラマは、Rustの `&T` がC++の `T&` のように動作すると仮定しがちです。表面的には似ていますが、混乱の原因となる根本的な違いがあります。Cプログラマはこのセクションをスキップしても構いません — Rustの参照については [所有権と借用](ch07-ownership-and-borrowing.md) で詳しく説明します。

#### 1. 右辺値参照やユニバーサル参照は存在しない

C++では、`&&` は文脈に応じて2つの意味を持ちます：

```cpp
// C++: && は文脈によって意味が異なります:
int&& rref = 42;           // 右辺値参照 — 一時オブジェクトにバインド
void process(Widget&& w);   // 右辺値参照 — 呼び出し元は std::move が必要

// ユニバーサル（転送）参照 — 推論されるテンプレート文脈:
template<typename T>
void forward(T&& arg) {     // 右辺値参照ではない！ T& または T&& として推論される
    inner(std::forward<T>(arg));  // 完全転送
}
```

**Rustでは、このようなものは一切存在しません。** `&&` は単なる論理積（AND）演算子です。

```rust
// Rust: && は単なるブール演算の AND
let a = true && false; // false

// Rustには右辺値参照、ユニバーサル参照、完全転送は存在しません。
// その代わり:
//   - Copyトレイトを実装していない型では、ムーブがデフォルト（std::move は不要）
//   - ジェネリクス + トレイト境界がユニバーサル参照を代替
//   - 一時オブジェクトへのバインドの区別はなく、値は単なる値

fn process(w: Widget) { }      // 所有権を取得（C++の値渡しパラメータ + 暗黙のムーブに相当）
fn process_ref(w: &Widget) { } // 不変で借用（C++の const T& に相当）
fn process_mut(w: &mut Widget) { } // 可変で借用（C++の T& に相当するが、排他的）
```

| C++の概念 | Rustの同等物 | 備考 |
|-------------|-----------------|-------|
| `T&`（左辺値参照） | `&T` または `&mut T` | Rustでは共有参照と排他参照に分離 |
| `T&&`（右辺値参照） | 単なる `T` | 値渡し = 所有権の取得 |
| テンプレート内の `T&&`（ユニバーサル参照） | `impl Trait` または `<T: Trait>` | ジェネリクスが転送を代替 |
| `std::move(x)` | `x`（そのまま使用） | ムーブがデフォルト |
| `std::forward<T>(x)` | 同等のものは不要 | 転送すべきユニバーサル参照が存在しない |

#### 2. ムーブはビット単位（memcpy）— ムーブコンストラクタは存在しない

C++において、ムーブは*ユーザー定義の操作*（ムーブコンストラクタ / ムーブ代入演算子）です。Rustにおいて、ムーブは常に値の**ビット単位の memcpy** であり、移動元は無効化されます：

```rust
// Rustのムーブ = バイト列をmemcpyし、移動元を無効としてマークする
let s1 = String::from("hello");
let s2 = s1; // s1のバイト列がs2のスタックスロットにコピーされる
              // s1は無効化される — コンパイラがこれを強制する
// println!("{s1}"); // ❌ コンパイルエラー: ムーブ後の値の使用
```

```cpp
// C++のムーブ = ムーブコンストラクタの呼び出し（ユーザー定義！）
std::string s1 = "hello";
std::string s2 = std::move(s1); // string のムーブコンストラクタを呼び出す
// s1 は「有効だが未規定の状態」のゾンビとなる
std::cout << s1; // コンパイル可能！ 何か（通常は空文字列）が出力される
```

**結果として生じるメリット**:
- Rustには「Rule of Five（5つの特殊メンバ関数のルール）」がありません（コピーコンストラクタ、ムーブコンストラクタ、コピー代入、ムーブ代入、デストラクタを定義する必要がない）
- ムーブ後の「ゾンビ」状態が存在しない — コンパイラが単にアクセスを防止します
- ムーブに関して `noexcept` を考慮する必要がない — ビット単位のコピーは例外をスローし得ません

#### 3. 自動逆参照（Auto-Deref）: コンパイラが間接参照を透過的に解決

Rustは、`Deref` トレイトを介して、ポインタやラッパーの多重層を自動的に逆参照します。これにはC++に対応する機能がありません：

```rust
use std::sync::{Arc, Mutex};

// ネストされたラッパー: Arc<Mutex<Vec<String>>>
let data = Arc::new(Mutex::new(vec!["hello".to_string()]));

// C++では、各層で明示的なロック解除や手動の間接参照が必要になります。
// Rustでは、コンパイラが Arc → Mutex → MutexGuard → Vec を自動逆参照します:
let guard = data.lock().unwrap(); // Arc が Mutex に自動逆参照される
let first: &str = &guard[0];      // MutexGuard→Vec (Deref), Vec[0] (Index),
                                   // &String→&str (Deref型強制)
println!("First: {first}");

// メソッド呼び出しも自動逆参照されます:
let boxed_string = Box::new(String::from("hello"));
println!("Length: {}", boxed_string.len());  // Box→String、そして String::len()
// (*boxed_string).len() や boxed_string->len() は不要
```

**Deref型強制（Deref coercion）**は関数の引数にも適用されます — コンパイラは型が一致するように逆参照を挿入します：

```rust
fn greet(name: &str) {
    println!("Hello, {name}");
}

fn main() {
    let owned = String::from("Alice");
    let boxed = Box::new(String::from("Bob"));
    let arced = std::sync::Arc::new(String::from("Carol"));

    greet(&owned);  // &String → &str  （1回のDeref型強制）
    greet(&boxed);  // &Box<String> → &String → &str  （2回のDeref型強制）
    greet(&arced);  // &Arc<String> → &String → &str  （2回のDeref型強制）
    greet("Dave");  // 既に &str — 型強制は不要
}
// C++では、それぞれのケースで .c_str() や明示的な変換が必要になります。
```

**Derefチェーン**: `x.method()` を呼び出すと、Rustのメソッド解決はレシーバの型 `T`、次に `&T`、次に `&mut T` を試行します。一致するものがない場合、`Deref` トレイトを介して逆参照し、ターゲット型で繰り返します。これは複数の層を通じて継続します — これが `Box<Vec<T>>` が `Vec<T>` と同様に「自然に機能する」理由です。Deref*型強制*（関数の引数用）は独立した関連メカニズムであり、`Deref` 実装を連鎖させることで `&Box<String>` を `&str` に自動的に変換します。

#### 4. null参照やオプション参照は存在しない

```cpp
// C++: 参照はnullになれませんがポインタはnullになれるため、境界が曖昧です
Widget& ref = *ptr;  // ptr が null の場合 → 未定義動作 (UB)
Widget* opt = nullptr;  // ポインタを介した「オプションの」参照
```

```rust
// Rust: 参照は常に有効です — ボローチェッカによって保証されます
// 安全なコード内で null やダングリング参照を作成する方法はありません
let r: &i32 = &42; // 常に有効

// 「オプションの参照」は明示的に表現されます:
let opt: Option<&Widget> = None; // 明確な意図、nullポインタは不要
if let Some(w) = opt {
    w.do_something(); // 値が存在する場合にのみ到達可能
}
```

#### 5. 参照の再バインド（Reseat）はできない

```cpp
// C++: 参照はエイリアス（別名）であり、再バインドできません
int a = 1, b = 2;
int& r = a;
r = b;  // これは b の値を a に代入します — r を再バインドするわけではありません！
// a は 2 になり、r は依然として a を参照しています
```

```rust
// Rust: let 束縛はシャドーイングできますが、参照は異なる規則に従います
let a = 1;
let b = 2;
let r = &a;
// r = &b;   // ❌ 不変変数への代入はできません
let r = &b;  // ✅ しかし新しい束縛で r をシャドーイングできます
             // 古い束縛は破棄され、参照が再バインドされたわけではありません

// mut を使用した場合:
let mut r = &a;
r = &b;      // ✅ r は b を指すようになります — これは（中身への代入ではなく）再バインドです
```

> **メンタルモデル**: C++では、参照は1つのオブジェクトに対する永続的なエイリアスです。
> Rustでは、参照は通常の値束縛規則に従う値（ライフタイム保証が付いたポインタ）です — デフォルトで不変であり、`mut` と宣言された場合にのみ再バインド可能です。
