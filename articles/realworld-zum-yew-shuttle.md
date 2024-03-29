---
title: "YewとaxumとShuttleで RealWorld example を書いてみた"
emoji: "😺"
type: "tech"
topics: ["Rust", "Yew", "axum", "shuttle", "realworld"]
published: true
---

フルスタックエンジニアになるために普段触らないWebフロントエンドの練習をしようと思い、[RealWorld example apps](https://github.com/gothinkster/realworld) というMediumクローンのような現実のWebアプリに即したサンプルアプリを実装してみました。

https://github.com/tanakh/axum-yew-shuttle-realworld-example-app

デモはここで動いています。

https://realworld-axum-yew-shuttle.shuttleapp.rs/

昨今はWebAssemblyのおかげで、WebフロントエンドもJavaScriptやJavaScriptへのトランスパイラ言語を用いなくても問題なくかけるようになってきている「はず」なので、バックエンド含めてすべてRustで実装することにしました。それと、最近微妙に気になっていた[shuttle](https://www.shuttle.rs/)というRustのコードを無料でお手軽にホスティングしてくれるサービスを使ってみることにしました。

# アプリ仕様

詳しくは[ここ](https://realworld-docs.netlify.app/docs/intro)にある通りなのですが、

https://realworld-docs.netlify.app/docs/intro

簡単にまとめると、

* ユーザー登録、情報取得、情報変更

* JWTによるトークンベースのユーザー認証

* ユーザーのフォロー、フォロー解除

* 記事の作成、取得、変更、削除

* 記事へのタグ付け

* 記事のファボ、ファボ削除、記事のファボ数の取得

* 記事へのコメントの追加、記事へのコメントの取得、コメントの削除

* 全記事、フォローしているユーザーの記事、特定のタグがついている記事、自分の記事、自分がフォローしているユーザーの記事のリストそれぞれによるフィード閲覧機能、およびページネーション

が実装できていれば良いようです。CSSや各ページのテンプレートは用意されています。

* ユーザーの削除

* 記事のタグの変更

* コメントの編集

などはあってしかるべきな気がしましたが、バックエンドAPIにそういう機能がなかったので、実装しなくてよさそうでした。まあそもそもの仕様自体が結構フワっとしていて、GitHubのIssueなんかでいろいろ新バージョンの議論があったりしたので、あまり細かいことは気にしなくていいのかなと思いました。

# バックエンド実装

というわけで、まずはバックエンドから実装していきます。shuttleでは[いろいろなHTTPバックエンドフレームワーク](https://docs.shuttle.rs/introduction/what-is-shuttle)

> - First-class support for popular Rust frameworks - Rocket, Axum, Tide, Poem and Tower

が利用可能になっているようですが、使ったことがあるactix-webはasyncランタイムのかみ合わせのせいか、まだサポートされていないようでした。なので今回は[axum](https://github.com/tokio-rs/axum)を使うことにしました。

shuttleでaxumを使う方法は、公式ドキュメントにそれなりに書いてあります。

https://docs.shuttle.rs/examples/axum

基本的にはコードはライブラリとして用意して、`#[shuttle_service::main]`をつけた関数をshuttleのランタイムがいい感じに呼び出してくれるようです。

```rust
use axum::{routing::get, Router};
use sync_wrapper::SyncWrapper;

async fn hello_world() -> &'static str {
    "Hello, world!"
}

#[shuttle_service::main]
async fn axum() -> shuttle_service::ShuttleAxum {
    let router = Router::new().route("/hello", get(hello_world));
    let sync_wrapper = SyncWrapper::new(router);

    Ok(sync_wrapper)
}

```

データベースなどにアクセスしたい場合は、

```rust
#[shuttle_service::main]
async fn axum(#[shuttle_aws_rds::Postgres] pool: PgPool) -> ShuttleAxum {
    ...
}
```

のように引数に必要とするリソースを書くと、自動で必要なものがプロビジョニングされて、何も考えなくてもプログラムから使える状態にしてくれるみたいです。Infrastructure-From-Codeと呼ばれるパラダイムのようです。あんまり良く知らなかったのですが、別途インフラ設定のコードを書くのに比べると、リソースの定義がそれを必要とするコードに直接書かれているし、コンパイラのチェックも入れられるので、たしかに筋が良いような感じを受けました。

shuttle依存の部分はここだけなので、あとはaxumを用いた普通のREST APIのコードを書いていくだけです。

axumはRustのデファクト的なasyncランタイムであるところのtokioのチームによって開発されているフレームワークらしく、ハンドラーをproc-macroを使わずプレーンな関数として記述するのでテストがしやすかったり、各ハンドラーが必要とするリソースがちゃんとセットされているかどうかなんかをTraitを巧みに用いて静的にチェックできるようになっていたりして、また、[tower](https://github.com/tower-rs/tower)というネットワークサーバー・クライアントを構築するためのモジュラーなコンポーネント上に構築されているのHTTPミドルウェア[tower-http](https://github.com/tower-rs/tower-http)で記述したサービスをサーブする形の、tokioランタイムを用いたWebサーバー実装という形になっているので、tower-httpのミドルウェア（compressionとか静的ディレクトリサーブとか）を組み合わせることもできて、そういうruntime-agnosticなミドルウェアと直行性があるクリーンでモジュラリティーな設計として評価も高いようです。

個人的にはルーティングにproc-macroを使うのは単純に楽だしそれは別にいいんじゃないかとか思ったり、型エラーが出た時に何が原因でコンパイルが通らないのか正直全然わからなかったりして、良し悪しな部分もあるのかなと思ったりもしましたが、わりと理想的な抽象化がなされているのはすごいと思うところでもあり、（たぶん）そういったコンパイルがなんで通らないのか全然わからない問題のためにか[axum-macros](https://docs.rs/axum-macros/latest/axum_macros/attr.debug_handler.html)のようなエラーメッセージ改善のためのproc-macroがあったりして、まあそういうアプローチもあるかとか思ったりもしました。

## データベース

さて、データを保存するためにデータベースを作ることになりますが、今回はPostgreSQLでやることにしました。shuttleがサポートしている選択肢としてはPostgreSQLとMySQLとMariaDBがあるようですが、正直疎いのでよくわかりません。今回のような要件だとどれでもこなせるだろうということで適当に選びました。RDBじゃない選択肢もある気がしましたが、ここ10年ぐらいで一周して結局RDBでいいやんという流れになってきている気もします。結構いろんなクエリーも要求されますし、習作だし極端なスケーラビリティーも考えなくていいですし。

で、RDBを使うとなると次はORMを使うか、プレーンなSQLで実装するかという選択肢があるわけですが、Rustのデファクト的なものだと、前者は[Diesel](https://diesel.rs/)、後者は[SQLx](https://github.com/launchbadge/sqlx)になるようです。Dieselは以前使ったことがあったので、今回はSQLxを使ってみることにしました。SQLxの特徴としては、コンパイル時に指定したSQLサーバーからテーブル情報を取得したりクエリ文字列の構文チェックしたりRustのstructとして正しくフェッチできるのかどうかをチェックしたりしてくれるのがあります。

https://github.com/tanakh/axum-yew-shuttle-realworld-example-app/blob/2083ffe59b5d580115abfd13483b3f742a3fabf5/backend/src/api.rs#L211-L226

とまあこんな感じで、使うバックエンドSQLサーバーの構文にマッチしているか、フェッチした結果を構造体として取得したい時にちゃんとフィールドが揃っているか、型が間違っていないかを全部チェックしてくれます。

このあたりを全部ランタイムで行うオプションも提供されてはいますが、せっかくなのでマクロで静的にチェックするようにしました。SQLのデバッグも結構めんどくさかったりするので、コンパイルが通れば少なくとも正しくクエリが実行されることは保証されるのは結構楽になったと思います。一方で、コンパイル時にテーブルが存在するDBへの接続が必要になるので、コンパイル環境の用意が多少面倒になります。しかしこれは、オフラインモードという、ある環境で事前コンパイルして、メタデータを生成してポータブルにコンパイルできるようにできるツールも用意されていましたし、基本的にはポータブルにメリットを享受できる形にはなっていると思います。今回のケースでも、shuttleではデプロイの際にそういったDBへの接続が用意できるとは限らないリモートでパッケージをコンパイルすることになるので、この機能は重要でした。

## 認証周り

今回のアプリでは、サードパーティーの認証サービスではなく、自前でアカウント登録と認証を実装しないといけないようなので、そこら辺もどうやるか決めないといけません。パスワード周りは、まさか平文で保存するわけにはいかないので、何らかのソルト付きハッシュを選択します。[password_hash](https://github.com/RustCrypto/traits/tree/master/password-hash)というクレートを用いて、[Argon2](https://ja.wikipedia.org/wiki/Argon2)でハッシュすることにしました。基本的にパスワードハッシュ関数は、ブルートフォースアタックを防ぐために重い処理になっていたり、並列数を制限するためにメモリを食うようになっていたりしますが、Argon2は比較的新しくてその辺のバランスが良いようです。実際scryptと適当に比較してみたところ、CPU時間はかなり少なく済むようでした。その分メモリを使うようになっているようですが、数十MB程度のようなので十分許容できるものでした。

パスワードが認証できればJWTトークンを発行してユーザーに渡すわけですが、これはRSAの秘密鍵で署名したJSONデータなので、サーバー側でSecretデータを持つことになります。shuttleでは[SecretStore](https://docs.shuttle.rs/resources/shuttle-secrets)としてこれをリソースとしてプロビジョニングしてくれるプラグインがあったので、これを使いました。

## API実装

というわけ方向性が決まればAPIを作っていきます。

https://realworld-docs.netlify.app/docs/specs/backend-specs/endpoints

ここに書いてあるエンドポイントを一つ一つ実装していきます。使うテクノロジーを決めたら、あとはシコシコ実装していくだけ･･･ですが、SQLに慣れていないので、あれをやるにはどうするのか、というのにいちいち引っかかって個人的には結構大変でした。Google検索でやりたいことを検索すると、マイルドに言えば玉石混交な記事が大量に出てくるので、それも微妙に辛かった点ではあります。

## テスト

https://github.com/gothinkster/realworld/tree/main/api

公式にバックエンドAPIのテストが用意されています。ですが、ほんの最低限のテストしか入っていないので、割とガバガバでも通ってしまいます。もうちょっと網羅的なテストが用意されていてくれたら楽だったのですが、そういうのを作るところから練習なのかもしれません。

# フロントエンド実装

RustのWebフロントエンドフレームワークもたくさんあって、生まれては消えてしてたりしているので、どれを選ぶかも結構悩ましい問題です。ここのFrontendのところにそれなりにちゃんと開発されているものが並んでいます。

https://www.arewewebyet.org/topics/frameworks/

今回は[Yew](https://yew.rs/)を選びました。YewはReact風のHTMLのDSLとカスタムコンポーネントをhookで更新していくといったもので、Reactになじみのある人ならとっつきやすいかもしれません。まあ、私はReactになじみがあるわけではないのですが。

ほかにも同様にReact系？のSycamore、Dioxus、Elmインスパイア系のSeedや、icedなんかがあるようです。Seedのメンテナは今はMoonZoonというフルスタックフレームワークを作っているようですが、これはまだ成熟していないように見えました。

一通り触ってみて、個人的にはYewが一番普通に使えるかなという印象を受けました。

```rust
use yew::prelude::*;

#[function_component]
fn App() -> Html {
    html! {
        <h1>{ "Hello World" }</h1>
    }
}

fn main() {
    yew::Renderer::<App>::new().render();
}
```

`#[function_component]` でアノテートされた関数がコンポーネントになって、`html`マクロで描画するHTMLを返します。

```rust
#[derive(Properties, PartialEq)]
struct FooProps {
    messages: Vec<String>,
}

#[function_component]
fn Foo(props: &FooProps) -> Html {
    html! {
        <ul>
        { for props.messages.iter().map(|s| html! {
            <li>{s}</li>
        }) }
        </ul>
    }
}

#[function_component]
fn App() -> Html {
    let messages = vec![
        "Hello".to_string(),
        "World!".to_string(),
    ];

    html! {
        <h1>{ "Foo" }</h1>
        <Foo messages={messages} />
    }
}

```

定義したコンポーネントは`html`マクロから通常のタグのように呼び出せます。プロパティーを渡すこともできて、これはもちろん型チェックされます。TypeScriptによるReactの機能とかなり1対1対応しているように見えました。

静的型付けとは別に、Rust固有の煩雑な点が少し気になりました。ライフタイムの都合上、例えばコールバック関数などが参照する値はそのコンポーネントのレンダリングが終わった後も生きている必要があるのですが、例えば公式のカウンターのサンプルに、

```rust
use yew::prelude::*;

#[function_component]
fn App() -> Html {
    let counter = use_state(|| 0);
    let onclick = {
        let counter = counter.clone();
        move |_| {
            let value = *counter + 1;
            counter.set(value);
        }
    };

    html! {
        <div>
            <button {onclick}>{ "+1" }</button>
            <p>{ *counter }</p>
        </div>
    }
}
```

こういったものがあって、これは`counter`という再描画のトリガーとなるフックがあって、ボタンのクリックでそれをインクリメントする`onclick`というクロージャが呼び出されるんですが、これは`App`コンポーネントの描画終了後に呼び出されるわけなので、このクロージャのライフタイムは`'static`にならないといけないわけです。つまりそこから参照される変数はApp内のローカル変数への参照ではダメで、クロージャ内にmoveされている必要があるのです。ところがこの変数は当然`html!`の中でカウンタ値を表示する際にも参照されるので、ここで単にmoveしてはいけません。moveする前にcloneしてやる必要があるので、`onclick`の実装が次のようになっているわけです。

```rust
let onclick = {
    let counter = counter.clone();
    move |_| {
        let value = *counter + 1;
        counter.set(value);
    }
};
```

これだけならいいんですが、参照するものが増えてくると、どんどんmoveするためにcloneしてクロージャを返すコードが増えてくる。auth contextのステートやら、データフェッチのasync stateやら、フォームから値を取り出すためのノードのrefやら、そういったフックのクローンがずらりと並ぶことになる。

https://github.com/tanakh/axum-yew-shuttle-realworld-example-app/blob/2083ffe59b5d580115abfd13483b3f742a3fabf5/frontend/src/setting.rs#L36-L48

こういったコードがいたるところに発生するわけです。

このcloneしているフック自体は単なる何らかのポインタだと思うので、実行コストが云々という話ではないのですが、単純にめんどくさすぎます。GCがある言語なら何も考えなくてもいいところではありますが、move semanticsを取っているRustではそうもいかない。何かうまい書き方があるような気がしないでもないけれど、あんまり考えてても前に進まないので、結局何も考えずにcloneを書きまくることになりました。

何かおかしなことをしているのかと思って、公式のexampleを見ても同じようなことをしていたり、あるいは、Struct Componentという関数ではなくstructとしてComponentを実装するインターフェースもあるんで、そちらを使えばフックに対して関数より広い構造体スコープのライフタイムを持たせられるのでこういった記述は回避できるようですが、それはそれでコンポーネントの実装そのものがかなり煩雑になるように見えました。

とは言え、こうしてcloneしまくらなきゃいけないというならcloneを書けばいいんです。Yewだとコンポーネントのプロパティーもclone前提で、cloneコストが大きなものは`Rc`で包めという方針になっているっぽい（？）んですが、Yew以外のライブラリだと、そういったcloneを避けるために、ライフタイムを明示させてコンテクストを明示的に引数として取りまわすようにして、そのライフタイムがかみ合うことを保証させるといったつくりになっているものもあって、cloneから解放されはするかもしれないけど、それはそれでしんどそうでした。もうちょっとうまいやり方はないのか、そういう疑問からいろんなフレームワークが生まれていくんだろうなあとか思ったりしました。

## アーキテクチャ

とまあ、そういうわけでまたシコシコと一つずつページを実装していきました。全体としてはSPAとして構築して、Yewに用意されているブラウザルーターを使ってルーティングしていくことにしました。

今回のアプリの仕様では、URLとして `http://example.com/foo/bar` のような形じゃなく、`http://example.com/#/foo/bar` のようにハッシュ以降にパスを置くようにせよとこのことだったので、yew-routerに用意されている[HashRouter](https://docs.rs/yew-router/latest/yew_router/router/struct.HashRouter.html)を用いました。ドキュメントには、できる限り普通の形式のルーターを利用してこれは最後の手段として使うようにと書いてありましたが、しょうがないのでこれを使いましたが、とりあえずは動いているように見えました。ハッシュルーターを使うのは、サーバーのルーティングを簡略化したいというのがあるんでしょうか？別にそんな面倒ではないはずですが。

SPAでWebアプリを作るとなると、フロントエンドからバックエンドAPIを叩いて表示するデータを取得する必要がありますが、Web APIの直接的なバインディングである[web_sys](https://rustwasm.github.io/wasm-bindgen/api/web_sys/)のAPIを直接叩くのは少々面倒ということで、そういうAPIに対するハイレベルなライブラリである[gloo_net](https://docs.rs/gloo-net/latest/gloo_net/)を使いました。JSオブジェクトへの変換などが隠蔽されていて、serdeのインターフェースを用いた自動的なシリアライズ/デシリアライズが行われて、Rustの世界で完結するようになっていてこれはかなり楽でした。

SSRとhydrationの仕組みも用意されているようなので、それを使えばレスポンスを良くすることもできますが、今回は習作なのでそこまではやらないことにしました。

## テストデータ

Realworldにはテストデータが用意されていないみたいで、いろいろ探しても見つからなかったので、そこを用意するのも練習かと思って適当に生成するものを作りました。[fake](https://github.com/cksac/fake-rs)というのが使いやすそうだったのでこれで実装しました。

```rust
#[derive(Debug, Dummy)]
struct User {
    #[dummy(faker = "Name()")]
    name: String,
    #[dummy(faker = "FreeEmail()")]
    email: String,
    #[dummy(faker = "Password(8..16)")]
    password: String,
    #[dummy(faker = "Sentence(5..11)")]
    bio: String,
}

let users = fake::vec![User; USER_NUM];
```
こんな感じでderiveするだけでダミーデータを作れます。人名やメールアドレスやパスワードやセンテンスやらいろいろ用意されているので、大抵はプリセットで何とかなる気がします。

# デプロイ

アプリを実装して、ローカルで動かすには `cargo shuttle run` としてやればローカルにサーバーが立ってshuttleアプリを実行できます。フロントエンドはあらかじめ [trunk](https://trunkrs.dev/) でコンパイルしておきます。Rust向けのWebフロントエンドビルダーはいろいろありますが、最近はtrunkが勢いがある感じなんですかね。

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Conduit</title>
    <link href="//code.ionicframework.com/ionicons/2.0.1/css/ionicons.min.css" rel="stylesheet" type="text/css">
    <link
        href="//fonts.googleapis.com/css?family=Titillium+Web:700|Source+Serif+Pro:400,700|Merriweather+Sans:400,700|Source+Sans+Pro:400,300,600,700,300italic,400italic,600italic,700italic"
        rel="stylesheet" type="text/css">
    <link rel="stylesheet" href="//demo.productionready.io/main.css">

    <link data-trunk rel="rust" data-wasm-opt="z" />
</head>
</html>
```

こんな感じのbodyが空の`index.html`を用意すれば、あとは`trunk serve`コマンドを実行するだけでwasmにコンパイルしてサーバーが立って変更をウォッチして自動的に再コンパイルしてくれる感じで、使用感はかなり良かったです。よくある感じの、scaffoldして初見ではよくわからないスケルトンと戯れるなんてこともなく、普通に`cargo new`したプロジェクトで問題なく使えます。

設定は`index.html`のヘッダーに`<link>`タグに書いていく感じになっています。sassなんかを指定しておくとこれも自動でコンパイルしてくれます。

```rust
<link data-trunk rel="rust" data-wasm-opt="z" />
```

のように、wasm生成時のコンパイラオプションやwasmオプティマイザの引数なんかもここで渡す形になっています。

別途設定ファイルを用意すればバックエンドAPIサーバーへのプロキシを書くこともできて、特に機能的に不満は感じませんでした。とにかく使い始めるのに必要な知識や設定が少ないのが良かったです。

そんなこんなでローカルで満足に動くようになったら、今度はいよいよshuttleのサーバーにデプロイすることになります。基本的にはバックエンドサーバーのディレクトリで、

```sh
$ cargo shuttle project new
$ cargo shuttle deploy
```

でデプロイできます。デプロイできるはずなんですが、結構はまりどころは多かったです。

まず第一に、デプロイコマンドが結構失敗します。これは何回もやればうまく行ったりうまく行かなかったりするので、まあ何とかなるところではあります。そもそもビルドサービスが起動しなかったり、かと思ったらコンパイル結果が中途半端にキャッシュされたりして修正がうまく反映されなかったりしてよくわからないバグの原因になったので、毎回 `cargo shuttle project rm` でプロジェクト削除から作り直しでやるのが無難かなと思いました。

次に困ったのが、shuttleのデプロイは手元のコードをアーカイブして、リモートのコンテナで`cargo shuttle run`しているっぽいのですが、その際に`.gitignore`を参照して無視するファイルを決めて、かつ`target/`ディレクトリを強制無視して、かつシークレットを指定するファイルを強制インクルードするような結構アドホックなコードになっていたので、なんかファイルがなくて失敗するぞみたいな現象の追及が結構面倒でした。オープンソースなのでコードを読めばわかるとはいえ、コードを読まないと利用に支障が出るのは辛いところです。実際に困ったのは、フロントエンドのビルド済みHTMLファイルとwasmファイルをshuttleにサーブさせるところなワケですが、こういったバイナリは普通は`.gitignore`に指定してレポジトリに入れないようにするはずなので、見事にこの現象にハマるわけです。Issueのようなものは立っていたので、そのうち何とかなるのかもしれません。

･･･と思っていたら、数日前にリリースされた最新版で[Static Folder](https://docs.shuttle.rs/resources/shuttle-static-folder)をデプロイする機能が追加されていて、そういったものでその辺に対処していく方向なのかなと思いましたが、これ自体はフロントエンドのビルドとはあんまり関係ないですし、アーカイブのファイル除外設定もいじれるようになっているわけではないので、今のところは結局wasmなどはレポジトリに含めてしまうしかないようです。

加えて、`shuttle_static_folder`を使った静的フォルダサーブが、手元では動くのにデプロイしたらファイルが読めないとかでうまく動きませんでした。新しい機能とはいえ、そこをテストしてないはずはないので、何かおかしいことをしているのか、それとも本当にバグっているのかはわかりませんが、とにかく動かないのは仕方がないので、フォルダ自体はアーカイブされているようなので、この機能を使わずに普通にtower-httpの静的フォルダミドルウェアを使えば動きました。

最後に一番困ったのが、shuttleのサーバーによるビルドが`rust-toolchain`ファイルを参照してしまうようで、これが自分のクレートだけじゃなくて、依存するすべてのクレートのものを見てしまうようです。そういったクレートが別々のrust-toolchainに従う形でコンパイルされてしまって、特にproc-macro周りでバージョンがかみ合わずにコンパイルが通らなくなってしまいました。手元では怒らない現象なので、どうしてそんなことになってしまっているのかはわからないですが、これもIssueには上がっていて特定のクレートを使うと変なバージョンの処理系が使われてエラーになるみたいな感じで報告されていました。これは原因がわかってもこちらでとれる対策がないので、`rust-toolchain`を含むライブラリへの依存を捨てるという他ありませんでした。幸い代替が見つからないようなものはなかったので何とかなりましたが、選択肢が少なくなりますし、どうしても使えないと困るものが使えないと困るので、いずれは何とかなってほしいところです。

そんなこんなで、いくつか引っかかったところはありましたが、一つずつ対処していったら、後はうまく動いてくれました。

# 所感

というわけで、現実に即したWebアプリをテーマにしたアプリのデモをバックエンドとフロントエンド含めて書いてみました。いくつか感想を書いていこうと思います。

## バックエンド

RustによるWebサービスのバックエンドREST APIの実装はかなり成熟していると感じました。パフォーマンス、機能、使いやすさ、安定性どれをとっても不満はなかったです。JSONのシリアライズ、デシリアライズはserde_jsonで問題なく行えますし、Webサーバーフレームワークも、十分に検証されたasyncライブラリの上に、これまた十分に検証されたHTTPライブラリを組み合わせる形で、非常にモジュラーに構成されています。今回はaxumを使いましたが、使いやすさやシンプルさなど各々方向性の違う多くのフレームワークが堅牢なライブラリ群の上に作られており、いろいろな選択肢から選べるのもRustエコシステムの良いところだと思いました。

## フロントエンド

フロントエンドフレームワークは群雄割拠という感じですが、wasm自体がまだまだフロンティアといった感じで、新しいのが生まれては消えてという雰囲気なのもさもありなんというところでしょうか。基本的にはすでに成功した他の言語のフレームワークのパラダイムをなぞっているものが多く、特にReactに慣れている人なら比較的すんなりと使えるものが多いのではないでしょうか。今回は一番ユーザーの多そうなYewを使いましたが、SSRやhydrationの機能は割と最近追加されたもののようで、またコンポーネントライブラリなどのエコシステムのラインナップもまだまだといった感じで、成長の余地はまだまだありそうでした。そもそもWebフロントエンド業界ではRustの存在感自体が今のところそんなにあるわけではないわけですし。とはいえ、Yew自体はよくできており、動作におかしな点も全然なかったので、もっと発展していけば面白くなるかなと思いました。

気になるのがバイナリのサイズでしょうか。[ここにあるような](https://rustwasm.github.io/docs/book/reference/code-size.html)最適化をしてdead code eliminationした後のサイズが900KB程度で、gzip圧縮された転送サイズが300KBぐらいのようでした。JavaScript基準では大きく見えますが、wasmはパーズや実行が速いので、そこまでの問題にはならないのかなという気がしました。回線が細いとダウンロードに時間がかかってしまって、コンテンツが描画されるまでちょっとラグができてしまうので、SSRという選択肢を考えるのもいいのかもしれません。

## shuttle

今回気になって使ってみたshuttleですが、今のところ生煮え感は否めません。でもまだalphaということなので仕方がないのかもしれません。Rustで書いたWebサービスをInfrastructure-from-codeでコマンド一つでデプロイで基本無料というコンセプトは非常に攻めていて興味深いものなので、もっと成熟すれば面白い選択肢になってくる気がします。ビルド周りのトラブルが多かったですが、公式のブログに[次世代の展望](https://www.shuttle.rs/blog/2022/10/21/shuttle-next)について記事が書かれていて、将来的にはwasmから各種DBなどへのブリッジを用意して、バイナリポン置きでサクっとデプロイみたいな方向性になるかもしれないので、ビルドに関してはそれで解決しようということなのかもしれません。非常に夢がある話ですが、現時点ではまだ形が見えてないような感じなので、期待して待ちたいと思います。
