# Rustの並行性

> **学習目標:** Rustの並行性モデル — スレッド、`Send`/`Sync` マーカートレイト、`Mutex<T>`、`Arc<T>`、チャンネル、そしてコンパイラがコンパイル時にデータ競合（data race）を防止する仕組みを学びます。使用しないスレッドセーフ機能に対する実行時オーバーヘッドはゼロです。

- Rustは、C++の `std::thread` と同様に、組み込みで並行性をサポートしています
    - 決定的な違い: Rustは `Send` と `Sync` というマーカートレイトを通じて、**データ競合をコンパイル時に防止します**
    - C++では、mutexなしで `std::vector` を複数スレッド間で共有することは未定義動作（UB）ですが、コンパイルは通ってしまいます。Rustでは、そもそもコンパイルエラーになります。
    - Rustの `Mutex<T>` はアクセスだけでなく**データそのものをラップします** — つまり、ロックを取得しない限り物理的にデータを読み書きできません
- `thread::spawn()` を使用すると、クロージャ `||` を並列に実行する独立したスレッドを生成できます
```rust
use std::thread;
use std::time::Duration;
fn main() {
    let handle = thread::spawn(|| {
        for i in 0..10 {
            println!("スレッド内カウント: {i}!");
            thread::sleep(Duration::from_millis(5));
        }
    });

    for i in 0..5 {
        println!("メインスレッド: {i}");
        thread::sleep(Duration::from_millis(5));
    }

    handle.join().unwrap(); // handle.join() は生成されたスレッドが終了するのを待機・保証します
}
```

# Rustの並行性
- 周囲の環境から変数を借用する必要がある場合は、`thread::scope()` を使用できます。これは、`thread::scope` が内部のスレッドが終了するまで待機するため安全に機能します
- `thread::scope` を使わずにこの処理を実行してみて、どのような問題（ライフタイムエラー）が発生するか確認してみてください
```rust
use std::thread;
fn main() {
  let a = [0, 1, 2];
  thread::scope(|scope| {
      scope.spawn(|| {
          for x in &a {
            println!("{x}");
          }
      });
  });
}
```
----
# Rustの並行性
- `move` キーワードを使用して、所有権をスレッドに移動（ムーブ）することもできます。`[i32; 3]` のような `Copy` 型の場合、`move` キーワードによってデータがクロージャ内にコピーされるため、元のデータも引き続き使用可能です
```rust
use std::thread;
fn main() {
  let mut a = [0, 1, 2];
  let handle = thread::spawn(move || {
      for x in a {
        println!("{x}");
      }
  });
  a[0] = 42;    // スレッドに送られたコピーには影響しません
  handle.join().unwrap();
}
```

# Rustの並行性
- `Arc<T>` を使用すると、複数のスレッド間で*読み取り専用*の参照を共有できます
    - `Arc` は Atomic Reference Counted（アトミック参照カウント）の略です。参照カウントが 0 になるまで参照先のメモリは解放されません
    - `Arc::clone()` は、データを複製（ディープコピー）することなく、参照カウントをインクリメントするだけです
```rust
use std::sync::Arc;
use std::thread;
fn main() {
    let a = Arc::new([0, 1, 2]);
    let mut handles = Vec::new();
    for i in 0..2 {
        let arc = Arc::clone(&a);
        handles.push(thread::spawn(move || {
            println!("スレッド {i}: {arc:?}");
        }));
    }
    handles.into_iter().for_each(|h| h.join().unwrap());
}
```

# Rustの並行性
- `Arc<T>` は `Mutex<T>` と組み合わせることで、可変（ミュータブル）な参照を共有できるようになります。
    - `Mutex` は保護対象のデータをガードし、ロックを保持しているスレッドのみがアクセスできるように保証します。
    - `MutexGuard` はスコープを抜けると自動的に解放されます（RAII）。注: `std::mem::forget` を使用した場合はガードがリークする可能性があるため、「アンロックを忘れることが不可能」という表現の方が「リークすることが不可能」よりも正確です。
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..5 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
            // ここで MutexGuard がドロップされ、ロックが自動的に解放されます
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("最終カウント: {}", *counter.lock().unwrap());
    // 出力: 最終カウント: 5
}
```

# Rustの並行性: RwLock
- `RwLock<T>` は、**複数の並行リーダー**または**1つの排他的なライター**を許可します — C++のリード/ライトロックパターン（`std::shared_mutex`）と同様です
    - 書き込みよりも読み取りの頻度が圧倒的に高い場合（設定データ、キャッシュなど）に `RwLock` を使用します
    - 読み取りと書き込みの頻度が同等である場合や、クリティカルセクションが短い場合は `Mutex` を使用します
```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let config = Arc::new(RwLock::new(String::from("v1.0")));
    let mut handles = Vec::new();

    // 5つのリーダー（読み取りスレッド）を生成 — すべて並行実行可能
    for i in 0..5 {
        let config = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            let val = config.read().unwrap();  // 複数のリーダーが同時にアクセス可能
            println!("リーダー {i}: {val}");
        }));
    }

    // 1つのライター（書き込みスレッド） — すべてのリーダーが終了するまでブロック
    {
        let config = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            let mut val = config.write().unwrap();  // 排他アクセス
            *val = String::from("v2.0");
            println!("ライター: {val} に更新しました");
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

# Rustの並行性: Mutex poisoning（ポイズニング）
- スレッドが `Mutex` または `RwLock` を保持したまま**パニック**を起こすと、ロックは**ポイズン状態（汚染状態）**になります
    - その後 `.lock()` を呼び出すと `Err(PoisonError)` が返されます — データが不整合な状態にある可能性があるためです
    - データが依然として有効であると確信できる場合は、`.into_inner()` を使用して復旧させることができます
    - これに相当する機能はC++にはありません — `std::mutex` にはポイズニングの概念がなく、パニック（例外）したスレッドは単にロックを保持したままになるか解放されるだけです
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));

    let data2 = Arc::clone(&data);
    let handle = thread::spawn(move || {
        let mut guard = data2.lock().unwrap();
        guard.push(4);
        panic!("おっと!");  // ここでロックがポイズン（汚染）状態になる
    });

    let _ = handle.join();  // スレッドがパニックした

    // 以降の lock 呼び出しは Err(PoisonError) を返す
    match data.lock() {
        Ok(guard) => println!("データ: {guard:?}"),
        Err(poisoned) => {
            println!("ロックがポイズン状態でした! 復旧中...");
            let guard = poisoned.into_inner();  // いずれにせよデータにアクセス
            println!("復旧されたデータ: {guard:?}");  // [1, 2, 3, 4] — パニック前に push は成功していた
        }
    }
}
```

# Rustの並行性: アトミック（Atomics）
- 単純なカウンタやフラグには、`std::sync::atomic` の型を使用することで `Mutex` のオーバーヘッドを回避できます
    - `AtomicBool`、`AtomicI32`、`AtomicU64`、`AtomicUsize` などがあります
    - C++の `std::atomic<T>` に相当し、メモリ順序モデル（メモリオーダリング）も同一です（`Relaxed`、`Acquire`、`Release`、`SeqCst`）
```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicU64::new(0));
    let mut handles = Vec::new();

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1000 {
                counter.fetch_add(1, Ordering::Relaxed);
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("カウンタ: {}", counter.load(Ordering::SeqCst));
    // 出力: カウンタ: 10000
}
```

| プリミティブ | 使用場面 | C++での相当機能 |
|-----------|-------------|----------------|
| `Mutex<T>` | 一般的な可変の共有状態 | `std::mutex` + 手動でのデータ関連付け |
| `RwLock<T>` | 読み取り頻度が高いワークロード | `std::shared_mutex` |
| `Atomic*` | 単純なカウンタ、フラグ、ロックフリーパターン | `std::atomic<T>` |
| `Condvar` | 条件が真になるまで待機する場合 | `std::condition_variable` |

# Rustの並行性: Condvar
- `Condvar`（条件変数）を使用すると、**別のスレッドが条件の変更を通知（シグナル）するまでスレッドをスリープ**させることができます
    - 常に `Mutex` とペアで使用します — 基本パターンは「ロック取得 → 条件チェック → 未完了なら待機 → 準備完了したら処理実行」です
    - C++の `std::condition_variable` / `std::condition_variable::wait` に相当します
    - **偽の目覚め（spurious wakeup）**を処理するため、常にループ内で条件を再チェックしてください（または `wait_while`/`wait_until` を使用）
```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));

    // シグナルを待機するワーカースレッドを生成
    let pair2 = Arc::clone(&pair);
    let worker = thread::spawn(move || {
        let (lock, cvar) = &*pair2;
        let mut ready = lock.lock().unwrap();
        // wait: シグナルを受信するまでスリープ（偽の目覚めに備えて必ずループ内で再チェック）
        while !*ready {
            ready = cvar.wait(ready).unwrap();
        }
        println!("ワーカー: 条件が満たされたため処理を続行します!");
    });

    // メインスレッドで処理を行い、ワーカースレッドにシグナルを送信
    thread::sleep(std::time::Duration::from_millis(100));
    {
        let (lock, cvar) = &*pair;
        let mut ready = lock.lock().unwrap();
        *ready = true;
        cvar.notify_one();  // 待機中のスレッドを1つ起床（notify_all() はすべて起床）
    }

    worker.join().unwrap();
}
```

> **Condvar とチャンネルの使い分け:** スレッド間で可変状態を共有し、その状態に関する条件（例:「バッファが空でない」など）を待機する必要がある場合は `Condvar` を使用します。スレッド間で*メッセージ*を受け渡す必要がある場合はチャンネル（`mpsc`）を使用します。一般にチャンネルの方が処理の流れを把握しやすくなります。

# Rustの並行性
- Rustのチャンネルを使用すると、`Sender`（送信側）と `Receiver`（受信側）の間でメッセージを交換できます
    - これは `mpsc`（`Multi-Producer, Single-Consumer`、複数プロデューサ・単一コンシューマ）と呼ばれるパラダイムを採用しています
    - `send()` と `recv()` の両方がスレッドをブロックする可能性があります
```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    tx.send(10).unwrap();
    tx.send(20).unwrap();
    
    println!("受信: {:?}", rx.recv());
    println!("受信: {:?}", rx.recv());

    let tx2 = tx.clone();
    tx2.send(30).unwrap();
    println!("受信: {:?}", rx.recv());
}
```

# Rustの並行性
- チャンネルはスレッドと組み合わせて使用できます
```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();
    for _ in 0..2 {
        let tx2 = tx.clone();
        thread::spawn(move || {
            let thread_id = thread::current().id();
            for i in 0..10 {
                tx2.send(format!("メッセージ {i}")).unwrap();
                println!("{thread_id:?}: メッセージ {i} を送信しました");
            }
            println!("{thread_id:?}: 完了");
        });
    }

    // オリジナルの送信側（tx）をドロップし、複製されたすべての送信側がドロップされた時点で rx.iter() が終了するようにする
    drop(tx);

    thread::sleep(Duration::from_millis(100));

    for msg in rx.iter() {
        println!("メイン: {msg} を受信しました");
    }
}
```



## Rustがデータ競合を防止できる理由: Send と Sync

- Rustは2つのマーカートレイトを使用して、コンパイル時にスレッドセーフ性を保証します:
    - `Send`: 別のスレッドに安全に**所有権を転送（トランスファー）**できる型に実装されます
    - `Sync`: 複数スレッド間で（`&T` を通じて）安全に**共有**できる型に実装されます
- ほとんどの型は自動的に `Send + Sync` になります。注目すべき例外は以下の通りです:
    - `Rc<T>` は `Send` でも `Sync` でも**ありません**（スレッド間では `Arc<T>` を使用してください）
    - `Cell<T>` と `RefCell<T>` は `Sync` では**ありません**（`Mutex<T>` または `RwLock<T>` を使用してください）
    - 生ポインタ（`*const T`、`*mut T`）は `Send` でも `Sync` でも**ありません**
- これが、コンパイラがスレッド間で `Rc<T>` を使用することを拒絶する理由です — 単純に `Send` を実装していないためです
- `Arc<Mutex<T>>` は、`Rc<RefCell<T>>` のスレッドセーフ版に相当します

> **直感的な理解** *(Jon Gjengset 氏による解説)*: 値をオモチャとして考えてみてください。
> **`Send`** = 自分のオモチャを別の子供（スレッド）に**あげてしまう**ことができる — 所有権の転送が安全であることを意味します。
> **`Sync`** = 自分のオモチャで**同時に他の子供たちも一緒に遊ばせる**ことができる — 参照の共有が安全であることを意味します。
> `Rc<T>` は壊れやすい（非アトミックな）参照カウンタを持っているため、人に渡したり共有したりするとカウントが壊れてしまいます。そのため、`Send` でも `Sync` でもありません。


# 演習: マルチスレッド単語カウント

🔴 **チャレンジ課題** — スレッド、Arc、Mutex、HashMap の組み合わせ

- テキスト行の `Vec<String>` が与えられたとき、各行ごとにスレッドを生成してその行内の単語数をカウントしてください
- 結果の収集には `Arc<Mutex<HashMap<String, usize>>>` を使用してください
- すべての行を通じた合計単語数を出力してください
- **ボーナス課題**: 共有状態の代わりにチャンネル（`mpsc`）を使って実装してみてください

<details><summary>解答例（クリックして展開）</summary>

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let lines = vec![
        "the quick brown fox".to_string(),
        "jumps over the lazy dog".to_string(),
        "the fox is quick".to_string(),
    ];

    let word_counts: Arc<Mutex<HashMap<String, usize>>> =
        Arc::new(Mutex::new(HashMap::new()));

    let mut handles = vec![];
    for line in &lines {
        let line = line.clone();
        let counts = Arc::clone(&word_counts);
        handles.push(thread::spawn(move || {
            for word in line.split_whitespace() {
                let mut map = counts.lock().unwrap();
                *map.entry(word.to_lowercase()).or_insert(0) += 1;
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let counts = word_counts.lock().unwrap();
    let total: usize = counts.values().sum();
    println!("単語の出現頻度: {counts:#?}");
    println!("合計単語数: {total}");
}
// 出力例（順序は異なる場合があります）:
// 単語の出現頻度: {
//     "the": 3,
//     "quick": 2,
//     "brown": 1,
//     "fox": 2,
//     "jumps": 1,
//     "over": 1,
//     "lazy": 1,
//     "dog": 1,
//     "is": 1,
// }
// 合計単語数: 13
```

</details>
