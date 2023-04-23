---
title: "Next13のApp RouterでMDXを直接ページにする方法"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Nextjs", "AppRouter"]
publication_name: "praha"
published: true
---

Next.jsの[App Router](https://beta.nextjs.org/docs) (app directory)で[MDX](https://mdxjs.com/)を使用したくて調べました。ほぼ[公式のドキュメント](https://beta.nextjs.org/docs/guides/mdx)通りなのですが、MDXファイルを直接ページとして表示するためには追加の設定が必要だったので、備忘録として手順を残します。

## 結論

- `@next/mdx` をインストールして設定する
- **nextjs.config.jsの `pageExtensions` に `'mdx'` を追加する**

## 詳細な手順

### 0. App Routerを有効にする

App Routerが有効になっていることを確認します。

```js: next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    appDir: true,
  },
}

module.exports = nextConfig
```

### 1. `@next/mdx` をインストールする

Peer Dependencies もまとめてインストールします。 ( `@mdx-js/loader` は無いと動きませんでした)

```bash
npm install @next/mdx @mdx-js/loader @mdx-js/react
```

のちのち使用する型定義があるので、 `@types/mdx` もインストールしておくと良いです。

```bash
npm install @types/mdx
```

### 2. プロジェクトのルートに `mdx-components.tsx` を追加する

[ドキュメント](https://beta.nextjs.org/docs/guides/mdx)のコードそのままです。

:::message
配置する場所は `src/` ディレクトリではなく、プロジェクトのルートディレクトリなので注意してください。
:::

```tsx: mdx-components.tsx
import type { MDXComponents } from 'mdx/types'
    // This file is required to use MDX in `app` directory.
    export function useMDXComponents(components: MDXComponents): MDXComponents {
      return {
        // Allows customizing built-in components, e.g. to add styling.
        // h1: ({ children }) => <h1 style={{ fontSize: "100px" }}>{children}</h1>,
        ...components,
      }
    }
```

### 3. next.config.jsにwithMDXの設定を行う

```js: next.config.js
const withMDX = require('@next/mdx')()

/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    appDir: true,
  },
}

module.exports = withMDX(nextConfig)
```

ここまで設定すれば、page.tsxからmdxファイルをインポートすることでmdxファイルを表示できるようになります。mdxファイルを直接ページとして表示するためには、さらに次の手順が必要です。

### 4. `pageExtensions` に `'mdx'` を追加

:::message
JavaScriptの場合は、tsxの代わりにjsxを記述してください。
:::

```js: next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  pageExtensions: ['tsx', 'mdx'], // ここを追加
  experimental: {
    appDir: true,
  },
}

module.exports = withMDX(nextConfig)
```

### 5. MDXファイルをページとして使用する (完了)

`app/about/page.mdx` を作成すると、`/about` ページとして表示されます。[ルートレイアウト](https://beta.nextjs.org/docs/routing/pages-and-layouts#root-layout-required) (`app/layout.tsx`) も、もちろん反映されます。

:::message
`app/about.mdx` ではないので注意してください。
:::

```mdx: app/hello.mdx
## About

This is a demo for rendering MDX as a page.
```

![mdx as page](/images/mdx-as-page.png)

### 6. (オプション) `mdxRs` を有効にする

mdxRsを有効にすると、Rustを使用してmdxファイルをコンパイルするのでビルドが高速になります。

```js: next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  pageExtensions: ['tsx', 'mdx'],
  experimental: {
    appDir: true,
    mdxRs: true, // ここを追加
  },
}

module.exports = withMDX(nextConfig)
```

## 補足: MDXファイルをコンポーネントとして使用する

[ドキュメント](https://beta.nextjs.org/docs/guides/mdx)に記載されている方法です。mdxファイルをコンポーネントとしてインポートできるので、`page.tsx` ファイルでインポートして使用することで、MDXのコンテンツを表示できます。

```mdx: contents/sample-text.mdx
## Sample Text

This is a demo for rendering MDX as a server component.
```

```tsx: pages/hello.tsx
import SampleText from "@/contents/sample-text.mdx"

export default function Home() {
  return <SampleText />
}
```

![mdx as component](/images/mdx-as-component.png)

## おわり

App Router楽しいですね。MDXをページとして使用するサンプルを置いて置くのでご自由にお使いください。

[sushidesu/next-app-router-with-mdx-page: next-app-router-with-mdx-page](https://github.com/sushidesu/next-app-router-with-mdx-page)

## 参考

- [Markdown and MDX | Next.js](https://beta.nextjs.org/docs/guides/mdx)
- [@next/mdx - npm](https://www.npmjs.com/package/@next/mdx)
- [`page.mdx` in the `app` directory. · vercel/next.js · Discussion #42882](https://github.com/vercel/next.js/discussions/42882)
- [next.js/examples/app-dir-mdx at canary · vercel/next.js](https://github.com/vercel/next.js/tree/canary/examples/app-dir-mdx)
