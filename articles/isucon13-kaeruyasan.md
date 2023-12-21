---
title: "ISUCON13に参加しました（かえるやさん）"
emoji: "🐸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["isucon"]
published: false
---

ISUCON13にチーム「かえるやさん」として参加しました。 最高スコアは11,248、最終スコアは10,678で、ISUCON13 受賞チームおよび全チームスコア 全694チーム中228位でした。参加言語はGo言語で、ISUCONへの参加は2回目になります。

![チーム紹介スライド](/images/isucon13-team-introduction-kaeruyasan.png)
*かえるは売りませんでした*

## 事前準備

### 参加登録

はじめの難関として、参加登録があります。去年のISUCON12では申し込み開始から2分44秒で第一期の220枠が埋まっており、今年も激戦となることが予想されました。

https://twitter.com/isucon_official/status/1531824879560888322?s=20

以下は、早く申し込むための心構えです。

- 早く起きる
- GitHubにログインしておく
- ISUCONポータルを開いておく
- ISUCONポータルにログインしておく（画面が狭いとログインボタンが出ないので注意）
- 入力項目はあとから変更可能な項目も多いので、焦らず入力する
- チーム番号が振られていれば、参加登録完了！

準備したかいもあり、無事に15番目に参加登録することができました。

![チーム情報が記載されている画面のスクリーンショット。チーム番号15 かえるやさん と記載されている](/images/isucon13-team-number-15.png)
*チーム番号15 かえるやさん*

### 素振り

[『達人が教えるWebパフォーマンスチューニング 〜ISUCONから学ぶ高速化の実践』](https://www.amazon.co.jp/dp/4297128462) の付録Aを片手に [catatsuy/private-isu](https://github.com/catatsuy/private-isu) のパフォーマンスチューニングを行いました。

実際に手を動かしながらパフォーマンスチューニングを行うという内容で、本番当日にそのまま活かせる内容も多くとても役に立ちました。付録の途中までしか進められませんでしたが、はじめ700点くらいだったスコアが最終的に100,000点くらいまで上がって楽しかったです。

## 当日

9:30に集合してISUCON13のオンラインライブ中継を見ました。パフォーマンス改善の題材となるサービスを紹介する[出題動画](https://www.youtube.com/watch?v=OOyInZbM85k)が流れるのですが、とても良くてテンションが上りました。

![ISUCON13 出題動画のスクリーンショット](/images/isucon13-introduction.png)
*かわいい*

その後、昼ごろまでにリポジトリのセットアップを終え、少しずつ改善に手を入れていきました。他のメンバーが環境構築したりマニュアルを読んでいる傍ら、自分はMiroのメッセージボードがあったので少しでも爪痕を残そうとお絵かきをしていました。あとから配信を見返したところ映っていたいたので頑張って書いてよかったです。([5:24:27あたり](https://www.youtube.com/live/YJ1_JnuZp0U?si=fAVdi16kWV_XCAa2&t=19467))

![かえるやさんへの応援メッセージ](/images/isucon13-cheer-for-kaeruyasan.png)
*配信に映る かえるやさんへの応援メッセージ*

大きくスコアが改善した変更としてはインデックスの追加とN+1改善でした。自分はDNSの設定やNGワード周りの処理の改善を行いました。チーム全体で他には以下のことをやりました。こうして振り返るとインデックスを貼ってばかりですね。

- `10:50` リポジトリのセットアップ
- `11:01~11:37` インデックス貼る
- `11:47` 配信周りのN+1改善
- `12:12` ADMIN PREPARE 無効化
- `13:21` アイコン改善
- `14:56` 配信統計情報周りのN+1改善
- `15:10` インデックス貼る
- `15:12` TTL設定
- `15:28` インデックス貼る
- `16:45` アイコン改善
- `17:14` NGワードのN+1改善
- `17:15` インデックス貼る
- `17:55` 急いでインデックスを貼りなおす

他にもいくつかスコアが上がらなかったため取り入れなかった変更があるのですが、実装にミスがあったわけではなく、貼ったつもりのインデックスが貼られていなかっただけだったということが終盤になって発覚しました。競技終了5分前に大急ぎでインデックスを貼り直し、お祈りしながら競技終了となりました。DBの変更を反映する部分ははじめに確認しておくべきでした 😢

## 次回に向けて

何も出来ずに終わったISUCON12に比べて多少は「パフォーマンスチューニング」っぽいことが出来たかなーと思います。来年も頑張るぞ！