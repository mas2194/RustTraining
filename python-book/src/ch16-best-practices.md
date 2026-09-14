## Python開発者のためのイディオマティックRust

> **学ぶこと:** 身につけるべき10の習慣、よくある落とし穴とその対策、体系的な3ヶ月の学習ロードマップ、
> Python→Rustの「ロゼッタストーン」対応表、および推奨される学習リソースについて学びます。
>
> **難易度:** 🟡 中級

```mermaid
flowchart LR
    A["🟢 1〜2週目<br/>基礎固め<br/>「なぜコンパイルが通らない？」"] --> B["🟡 3〜4週目<br/>コア概念<br/>「なるほど、守ってくれてるのか」"] 
    B --> C["🟡 2ヶ月目<br/>中級<br/>「なぜこれが重要かわかってきた」"]
    C --> D["🔴 3ヶ月目以降<br/>上級<br/>「コンパイル時にバグを捕捉できた！」"]
    D --> E["🏆 6ヶ月目<br/>習熟<br/>「どの言語でもより良いプログラマになれた」"]
    style A fill:#d4edda
    style B fill:#fff3cd
    style C fill:#fff3cd
    style D fill:#f8d7da
    style E fill:#c3e6cb,stroke:#28a745
```

### 身につけるべき10の習慣

1. **`if isinstance()` の代わりに列挙型（enum）に対する `match` を使用する**
   ```python
   # Python                              # Rust
   if isinstance(shape, Circle): ...     match shape { Shape::Circle(r) => ... }
   ```

2. **コンパイラの案内に従う** — エラーメッセージを注意深く読みましょう。Rustのコンパイラはあらゆる言語の中でも最高峰です。何が間違っているかだけでなく、それをどう修正すべきかも教えてくれます。

3. **関数の引数には `String` よりも `&str` を優先する** — 最も汎用的な型を受け取るようにします。`&str` であれば、`String` も文字列リテラルもどちらも受け付けることができます。

4. **インデックスによるループではなくイテレータを使用する** — イテレータチェーンを使用する方がイディオマティックであり、`for i in 0..vec.len()` よりも高速に動作することが多いです。

5. **`Option` と `Result` を使いこなす** — 何でも `.unwrap()` で済ませようとしないでください。`?` 演算子、`map`、`and_then`、`unwrap_or_else` などを活用しましょう。

6. **トレイトを積極的に導出（derive）する** — ほとんどの構造体に `#[derive(Debug, Clone, PartialEq)]` を付与すべきです。コストは実質ゼロで、テストやデバッグが大幅に容易になります。

7. **`cargo clippy` を日常的に活用する** — スタイルやコードの正当性に関する何百もの問題を検出してくれます。Pythonにおける `ruff` のように、常にclippyのアドバイスに従いましょう。

8. **ボローチェッカと戦わない** — ボローチェッカとの戦いに苦労しているなら、データ構造の設計に問題がある可能性が高いです。所有権の所在が明確になるようにリファクタリングしましょう。

9. **状態機械（ステートマシン）には列挙型を使用する** — 文字列フラグや真偽値の代わりに、列挙型（enum）を使用します。コンパイラがすべての状態の網羅を保証してくれます。

10. **まずはクローン、最適化は後から** — 学習初期の段階では、所有権の複雑さに悩まされないよう、自由に `.clone()` を使ってください。プロファイリングを行って真に必要な箇所だけを最適化しましょう。

### Python開発者が陥りがちな共通の誤り

| 誤り | 理由 | 対策 |
|------|------|------|
| 至る所で `.unwrap()` を使用 | 実行時パニックの原因になる | `?` 演算子や `match` を使用する |
| `&str` ではなく `String` を引数に取る | 不要なメモリ割り当てが発生する | 引数には `&str` を使用する |
| `for i in 0..vec.len()` を使用 | イディオマティックではない | `for item in &vec` を使用する |
| clippyの警告を無視する | 簡単な改善の機会を逃す | `cargo clippy` を実行して指示に従う |
| 過剰な `.clone()` の呼び出し | パフォーマンスのオーバーヘッド | 所有権の構造を見直す（リファクタリング） |
| 巨大な `main()` 関数 | テストが困難になる | `lib.rs` へロジックを切り出す |
| `#[derive()]` を使わない | 車輪の再発明になる | 一般的なトレイトを自動導出する |
| エラー発生時にすぐパニックさせる | 回復不能になってしまう | `Result<T, E>` を返す |

***

## パフォーマンスの比較

### ベンチマーク：一般的な操作
```text
操作                   Python 3.12    Rust (release)    高速化率
─────────────────────  ────────────   ──────────────    ─────────
Fibonacci(40)          約25秒         約0.3秒           約80倍
1,000万個の整数のソート 約5.2秒        約0.6秒           約9倍
100MBのJSONパース      約8.5秒        約0.4秒           約21倍
正規表現 100万件マッチ   約3.1秒        約0.3秒           約10倍
HTTPサーバー (req/s)   約5,000        約150,000         約30倍
1GBファイルのSHA-256   約12秒         約1.2秒           約10倍
100万行のCSVパース     約4.5秒        約0.2秒           約22倍
文字列結合             約2.1秒        約0.05秒          約42倍
```

> **注意**: C拡張（NumPyなど）を利用したPythonコードの場合、数値計算における差は大幅に縮まります。上記のベンチマークは、純粋なPython（Pure Python）と純粋なRustを比較したものです。

### メモリ使用量
```text
Python:                                 Rust:
─────────                               ─────
- オブジェクトヘッダー: 28バイト/オブジェクト - オブジェクトヘッダーなし
- int: 28バイト（0であっても）          - i32: 4バイト、i64: 8バイト
- str "hello": 54バイト                 - &str "hello": 16バイト（ポインタ + 長さ）
- 1,000個の整数リスト: 約36KB           - Vec<i32>: 約4KB
  (8KBのポインタ + 28KBのintオブジェクト)
- 100要素のdict: 約5.5KB                - 100要素のHashMap: 約2.4KB

典型的なアプリケーション全体のベースライン:
- Python: 50〜200MB                     - Rust: 1〜5MB
```

***

## よくある落とし穴と解決策

### 落とし穴1: 「ボローチェッカに怒られる」
```rust
// 問題: コレクションを反復処理しながら要素を変更しようとする
let mut items = vec![1, 2, 3, 4, 5];
// for item in &items {
//     if *item > 3 { items.push(*item * 2); }  // ❌ 借用中に可変借用することはできない
// }

// 解決策1: 変更内容を収集し、反復処理後に適用する
let additions: Vec<i32> = items.iter()
    .filter(|&&x| x > 3)
    .map(|&x| x * 2)
    .collect();
items.extend(additions);

// 解決策2: retain / extend などの専用メソッドを使用する
items.retain(|&x| x <= 3);
```

### 落とし穴2: 「文字列型が多すぎる」
```rust
// 迷ったときの基本方針:
// - 関数の引数には &str を使う
// - 構造体のフィールドや戻り値には String を使う
// - 文字列リテラル（"hello"）は &str が期待されるすべての場所で利用可能

fn process(input: &str) -> String {    // &str を受け取り、String を返す
    format!("処理結果: {}", input)
}
```

### 落とし穴3: 「Pythonのシンプルさが恋しい」
```rust
// Pythonの1行コード:
// result = [x**2 for x in data if x > 0]

// Rustにおける同等のコード:
let result: Vec<i32> = data.iter()
    .filter(|&&x| x > 0)
    .map(|&x| x * x)
    .collect();

// 記述量は増えますが、以下のメリットがあります:
// - コンパイル時に型安全
// - 10〜100倍高速
// - 実行時型エラーが決して発生しない
// - メモリ割り当てが明示的（.collect()）
```

### 落とし穴4: 「REPLはどこ？」
```rust
// Rustには標準の対話型REPLがありません。代わりに以下を活用します:
// 1. `cargo test` をREPL代わりにする — 試行錯誤用の小さなテストを書く
// 2. 素早い実験には Rust Playground (play.rust-lang.org) を使用する
// 3. 簡単なデバッグ出力には `dbg!()` マクロを使用する
// 4. ファイル保存時に自動でテストを実行するには `cargo watch -x test` を使用する

#[test]
fn playground() {
    // これを「REPL」として活用する — `cargo test playground` で実行
    let result = "hello world"
        .split_whitespace()
        .map(|w| w.to_uppercase())
        .collect::<Vec<_>>();
    dbg!(&result);  // 出力: [src/main.rs:5] &result = ["HELLO", "WORLD"]
}
```

***

## 学習ロードマップとリソース

### 1〜2週目: 基礎固め
- [ ] Rustをインストールし、VS Code + rust-analyzer環境をセットアップする
- [ ] 本ガイドの第1〜4章（型、制御構文など）を修了する
- [ ] 簡単なPythonスクリプトをRustに移植する小さなプログラムを5つ書く
- [ ] `cargo build`、`cargo test`、`cargo clippy` の操作に慣れる

### 3〜4週目: コア概念
- [ ] 第5〜8章（構造体、列挙型、所有権、モジュール）を修了する
- [ ] Pythonで書かれたデータ処理スクリプトをRustで書き直す
- [ ] `Option<T>` と `Result<T, E>` の扱いが自然になるまで練習する
- [ ] コンパイラのエラーメッセージを丁寧に読む（コンパイラが教えてくれています）

### 2ヶ月目: 中級
- [ ] 第9〜12章（エラー処理、トレイト、イテレータ）を修了する
- [ ] `clap` と `serde` を使用したCLIツールを作成する
- [ ] 既存のPythonプロジェクトのボトルネック向けにPyO3拡張機能を書く
- [ ] 内包表記と同じくらい自然にイテレータチェーンが書けるようになるまで練習する

### 3ヶ月目: 上級
- [ ] 第13〜16章（並行性、unsafe、テスト）を修了する
- [ ] `axum` と `tokio` を使ったWebサービスを作成する
- [ ] オープンソースのRustプロジェクトにコントリビュートしてみる
- [ ] 理解を深めるために『Programming Rust』（オライリー本）を読む

### 推奨リソース
- **The Rust Book**: https://doc.rust-lang.org/book/ （公式ドキュメント、必読）
- **Rust by Example**: https://doc.rust-lang.org/rust-by-example/ （実践形式で学ぶ）
- **Rustlings**: https://github.com/rust-lang/rustlings （演習課題集）
- **Rust Playground**: https://play.rust-lang.org/ （ブラウザ上で動くコンパイラ）
- **This Week in Rust**: https://this-week-in-rust.org/ （週刊ニュースレター）
- **PyO3 Guide**: https://pyo3.rs/ （Python ↔ Rust ブリッジガイド）
- **Comprehensive Rust** (Google): https://google.github.io/comprehensive-rust/

### Python → Rust ロゼッタストーン（対比表）

| Python | Rust | 該当章 |
|--------|------|--------|
| `list` | `Vec<T>` | 5 |
| `dict` | `HashMap<K,V>` | 5 |
| `set` | `HashSet<T>` | 5 |
| `tuple` | `(T1, T2, ...)` | 5 |
| `class` | `struct` + `impl` | 5 |
| `@dataclass` | `#[derive(...)]` | 5, 12a |
| `Enum` | `enum` | 6 |
| `None` | `Option<T>` | 6 |
| `raise`/`try`/`except` | `Result<T,E>` + `?` | 9 |
| `Protocol` (PEP 544) | `trait` | 10 |
| `TypeVar` | ジェネリクス `<T>` | 10 |
| `__dunder__` 特殊メソッド | 各種トレイト（Display, Add など） | 10 |
| `lambda` | `\|args\| body` | 12 |
| ジェネレータ `yield` | `impl Iterator` | 12 |
| リスト内包表記 | `.map().filter().collect()` | 12 |
| `@decorator` | 高階関数またはマクロ | 12a, 15 |
| `asyncio` | `tokio` | 13 |
| `threading` | `std::thread` | 13 |
| `multiprocessing` | `rayon` | 13 |
| `unittest.mock` | `mockall` | 14a |
| `pytest` | `cargo test` + `rstest` | 14a |
| `pip install` | `cargo add` | 8 |
| `requirements.txt` | `Cargo.lock` | 8 |
| `pyproject.toml` | `Cargo.toml` | 8 |
| `with`（コンテキストマネージャ） | スコープベースの `Drop` | 15 |
| `json.dumps/loads` | `serde_json` | 15 |

***

## Python開発者への結びの言葉

```rust
Pythonが恋しくなる瞬間:
- REPLによるインタラクティブな試行錯誤
- 素早いプロトタイピングのスピード
- 豊富なML/AIエコシステム（PyTorchなど）
- 「とりあえず動く」動的型付けの手軽さ
- pip install による即座の利用

Rustから得られる大きな恩恵:
- 「コンパイルが通れば動く」という絶対的な安心感
- 10〜100倍のパフォーマンス向上
- 実行時型エラーの完全な撲滅
- None/null参照によるクラッシュの解消
- 真の並列処理（GILからの解放！）
- 単一バイナリによる簡単なデプロイ
- 予測可能なメモリ使用量
- あらゆる言語の中でも最も親切なコンパイラエラーメッセージ

習得までの道のり:
1週目:   「なぜコンパイラはこんなに私を嫌うのか？」
2週目:   「なるほど、実はバグから私を守ってくれているのか」
1ヶ月目: 「なぜこの仕組みが重要なのか理解できてきた」
2ヶ月目: 「本番障害になり得たバグをコンパイル時に捕捉できた！」
3ヶ月目: 「もう型の付いていないコードには戻りたくない」
6ヶ月目: 「Rustを学んだことで、他のどの言語を書くときもより良いプログラマになれた」
```

---

## 演習問題

<details>
<summary><strong>🏋️ 演習問題：コードレビューのチェックリスト</strong> (クリックして展開)</summary>

**課題**: Python開発者が書いた以下のRustコードをレビューし、よりイディオマティックにするための改善点を5つ指摘してください:

```rust
fn get_name(names: Vec<String>, index: i32) -> String {
    if index >= 0 && (index as usize) < names.len() {
        return names[index as usize].clone();
    } else {
        return String::from("");
    }
}

fn main() {
    let mut result = String::from("");
    let names = vec!["Alice".to_string(), "Bob".to_string()];
    result = get_name(names.clone(), 0);
    println!("{}", result);
}
```

<details>
<summary>🔑 解答例</summary>

5つの改善点:

```rust
// 1. Vec<String> ではなく &[String] を受け取る（ベクタ全体の所有権を奪わない）
// 2. インデックスには i32 ではなく usize を使用する（インデックスは常に非負）
// 3. 空文字列ではなく Option<&str> を返す（型システムを活用する！）
// 4. 手動で範囲チェックを行う代わりに .get() メソッドを使用する
// 5. main 内で clone() を使わず、参照を渡す

fn get_name(names: &[String], index: usize) -> Option<&str> {
    names.get(index).map(|s| s.as_str())
}

fn main() {
    let names = vec!["Alice".to_string(), "Bob".to_string()];
    match get_name(&names, 0) {
        Some(name) => println!("{name}"),
        None => println!("見つかりませんでした"),
    }
}
```

**重要なポイント**: Rustでパフォーマンスや可読性を損なうPythonの習慣：何でもクローンする（借用を使う）、`""` のような番兵（センチネル）値を使う（`Option` を使う）、借用で済む場面で所有権を奪う、インデックスに符号付き整数を使う。

</details>
</details>

***

*PythonプログラマーのためのRustトレーニングガイド 完*
