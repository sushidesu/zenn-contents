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
type Color = typeof colors[number]; // = "red" | "green" | "blue"

declare const input: string;
if (colors.includes(input)) {
  // do something
}
```

しかし、上記のコードはエラーになります。

```console
> tsc --noEmit

hello.ts:5:21 - error TS2345: Argument of type 'string' is not assignable to parameter of type '"red" | "green" | "blue"'.

5 if (colors.includes(input)) {
                      ~~~~~
```

なぜなら、 `Array.includes()` の定義で引数 `searchParams` の型は `T` になっており^[`ReadonlyArray` も同様です。]、関係ない値 (今回は `string`)を渡せないようになっているためです。


```ts: lib.es2016.array.include.d.ts
interface Array<T> {
    /**
     * Determines whether an array includes a certain element, returning true or false as appropriate.
     * @param searchElement The element to search for.
     * @param fromIndex The position in this array at which to begin searching for searchElement.
     */
    includes(searchElement: T, fromIndex?: number): boolean;
}
```

この型エラーを回避しつつ絞り込むためにはいくつか方法があります^[[TypeScript: Array.includes on narrow types](https://fettblog.eu/typescript-array-includes/)]。一番手軽なのは一度キャストする方法でしょうか。

```ts
if (colors.includes(input as Color)) {
  // do something
}
```

---

さて、ここで以下の疑問が生まれます。

キャストを使用せずに絞り込むことはできないのか？ `Array<T>.includes(searchParams: T)` の `searchParams` は `T` ではなく `unknown` ではだめなのか？すでに Union Types を満たすことが判明している値をincludesに渡す意味はあるのか？

しかし先程確認したように、TypeScriptの定義はそうなっていません。ここにはそれなりの理由があるのであろうと推測し、なぜ、このような定義になっているのかIssueを辿って調べました。

## なぜ、このような型定義になっているのか

### 結論

- 本来、 `searchParams` は `T` の supertype (`T | super T`) を受け付けるべき^[> this issue is a complaint about being unable to search for supertypes in an array (which should be valid). https://github.com/microsoft/TypeScript/issues/26255#issuecomment-733914018]だが、この記法は実装されていない
- 実装されるまでの選択肢として、以下がある
  - `T` にする (現状)
  - `unknown` にする
- `unknown` の場合、意図しない使用用途に警告を出せなくなるので、良くない
  - `["foo"].includes(0)` を許可してしまう :man_no_good:
- `unknown` の場合、ユーザーは subtype のみに制限する挙動 ( `searchParams: T` ) を選択できない
- よってより良い `T` を選択している

### 関連するIssue

この件に関連するIssueはかなりの量があります。

- [Current type for `Array.prototype.includes` may reject valid use cases · Issue #53904 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/issues/53904)
- [Array Includes incorrectly uses generic type · Issue #53275 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/issues/53275)
- [`Array.prototype.includes` should be narrow on union of enum types instead of throwing · Issue #49906 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/issues/49906)

かなり古くからあり、多くのIssueから参照されているIssueは [`#26255`](https://github.com/microsoft/TypeScript/issues/26255) です。主にこのIssueを中心に議論を追っていくのが良さそうです。

### [`Array.includes` type is too narrow · Issue #26255 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/issues/26255)

コメントが盛り上がっておりややこしいのですが、 `Array.includes` が受け付ける `searchParams` の型については大きく2つの論点があるようです。

1. `searchParams` は supertype であるべき
1. `searchParams` は なんでも受け付けるべき

#### 1. `searchParams` は supertype であるべき

該当Issueの大本の提案の趣旨です^[Issue本文には *subtype* を受け入れるべきと書かれていますが、用語の間違いで実際は *supertype* のことを言っているようです。 https://github.com/microsoft/TypeScript/issues/26255#issuecomment-411120022]。

supertypeは基本型、上位型と呼ばれ^[ https://zenn.dev/estra/articles/typescript-type-set-hierarchy]、ある型の派生元の型を表します。またsubtypeは派生型、下位型と呼ばれ、supertypeを継承しつつ拡張した型を表します。

`T extends string` と記述したときの `T` がsubtypeで、 `string` がsupertypeにあたります。一番初めに挙げた例で言うと、 `Color` は `string` のsubtypeであり、 `string` は `Color` のsupertypeです。

現在、TypeScriptに supertype を表現するための記法はありませんが、仮に `T` の supertype を `super T` と表すとすると、`Array<T>.includes` の `searchParams` は `T | super T` と表すことができます。

```ts
interface Array<T> {
  includes(searchElement: T | super T, fromIndex?: number): boolean;
}
```

`searchParams` は同じ型もしくは派生元の型 (`super T`) を受け取ることができるので、 `Array<Color>.includes` には `Color` に加えて派生元である `string` を渡すことができます。

```ts
const colors: Color[] = ["red", "blue", "green"];
declare const input: string;
colors.includes(input); // OK
```

しかし派生元ではない `number` や `boolean` は渡すことができません。

```ts
const colors: Color[] = ["red", "blue", "green"];
declare const input: string;
colors.includes(input.length); // NG
colors.includes(input === "red"); // NG
```

このようにsupertypeを使用することで、意図しない使用方法には警告を出しつつ、より柔軟な型を受け付けることができるようになります。

この仕様は前向きに検討されており、 [Enable type parameter lower-bound syntax · Issue #14520 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/issues/14520) で現在検討中のようです。

ということで当該Issueは `#14520` の重複としてすぐに閉じられている^[ https://github.com/microsoft/TypeScript/issues/26255#issuecomment-411326965]のですが、もう一つの観点のコメントで盛り上がっています。

#### 2. `searchParams` は なんでも受け付けるべき

そもそもなぜ、 `Array<T>.includes` の `searchParams` が `T` に制限されているのか？という趣旨のコメントです。

これが有用なパターンはあるのか、むしろ特定の値が配列に含まれているかを判定するユースケースにおいてキャストを強いることになり、過剰なのではないか？という主張が複数ユーザーによりなされています。^[https://github.com/microsoft/TypeScript/issues/26255#issuecomment-681313150, https://github.com/microsoft/TypeScript/issues/26255#issuecomment-479537297, https://github.com/microsoft/TypeScript/issues/26255#issuecomment-736274359]

こちらに関しては現在のところ取り入れる予定はないようです。

なぜなら何でもあり ( `unknown` ) の場合、意図しない使用方法に対して警告を出すことができず、TypeScriptの魅力の1つであるエラーを見つけ出すことができなくなってしまいます。

> Part of TypeScript's value proposition is to catch errors; failing to catch an error is a reduction in that value and is something we have to weigh carefully against "Well maybe I meant that" cases.
>
> https://github.com/microsoft/TypeScript/issues/26255#issuecomment-479557916

> TS has many checks that are based on our judgment about what's "over the line" in terms of intentional vs unintentional behavior of JavaScript.
>
> https://github.com/microsoft/TypeScript/issues/26255#issuecomment-735995564

```ts
function foo(array: string[], content: string, index: number) {
  if (array.includes(index)) { // contentではなく誤ってindexを渡してしまっている
    // Do something important
  }
}
```

上記のコードはJavaScriptの文法上間違ったコードではありません。しかし意図しない使用方法であることは明らかです。もし何でも受け付けるようにしてしまうと、このような間違いを見つけることができなくなってしまいます。

また、何でも受け付ける状態をデフォルトにしてしまうと、オーバーロードによってより厳しい型チェックの動作を選択できなくなってしまいます。

> If we made the opposite decision and said anything goes, then no one would be able to opt in to the "subtypes only" behavior because there is no mechanism for removing an overload.
> 
> https://github.com/microsoft/TypeScript/issues/26255#issuecomment-736680802

現状の宣言の場合はユーザーが `searchParams: unknown` を宣言し制限を緩めることができますが、もともと `searchParams: unknown` である場合は、ユーザーが `searchParams: T` を宣言して制限しようとしても、 TypeScriptの仕様により より一般的な型が優先されるため `unknown` が優先されてしまいます。

ということで、本来は `T | super T` にしたいが、まだ `super T` 記法は実装されておらず、残った選択肢の中でより厳しい動作である `T` に制限されているようでした。

## 補足: なぜ `Array.includes()` した後の値は絞り込まれないのか？

該当Issueには関連する議論として `Array.includes()` した後の値は絞り込まれないのか？という議論があります。

`includes()` によってある集合の要素であることを確認したのだから、型が推論されていてほしいという主張です。

```ts
const colors: Color[] = ["red", "blue", "green"];
declare const input: string;
if (colors.includes(input)) {
  const color: Color = input // input is Color！
}
```

こちらは [Suggestion: one-sided or fine-grained type guards · Issue #15048 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/issues/15048) にて検討されています。

## おわりに

歴史的経緯とかかな〜と思いきや、慎重に検討された上で最善な仕様が選択されていることがわかりました。あまり議論は進んでいないように見えますが、いつかまとまると嬉しいですね。

それにしても、いくつも同じIssueが立っていたり、すでに説明されているのにも関わらず何度も同じ趣旨のコメントが発生していて、これに対応するのは骨が折れそうです。（GitHubのコメントって議論に向かないのでは 🙄）

とはいえ、気になっていたことがわかって嬉しいし、検討中の仕様を2つも知ることができたのでIssueを読むのは楽しいです。Issueを読もう！
