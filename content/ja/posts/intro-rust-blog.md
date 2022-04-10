---
type: Archive
title: Rustを学ぶ際に見るべき海外サイトをまとめる
date: 2022-04-10
description: Rust勉強しながら英語もできたらいいよねシリーズ
titleWrap: wrap
tags: 
- rust
image: images/summary/rust.png
---

日本語の公式ドキュメントもあって充実してはいるものの、Rustのイディオムっぽい書き方やキレイなコードの書き方、アンチパターン等のTipsは自分で書いて学んでいくよりも、積極的に先人の知恵を借りたいところ。

2022年4月時点の私個人の感覚では、ブログやサイトは海外の方が圧倒的に多く、日本人が日本語のみで得られる知見はかなり限定されてしまう。
そこで、私が最近情報収集として見るようにしているブログを紹介する。


# fasterthanli.me
海外では壮絶な読者を抱えるブログ。ポストされた記事のうち、多くがRustに関する記事となっている。
サンプルコードも充実していて、要所でキャラクターがポイントとなるコメントを入れてくれる。
1記事あたりの文章量もかなり多い(1時間かけても読み終わらないものもある)のだが、ポップな語りや説明が多いのでとても読みやすい。
Rustのコアな部分を対象とした記事から、かなり初心者向けの内容もあるので、初級者から上級者までお勧めできるブログ。
![](/fasterthanli.png)

{{< blogcard url="https://fasterthanli.me/" >}}  


# Baby steps
著者がRustの開発チームに所属しているだけあって、Rustブログの中ではかなりの古参。2011年ごろからブログを掲載している。1記事当たりの文章量はfasterthanli.meに比べるとかなりコンパクトで、「非同期処理例外ではPanicにすべき?非同期キャンセル(async cancellation)すべき?」などのように実践的な内容を取り扱うことが多い。基礎を抑えたうえで、Rustをもっと深く知りたい！ベストプラクティスやTipsを学びたい！という人におススメ。
![](2022-04-10-21-58-38.png)


{{< blogcard url="https://smallcultfollowing.com/babysteps/" >}}  


# Chris Biscardi's Digital Garden
Rust以外にもAws, js等の記事も取り扱っている。そして今回紹介するブログの中で一番UIがキレイ。記事自体もBaby stepsやfasterthanli.meに比べてとっつきやすいものが多い。「for loopとiteratorどっちがよいか？」等。英語の文章量も少ないので先述のブログで心が折れかけたときはこのブログで元気を取り戻す。
![](2022-04-10-21-59-18.png)
{{< blogcard url="https://www.christopherbiscardi.com/garden/" >}}  


# lpalmieri.com
ルーカス・パルミエーリという界隈では結構有名な人らしく、RustのPodcastといえば！の「Rustacean Studio」や「Building with Rust」にもゲスト出演をしていた。(家事をしながら聞いていた時に偶然発見した)
それ故、実力者であるからかブログはかなり濃ゆい。かなり厳密に書いてあるので、日々の情報収集として読むにはハードルが高いが[How To Write A REST Client In Rust](https://www.lpalmieri.com/posts/how-to-write-a-rest-client-in-rust-with-reqwest-and-wiremock/)のような自分が実際に経験したことのある内容や、苦労した領域に関する部分については一読すべし！


{{< blogcard url="https://podcasts.google.com/feed/aHR0cHM6Ly9ydXN0YWNlYW4tc3RhdGlvbi5vcmcvcG9kY2FzdC5yc3M/episode/cnVzdGFjZWFuLXN0YXRpb24vZXBpc29kZS8wMzYtbHVjYS1wYWxtaWVyaS8" >}} 
![](2022-04-10-21-57-28.png)

{{< blogcard url="https://www.lpalmieri.com/" >}}  


# 最後に
私の場合は、Feedly使ってTech系のブログをまとめて見るようにしています。(他によく見るのは[DEVCommunity](https://dev.to/)など)
海外のサイトを追っかけていると日本ではまだ出ていないような情報が当たり前にフィードに出てくるので、そのあたりに日本と海外の情報が回る早さの差を感じました。  


日本語で勉強していても難しいRustを英語で勉強するのはそりゃ難しいよね。と理解しつつもそこはあきらめずに。
技術に関する文章は繊細な部分も多く、翻訳機にかけるとニュアンスが若干異なる場合も少なくないのでなるべく原文のまま理解できるようになりたい。なんてことを考えていました。

