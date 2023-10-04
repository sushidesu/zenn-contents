---
title: "typescript-eslintのrecommendedって2種類あんねん"
emoji: "👯‍♀️"
type: "tech"
topics: ["typescript", "eslint"]
published: false
---

## 結論

`eslint:recommended` や `plugin:@typescript-eslint/recommended` を使用するだけでは [`no-floating-promises`](https://typescript-eslint.io/rules/no-floating-promises/) や [`restrict-plus-operands`](https://typescript-eslint.io/rules/restrict-plus-operands/) などの重要^[個人の感想です]なルールが有効になりません。

`plugin:@typescript-eslint/recommended-type-checked` を使用すると、これらのルールをまとめて有効にできます。

## recommended

TypeScriptのプロジェクトではともに導入されることが多いESLintとtypescript-eslintですが、公式ドキュメントの [Getting Started | typescript-eslint](https://typescript-eslint.io/getting-started) に則って進めると、以下のようなconfigになっていることが多いのではないでしょうか。

```js
module.exports = {
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended'
  ],
  // ...
};
```

しかし、この設定だけではawait忘れを検知する [`no-floating-promises`](https://typescript-eslint.io/rules/no-floating-promises/) や、不自然な加算を検知する [`restrict-plus-operands`](https://typescript-eslint.io/rules/restrict-plus-operands/) が有効になりません。recommendedだからこれを使用しておけば安心、というわけではなかったのです...。

## 生き別れの兄弟、recommended-type-checked

[typescript-eslintが提供するconfig](https://github.com/typescript-eslint/typescript-eslint/tree/main/packages/eslint-plugin/src/configs) の中には `plugin:@typescript-eslint/recommended` の他にもう1つ recommended と名のつくconfigがあります。それが `plugin:@typescript-eslint/recommended-type-checked` です。これは recommended の中でも型情報が必要なルールをまとめたものになります。

設定の詳細は [packages/eslint-plugin/src/configs/recommended-type-checked.ts](https://github.com/typescript-eslint/typescript-eslint/blob/main/packages/eslint-plugin/src/configs/recommended-type-checked.ts) から確認できます。[`plugin:@typescript-eslint/recommended`](https://github.com/typescript-eslint/typescript-eslint/blob/d948dc4a21ad8e15eec152c0cf2fdda819ea4a3a/packages/eslint-plugin/src/configs/recommended.ts#L11-L30) と比較すると 20行近く有効なルールが増えていますね。

この中に、await忘れを検知する `no-floating-promises` や 不自然な加算を検知する `restrict-plus-operands` を有効にする設定が含まれています。

繰り返しになりますが、これらのルールは、 `eslint:recommended` や `plugin:@typescript-eslint/recommended` では有効になりません。個別に有効にするか、 `recommended-type-checked` をextendsする必要があります。

```js
module.exports = {
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended-type-checked',
  ],
  // ...
};
```

## 別れた経緯

リリースノートによると、これらのルールはもともと recommended に入っていましたが、 [v2.0.0](https://github.com/typescript-eslint/typescript-eslint/releases/tag/v2.0.0) にて分離されたようです。 導入を容易にするため、デフォルトのルールを速くするため、とのこと。

> This is a slightly expanded set of rules, intended to be used in conjunction with plugin:@typescript-eslint/recommended. These rules specifically require type information. We separated these rules into a separate config to ease adoption and to make the base recommended "fast-by-default".


### おわりに

みなさんは知っていましたか？自分は知らなかったのでびっくりしました... より厳しいルール集の [`strict-type-checked`](https://github.com/typescript-eslint/typescript-eslint/blob/main/packages/eslint-plugin/src/configs/strict-type-checked.ts) もあるので、厳しさを求める場合はそちらもおすすめです。

## 参考

- https://typescript-eslint.io/linting/typed-linting/
- https://github.com/typescript-eslint/typescript-eslint/pull/846
