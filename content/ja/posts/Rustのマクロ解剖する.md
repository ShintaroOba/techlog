---
type: Archive
title: Yewを解剖し、マクロの動作を理解する
date: 2022-06-27
description: マクロ
titleWrap: wrap
tags: 
- Rust
image: images/summary/rust.png
---

Rustを業務の傍らでやんわりと勉強している中で、[Yew](https://github.com/yewstack/yew)を触る機会があった。
これはRustでReactライクなフロントエンドの実装ができるOSSなのだが、非常によくできていて、簡単に言うとReactのような実装をRustで記述できるというもの。  
興味本位で軽く触ってみたが、Reactに負けず劣らずの表現がYewで可能なことに感動し、どうしてもYewの中身を知ってみたくなった。  

これを機にRustのマクロを中心にYewが内部でどのように動作しているのかを見ていく。

当記事で出てくるソースコードはこちらから。
[https://github.com/ShintaroOba/wasm-md-editor](https://github.com/ShintaroOba/wasm-md-editor
)


## Rustにおけるマクロとは
メタプログラミングと呼ばれており、コードをコードによって生成するための機能。Rustではこれがコンパイル時に行われる。
マクロは宣言的マクロ、手続き型マクロの2種類に分類することができ、
``macro_rules!``で記載されるマクロが宣言的マクロ、``#[some_attribute]``で記載されるものは手続き的マクロと呼ばれているが、今回メインで紹介したいYewで利用されるアトリビュートというのはこの手続き的マクロに該当する。
手続き型マクロの中には、``#[proc_macro]``、``#[proc_macro_attribute]``等があり、部分的にlintを無効化したり(例えばDeadCodeを許容する)、環境に応じたコンパイル自の挙動制御(下記参照)など。

````rs
// 実行環境がLinuxの場合のみコンパイルされる
#[cfg(target_os="Linux")]
fn are_you_on_linux(){
    println!("You are runnnig linux!");
}

````


### つまるところ何がうれしいのか
複雑な内部動作を隠蔽し、開発者にとって可読性の高い状態で構造体や関数に付加情報を与えることができる。また、構造体や関数の単位での付加情報の一覧性にも優れる。

````rs
#[derive(Debug, Eq, PartialEq)]
pub struct Foo(i16);

#[derive(Debug, Copy, Eq, PartialEq)]
pub struct Bar(i32);

````

上記のDeriveはプレリュードで提供されるようなよく使われるトレイトを宣言的に継承させ、derive()内に記載したトレイトの振るまいを持たせることができる。

## Yewでのアトリビュート
先ほど説明したマクロの中でも、foo!や#[derive(Foo)]などとは異なる、関数や構造体に対して付与するマクロのことをアトリビュートと呼ぶ。

[Rust-by-example](https://doc.rust-jp.rs/rust-by-example-ja/attribute.html)には以下のように書かれていて、定義した関数や構造体を拡張するために使われる。
> アトリビュートはモジュール、クレート、要素に対するメタデータです。以下がその使用目的です。
> - コンパイル時の条件分岐
> - クレート名、バージョン、種類（バイナリか、ライブラリか）の設定
> - リントの無効化
> - コンパイラ付属の機能（マクロ、グロブ、インポートなど）の使用
> - 外部ライブラリへのリンク
> - ユニットテスト用の関数を明示
> - ベンチマーク用の関数を明示

### #[function_component]
Yewでは#[function_component]を使った関数の定義によってコンポーネントを構築していくが、この関数に付与するマクロがアトリビュートに当たる。

````rs
use yew::prelude::*;

#[function_component(Home)]
pub fn home() -> Html {
   
    html! {
        <h1>{"Welcome to my editor!"}</h1>
       
    }
}
````

ここでは、Home画面を構成するHomeコンポーネントを定義。``html!``マクロでは、インナーブロックで与えられたHTMLタグを処理し、HTMLとして返却する。



画面ではこう表示される。(ボタンは別で配置している)

![welcome](/welcome-my-editor.png)


#[function_component]アトリビュートの中身はこれ。
````rs
#[proc_macro_attribute]
pub fn function_component(
    attr: proc_macro::TokenStream,
    item: proc_macro::TokenStream,
) -> proc_macro::TokenStream {
    let item = parse_macro_input!(item as FunctionComponent);
    let attr = parse_macro_input!(attr as FunctionComponentName);

    function_component_impl(attr, item)
        .unwrap_or_else(|err| err.to_compile_error())
        .into()
}
````
- ``proc_macro_attribute``がfunction_component()関数がCustom Attributeであることを示しているため、利用側で#[function_component]とアトリビュートを付与した際にこの関数がリンクされる。
- 手続き的マクロを定義する関数には、``TokenStream``を入力として受け取り``TokenStream``を出力として返す。
- アトリビュートを付与したソースコードが入力値としてTokenStreamに変換され、それを基にマクロが生成するソースコードがTokenStreamとして返却される。
- 引数の１つ目である``attr: proc_macro::TokenStream``は呼び出し側(``#[function_component(Home)]``)の``Home``を指しているのに対し、2つ目の``item: proc_macro::TokenStream``は``#[function_component(Home)]``を付与した関数の中身に対応している。(function_componentの例ではitemはFunctionComponent、attrはFunctionComponentNameに対応)
- ``parse_macro_input!``はTokenStreamのトークン列を構文木にパース。

一般的に、parse_macro_input!によって構文木にパースされたTokenStreamは、このあと``quote``マクロによって再度TokenStreamに変換され、マクロ呼び出し元の結果として返却される。
マクロを定義するlib.rsにはアトリビュートに関する詳細な実装を書けないため、アトリビュート本体の実装は別のクレートの関数を呼び出す形式をとることが多い。
Yewの#[functionComponent]においてもその流れは変わらず、上述の流れでパースされた構文木は``function_componnet_impl``関数内でquote!マクロが呼ばれてトークン列に変換されている。

````rs
pub fn function_component_impl(
    name: FunctionComponentName,
    component: FunctionComponent,
) -> syn::Result<TokenStream> {
    let FunctionComponentName { component_name } = name;

    // ・・・略

    let quoted = quote! {
        #[doc(hidden)]
        #[allow(non_camel_case_types)]
        #[allow(unused_parens)]
        #vis struct #function_name #impl_generics {
            _marker: ::std::marker::PhantomData<(#phantom_generics)>,
        }

        impl #impl_generics ::yew::functional::FunctionProvider for #function_name #ty_generics #where_clause {
            type TProps = #props_type;

            fn run(#arg) -> #ret_type {
                #block
            }
        }

        #(#attrs)*
        #[allow(type_alias_bounds)]
        #vis type #component_name #impl_generics = ::yew::functional::FunctionComponent<#function_name #ty_generics>;
    };

    Ok(quoted)
}

````

### html!
html!のようなマクロは手続き的マクロの一種で、関数風マクロとも呼ばれる。宣言的マクロと非常に似ているが、``macro_rules!``ではなく、``#[proc_macro]``を利用し、手続き的マクロ同様にTokenStreamを受け取って生成したTokenStreamを返す点で違いが明確にある。
html!を使ったサンプル実装を下記に示す。

````rs
use yew::prelude::*;

#[function_component(Home)]
pub fn home() -> Html {
    html! {
            <h1>{"Welcome to my editor!"}</h1>
    }
}

````

````rs
use crate::{components::home::Home, Routing};
use stylist::style;
use yew::prelude::*;
use yew_router::{history::History, hooks::use_history};

#[function_component(Top)]
pub fn top() -> Html {
    let container = style!(
        r#"
        display: flex;
        flex-direction: column;
        align-items: center;
        "#
    )
    .expect("Failed to styled.");
    let button = style!(
        r#"
        color: #ffffff;
        width: 200px;
        padding: 10px;
        background-color: #1976d2;
        box-shadow: 0 3px 5px rgba(0, 0, 0, .3);
        -webkit-box-shadow: 0 3px 5px rgba(0, 0, 0, .3);
        :hover {
            background: #115293;
            margin-top: 3px;
        }
        "#
    )
    .expect("Failed to styled.");
    let history = use_history().unwrap();
    let onclick = Callback::once(move |_| history.push(Routing::Editor));

    html! {
        <>
            <div class={container}>
                <Home />
                <button class={button} {onclick}>{"Start"}</ button>
            </div>
        </>
    }

````
- Homeコンポーネントは「Welcome to my editor!」を返すだけのシンプルなコンポーネント。
- Topコンポーネントは、Homeコンポーネントを呼び出しており、Startボタンを配置するTopページを表している。
- styleは[stylist](https://github.com/futursolo/stylist-rs)を利用しており、Reactのような宣言的なスタイルの定義が可能。

ここまでの説明でやっと冒頭の画像のコンポーネントの画像に戻る。
![welcome](/welcome-my-editor.png)

````rs
#[proc_macro_error::proc_macro_error]
#[proc_macro]
pub fn html(input: TokenStream) -> TokenStream {
    let root = parse_macro_input!(input as HtmlRootVNode);
    TokenStream::from(root.into_token_stream())
}
````