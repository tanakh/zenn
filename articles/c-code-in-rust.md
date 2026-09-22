---
title: "C言語のコードをRustの中に書くライブラリを作った"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "C"]
published: false
---

C言語のコードをRustの中に書けるようにするライブラリ [cinrs](https://github.com/tanakh/cinrs) を作りました。

https://github.com/tanakh/cinrs

こんな風にCのコードをRustの中に書いて、Rustから呼び出したりできます。

```rust
use cinrs::c99;

c99! {
    int fact(int n) {
        return n == 0 ? 1 : n * fact(n - 1);
    }
}

println!("fact(10) = {}", unsafe { fact(10) }); // fact(10) = 3628800
```

ここでは `fact` という関数が定義され、Rustから呼べる状態になります。

# 概要

マクロ内で定義されたCの関数は `core::ffi` の対応する型にマッピングされ、`unsafe` としてマークされます。

```rust
pub unsafe extern "C" fn fact(mut n: ::core::ffi::c_int) -> ::core::ffi::c_int
```

基本的に `core::ffi` の型を参照するので、`no_std` 環境でも問題なく動作します。可変長配列（詳しくは後述）と `_Thread_local` の使用時のみ `std` を必要とします。

C言語の各標準に対して、`c89!`, `c99!`, `c11!`, `c17!`, `c23!` のマクロが定義されていて、それぞれ対応する標準規格でコンパイルされます。それに加えて、`gnu89!`, `gnu99!`, `gnu11!`, `gnu17!`, `gnu23!` のマクロも定義されており、これはそれぞれの規格に加えてGNU C拡張が有効になった状態でコンパイルされます。


`fact` のような、計算のみを行うような安全な関数は `__attribute__((cinrs_safe))` 、あるいは、C23形式ののattribute `[[cinrs_safe]]`でアノテートすることで、safeであると推論させることができます。

```rust
use cinrs::{c23, c99};

c99! {
    __attribute__((cinrs_safe)) int safe_fact(int n) {
        return n == 0 ? 1 : n * safe_fact(n - 1);
    }
}

c23! {
    [[cinrs_safe]] int safe_fact2(int n) {
        return n == 0 ? 1 : n * safe_fact2(n - 1);
    }
}

println!("fact(10) = {}", safe_fact(10));
println!("fact(10) = {}", safe_fact2(10));
```

safetyのチェックはRustの型システムによって行われ、safeであると推論できなかった場合はコンパイルエラーになります。自動でsafeであると推論できる関数から`unsafe`を外すこともできるかもしれませんが、safeであることをアノテートした関数以外は`unsafe`関数として定義されます。これは、関数のシグネチャに関してはnominalであるのが好ましいであろうという設計判断です。

`struct` も定義できます。`#[repr(C)}` のレイアウトで自然な形でRustの型として定義されます。

```rust
c99! {
    typedef struct { double x, y; } Vec2;

    __attribute__((cinrs_safe)) double dot(Vec2 a, Vec2 b) {
        return a.x * b.x + a.y * b.y;
    }
}

let d = dot(Vec2 { x: 1.0, y: 2.0 }, Vec2 { x: 3.0, y: 4.0 });
println!("dot = {d}");
```

`#include` などのプリプロセッサも利用できます。Cの標準ヘッダーは `cinrs` にバンドルされて提供されます。

```rust
c99! {
    #include <stdio.h>

    void greet(const char *name) {
        printf("hello, %s\n", name);
    }
}

unsafe { greet(c"cinrs".as_ptr()) };
```

もちろん、システムのヘッダーを参照することも可能です。

```rust
c99! {
    // デフォルトではシステムパスの検索がオフになっているので有効にする
    #pragma cinrs system_include

    #include <sys/stat.h>

    long file_size(const char *path) {
        struct stat st;
        return stat(path, &st) == 0 ? (long) st.st_size : -1;
    }
}

let size = unsafe { file_size(c"Cargo.toml".as_ptr()) };
println!("file size = {size}");
```

Cコード内にエラーがある場合、きちんと正確な場所がコンパイルエラーとして報告されます。

```rust
c99! {
    int add(int a, int b) {
        return a + ;
    }
}
```

```sh
error: expected expression, found ';'
 --> src/main.rs:5:20
  |
5 |         return a + ;
  |                    ^
```

Visual Studio Code の rust-analyer 拡張でも正確な位置にエラーメッセージが表示されます。

![alt text](/images/cinrs-compile-error.png)

Rustのproc-macroの形を取っているため、C言語のトークンであるが、Rustのトークンではないようなものを含むコードは、そのままではマクロ中に記述できません。

```rust
c99! {
    #include <wchar.h>
    const wchar_t *wchar_lit() { return L"ab"; }
}
```

```sh
error: prefix `L` is unknown
 --> src/main.rs:5:41
  |
5 |     const wchar_t *wchar_lit() { return L"ab"; }
  |                                         ^ unknown prefix
  |
  = note: prefixed identifiers and literals are reserved since Rust 2021
help: consider inserting whitespace here
  |
5 |     const wchar_t *wchar_lit() { return L "ab"; }
  |                                          +
```

その場合は、Cコードを文字列として渡すことで対応できます。

```rust
c99! { r#"
    #include <wchar.h>
    const wchar_t *wchar_lit() { return L"ab"; }
"# }
```

ただ、文字列として渡した場合は、細かいエラー箇所をエラーメッセージとして報告することが不可能になります（Rustのエラーメッセージとしていい感じにビジュアライズはされませんが、一応位置は表示されます）。

```rust
c99! { r#"
    int add(int a, int b) {
        return a + ;
    }
"# }
```

```sh
error: expected expression, found ';' (at line 3, column 20 of the C source)
 --> src/main.rs:3:8
  |
3 |   c99! { r#"
  |  ________^
4 | |     int add(int a, int b) {
5 | |         return a + ;
6 | |     }
7 | | "# }
  | |__^
```

![alt text](/images/cinrs-str-error.png)

これは、proc-macroにトークンの部分スパンを通知させる方法がないことから来る制限ですが、nightlyには `proc_macro_span` というフィーチャーがあって、その機能を使えばうまくやれるので、`cinrs` の `nightly` フィーチャーを有効にしたうえで、nightly toolchainを用いてコンパイルすれば、的確なエラーメッセージを表示できます。

```sh
$ cargo +nightly build 
error: expected expression, found ';'
 --> src/main.rs:5:20
  |
5 |         return a + ;
  |                    ^
```

https://github.com/rust-lang/rust/issues/54725

これがデフォルトで使えるようになるとだいぶ使い勝手が良くなると思うので、この機能が一刻も早くstableになるように皆さんも一緒にお祈り下さい。

また、`include_c99!` など、ファイルパスを指定してファイルをCコードとして取り込むマクロも用意されています（Rustの `std::include!` マクロのC版のような雰囲気）。

```rust
include_c99!("foo.c");
```

単純に、指定したファイルの中身を `c99!` などのマクロに文字列として渡した時と同じ挙動になります。

# 仕組み

各マクロ呼び出しはC言語の "翻訳単位(translation unit)" としてコンパイルされます。なので、同じプログラム内にいくつマクロ呼び出しを書いても大丈夫です。各マクロ呼び出しでは proc-macro に渡されたトークン列を文字列化して、これをプリプロセス、パーズ、意味解析を行ったうえで、同じセマンティクスるになるRustコードを生成します。ですので、一見Cのコードに見えるコードが埋め込まれますが、コンパイルされるのは正真正銘のRustのコードになります。このマクロで囲むだけでRustへの移植完了です。お疲れさまでした。

なので、`cinrs` には完全なCのコンパイラが実装されています。C言語処理系への依存もありませんし、C言語処理系を実装するライブラリへの依存もありません。スタンドアロンなC言語処理系ということができると思います。

生成されるRustのコードは基本的には [`c2rust`](https://github.com/immunant/c2rust) に似たようなものだと思います。ただ、`cinrs` では生成されるコードを観察したり、手で修正したりすることを想定はしていません。[`cargo-expand`](https://github.com/dtolnay/cargo-expand) などを用いて変換されたRustコードを確認することはできますが、可読性に配慮しているといったことはありません（`c2rust`もそこらへんに配慮はされていませんが）。

マクロとしての使い勝手を自然にするために、proc-macro に渡されたトークンの位置情報からソースマップを構築して、エラーレポートの際には内部のエラー位置から元コードの位置に復元させることで、正確なエラー位置の報告を実現しています。これはRust 1.88あたりでstableになった `proc_macro::Span` 系のAPIによって可能になっています。割と最近の話なんですよね。

# 言語標準のサポート状況

詳細は https://github.com/tanakh/cinrs/blob/master/doc/c-status.md ここにあります。

基本的にはC99～C23のほとんどの機能が正しく実装されており、known-bugはほとんどないと思います。Rustにコンパイルする上で困難なものが残されているといった感じです。

未対応機能としては、

* `setjmp` / `longjmp`
  * 関数を呼ぶだけならできるけど･･･
* `_BitInt`, `_Imaginary`, 非プリミティブな `_Atomic`
* `alloca` および可変長配列 (variable length array, VLA)のスタック上での確保
  * VLAはスタックではなくヒープ上での確保になります（Rustはそもそもコンパイル時にスタックサイズが決まらないような状況を安全とみなしていない気がするので無理だと思われる）
* 可変長引数関数の定義
  * プロトタイプ宣言はできる
  * 定義自体もサポートはしているが、必要な機能がコンパイラのstableにない
  * https://github.com/rust-lang/rust/pull/155697
  * 現在betaチャンネル。なんと次バージョンでstable化予定
  * 2026年10月1日を待て
* `va_arg` から `struct` を取り出すようなコード
  * 部分的（16バイト未満のもの）についてはサポート
  * https://doc.rust-lang.org/std/ffi/trait.VaArgSafe.html このあたりでプリミティブ型しかサポートされていない
  * 各種ABIとメモリレイアウトを考慮しつつフルサポートするのはかなり大変だと思われる
* インラインアセンブリ
* intrinsic

などがあります。LinuxがRustをサポートするにつれて、このあたりのCとの相互運用機能が最近充実してきている感はあるので、もしかしたらこれらもstableなRustで無理なく可能になっていくかもしれません。

# ベンチマーク

詳しい値はこちら https://github.com/tanakh/cinrs/blob/master/doc/benchmarks.md にあります。

![alt text](/images/cinrs-benchmarks.png)

ベンチマークスイートは [The Computer Language Benchmaks Game](https://benchmarksgame-team.pages.debian.net/benchmarksgame/index.html) から標準的な機能で書かれたものをいくつか、古典的なベンチマークをいくつか、それと自前のマイクロベンチマークからなります。ベンチマーク全体の Geometric mean は、cinrs / gcc で 0.927、cinrs / clang で 1.071 となっており、Cコンパイラと比べてもほぼ遜色のない、あるいはむしろ速いものもあるという結果になりました。これはかなり良い結果だと思います。

statemachine というベンチが顕著に遅いですが、これは `goto` を自由に使うようなコードで、Rustには `goto` がないので、これを実現するために、cinrs ではそのようなコードはステートマシンとしてコンパイルしています。なので、冗長なコードになり、コンパイラの最適化も効きづらくなっていると思われますが、それでも2倍強の実行時間で収まっています。なお、後方への `goto` に限っては、Rustでもラベル付き `loop` の `break` という形で比較的自然に、かつ最適化の効きやすい形で記述できるので、（実際にはほかのベンチでも`goto`は使われているものの）重いペナルティーとなっているものは少ないです。

コンパイル時間に関しては、今回のベンチマークはどれも大きなコードではないので問題ない範囲だと思います。おおよそ gcc/clang と比べて2倍程度に収まっていると思います。C言語からRustへのコンパイルと、その生成されたRustコードのコンパイルを合わせてこの程度なので、オーバーヘッドとしては自然だと思われます。

# 似たプロジェクトとの違い

## inline-c

https://crates.io/crates/inline-c

似たようなシンタックスでC/C++のコードをRustに書くことができるライブラリですが、こちらはCコードを一時ファイルに書き出して、Cコンパイラでコンパイルするというもののようです。また、埋め込んだCコードをRustから呼び出したりすることはできず、Rustプログラムで生成されたC APIをテストするという用途が主目的のようです。

## cpp

https://crates.io/crates/cpp

こちらはC++コードをコード片として記述して、Rustと値のやり取りもできるライブラリのようです。`build.rs` でプリプロセスしつつ、コード片をC++コンパイラでコンパイルして、実行時にはオブジェクトファイルをリンクして実行する形のようです。任意のコードを記述するというよりは、グルーコードを書いてC++で書かれたコードをRustから呼び出すことを目的としていそうです。[`cxx`](https://crates.io/crates/cxx)よりも緩い使い方ができるのがウリなのかな。

## c2rust

https://github.com/immunant/c2rust

CのコードをRustに変換するスタンドアプリのプログラムです。CのプログラムをRustに移植する際によく使われているイメージです。c2rustが生成するコードは非常にナイーブなコードですが、これをインクリメンタルにRustっぽいコードに修正していくというのが一般的でしょうか。ポン出しでRust移植したと言い張っているケースもあるような気がします。

# こんなの何に使うんですか？

何に使えばいいんでしょうか。面白いアイデアがあれば是非試してみてください。

## CプログラムのインクリメンタルなRust移植

マクロが生成するのは正真正銘のRustコードですので、cinrsのマクロ内にCのコードを書けばそれでRust移植完了です。また、そこからCのコードをインクリメンタルにRust側に書き換えていくことで、インクリメンタルに安全なコードにしていくという使い方もできるかもしれません。

## 部分的にプログラムをCで書きたい

どうしてもRustでうまく書けない部分があったとき、unsafe RustのシンタックスシュガーとしてC言語でコードを書けると嬉しいこともあるのかもしれないし、ないかもしれません。

https://docs.rust-embedded.org/book/interoperability/c-with-rust.html

こういったことをやりたい場合に、別ファイルやビルドスクリプトを作らずに、シンプルにできるかもしれません。

## [bindgen](https://crates.io/crates/bindgen) の代わりに

RustからCのライブラリを呼び出す際に、ヘッダファイルからRustのFFI呼び出しを生成してくれる bindgen というツールがあります。

bindgenはlibclangが必要だし、`build.rs` を書かないといけない。`build.rs` が生成した `.rs` ファイルをincludeしないといけない。でも、cinrsならこれだけで済みます。

```rust
use cinrs::c99;
use std::ffi::CStr;

mod z {
    c99! {
        #pragma cinrs system_include
        #pragma cinrs link "z"
        #include <zlib.h>
    }
}

fn main() {
    let version = unsafe { CStr::from_ptr(z::zlibVersion()) };
    println!("zlib version: {}", version.to_str().unwrap());
}
```

```sh
$ cargo run -q
zlib version: 1.3.1
```

使いたいライブラリのbinding(*-sysクレートとか)がない。そういう時に、お手軽にCのAPIを呼び出したい。そんな時に。

## [bitfield](https://crates.io/crates/bitfield) などの代わりに

Rustでもビットフィールドが使いたくなる時がたまにあります。そういったことをする[既存のcrateがたくさん](https://crates.io/search?q=bitfield)ありますが、`cinrs` でもビットフィールドをRustの型として定義できます。しかもCと同じ構文で、Cと同じメモリレイアウトで。これが牛刀というやつですね。

```rust
use cinrs::c99;

c99! {
    struct RiscvRtype {
        int opcode: 7;
        int rd: 5;
        int funct3: 3;
        int rs1: 5;
        int rs2: 5;
        int funct7: 7;
    };
}

impl Default for RiscvRtype {
    fn default() -> Self {
        unsafe { std::mem::zeroed() }
    }
}

impl From<u32> for RiscvRtype {
    fn from(value: u32) -> Self {
        unsafe { std::mem::transmute(value) }
    }
}

impl std::fmt::Display for RiscvRtype {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "RiscvRtype {{ opcode: {:#x}, rd: {}, funct3: {}, rs1: {}, rs2: {}, funct7: {:#x} }}", self.opcode(), self.rd(), self.funct3(), self.rs1(), self.rs2(), self.funct7())
    }
}

fn main() {
    println!("Size: {}", std::mem::size_of::<RiscvRtype>());
    let inst = RiscvRtype::from(0x00C58533); // add x10, x11, x12
    println!("{inst}");
}
```

```sh
$ cargo run -q
Size: 4
RiscvRtype { opcode: 0x33, rd: 10, funct3: 0, rs1: 11, rs2: 12, funct7: 0x0 }
```

## C言語のコードの検証

cinrsで生成されるコードはRustなので、`miri` で検証できます。

```rust
use cinrs::c99;

c99! {
    #include <stdlib.h>

    int use_after_free() {
        int* p = (int*)malloc(sizeof(int));
        *p = 42;
        free(p);
        return *p;
    }
}

fn main() {
    let result = unsafe { use_after_free() };
    println!("{}", result);
}
```

簡単な use-after-free の例ですが、これを `miri` で実行してみます。

```sh
$ cargo +nightly miri run -q
error: Undefined Behavior: memory access failed: alloc177 has been freed, so this pointer is dangling
  --> src/main.rs:10:16
   |
10 |         return *p;
   |                ^^ Undefined Behavior occurred here
   |
   = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
   = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
help: alloc177 was allocated here:
  --> src/main.rs:7:24
   |
 7 |         int* p = (int*)malloc(sizeof(int));
   |                        ^^^^^^
help: alloc177 was deallocated here:
  --> src/main.rs:9:9
   |
 9 |         free(p);
   |         ^^^^
   = note: stack backtrace:
           0: __cinrs_unit_657556fd::use_after_free
               at src/main.rs:10:16: 10:18
           1: main
               at src/main.rs:15:27: 15:43

note: some details are omitted, run with `MIRIFLAGS=-Zmiri-backtrace=full` for a verbose backtrace

error: aborting due to 1 previous error
```

という風に、しっかりエラーとして検出されて、Cコードの位置をわかりやすく出してくれます。

まあ、C言語でもアドレスサニタイザーとかを使えば、この類のエラーは検出してくれるのですが。

:::details asan result
```sh
❯ gcc use-after-free.c -fsanitize=address && ./a.out 
=================================================================
==3272533==ERROR: AddressSanitizer: heap-use-after-free on address 0x77980e9e0010 at pc 0x6449e86e22e7 bp 0x7ffc80c86fb0 sp 0x7ffc80c86fa0
READ of size 4 at 0x77980e9e0010 thread T0
    #0 0x6449e86e22e6 in use_after_free (/home/tanakh/tmp/cinrs-test/a.out+0x12e6) (BuildId: f8b952ac836dc529c06e62028795e3f516696b01)
    #1 0x6449e86e22ff in main (/home/tanakh/tmp/cinrs-test/a.out+0x12ff) (BuildId: f8b952ac836dc529c06e62028795e3f516696b01)
    #2 0x7b780f62a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #3 0x7b780f62a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #4 0x6449e86e2184 in _start (/home/tanakh/tmp/cinrs-test/a.out+0x1184) (BuildId: f8b952ac836dc529c06e62028795e3f516696b01)

0x77980e9e0010 is located 0 bytes inside of 4-byte region [0x77980e9e0010,0x77980e9e0014)
freed by thread T0 here:
    #0 0x7b780fb2a3ff in free ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:51
    #1 0x6449e86e22af in use_after_free (/home/tanakh/tmp/cinrs-test/a.out+0x12af) (BuildId: f8b952ac836dc529c06e62028795e3f516696b01)
    #2 0x6449e86e22ff in main (/home/tanakh/tmp/cinrs-test/a.out+0x12ff) (BuildId: f8b952ac836dc529c06e62028795e3f516696b01)
    #3 0x7b780f62a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #4 0x7b780f62a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #5 0x6449e86e2184 in _start (/home/tanakh/tmp/cinrs-test/a.out+0x1184) (BuildId: f8b952ac836dc529c06e62028795e3f516696b01)

previously allocated by thread T0 here:
    #0 0x7b780fb2b60f in malloc ../../../../src/libsanitizer/asan/asan_malloc_linux.cpp:67
    #1 0x6449e86e225e in use_after_free (/home/tanakh/tmp/cinrs-test/a.out+0x125e) (BuildId: f8b952ac836dc529c06e62028795e3f516696b01)
    #2 0x6449e86e22ff in main (/home/tanakh/tmp/cinrs-test/a.out+0x12ff) (BuildId: f8b952ac836dc529c06e62028795e3f516696b01)
    #3 0x7b780f62a600 in __libc_start_call_main ../sysdeps/nptl/libc_start_call_main.h:59
    #4 0x7b780f62a717 in __libc_start_main_impl ../csu/libc-start.c:360
    #5 0x6449e86e2184 in _start (/home/tanakh/tmp/cinrs-test/a.out+0x1184) (BuildId: f8b952ac836dc529c06e62028795e3f516696b01)

SUMMARY: AddressSanitizer: heap-use-after-free (/home/tanakh/tmp/cinrs-test/a.out+0x12e6) (BuildId: f8b952ac836dc529c06e62028795e3f516696b01) in use_after_free
Shadow bytes around the buggy address:
  0x77980e9dfd80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x77980e9dfe00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x77980e9dfe80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x77980e9dff00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x77980e9dff80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x77980e9e0000: fa fa[fd]fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x77980e9e0080: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x77980e9e0100: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x77980e9e0180: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x77980e9e0200: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x77980e9e0280: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==3272533==ABORTING
```
:::

## C言語処理系が無いが、なぜかRustは使えるときに、なんとかしてCのプログラムをコンパイルしたい

そんな時もあるかもしれない。

# あとがき

さらに詳しくは [レポジトリのドキュメント](https://github.com/tanakh/cinrs/tree/master) をご覧ください。詳しい機能の一覧や、仕様準拠度の詳細、既知の制限事項、ベンチマークコード、クロスコンパイルの扱いなどが書かれています。

こののアイデア自体はずっと前からあったのですが、stable Rustにproc-macro周りの機能が足りてなかったり、C言語との連携機能（特にva_arg周り）が足りてなかったりして、当時はあまり満足のいくものが作れない状況で（stableで動くものが望ましいし）、作りかけでずっと放置していました。それに加えて、こういう面倒くさいものを作るのも楽な時代になってきたので、今作ればいいものができるんじゃないか？というのでこの度完成させることができました。当初はC99だけサポートしようと思っていたのですが、無駄にC89～C23までのすべての規格が可能な範囲で実装されました。結果として、面白いものができたと思います。是非みなさん使ってみて、面白い使い方を発見してくださると嬉しいです。
