---
title: "prpr: Rust 向けのシンプルで高速な Parser generator"
emoji: "👅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "parser", "PEG"]
published: false
---

[prpr (PackRat Parser in Rust)](https://github.com/tanakh/prpr) という Rust 向けの新しい Parser generator ライブラリを作りました。

https://github.com/tanakh/prpr

詳しい使い方を書いたドキュメントがありますので、使ってみたいと思った方はぜひこちらをご覧になってください。

https://tanakh.github.io/prpr/

開発の背景についてちょっと説明しておきますと、これ自体はずいぶん昔から開発していたものなのですが、あまり公開するに足るクオリティーに達する前に放置してしまってずっと寝かせていたんですが、Rustも当時からだいぶバージョンが上がって、proc-macro 周りの機能も充実してきたので、今作れば思ったようなものが作れるかなという目論見で新たに作り直しました。かなり自分では良いものができたんじゃないかと思います。Rust の parser generator 環境もあれからかなり変わってきていて、似たようなコンセプトのライブラリはすでにいくつかあったりしますが、それでもまだリリースする意味はあるかなと思っております。

# 概要

prpr は [Parsing Expression Grammar (PEG)](https://ja.wikipedia.org/wiki/Parsing_expression_grammar) と呼ばれる形式文法を採用しています。PEGは、[Packrat parser](https://en.wikipedia.org/wiki/Packrat_parser) によって入力サイズに対して線型時間で解析できることが知られており、cut演算子やtry演算子などによるバックトラック制御なしに最悪計算量の保証された構文解析が実現できます。prpr では実装上の工夫を行うことで、実際に全体的にかなり高速なコードが生成できています。

また、PEGでは基本的に別途トークナイザーを必要としないということが特徴として語られることがしばしばありますが[^1]、確かにトークナイザーなしでの構文解析をPEGでは本質的な制約なしに行うことが可能ではあるが、実際のところ空白の読み飛ばしだったり、デリミタの処理だったりが入って結局煩雑になりがちです（コンビネーターベースのものでも結局似たようなことをやる羽目にはなったりする）。prpr ではデフォルトで自動の空白スキップやデリミタ制御が用意されており、カスタマイズも可能です。

prpr は構文に余分なノイズがなるべく入らないように設計しました。つまり、パーザーのコードを読んでいて「おや？」となるような謎の記号がなるべく出てこないように注意を払いました。基本的にはPEGのルールをそのまま素直に書いて、そこに受理アクションを混ぜ込めるようにしたという形です。覚えやすく書きやすく、それでいて表現力が制限されず、高速なコードが生成されて、なおかつ適切なエラーレポートが出るような、そういう理想です。そしてこれらはおよそ実現できていると思います。

prpr は prpr 自身を用いてパーズされます。文法は[自身で完全に定義されており](https://tanakh.github.io/prpr/reference/prpr-peg.html)、コードは短く簡素にまとまっているはずなので、これを参照することで実際の完全な文法を把握することができます。文法は `proc-macro` でRustのコードに埋め込んで直接記述できるほか、独立したプログラムファイルとして生成することもできます。生成したプログラムは他のライブラリあるいはランタイムへの依存なしに動作させられます。prpr 自身のブートストラップはこれを用いて実現されています。

[^1]: LR(1) / LALR(1) などでは先読みトークン数を1とすることでサイズの小さいオートマトンを生成できるようにしているため、入力トークンの単位が「文字」だとコンフリクト制御や状態数爆発で大変なことになりそう。PEGでは先読みトークン数に制限はないし、それに加えて選択に優先順位があるので文字列入力で問題ないという話だと思われる。

# 例

まずは簡単な例を示します。

```rust
prpr::grammar! {
    pub greeting = "hello" "world"
}

fn main() {
    assert_eq!(greeting("hello world"), Ok(()));
    assert!(greeting("  hello\n  world  ").is_ok());
    assert!(greeting("hello").is_err());
    assert!(greeting("helloworld").is_err());
}
```

ルール `greeting` は、`"hello"` と `"world"` の連続した文字列を受理します。`pub` は当該ルールの visibility を指定するものです。ルールはRustの関数として定義されるので、そこに `pub` を付けるという指示です。

prpr ではデフォルトで空白読み飛ばしが行われるので、`"hello world"` というという文字列や `"  hello\n  world  "` という文字列はどちらも受理されます。また、要素間の区切りにはデリミタが要求されるので、`"helloworld"` のような文字列は受理されません。これは例えばプログラミング言語のパーザーを書くときに `if x == 0 { ... }` のようなコードがあったとして、これを `ifx == 0 { ... }` などと書いてしまったときに受理されないようにするために望ましいデフォルト動作です。トークナイザーなしで現実的な構文を定義しようとするなら重要になってきます。もちろん、空白読み飛ばしやデリミタ要求を行わないようにする機能もあります。例えば、

```rust
prpr::grammar! {
    pub atom greeting = "hello" "world"
}
```

ルール名に `atom` を付ければ、これは atomic なルール、いわゆる「トークン」として扱うようになります。`atom` ルール内では一切の暗黙の空白制御やデリミタの処理が行われず、書いてあるものを strict に受理します。

```rust
prpr::grammar! {
    pub greeting = <"hello" "world">
}
```

あるいは、`<e>` という形で囲めば、その中はatomicとして処理されます。この場合、`"hello world"` という文字列は受理されなくなりますが、`greeting` 自体は `atom` ではないので、先頭の空白除去と、組み合わせた際のデリミタ処理は行われます。

```rust
prpr::grammar! {
    pub greeting = "say" <"hello" "world">
}
fn main() {
    assert_eq!(greeting("say helloworld"), Ok(()));
    assert!(greeting("sayhelloworld").is_ok());
}
```

と言った具合に、`say` と `helloworld` が繋がっていると受理されないという形になります。

## ルールの値

ルールには型がつけられて、返す値をアクションで書くことができます。

```rust
prpr::grammar! {
    pub atom number: i64 = s:([0-9]+) { s.into_iter().collect::<String>().parse().unwrap() }
}
fn main() {
    assert_eq!(number("42"), Ok(42));
    assert!(number("4 2").is_err());
}
```

任意の式の後に、`{ ... }` でRustの式のアクションを記述でき、束縛した変数を参照できます（この例だと `s:([0-9]+)`）。

ちなみに、前の例（`greeting`）のような型がつけられていないルールは `()` を返すルールだと解釈されます。Rustでいうところの返り値の無い関数と同じ振る舞いだと考えてもらえば一貫性があると思います。

変数の型は、束縛した式によって決まります（詳細は[こちら](https://tanakh.github.io/prpr/guide/values-and-actions.html#the-values-of-expressions)）。ここでは文字の繰り返しなので `s` の型は `Vec<char>` になります。これを `String` に変換して、`parse` しているという形です。

これは効率が悪いので、もうちょっと効率の良い書き方もできます。

```rust
prpr::grammar! {
    pub atom number: i64 = s:([0-9]+) { #s.parse().unwrap() }
}
```

変数参照の際に `#` を付けると、値の代わりに、解析に成功した入力の範囲の `&str` が取得できます。この場合値は構築されないので、非常に効率の良いコードが生成されます。

また、変数に `@` を付けると、マッチした範囲が `Range<usize>` で取得できます。これは span 付き AST を作るときなどに便利です。また、入力のライフタイムとして `'i` が定義されており、ルールに対する型の場所で使うことができます。これはゼロコピーでの構文解析に利用できます。

```rust
use std::range::Range;

#[derive(Debug, PartialEq)]
pub struct Assign<'a> {
    name: &'a str,
    name_span: Range<usize>,
    span: Range<usize>,
}

prpr::grammar! {
    pub assign: Assign<'i> = n:ident "=" ident {
        Assign { name: #n, name_span: @n, span: @_ }
    }

    atom ident = [a-z]+
}
```

`#_` や `@_` という特別な変数も定義されていて、これはそれぞれシーケンス全体の文字列と範囲に相当します。

詳細は [こちら](https://tanakh.github.io/prpr/guide/values-and-actions.html#the-text-and-the-span-of-a-binding) をご参照ください。

## 四則演算の例

よくある四則演算の式を解析して計算する例を示します。

```rust
prpr::grammar! {
    pub expr: i64 = l:term rs:(("+" / "-") term)* {
        rs.into_iter().fold(l, |a, (op, b)| if op == "+" { a + b } else { a - b })
    }
    term: i64 = l:fact rs:(("*" / "/") fact)* {
        rs.into_iter().fold(l, |a, (op, b)| if op == "*" { a * b } else { a / b })
    }
    fact: i64 = number / "(" e:expr ")" { e }

    #[label = "number"]
    atom number: i64 = s:([0-9]+) { #s.parse().unwrap() }
}

fn main() {
    assert_eq!(expr("1 + 2 * 3"), Ok(7));
    assert_eq!(expr("(1 + 2) * 3"), Ok(9));
    assert_eq!(expr("10 - 2 - 3"), Ok(5));
    assert_eq!(expr("-(4 / 2)"), Ok(-2));
}
```

`#[label = "..."]` というのものがありますが、これはエラーレポートに使われます。prpr では、構文解析に失敗した場合に、失敗した一番後ろの位置で、次に何が来るべきだったかの集合を計算してエラーメッセージを生成します。ここで `#[label = "..."]` が付いていなかったら、例えば `"(1 + foo) * 3"` を入力した場合、

```
Error: error: expected one of `(` or `[0-9]`, found `f`
 --> input:1:6
  |
1 | (1 + foo) * 3
  |      
```

というような具体的な文字を示すエラーが出ます。ラベルを付けていたら、

```
Error: error: expected one of `(` or number, found `f`
 --> input:1:6
  |
1 | (1 + foo) * 3
  |      ^
```

のように、`(` もしくは `number` が期待されるところに違う文字が来ましたよという、よりわかりやすいエラーメッセージになります。

# 構文

以下に prpr の構文全体を載せておきます。基本的にはPEGにあるものおよびシンタックスシュガーです（アクション関係以外）。

| expression                          | matches                                                      |
| ----------------------------------- | ------------------------------------------------------------ |
| `e1 / e2`                           | `e1` または `e2`                                             |
| `e1 e2`                             | `e1` に続けて `e2`                                           |
| `e1 e2 { code }`, `e1 e2 &{ code }` | アクション、または述語アクション                             |
| `name:e`                            | `e`、値を `name` に束縛する                                  |
| `&e`, `!e`                          | 空文字列。`e` を先読みして成功するか (`&`)、失敗するか (`!`) |
| `e*`, `e+`, `e?`                    | `e` の0回以上、1回以上、あるいは optional                    |
| `(e)`                               | `e`                                                          |
| `<e>`                               | `e` をトークンとして                                         |
| `(e, sep)*`, `(e, sep)+`            | `sep` で区切られた `e` の0回以上もしくは1回以上の繰り返し    |
| `"abc"`, `'c'`                      | リテラル                                                     |
| `[a-z_]`, `[^ '"']`                 | 文字クラス                                                   |
| `.`                                 | 任意の1文字                                                  |
| `rule`, `path::to::rule`            | ルール                                                       |

# 空白およびデリミタの制御

特殊なビルトインルールとして、`whitespace`, `comment`, `spaces`, `word_char` の4つが定義されています。`spaces` は `(whitespace / comment)*` と定義され、その他はデフォルトでは

```rust
atom whitespace = [' ' '\t' '\r' '\n']
atom comment = []    // never matches: there are no comments
atom word_char = [a-zA-Z0-9_]
```

と定義されます。これらはユーザーが再定義することで上書きすることができます。

`comment` は実装したい文法に合わせて、適宜定義します。

`whitespace` は、例えば改行が重要な文法なら、改行を外した定義にするなど変更できます。

non-atom なルールに対して、prpr は `spaces` を先頭とシーケンス間に自動で挿入します。また、シーケンス間で直前の要素の最後の文字と、次の要素の最初の文字が両方とも `word_char` にマッチする場合、デリミタ制約違反としてリジェクトする処理が自動で挿入されます。これも `word_char` を上書きすることで挙動を変更できます。また、`#[boundary(e)]`、`#[no_boundary]` を付けることで、さらなる細かい制御も可能です（くわしくは[こちら](https://tanakh.github.io/prpr/guide/whitespace.html#delimiters)）。

# メモ化

PEGというのは、Packrat parser によって解析することで、最悪計算量を入力の線形時間に抑えられます。Packrat Parser とは、非常に簡単に言えば、ルールごとに入力の位置を決めたときの解析結果をメモ化すれば `O(N*|rule|)` で計算できるという話です。

しかしながら prpr は Packrat parser を僭称しておきながら、実際にはパフォーマンスのためにデフォルトではメモ化を行いません。実際の文法ではそこまでバックトラックが発生しないようになっていることも多いからです。メモ化を行うと解析結果を格納しておくテーブルが必要になるし、さらにRustだとその値が `Clone` を実装する必要もあります（大きなデータなら `Rc<T>` で包む必要もあるかもしれない）。

なので、prpr では、オプトインでルールごとにメモ化を行うかどうかを指定する設計を採用しています。

```rust
prpr::grammar! {
    pub term: i64
        = "(" t:term ")" "!" { t + 1 }
        / "(" t:term ")" { t }
        / "x" { 0 }
}
```

例えばこういう文法があったとします。これは `!` が見つからなければ毎回バックトラックが発生して、入力によっては指数関数的な計算時間がかかってしまいます。

```rust
fn main() {
    let input = format!("{}x{}", "(".repeat(1000), ")".repeat(1000));
    assert_eq!(term(&input), Ok(0));
}
```

```sh
$ cargo run -r
    Finished `release` profile [optimized] target(s) in 0.01s
     Running `target/release/prpr-test`
<hangs...>
```

`#[memo]` をルールにつけると、そのルールはメモ化されるようになります。

```rust
prpr::grammar! {
    #[memo]
    pub term: i64
        = "(" t:term ")" "!" { t + 1 }
        / "(" t:term ")" { t }
        / "x" { 0 }
}
```

```sh
$ time cargo run -r
    Finished `release` profile [optimized] target(s) in 0.01s
     Running `target/release/prpr-test`

real    0m0.020s
user    0m0.008s
sys     0m0.012s
```

では、実際どこに `#[memo]` を付ければいいのか？という話になると思いますが、prpr では計算量が爆発しそうな場所を静的に解析してレポートする機能があります。

`grammar!` の先頭に `#![lint]` を付ければ、解析が有効になって、メモ化すべき場所をエラーとしてレポートします。

```rust
prpr::grammar! {
    #![lint]

    pub term: i64
        = "(" t:term ")" "!" { t + 1 }
        / "(" t:term ")" { t }
        / "x" { 0 }
}
```

```bash
error: `term` may be parsed more than once at the same position by the alternatives of this choice
       = note: add `#[memo]` to `term`, factor out what the alternatives start with, or add `#[no_memo]` to `term` if this is intended
 --> src/main.rs:6:17
  |
6 |         / "(" t:term ")" { t }
  |                 ^^^^
```

メッセージにもある通り、`#[no_memo]` を付ければ黙らせることもできます。

lint という通り、この機能はデフォルトでオンにして、warning として出したいところではあるのですが、`proc-macro` で warning を出す機能は、例によって例のごとく[塩漬けになっている](https://github.com/rust-lang/rust/issues/54140)ので、こういう形になりました。

# エラーレポート

各ルールは `fn(&str) -> Result<T, ParseError>` の型を持つ関数として生成されます。`ParseError` は、エラーの位置と、期待されるものの集合と、実際に見つかったものの情報を含んでいます。

prpr には組み込みで rustc 風のエラーメッセージを出す機能がついています。

```rust
prpr::grammar! {
    pub list: Vec<i64> = "[" xs:(number, ",")* "]" { xs }

    #[label = "number"]
    atom number: i64 = s:([0-9]+) { #s.parse().unwrap() }
}

fn main() {
    let input = "[1, 2,]";
    match list(input) {
        Ok(_) => {}
        Err(err) => eprintln!("{}", err.render(input, "<input>")),
    };
}
```

`ParseError::render()` でそれっぽくレンダリングされます。

```sh
error: expected number, found `]`
 --> <input>:1:7
  |
1 | [1, 2,]
  |      ^
```

別のエラーレポーティングライブラリを使いたい場合は、それらと簡単に組み合わせることもできます。

[ariadne](https://crates.io/crates/ariadne) なら、

```rust
use ariadne::{Color, Label, Report, ReportKind, Source};

fn report(path: &str, src: &str, e: &prpr::ParseError) {
    let span = std::ops::Range::from(e.span());
    Report::build(ReportKind::Error, (path, span.clone()))
        .with_message(e.to_string())
        .with_label(Label::new((path, span)).with_message("here").with_color(Color::Red))
        .finish()
        .eprint((path, Source::from(src)))
        .unwrap();
}
```

```rust
Error: expected number, found `]`
   ╭─[ input.txt:1:7 ]
   │
 1 │ [1, 2,]
   │       ┬
   │       ╰── here
───╯
```

[miette](https://crates.io/crates/miette) なら、

```rust
use miette::{LabeledSpan, NamedSource, miette};

fn report(path: &str, src: &str, e: &prpr::ParseError) -> miette::Report {
    let span = std::ops::Range::from(e.span());
    miette!(labels = vec![LabeledSpan::at(span, "here")], "{e}")
        .with_source_code(NamedSource::new(path, src.to_string()))
}
```

```sh
  × expected number, found `]`
   ╭─[input.txt:1:7]
 1 │ [1, 2,]
   ·       ┬
   ·       ╰── here
   ╰────
```

こんな感じで出せます。詳しくは[こちら](https://tanakh.github.io/prpr/guide/errors.html#displaying-errors)。

# スタンドアロン実行

`prpr-cli` を用いると、prpr の文法定義ファイルから `.rs` ファイルを生成できます。

まずは `prpr` コマンドをインストールします。

```sh
$ cargo install prpr-cli
```

`prpr gen` で `.peg` ファイル（`prpr::grammar!` の内容を単独のファイルに書いたもの）から Rust のコードが生成されます。

```sh
$ prpr gen syntax.peg -o syntax.rs
```

このファイルは `prpr-runtime` クレートに依存するコードになります。

`--inline-runtime` を付けると、ランタイムがバンドルされた依存クレートなしのコードになります。

```sh
$ prpr gen syntax.peg --inline-runtime -o syntax.rs
```

複数の文法ファイルから複数のモジュールを生成して、リンクしたいような場合、それぞれにランタイムが入ってしまうと困るので、そういうのを回避するような使い方もできます。

詳しくは[こちら](https://tanakh.github.io/prpr/codegen/cli.html)を参照してください。

# prpr を用いた Procedural Macro の定義

proc-macro は楽しいけど作るのちょっと面倒ですよね。prpr を用いてさくっと作りましょう。

```rust
prpr::grammar! {
    // 文法を定義
    pub expr: Expr = ...
}

fn expand(src: &SourceText, e: Expr) -> Result<TokenStream, MacroError> {
    // ASTからRustのコードを生成
    ...
}

// これらを組み合わせてマクロにする
prpr::define_proc_macro!(calc, expr, expand);

// 👆 define_proc_macro! はこう展開される
// #[proc_macro]
// pub fn example(input: proc_macro::TokenStream) -> proc_macro::TokenStream {
//     SourceText::with(input.into(), |src| {
//         let expr = expr(src.text())?;
//         expand(src, expr)
//     })
//     .into()
// }
```

`SourceText` という `TokenStream` から元の位置を保持したテキストを生成して、それを prpr でパーズして、結果をRustコードに変換という形です。エラーの場合は、prpr の `ParseError` の位置情報から元のマクロ呼び出しの位置情報を復元して、自動的に適切な位置にエラーメッセージを表示させることができます。`prpr::grammar!` 自身も、同じ仕組みを用いていい感じにエラーメッセージを出しています。詳しくは[こちら](https://tanakh.github.io/prpr/codegen/proc-macros.html)を参照してください。

# ベンチマーク

JSONのパーザーで、[rust-peg](https://crates.io/crates/peg)、[pest](https://crates.io/crates/pest)、[nom](https://crates.io/crates/nom)、[winnow](https://crates.io/crates/winnow) との比較を行いました。

実際のコードは[こちら](https://github.com/tanakh/prpr/tree/master/benchmarks)。

prpr では、このようなコードでさくっと実装されています。

```rust
#[derive(Clone, Debug, PartialEq)]
pub enum Value {
    Null,
    Bool(bool),
    Number(f64),
    String(String),
    Array(Vec<Value>),
    Object(Vec<(String, Value)>),
}

prpr::grammar! {
    pub json: Value
        = object
        / array
        / s:string { Value::String(s) }
        / n:number { Value::Number(n) }
        / "true" { Value::Bool(true) }
        / "false" { Value::Bool(false) }
        / "null" { Value::Null }

    object: Value = "{" kvs:(member, ",")* "}" { Value::Object(kvs) }
    member: (String, Value) = k:string ":" v:json { (k, v) }
    array: Value = "[" es:(json, ",")* "]" { Value::Array(es) }

    atom string: String = '"' s:chunk* '"' { unescape(#s) }
    atom chunk = [^ '"' '\\' '\0'-'\x1F']+ / "\\" (['"' '\\' '/' b f n r t] / "u" hex hex hex hex)
    atom hex = [0-9a-fA-F]

    atom number: f64 = s:("-"? ("0" / [1-9] [0-9]*) ("." [0-9]+)? ([eE] [+-]? [0-9]+)?) {
        #s.parse().unwrap()
    }
}
```

結果は以下の通り。実行環境は手元の Ryzen 9950X + WSL2 です。


| document                   |    prpr |     peg |     pest |      nom |   winnow | serde_json |
| -------------------------- | ------: | ------: | -------: | -------: | -------: | ---------: |
| canada.json (2.2 MB)       | 8.01 ms | 8.86 ms | 35.67 ms | 12.75 ms | 14.73 ms |    4.64 ms |
| citm_catalog.json (1.7 MB) | 2.37 ms | 2.95 ms | 28.63 ms |  4.69 ms |  6.05 ms |    2.69 ms |
| twitter.json (0.6 MB)      | 1.42 ms | 1.62 ms | 14.06 ms |  2.24 ms |  3.00 ms |    1.29 ms |

もうちょっと複雑な文法の、GraphQLでも既存のライブラリと比較しました。prpr の実装は[こちら](https://github.com/tanakh/prpr/blob/master/benchmarks/src/graphql.peg)。

作っている構文木が違うので、完全にフェアな比較になっているかは疑問が残るかもしれませんが、なるべく他のライブラリと機能的に似たような感じになるようにしているはずです（apollo-parser は空白やコメントも保持しているのでちょっと仕事量が多いですが）。

| document                |    prpr | graphql-parser | async-graphql-parser | apollo-parser | cynic-parser |
| ----------------------- | ------: | -------------: | -------------------: | ------------: | -----------: |
| github.graphql (1.6 MB) | 5.49 ms |       10.95 ms |             57.27 ms |      13.35 ms |      8.14 ms |

いずれも割と良い性能が出ていると思います。

# まとめ

構文解析っていうのはプログラム書いていれば当たり前に必要になるものですけど、どうにもしっくりくるものがないなあと思って幾星霜、ようやく自分の考えるそれなりにしっくりくるようなものがリリースできました。まだ足りていない機能などもいろいろあると思いますが、皆さんぜひ使ってみてください。そして感想や機能の要望などをいただけると幸いです。ここでは軽い紹介なので細かい部分まで説明はできていませんが、[Book](https://tanakh.github.io/prpr/) を読んでいただければ一通りのことは理解できるようになっていると思います。ライブラリ名がなんかやだとかそういう意見は受け付けておりません。
