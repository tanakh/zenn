---
title: "自作のRust製エミュレーターをWebブラウザーで動くようにした"
emoji: "🎮"
type: "tech"
topics: ["wasm", "emulator", "rust"]
published: true
---

最近Rustで[ちょい](https://zenn.dev/tanakh/articles/gb-emulator-in-rust)[ちょい](https://zenn.dev/tanakh/articles/gba-emulator-in-rust)[書いてた](https://zenn.dev/tanakh/articles/nes-and-snes-emulator-in-rust)エミュレーターをWebブラウザーで動くようにしました。

https://meru-emu.com

現在のところ、[ゲームボーイ・ゲームボーイカラー](https://github.com/tanakh/tgbr/)、[ゲームボーイアドバンス](https://github.com/tanakh/tgba)、[ファミコン](https://github.com/tanakh/sabicom)、[スーパーファミコン](https://github.com/tanakh/super-sabicom)が動きます。

当初からブラウザーでも動かすことを念頭に置いていたので、コア開発が一段落したら対応させたいと思っていました。実際に動かすに当たっては思い通りに進んだところもあり、そうでないところもありといったところで、ところどころで気づいたところを書いておいたら誰かの参考になるかもしれないので、とりとめのない話ですが書き残しておくことにします。

一応ウェブアプリなのにJavaScriptもTypeScriptも、ReactもAngularも、npmもwebpackも使わない、インド人完全無視カレーみたいなものになりましたが、それでもそこそこに良い感じにできた気はするので、私のようにフロントエンドから取り残されてきた人にとってはおススメできるのかもしれません。

# Rustのコードをブラウザーで動かすためには

Rustは処理系がwasmの出力をサポートしていて、実際にwasmのバイナリを出力させるのもとても簡単なので、ローレベルな開発だけではなく、そういったところもある程度売りになっているプログラミング言語だと思います。実際私のエミュレーターもほとんど手を加えずにwasmにコンパイルすることができました。Rustでwasmプロジェクトをスタートする手順は https://rustwasm.github.io/book/ にまとまっていますが、wasm-packを入れろとかnpmを入れろとかテンプレからプロジェクト生成しろとか、すでにあるものを動かしたい場合にはちょっと面倒にも見えます。一昔前だとそういった環境を整えるのがちょっと面倒だった印象がありましたが、現在ではそういったものに頼らずとも簡単にwasmにコンパイルしてデプロイできるようになっているようです。既存のRustプロジェクトに対してある日突然ブラウザーで動かしてみるか、と思い立った場合でも、特に何の問題もなく簡単にサポートを追加できると思います。その辺りはゲームフレームワークBevyのブラウザーサポートに関するページ https://bevy-cheatbook.github.io/platforms/wasm.html のほうが分かりやすいかもしれません。

具体的な手順としては、

1. Rustツールチェインにwasm32サポートを追加

```sh
$ rustup target add wasm32-unknown-unknown
```

2. コンパイルする

```sh
$ cargo build --target wasm32-unknown-unknown
```

これだけです。これでwasmが生成されます。何らかの理由でどこかがコンパイルエラーになるかもしれないので、それを直せば完了です。

生成したwasmバイナリの実行ですが、[`wasm-server-runner`](https://github.com/jakobhellermann/wasm-server-runner) というのを使えば楽です。`cargo` でインストールできます。

```sh
$ cargo install wasm-server-runner
```

`wasm-server-runner` に生成されたwasmを渡せば、

3. 実行する

```sh
$ wasm-server-runner target/wasm32-unknown-unknown/release/meru.wasm    
 INFO wasm_server_runner: compressed wasm output is 6.76mb large
 INFO wasm_server_runner::server: starting webserver at http://127.0.0.1:1334
```

といった感じで、渡されたwasmファイルをロードして`main()`を呼び出すJavaScriptのコードを含んだHTMLファイルをサーブするサーバーが立ちます。これをブラウザーでアクセスすればRustプログラムが動きます。とっても簡単。

毎回生成されたバイナリのパスを渡すコマンドを実行するのは面倒なので、`.cargo/config.toml` に、

```toml:.cargo/config.toml
[target.wasm32-unknown-unknown]
runner = "wasm-server-runner"
```

を追加しておけば、

```bash
$ cargo run --target wasm32-unknown-unknown
   Compiling meru v0.2.0 (C:\Users\tanak\repos\meru)
    Finished release [optimized] target(s) in 17.70s
     Running `wasm-server-runner target\wasm32-unknown-unknown\release\meru.wasm`
 INFO wasm_server_runner: compressed wasm output is 6.76mb large
 INFO wasm_server_runner::server: starting webserver at http://127.0.0.1:1334
```

のようなな感じでコマンド一つで実行できます。開発環境はWindowsでもLinuxでも問題なく動きました。一昔前はRustでwasmを生成する時は、`Cargo.toml` の `[lib]` に `crate-type = ["cdylib"]` を付けてライブラリとして出力させて、それを読み込むHTMLとJavaScriptを書いてみたいな感じだった気がするのですが、今やとりあえず実行するならJavaScriptを1行も書かなくて良くなりました。コミュニティーの進捗には頭が下がります。最終的にWebサイトとして公開するならもちろんガワとなるHTMLを書かないといけませんが、[`wasm-bindgen`](https://rustwasm.github.io/wasm-bindgen/reference/cli.html)を使えばwasmをロードするJavaScriptの部分は自動で生成できるので、結局私が書いたJavaScriptは以下の2行だけでした。

```html
<script type="module">
    import init from './meru.js';
    await init();
</script>
```

GitHub Actionsで[wasmにコンパイルしてGitHub Pagesで公開する用のArtifactを作る](https://github.com/tanakh/meru/blob/master/.github/workflows/pages.yml)ようにすればデプロイまで自動です。最近はGitHubもブランチじゃなくてArtifactからPagesにデプロイできるようになっていたんですね。こういうWebアプリにとってはずいぶん楽になりました。

というわけで、wasmへのコンパイルでコンパイルエラーになった箇所を直してwasmバイナリを作るところまでは意外なほどすんなりいきました。ここからはすんなりいかなかったところを書いていこうと思います。

# 実行速度

Rustはあらゆるプログラミング言語の中でもトップレベルに効率の良いバイナリを生成してくれるプログラミング言語ではありますが、さすがにwasmにするとネイティブと比べて多少は遅くなります。ちゃんと計測したわけではないので正確なことは言えませんが、体感的には倍ほど遅い程度でしょうか。実装したコアの中で一番重いゲームボーイアドバンスが手元のマシン（Ryzen 3950X）でフレーム落ちなしで多少余裕を持って動いている程度の速度で、他のコアだと十分余裕をもって動いているといった感じです。なおFirefoxはChromeとEdgeに比べると明らかに遅かったので、手元のマシンではゲームボーイアドバンスはフルフレーム出ませんでした。その辺も考えると多少コアのパフォーマンスチューニングのモチベが出てきました。ただ今のところコードのシンプルさを優先して一切速度を意識したコードを書いていないので、それでこれだけ速度が出ていれば文句はないかといったところです。

ところで、ブラウザーだと開発ツールにプロファイラが組み込まれていて、ワンクリックで簡単にプロファイルが取れて、プロファイル結果を色々な形でビジュアライズできたのがすごく便利そうでした。VMだからやりやすいんだと思いますが、こんなにお手軽に使えるようになっているんですね。wasmにしても関数名などの情報はちゃんと出てくれていますね。

![](/images/gba-profile.png)

# wasmおよびブラウザーに対応していないライブラリ

wasmに対応していないライブラリを使っている場合は、当然ながら何とかして対応させるなり、使うのを諦めるなりする必要があります。当初からブラウザー対応を念頭に置いてはいたので、ライブラリ選定の時もwasm非対応のものをなるべく避けるように気を付けていました。

とはいえ、Rustで書かれたコードはほとんどは問題なくブラウザーでも動くので、大抵のライブラリは対応しているはずです。実際MERUは間接的に依存している物を含めると、360個程度のcrateを利用していることになっていますが、ほとんどは問題なくコンパイルして正しく動きました。問題となるのは主に2点です。1つはC言語のライブラリを使っているもの、もう1つはプラットフォーム固有のAPIを使っているものです。

## C言語のライブラリを使っているもの

RustではCへのFFIが比較的容易なので、多くのよく使われているCのライブラリへのバインディングが作られています。MERUではアーカイブファイルからのロードに対応するために [compress-tools](https://crates.io/crates/compress-tools) というcrateを使っています。これは [libarchive](https://www.libarchive.org/) というCのライブラリを用いて、多様なアーカイブ形式に対してシンプルで統一的なインターフェースを提供してくれるもので、ネイティブ環境で使う分には大変便利なものですが、残念ながらwasmでは使えません。

Rustはwasmにコンパイルできる。標準ライブラリのwasmサポートも最新のstdには入っている。C言語のコードもemscriptenなどを用いればCの標準ライブラリを呼んでいるようなコードもwasmにコンパイルできる。そして、RustはC言語のコードを簡単に呼び出せる。理屈としては問題なく動かせてもおかしくはないはずですが、どうやら両者が生成するインターフェースが噛み合わず、そこをRustのFFIでは現状透過的に扱えないといった感じで、理屈としてはすごく頑張れば動かせるのかもしれませんが、今回はそこを頑張りたい感じではなかったので、素直に諦めることにしました。

compress-toolsを使うのを諦めて、その代わりにzip-rsというピュアRustで書かれたzipの圧縮展開ができるライブラリがあったので、これを用いてzipだけサポートすることにしました。その他のアーカイブ形式は今回はサポートを見送りましたが、rarやtarはピュアRustのライブラリが存在しているようなので、対応しようと思えば出来そうです。7zは、lzmaのライブラリはあったものの7zのアーカイブ形式に対応するものは見つからなかったので、ちょっと大変そうです。いずれにせよ自力でlibarchiveのRust実装を作るみたいな話になってしまいますね。そういうのがあれば、他の人の役にも立つかもしれないので、作る価値はあるのかもしれません。

## プラットフォーム依存のAPIを使うもの

今回作ったものはGUIアプリなので、当然どこかでGUIのためのプラットフォーム固有のAPIが呼び出されているはずです。あるいはCで書かれたマルチプラットフォーム対応のGUIツールキットのバインディングなんかも候補に上がるかもしれませんが、それは先述の理由でブラウザー向けに動かしたい場合は採用が難しいです。Rustにはかなり多くのGUIライブラリがあって、選ぶのだけでも一苦労、というのは[前々回の記事](https://zenn.dev/tanakh/articles/gb-emulator-in-rust)にも書きましたが、数は豊富でもクオリティーは玉石混交、ある程度力を入れて開発されていたライブラリも開発終了したものも結構あり、さらにその中から手間をかけてブラウザーで動くようにJSのAPIを呼び出すバックエンドまで実装してくれているものはかなり限られていて、しかも将来的にサポートが期待できそうなものとなると実は選択肢はそんなにありません。今回はゲームエンジンとして[bevy](https://bevyengine.org/)を、メニュー画面のGUIツールキットとして[egui](https://github.com/emilk/egui)を採用しました。どちらも現時点で活発に開発されており、ブラウザー対応にも力を入れているように見えます。なので、基本的にはコンパイルするだけでネイティブと同じように動いてくれました。ファイルオープンダイアログのためにクロスプラットフォーム対応の[rfd](https://github.com/PolyMeilex/rfd)というのも使いましたが、これも主要OSおよびブラウザー向けの実装が存在していて、ちゃんと動いてくれました。ただ、ブラウザーではブロックするような呼び出しは行えないので、非同期APIのみの提供になっています。ネイティブ版ではブロッキング版を使っていたので、ここを書き換える必要がありました。

# 非同期対応

ブラウザーで動かすにあたって一番大変だったのは非同期への対応でした。ブラウザーでは（Rustのセマンティクスを満たすような、メモリを共有する）スレッドが1つしかないので、あらゆるブロッキング処理が禁止されているようです。ファイルオープンダイアログのようなユーザーの入力が終わるまで待たされるものは当然として、condvarによるウェイトも禁止されます。特に後者に関しては、stdの互換性のために関数自体はライブラリに用意されてい入るものの、ランタイムにパニックするような実装になっています。そういうのはコンパイル時点ではじいてほしいというのが心情ではありますが、並行処理を行っているライブラリでは頻出なのでそういうわけにもいかなかったのかもしれません。

特に困ったのが、async対応のために使おうと思ったasyncランタイムが、ことごとくcondverによる同期を行っていることです。Rustのasyncは、言語レベルでは非同期処理の生成を行うための構文と非同期処理を表現する型しか用意されておらず、それをどのように実行するかは別途ランタイムを利用する形になっています。言語および標準ライブラリは最低限の仕組みを用意して、それに加えて非同期処理に対する様々な操作を提供する[futures](https://rust-lang.github.io/futures-rs/)というライブラリが公式のコミュニティーから提供されていて、その上に各種ランタイムがある格好です。futures自体も一応ミニマルなランタイムを提供していますが、コミュニティーの推奨としては他の本格的なランタイムを使ってほしいようです。それで、そのランタイムのなかでメジャーなのが[tokio](https://tokio.rs/)というのと[async-std](https://async.rs/)というのの2つなんですが、どちらも非asyncからasyncを呼び出す手段がcondvarを使っています。しかし考えてもみれば、そういうことをするにはなんとかしてブロッキングしないといけないので、そもそも不可能なことなのかもしれません。何が可能で何が不可能なのか、どういう理屈でそうなっているのか、そこらへんをちゃんと把握しておく必要があったのかもしれません。

じゃあ全体をasyncにすればいいじゃんと思われるかもしれませんが、それも簡単には行きません。bevyのECSのシステムとして登録する関数はasyncに対応していないのです。そのbevyのシステムの非asyncの関数から、asyncのAPIを呼び出さないといけないという形です。そんなわけで、上からは非asyncのbevy、下からはasyncの入出力APIに挟まれ、身動きが取れません。

非async関数からasyncを呼び出さなきゃいけないなら、とりあえずspawnだけしてそれをチャンネルでどっかに飛ばして、mainをasyncにしてそこでawaitすればいいじゃんとか、どこかのasync関数で作ったExecutorをもらってそこで動かせば良いじゃんとも思いましたが、今度ははSendトレイトの制約が行く手を阻んできます。Rustではあるスレッドで作ったオブジェクトを別のスレッドに移動させても大丈夫だということを示すマーカートレイトのSendトレイトというものがあります。代表的なものだとスレッド間で移動や共有をされないことを前提に同期処理を省略している参照カウントスマートポインタの`Rc`や、生ポインタなどがあります。ブラウザー関係だと、JavaScriptのAPIが返すJsValueオブジェクトが非Sendになっています。そもそも同一スレッドで動かなきゃならない物なので、これも当然と言えば当然です。asyncランタイムでは当然タスクのスケジューリングのためにSendがついている型の方が実行の融通が付きますし、逆に非Sendのものに関しては大抵は大幅な制約が課せられています。非Sendのタスクを実行するには、典型的には登録されたタスクが同一スレッドで実行されるローカルExecutorを作って、そこに非Sendなタスクをspawnさせて、そのExecutorを実行する形になりますが、当たり前ですがローカルExecutor自体も非Sendになります。そしてブラウザーではローカルExecutorの実行でブロックすることはできないので、Executorの実行自体を非同期で行う関数を呼び出して、そのfutureをどこかでawaitするということになりますが、このfutureにはこれまた当たり前だけど非SendのオブジェクトであるローカルExecutorをコンテクストに含むことになるので非Sendがついてしまいます。なので、結局ローカルExecutorではどうにもならないということです。いやあ、Rustのこの辺の型システムは本当に腹が立つほどよくできていますね。できてはならないことが本当にできない。よくよく考えたらそうじゃん、という感じで、私自身の理解にもだいぶ役に立ちました。

と思って半ば詰んでるんじゃないかと思いながらも悶々と色々試していると、async-stdの[`async_std::task::block_on`](https://docs.rs/async-std/latest/async_std/task/fn.spawn_blocking.html)という関数がなぜかわからないけどなんとなく動くのを見つけました。なんかわからんけど解決した、と喜んでネイティブでも試してみるとハングして動かない。訝しみながらコードを見てみると、この関数かなり怪しくて、wasm版とネイティブ版で返り値の型から、タスクのトレイト制約から何から何まで違う。何から何まで違う関数なのに、同じ関数ですよと言う顔で堂々と存在している。wasm版はSendを要求しないのにネイティブ版ではちゃっかりSendを要求してくる。挙動がそもそも異なるワケです。`block_on`とかいう名前してるくせに、wasm版はブロックしないのです。タスクがどこかに投げられて、どこかのExecutorで実行されて、タスクの完了を待つすべがない。それがwasm版の挙動になります。かなり怪しいけど、他の手段も見つからないので、これをだましだまし使って何とかすることにしました。プリミティブとしてspawnするけどawaitできないものしかないことを前提に全体のコードを書き直して、最終的には何とか動くものになった気がします。

# 設定画面

それに合わせて設定画面のAPIも刷新しました。コアごとに設定項目が存在するので、それを設定するUIも定義したいんですが、前バージョンではUIを定義するシンプルなトレイトを用意して、それを使ってUI定義を書かせる感じにしていました。ところが、設定で何らかのファイルを指定したい場合に、ここにasyncが絡んでくるのでうまくいかないと思ったわけです。それに加えて、UIを定義するのがめんどくさすぎるというのがあります。よくよく考えたらJSONの設定ファイルをいじるためのGUIとか、自動で作れてもおかしくなさそうです。VSCodeの設定なんかも当初はJSON手書きでしたが、今ではGUIで設定できるようになっています。ああいうのって設定の仕様がJSON Schamaで定義されてるんだよな、そっからGUI自動生成やろうと思えばできるんじゃないかと思って色々探してみると、JavaScriptのライブラリがいろいろ見つかりました。Rustにもないかなと思って探してみましたが、さすがになさそうでした。やろうと思えばできるけどRustにはないなら自分で書くしかないか･･･ということでJSON SchemaからGUIを生成するコードを書きました。さくっと書けるかと思ったらこれが意外と大変で、もともとこれはバリデーションに使う用途なのか、同じようなことを表現するにも記述方法に冗長性がかなりあるようで、結局そこら辺を正確に解釈するのを諦めてとりあえずは限られた表現のみに対応して、のちのち充実させればいいかという感じで落ち着きました。なお、各コアが設定のJSON Schamaを出力できるようにするには、structにderiveつけるだけです。こちらは思惑通りものすごく簡単になりました。

# ファイルシステム

最後にもう一つ困ったのがファイル入出力です。これもstdに存在している標準ライブラリですが、ブラウザーでファイルシステムにアクセスすることはできないので、wasmターゲットでも関数は用意されているけど、呼ぶと死ぬタイプのやつになっています。ファイルをダイアログで選択してもらうのはブラウザーのAPIがデータごとくれるのでいいんですが、設定ファイルやセーブデータの読み書きに関しては何とかしないといけません。というわけで、ブラウザー側にミニマルなファイルシステムを作ってそこにデータを読み書きすることにしました。最近はIndexedDBというのが推奨されているらしいので、それを使いました。こういうのはウェブプログラミングをしていればよくあるニーズなんじゃないのかと思って探してみましたが、Rustのライブラリには今のところ存在しないみたいでした。JavaScriptには当然のようにそういうライブラリがあるみたいで、さすがにホームグラウンドの言語は年季が違うなあと思いました。JSON SchemaからのGUI生成もそうですが、そういった、こういうのあってもおかしくないよなというのが当然のようにあるのはうらやましい所です。と羨んでもしかたがないので、最低限のファイル読み書きとメタデータだけ実装したものを作って、全てのstd::fs周りの呼び出しをそちらを呼び出すように書き直しました。そして当然と言えば当然ですが、ファイルの読み書きがasyncになってしまうので、ファイルを読んでいる部分を非同期にして、残りの処理を分割してチャンネルでデータ受け渡しするようにして、ファイルを読んでデータを処理したいだけなのになんでこんなにコードを分割してあっちこっちで処理して苦労しないといけないのかと悶々としながらも書いていきました。bevyのシステムがasyncじゃないから仕方がない。理屈としては分かっていますけどね。

# 所感

というわけで問題を一つ一つ潰して、無事に自作のエミュレーターをブラウザーで動かすことができました。出来栄えには割と満足しています。このアプリの特筆性のある部分、つまりエミュレーターのコアの部分はすんなりと動いてくれました。とりあえずの画面と音が出るまでは本当にすぐだったので、結構感動しました。ゲームパッド対応もbevyが頑張ってくれてるのか、そのまま動いてくれてすごいと思いました。音に関しては、現代のブラウザーはユーザーの操作なしに音を出せない設定になっているらしく、オーディオストリームを起動直後に作ると死んでたりして多少の問題はありましたが、生成を遅延させればいいだけなので対応は簡単でした。それ以外の部分はそれなりに苦労しました。ブラウザーが一般的なメモリモデルを持ついわゆる普通のスレッドをサポートしてくれれば、たぶんRustのasyncランタイムもネイティブとの乖離が少なくなっていろいろ簡単にサポートできるようになるんじゃないのかなあとか思わずにいられませんが、それは難しいんでしょうね。

というわけで、ブラウザーで動くようになって試してもらうのも簡単になったと思うので、よろしければ触ってみてください。動きがおかしいなどの挙動を発見された方はバグレポートを頂けると嬉しいです。ではまた。
