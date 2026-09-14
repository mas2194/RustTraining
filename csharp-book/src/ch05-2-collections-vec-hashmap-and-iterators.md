## `Vec<T>` vs `List<T>`

> **学習内容:** `Vec<T>` vs `List<T>`、`HashMap` vs `Dictionary`、安全なアクセスパターン（なぜ Rust は例外をスローする代わりに `Option` を返すのか）、そしてコレクションにおける所有権への影響を学びます。
>
> **難易度:** 🟢 初級

`Vec<T>` は C# の `List<T>` に相当する Rust の型ですが、所有権セマンティクスが適用されます。

### C# `List<T>`
```csharp
// C# List<T> - 参照型、ヒープ割り当て
var numbers = new List<int>();
numbers.Add(1);
numbers.Add(2);
numbers.Add(3);

// メソッドに渡す - 参照がコピーされる
ProcessList(numbers);
Console.WriteLine(numbers.Count);  // まだアクセス可能

void ProcessList(List<int> list)
{
    list.Add(4);  // 元のリストを変更する
    Console.WriteLine($"Count in method: {list.Count}");
}
```

### Rust `Vec<T>`
```rust
// Rust Vec<T> - 所有型、ヒープ割り当て
let mut numbers = Vec::new();
numbers.push(1);
numbers.push(2);
numbers.push(3);

// 所有権を受け取る関数
process_vec(numbers);
// println!("{:?}", numbers);  // ❌ エラー: numbers はムーブされました

// 借用する関数
let mut numbers = vec![1, 2, 3];  // 便宜のための vec! マクロ
process_vec_borrowed(&mut numbers);
println!("{:?}", numbers);  // ✅ まだアクセス可能

fn process_vec(mut vec: Vec<i32>) {  // 所有権を取得
    vec.push(4);
    println!("メソッド内での要素数: {}", vec.len());
    // ここで vec はドロップ（破棄）される
}

fn process_vec_borrowed(vec: &mut Vec<i32>) {  // 可変借用
    vec.push(4);
    println!("メソッド内での要素数: {}", vec.len());
}
```

### ベクタの作成と初期化
```csharp
// C# List の初期化
var numbers = new List<int> { 1, 2, 3, 4, 5 };
var empty = new List<int>();
var sized = new List<int>(10);  // 初期容量

// 他のコレクションから
var fromArray = new List<int>(new[] { 1, 2, 3 });
```

```rust
// Rust Vec の初期化
let numbers = vec![1, 2, 3, 4, 5];  // vec! マクロ
let empty: Vec<i32> = Vec::new();   // 空の場合は型注釈が必要
let sized = Vec::with_capacity(10); // 容量を事前確保

// イテレータから
let from_range: Vec<i32> = (1..=5).collect();
let from_array = vec![1, 2, 3];
```

### 一般的な操作の比較
```csharp
// C# List の操作
var list = new List<int> { 1, 2, 3 };

list.Add(4);                    // 要素を追加
list.Insert(0, 0);              // 指定インデックスに挿入
list.Remove(2);                 // 最初に見つかった要素を削除
list.RemoveAt(1);               // 指定インデックスの要素を削除
list.Clear();                   // すべて削除

int first = list[0];            // インデックスアクセス
int count = list.Count;         // 要素数を取得
bool contains = list.Contains(3); // 含まれているか確認
```

```rust
// Rust Vec の操作
let mut vec = vec![1, 2, 3];

vec.push(4);                    // 要素を追加
vec.insert(0, 0);               // 指定インデックスに挿入
vec.retain(|&x| x != 2);        // 要素を削除（関数型スタイル）
vec.remove(1);                  // 指定インデックスの要素を削除
vec.clear();                    // すべて削除

let first = vec[0];             // インデックスアクセス（範囲外の場合はパニック）
let safe_first = vec.get(0);    // 安全なアクセス、Option<&T> を返す
let count = vec.len();          // 要素数を取得
let contains = vec.contains(&3); // 含まれているか確認
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
// Rust - Option ベースの安全なアクセス
fn safe_access(vec: &[i32], index: usize) -> Option<i32> {
    vec.get(index).copied()  // Option<i32> を返す
}

fn main() {
    let vec = vec![1, 2, 3];
    
    // 安全なアクセスパターン
    match vec.get(10) {
        Some(value) => println!("値: {}", value),
        None => println!("インデックスが範囲外です"),
    }
    
    // または unwrap_or を使用
    let value = vec.get(10).copied().unwrap_or(-1);
    println!("値: {}", value);
}
```

***

## HashMap vs Dictionary

Rust の `HashMap` は C# の `Dictionary<K,V>` に相当します。

### C# Dictionary
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
scores["Alice"] = 105;  // 既存の値を更新

// 安全なアクセス
if (scores.TryGetValue("Eve", out int score))
{
    Console.WriteLine($"Eve のスコア: {score}");
}
else
{
    Console.WriteLine("Eve は見つかりません");
}

// 反復処理
foreach (var kvp in scores)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}
```

### Rust HashMap
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
scores.insert("Alice".to_string(), 105);  // 既存の値を更新

// 安全なアクセス
match scores.get("Eve") {
    Some(score) => println!("Eve のスコア: {}", score),
    None => println!("Eve は見つかりません"),
}

// 反復処理
for (name, score) in &scores {
    println!("{}: {}", name, score);
}
```

### HashMap の操作
```csharp
// C# Dictionary の操作
var dict = new Dictionary<string, int>();

dict["key"] = 42;                    // 挿入 / 更新
bool exists = dict.ContainsKey("key"); // 存在確認
bool removed = dict.Remove("key");    // 削除
dict.Clear();                        // すべてクリア

// デフォルト値付きで取得
int value = dict.GetValueOrDefault("missing", 0);
```

```rust
use std::collections::HashMap;

// Rust HashMap の操作
let mut map = HashMap::new();

map.insert("key".to_string(), 42);   // 挿入 / 更新
let exists = map.contains_key("key"); // 存在確認
let removed = map.remove("key");      // 削除、Option<V> を返す
map.clear();                         // すべてクリア

// 高度な操作のための Entry API
let mut map = HashMap::new();
map.entry("key".to_string()).or_insert(42);  // 存在しない場合のみ挿入
map.entry("key".to_string()).and_modify(|v| *v += 1); // 存在する場合は変更

// デフォルト値付きで取得
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
    // println!("{}", key);   // ❌ エラー: key はムーブされました
    // println!("{}", value); // ❌ エラー: value はムーブされました
    
    // 参照経由のアクセス
    if let Some(name) = map.get("name") {
        println!("名前: {}", name);  // 値を借用
    }
}

// &str キーの使用（所有権の移動なし）
fn string_slice_keys() {
    let mut map = HashMap::new();
    
    map.insert("name", "Alice");     // &str のキーと値
    map.insert("age", "30");
    
    // 文字列リテラルなら所有権の問題は発生しない
    println!("名前の存在: {}", map.contains_key("name"));
}
```

***

## コレクションの操作

### 反復処理パターン
```csharp
// C# の反復処理パターン
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// インデックス付き for ループ
for (int i = 0; i < numbers.Count; i++)
{
    Console.WriteLine($"インデックス {i}: {numbers[i]}");
}

// Foreach ループ
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
    println!("インデックス {}: {}", i, num);
}

// 値に対する for ループ
for num in &numbers {  // 各要素を借用
    println!("{}", num);
}

// イテレータメソッド（LINQ に類似）
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
let evens: Vec<i32> = numbers.iter().filter(|&x| x % 2 == 0).cloned().collect();

// より効率的な、イテレータの消費
let doubled: Vec<i32> = numbers.into_iter().map(|x| x * 2).collect();
```

### Iterator vs IntoIterator vs Iter
```rust
// さまざまなイテレーションメソッドの理解
fn iteration_methods() {
    let vec = vec![1, 2, 3, 4, 5];
    
    // 1. iter() - 要素を不変借用 (&T)
    for item in vec.iter() {
        println!("{}", item);  // item は &i32
    }
    // vec はここでもまだ使用可能
    
    // 2. into_iter() - 所有権を取得 (T)
    for item in vec.into_iter() {
        println!("{}", item);  // item は i32
    }
    // vec はこれ以降使用不可
    
    let mut vec = vec![1, 2, 3, 4, 5];
    
    // 3. iter_mut() - 可変借用 (&mut T)
    for item in vec.iter_mut() {
        *item *= 2;  // item は &mut i32
    }
    println!("{:?}", vec);  // [2, 4, 6, 8, 10]
}
```

### 結果の収集（Collect）
```csharp
// C# - エラーの可能性があるコレクションの処理
public List<int> ParseNumbers(List<string> inputs)
{
    var results = new List<int>();
    foreach (string input in inputs)
    {
        if (int.TryParse(input, out int result))
        {
            results.Add(result);
        }
        // 無効な入力は暗黙的にスキップ
    }
    return results;
}
```

```rust
// Rust - collect による明示的なエラーハンドリング
fn parse_numbers(inputs: Vec<String>) -> Result<Vec<i32>, std::num::ParseIntError> {
    inputs.into_iter()
        .map(|s| s.parse::<i32>())  // Result<i32, ParseIntError> を返す
        .collect()                  // Result<Vec<i32>, ParseIntError> に収集される
}

// 代替案: エラーをフィルタリングして除外
fn parse_numbers_filter(inputs: Vec<String>) -> Vec<i32> {
    inputs.into_iter()
        .filter_map(|s| s.parse::<i32>().ok())  // Ok の値のみを保持
        .collect()
}

fn main() {
    let inputs = vec!["1".to_string(), "2".to_string(), "invalid".to_string(), "4".to_string()];
    
    // 最初のエラーで失敗するバージョン
    match parse_numbers(inputs.clone()) {
        Ok(numbers) => println!("すべてパース成功: {:?}", numbers),
        Err(error) => println!("パースエラー: {}", error),
    }
    
    // エラーをスキップするバージョン
    let numbers = parse_numbers_filter(inputs);
    println!("パース成功分: {:?}", numbers);  // [1, 2, 4]
}
```

---

## 演習問題

<details>
<summary><strong>🏋️ 演習: LINQ からイテレータへ</strong> (クリックして展開)</summary>

以下の C# LINQ クエリを、慣用的な Rust イテレータに翻訳してください:

```csharp
var result = students
    .Where(s => s.Grade >= 90)
    .OrderByDescending(s => s.Grade)
    .Select(s => $"{s.Name}: {s.Grade}")
    .Take(3)
    .ToList();
```

次の構造体を使用してください:
```rust
struct Student { name: String, grade: u32 }
```

成績が 90 以上の生徒の上位 3 名を `"名前: 成績"` の形式にフォーマットした `Vec<String>` を返してください。

<details>
<summary>🔑 解答</summary>

```rust
#[derive(Debug)]
struct Student { name: String, grade: u32 }

fn top_students(students: &mut [Student]) -> Vec<String> {
    students.sort_by(|a, b| b.grade.cmp(&a.grade)); // 降順ソート
    students.iter()
        .filter(|s| s.grade >= 90)
        .take(3)
        .map(|s| format!("{}: {}", s.name, s.grade))
        .collect()
}

fn main() {
    let mut students = vec![
        Student { name: "Alice".into(), grade: 95 },
        Student { name: "Bob".into(), grade: 88 },
        Student { name: "Carol".into(), grade: 92 },
        Student { name: "Dave".into(), grade: 97 },
        Student { name: "Eve".into(), grade: 91 },
    ];
    let result = top_students(&mut students);
    assert_eq!(result, vec!["Dave: 97", "Alice: 95", "Carol: 92"]);
    println!("{result:?}");
}
```

**C# との主な違い**: Rust のイテレータは（LINQ と同様に）遅延評価されますが、`.sort_by()` は即座にインプレースで実行されます（遅延評価される `OrderBy` はありません）。そのため、まずソートを行ってから、遅延操作をチェーンします。

</details>
</details>

***
