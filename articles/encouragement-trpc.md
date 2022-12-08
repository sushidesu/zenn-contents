---
title: "tRPCを導入したら個人開発がめちゃくちゃ捗った話"
emoji: "🦑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["trpc", "typescript", "zod", "swr", "個人開発"]
published: true
---

この記事は 「[個人開発Advent Calendar 2022](https://qiita.com/advent-calendar/2022/individual-developers)」 8日目の記事です。

## はじめに

https://twitter.com/splarate/status/1592819779882156032?s=20&t=vPwLumuT_MAqBtK43JMWjA

先日リリースされた、[Splarate](https://www.splarate.app/)というWebサービスの開発をお手伝いしています。そこで導入した[tRPC](https://trpc.io/)が驚くほど便利だったので、実際の体験を交えてその使いやすさを紹介します。

## tRPCとは？

> tRPC allows you to easily build & consume fully typesafe APIs without schemas or code generation.
>
> https://trpc.io/docs/

tRPCは、スキーマやコード生成なしで型安全なAPIを簡単に構築し、呼び出すことのできるライブラリです。

`tRPC is for full-stack TypeScript developers.` と謳われているように、TypeScriptに特化して作られており、TypeScript環境であれば大きな導入の手間なく使えるようになっています。

## なぜ導入したのか

Splarateは2人体制での開発でした。元々、バックエンドをGo言語で書こうとしていたこともあり、REST + APIスキーマの共有のために言語を問わず使用できる[OpenAPI](https://www.openapis.org/)を導入していました。しかし、以下のような問題がありました。

- スキーマを書くのが大変
- スキーマを書くのが大変
- スキーマを書くのが大変
- スキーマが守られている保証がない
  - (これについては、使い方が間違っていたと思います)

どの問題もOpenAPIに慣れていない、使い方が間違っているということに起因するものであり、OpenAPIは正しく使用すればとても良い技術です。しかしOpenAPIは正しく使うえるよう環境を整えるための手間が多く、2人体制の個人開発にはオーバースペックだと感じました。現状の、「なんちゃってOpenAPI」では、スキーマと実装が乖離しているのではないか？という不安が常にあり、安心して開発できない状態でした。

そんなとき、ちょうどtRPCが話題になっており、直感的にtRPCは今抱えてくれている問題を解決してくれそうだと感じました。すでにRESTでいくつかのAPIを開発していましたがそれを一旦止め、tRPCを導入してフルTypeScriptで書き換える方向に舵を切ることにしました。(本音: かっこよさそうだったので使ってみたかった)

![9/12にsushiが「tRPCを使います」と発言しているチャットのスクリーンショット](/images/i-will-use-trpc.png)
*決意の瞬間*

はじめの導入こそつまずきましたが、とてもTypeScriptと相性がよく、TypeScriptに慣れている筆者としてはかなり直感的に使うことができました。APIとクライアントのつなぎこみやリクエストのバリデーションにかかる労力が最小限になり、重要な機能の開発に注力できたため、無事リリースまでたどり着くことができました。導入してよかったです。

## よかったところ

たくさんあるのですが、実際に使用して特にによかった感じるところを3つ紹介します。

### 型安全性 (zodによるバリデーション)

これが一番大きいです。サーバーのコードを書くと、コード生成などのステップを挟むことなくリクエスト/レスポンスの型定義、クライアントのコードにまで型推論が行われます。クライアントとサーバーの間で型を直接共有できるので、「スムーズに実装できて快適/型情報が信頼できるので安心」です。

リクエストの定義に[zod](https://github.com/colinhacks/zod)[^1]を使用することも、とても体験が良いです。zodはTypeScriptによるスキーマの定義と、それをもとにバリデーションができるライブラリなのですが、tRPCではそのスキーマを使用してリクエストを定義します。型が一致しているかはもちろん、詳細なバリデーションもスキーマファーストで行えるため、バリデーションを忘れていないか？変な値がリクエストされてないか？と不安にならずにすみます。

とにかくわかりやすく、書きやすかったです。

```ts
trpc
  .router()
  .query("hello", {
    input: z.object({
      name: z.string().min(1).max(30),
    }),
    resolve: async ({ input }) => {
      // `input.name` は 1文字以上30文字以内のstring であることが保証されている！
      return `hello: ${input.name} !!`
    }
  })
```

### シリアライズ/デシリアライズ

一般的に、多くのWeb APIからのレスポンスはJSONで返されることが多いですが、JSONは `Date` や `Map` などのJavaScriptの組み込みオブジェクトや、 `undefined` を使用することができません。しかしtRPCはそのような値をシリアライズするための[Data Transformers](https://trpc.io/docs/data-transformers)という機能があるので、その設定さえしてしまえば特に意識することなく `Date` も `undefined` も使用できます。

個人的には、コード内に `null` と `undefined` が混在していると辛いため `undefined` に寄せたいと考えており、それを簡単に行えるのはとてもありがたかったです。

補足ですが、何も設定しなかった場合の[デフォルトのtransformer](https://github.com/trpc/trpc/blob/63daaab739e0eb2d330a977c50d823a66dd2038f/packages/server/src/transformer.ts#L61-L65)は何もしてくれず、さらにv10.2.0未満では[型推論にそれは反映されない](https://github.com/trpc/trpc/issues/1694)ため注意が必要です。実装と実際のレスポンスの型に乖離が発生してしまいます。[v10.2.0からは、transformerが設定されない場合の型も正しく推論されるようになった](https://github.com/trpc/trpc/pull/3261)ため、安心です :+1:

### インクリメンタルな開発

少人数で行う個人開発では、先にある程度スキーマを固めてから機能を開発するより、機能ごとに少しずつ開発していく方がやりやすかったです。

スキーマ駆動開発の、事前にインターフェースを決めて合意するみたいなフェーズは少人数の個人開発だとかなり無駄が多いと感じました。共有相手が1人しかいないので、事後報告でここ変えといたよ！と伝えれば良いだけなんですよね。

開発者の人数が2人と少ないことに加え、お互いがフロントエンド/バックエンドの両方を書くのもあって、1機能をフロントエンド/バックエンドまとめて1人で実装する方が、実装待ちで手持ち無沙汰になったり、手戻りになることも少なく、効率が良かったと感じています。

## 工夫したところ

tRPCとNext.jsを組み合わせるにあたって、困ったことや工夫したことをいくつか紹介します。

### SWRと組み合わせる

SWRを使いたかったので、 [tRPCとnext.jsの組み合わせで、withTRPCを使うほどでもない時のライトに使う方法](https://zenn.dev/terrierscript/articles/2022-08-18-trpc-nextjs-without-hoc) の記事を参考に、[Vanilla client](https://trpc.io/docs/vanilla)を使用しました。

最小限の設定で済むのと、普段通りのSWRの使い方ができるためこれで十分だなあという感想です。

#### keyを共通化する

余談ですがSWRでただデータを取得したいだけのときは、 `useSWR` をラップした関数を作らずに `key` の生成部分のみを関数に切り出して使用するのが良いかな〜と思っています。

SWRのキャッシュの単位は `key` であり、データ取得を行わないページで `key` を指定してキャッシュを更新したい、などのユースケースも考慮すると最小単位は `key` なのではないかという考えです。それと、 `useSWR` をラップした関数をあまり作りたくないんですよね。 `useSWR` がせっかく `error` だったり `isValidating` や `mutate (KeyedMutator)` を返してくれているでそのまま使いたい。

このあたり、やっている方がいたら意見を聞いてみたいです。

```tsx
// keyGenerator.ts
export type KeyGenerator<T extends string, U = undefined> = U extends undefined
  ? () => { key: T }
  : (query: U) => { key: T } & U


// getUserKey.ts
type GetUserQuery = {
  userId: string
}

const getUserKey: KeyGenerator<"getUser", GetUserQuery> = (query) => ({
  key: "getUser",
  userId: query.userId,
})

// App.tsx
const App = (): JSX.Element => {
  const user = useSWR(getUserKey(), ({ userId }) =>
    trpc.query("usersGet", { userId })
  )
  return (
    <div>
      {user.error && <p>{user.error}</p>}
      <p>{user.data.name}</p>
    </div>
  )
}
```

### 型情報の共有のしすぎに注意

tRPCではクライアントから簡単にバックエンドのコードを呼び出すことができるのですが、それがあまりにも簡単すぎるため注意が必要です。サーバーのコードで `return` したデータがそのままレスポンスの型になり、クライアントに返ってくるため、知らないうちにクライアントに見せてはいけない機密情報などがレスポンスに含まれてしまう可能性があります。

Splarateでは、サーバーとやり取りする用のモデルの型を置くための `/dto` ディレクトリを切り出して、サーバーから返す値はそれを守るようにしてみました。お手軽で良かったです。

zodを使用して[^1][レスポンスのスキーマを定義することもできる](https://trpc.io/docs/output-validation)ので、この機能を使用するのも良いかもしれません。

## おわりに

手戻りも発生するため、かなり迷った結果の[tRPC](https://trpc.io/)の導入だったのですが、本当に導入してよかったな〜と思います。かなり直感的に使用でき、開発も爆速になるのでぜひ使ってみてください。そして[Splarate](https://www.splarate.app/)もよろしくお願いします。

最後に、 [ChatGPT](https://openai.com/blog/chatgpt/) が考えてくれたtRPCのアスキーアートを紹介して終わります。ありがとうございました。

![chatGPTに「アスキーアートでtRPCを書いてください」と質問した結果のスクリーンショット。耳のある動物っぽいものがAAで描かれている。](/images/trpc-aa-from-chatgpt.png)
*かわいいですね*

[^1]: zodに限らず、yup、superstructなども使用可能です。 (https://trpc.io/docs/procedures#input-validation)
