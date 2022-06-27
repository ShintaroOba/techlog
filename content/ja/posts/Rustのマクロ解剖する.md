---
type: Archive
title: Yewを解剖し、アトリビュートの動作を理解する
date: 2022-06-27
description: マクロ
titleWrap: wrap
tags: 
- Rust
image: images/summary/rust.png
---

Rustを業務の傍らでやんわりと勉強している中で、[Yew](https://github.com/yewstack/yew)を触る機会があった。
これはRustでReactライクなフロントエンドの実装ができるOSSなのだが非常によくできていて、ReactでいうFunctional Componentを定義するには#[functional_component]というアトリビュートを任意のメソッドに対して宣言することで、宣言されたメソッドはFunctional Componentであるとみなされる。という作りになっていた。  
一利用者として、Reactライクな作りをRustの機構で再現していること、またReactに負けず劣らずの表現が可能なことに感動した私は、どうしてもYewの中身を知ってみたくなった。  

これを機にRustのアトリビュートを中心にYewが内部でどのように動作しているのかを見ていく。

当記事で出てくるソースコードはこちらから。https://github.com/ShintaroOba/wasm-md-editor


# Rustのアトリビュートとは
[Rust-by-example](https://doc.rust-jp.rs/rust-by-example-ja/attribute.html)にはこう書かれている。
> アトリビュートはモジュール、クレート、要素に対するメタデータです。以下がその使用目的です。
> - コンパイル時の条件分岐
> - クレート名、バージョン、種類（バイナリか、ライブラリか）の設定
> - リントの無効化
> - コンパイラ付属の機能（マクロ、グロブ、インポートなど）の使用
> - 外部ライブラリへのリンク
> - ユニットテスト用の関数を明示
> - ベンチマーク用の関数を明示

部分的にlintを無効化したり(例えばDeadCodeを許容する)、環境に応じたコンパイル自の挙動制御(下記参照)など。

````rs
// 実行環境がLinuxの場合のみコンパイルされる
#[cfg(target_os="Linux")]
fn are_you_on_linux(){
    println!("You are runnnig linux!");
}

````


## つまるところ何がうれしいのか
複雑な内部動作を隠蔽し、開発者にとって可読性の高い状態で構造体やメソッドに付加情報を与えることができる。また、構造体やメソッドの単位での付加情報の一覧性にも優れる。

````rs
#[derive(Debug, Eq, PartialEq)]
pub struct Foo(i16);

#[derive(Debug, Copy, Eq, PartialEq)]
pub struct Bar(i32);

````

上記のDeriveはプレリュードで提供されるようなよく使われるトレイトを宣言的に継承させ、derive()内に記載したトレイトの振る舞意を簡単に持たせることができる。

# Yewでのアトリビュート
Yewでは#[function_component]を使ってコンポーネントの定義を行う。
ここでは、Home画面を構成するHomeコンポーネントを定義。``html!``マクロでは、インナーブロックで与えられたHTMLタグを処理し、HTMLとして返却する。

````rs
use yew::prelude::*;

#[function_component(Home)]
pub fn home() -> Html {
   
    html! {
        <h1>{"Welcome to my editor!"}</h1>
       
    }
}
````

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
- ``proc_macro_attribute``がfunction_component()メソッドがCustom Attributeであることを示している。
#
