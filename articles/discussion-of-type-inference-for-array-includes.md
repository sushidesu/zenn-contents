---
title: "なぜ Union に含まれるかどうかを判定するのに `Array.includes()` を使用すると型エラーになるのか"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: false
---

TypeScriptで外部から受け取った値が Union Type に含まれるかどうかを検証したいとき、パッと思いつく方法として、 `Array.includes()` を使用するというものがあります。

```ts
const colors = ["red", "green", "blue"] as const;
type Color = typeof colors[number];

declare const input: string;
if (colors.includes(input)) {
  // do something
}
```

しかし、上記のコードはコンパイルエラーになります。

```console
$ tsc
articles/discussion-of-type-inference-for-array-includes.ts:9:3 - error TS2345: Argument of type 'string' is not assignable to parameter of type 'Color'.
```

なぜなら、 `Array.includes()` の型定義で `searchParams` 引数の型は `T` になっており、関係ない値 (今回は `string`)を渡せないようになっているためです。


```ts: lib.es2016.array.include.d.ts
interface ReadonlyArray<T> {
    /**
     * Determines whether an array includes a certain element, returning true or false as appropriate.
     * @param searchElement The element to search for.
     * @param fromIndex The position in this array at which to begin searching for searchElement.
     */
    includes(searchElement: T, fromIndex?: number): boolean;
}
```

この型エラーを回避しつつ絞り込むためにはいくつか方法があります[^1]。一番手軽なのは一度キャストする方法でしょうか[^1]。

```ts
if (colors.includes(input as Color)) {
  // do something
}
```

---

さて、ここで以下の疑問が生まれます。

型定義のオーバーライドやキャストを使用せずに安全に絞り込むことはできないのか？ `Array<T>.includes(searchParams: T)` の `searchParams` は `T` ではなく `string` や `unknown` ではだめなのか？すでに Union Types を満たすことが判明している値をincludesに渡してもあまり意味はないのではないか？

:::message
`searchParams: unknown` ではなく `searchParams: string` も良さそうなのですが、 `Array<0 | 1 | 2>` などの `string` 以外の Union Types も考慮に入れると `string` とは限らずやはり `unknown` のほうが取り回し良さそうに見えます[^2]。
:::

しかし先程確認したように、TypeScriptの定義はそうなっていません。ここにはそれなりの理由があるのであろうと推測し、なぜ、このような定義になっているのか？なぜ、`T` 以外の型を受け付けないのか？をIssueを辿って調べました。

## なぜ、このような型定義になっているのか？

### 結論

- 本来、 `searchParams` は `T` の super type (`T | super T`) を受け付けるべきだが、この記法は実装されていない
- 実装されるまでの選択肢として、以下がある
  - `T` にする (現状)
  - `any (unknown)` にする
- `any` にした場合、意図しない使用用途に警告を出せなくなるので、良くない
  - `["foo"].includes(0)` を許可してしまう
- anyにした場合、ユーザーはオーバーロードによって他方を選択できない
  - >If we made the opposite decision and said anything goes, then no one would be able to opt in to the "subtypes only" behavior because there is no mechanism for removing an overload. (該当issue)
- よってより良い `T` を選択している

### 関連するIssue

この件に関連するissueはかなりの量があります。

それらの大部分のissueから参照されているissueは `#26255` です。主にこのIssueを中心に見ていくと良さそうです。

### [`Array.includes` type is too narrow · Issue #26255 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/issues/26255)

コメントが盛り上がっておりややこしいのですが、 `Array.includes` が受け付ける `searchParams` の型については大きく2つの論点があるようです。

1. `searchParams` は super type であるべき
1. `searchParams` は なんでも受け付けるべき

#### 1. `searchParams` は super type であるべき

該当Issueの大本の提案の趣旨です。

super typeは 〜〜〜 と呼ばれるもので、〇〇です。

こちらは前向きに検討されており、[`Array.includes` type is too narrow · Issue #26255 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/issues/26255) で現在検討中のようです。

ということで該当Issueは役目を終えてすぐに閉じられているのですが、もう一つの観点のコメントで盛り上がっています。

#### 2. `searchParams` は なんでも受け付けるべき

そもそもなぜ、 `Array<T>.includes` の searchParams が `T` に制限されているのか？という趣旨のコメントです。これが有用なパターンはあるのか？大抵のユースケースでは有用でななく、むしろ union の絞り込みというユースケースにおいてキャストを強いることになるため、余計なのではないか？という主張が複数ユーザーによりなされています。

- https://github.com/microsoft/TypeScript/issues/26255#issuecomment-458013731
- https://github.com/microsoft/TypeScript/issues/26255#issuecomment-479537297

こちらに関しては現在のところ取り入れる予定はないようです。

- ECMAScriptの仕様で認められているので許可すべき、は誤り
  - stack overflow
- TypeScriptの価値の一つに警告がある
  - 制限を緩めることは、価値の低下につながる
  - 意図しない使用方法で警告を出せなくなる
    - `[0, 1, 2].includes("four")`
  - 煩わしい場合、JSという便利なものがあります :) らしいw
- `T` はオプトイン可能だが、`any` はオプトインができない

ということで、本来は `T | super T` にしたいが、現状は警告の価値と柔軟性から `T` に制限されているようでした。

## 補足: なぜ `Array.includes()` した後の値は絞り込まれないのか？

該当Issueには関連する議論として `Array.includes()` した後の値は絞り込まれないのか？という議論があります。

こういうのです。

こちらは [Suggestion: one-sided or fine-grained type guards · Issue #15048 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/issues/15048) にて検討されているようです。

## おわりに

歴史的経緯とかかな〜と思いきや、慎重に検討された上で最善な仕様が選択されていることがわかりました。議論はあまり議論は進んでいないように見えますが、いつか議論がまとまると嬉しいですね。

それにしても、いくつも同じissueが立っていたり、同じIssue内で説明されているのにも関わらず何度も同じ趣旨のコメントが発生していて、これに対応するのは骨が折れそうです。（GitHubのコメントって議論に向かないのでは :rolling_eyes:）

とはいえ、気になっていたことがわかって嬉しいし、技術論議が激しく交わされているのでIssueを読むのは楽しいです。Issueを読もう！
