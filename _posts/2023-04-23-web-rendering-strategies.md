---
layout: post
title:  "Next.js中的Web渲染策略-CSR, SSR, SSG, ISR（一）"
date:   2023-04-24 00:53:08 +0800
categories: jekyll update
---

## Intro ##  
最近在学习前端，接触到了Next.js以及Web渲染的四种策略，CSR，SSR，SSG，以及ISR。
首先简单解释一些术语：  
**SPA**（Single-Page Application）：  
单页应用。用新数据动态重写当前页面，而非加载整个页面，避免频繁的页面切换。比如我们常见的Gmail，Twitter，Discord等等都是典型的SPA。  
**CSR**（Client-Side Rendering）：  
客户端渲染。直接在浏览器中使用JavaScript渲染页面。逻辑，数据，路由等几乎所有工作都在客户端处理。  
**SSR**（Server-Side Rendering）：  
服务器端渲染。在服务器端进行渲染，将渲染好的HTML页面发送至浏览器或客户端。  
**SSG**（Static-Site Generation）：  
静态站点生成，同样在服务器端渲染，但是只在构建（build）时生成一次。数据只有在下次构建（rebuild）时才会更新。  
**ISR**（Incremental Static Generation）：  
增量式静态生成。在SSG的基础上，支持更新静态页面，而不必重新构建整个网站。  

## CSR ##  
CSR是我们使用React构建SPA时的默认方式。在Next.js中，我们有两种方式实现CSR。  
- 使用useEffect():  
  就像在纯粹的React中一样，我们在Next.js中可以使用useEffect hook。  

```js
import {useState, useEffect} from 'react'

export default function Home() {
  const [time, setTime] = useState(null)
  const [isLoading, setLoading] = useState(false)

  useEffect(() => {
    setLoading(true)
    fetch('https://worldtimeapi.org/api/ip')
    .then((res) => res.json())
    .then((data) => {
      setTime(data.datetime)
      setLoading(false)
    })
  }, [])

  if (isLoading) {
    return <p>Loading...</p>
  }

  return (
      <div>
        <h1>{time}</h1>
      </div>
  )
}
```  

- 使用SWR（Highly Recommend）：  
  SWR是Next.js团队创建的一个用于数据请求的轻量级React hook库。它提供了很多有用的特性，具体可以参考[官方文档](https://swr.vercel.app/)。  

```js
import useSWR from 'swr'

const fetcher = (...args) => fetch(...args).then((res) => res.json())

export default function Home() {
  const {data, error} = useSWR('https://worldtimeapi.org/api/ip', fetcher)

  if (error) {
    return <div>Failed to load</div>
  }
  if (!data) {
    return <div>Loading...</div>
  }

  return (
      <div>
        <h1>{data.datetime}</h1>
      </div>
  )
}
```

## SSR ##  
作为Next.js的主要卖点之一，SSR在Next.js中是通过在[page](https://nextjs.org/docs/basic-features/pages)（其实就是一个component）中导出一个`getServerSideProps`函数实现的。当我们请求时，Next.js会使用该函数返回的数据在服务器端预渲染当前页面，最后将渲染好的页面返回给浏览器。  
```js
export default function Home({data}) {
  return (
      <div>
        <h1>{data.datetime}</h1>
      </div>
  )
}

// This gets called on every request
export async function getServerSideProps() {
  // Fetch data from external API
  const res = await fetch('https://worldtimeapi.org/api/ip')
  const data = await res.json()

  // Pass data to the page via props
  return {props: {data}}
}
```

## SSG ##  
默认情况下，Next.js总是会试图在构建时（build）静态生成页面，如果该页面是被写死的（hard-coded）。在这种情况下，我们不需要任何额外操作。  
```js
export default function Home() {
  return (
      <div>
        <h1>Hello,World.</h1>
      </div>
  )
}
```
但是，如果我们页面上的一些内容来自于外部资源（例如headless CMS），那么我们需要使用`getStaticProps`函数帮助我们从外部请求数据。这样，在页面被静态创建时，该函数返回的数据会被填充在页面中。并且，像SSR一样，`getStaticProps`函数只能被从一个页面中导出。将它放在其他任何地方都不会起作用。  
```js
export default function Home({data}) {
  return (
      <div>
        <h1>{data.datetime}</h1>
      </div>
  )
}

// This gets called at build time on server-side
export async function getStaticProps() {

  const res = await fetch('https://worldtimeapi.org/api/ip')
  const data = await res.json()

  return {props: {data}}
}
```
在某些情况下，我们可能有多个页面，并且每个页面的路径都取决于该数据本身。比如说一个用户信息系统，拥有很多用户，每个用户都有一个详情页，我们想让每个页面的URL格式为`users/[uid]`。同时，我们想要静态创建所有这些页面，从而用户可以飞速地访问这些页面。Next.js可以帮助我们很轻松地做到这一点，唯一要做的就是添加一个`getStaticPaths`函数。该函数帮助我们定义了一组需要被静态生成的页面，并返回一个数组。当我们从一个使用了动态路由的页面上导出`getStaticPaths`时，Next.js会帮助我们预渲染所有被指定的路径，通过遍历该函数返回的数组，并在每个元素上执行`getStaticProps`的方式（此处涉及到Next.js的[Dynamic Routes](https://nextjs.org/docs/routing/dynamic-routes)相关知识）。  
```js
export default function UserDetail({data}) {
  return (
      <div>
        // ...some contents
      </div>
  );
}

// Generates `/users/1`, `/users/2`...
export async function getStaticPaths() {
  const res = await fetch('https://userAPI/users')
  const users = await res.json()

  const paths = users.map((user) => ({
    params: {id: user.id.toString()},
  }))

  return {paths, fallback: false} // fallback can also be true or 'blocking'
}

// `getStaticPaths` requires using `getStaticProps`
// params is from `getStaticPaths`
export async function getStaticProps({params}) {
  const res = await fetch(`https://userAPI/users/${params.id}`)
  const data = await res.json()

  return {props: {data}}
}
```

## ISR ##  
由于SSG过于静态，很多场景都不完全适用，因此我们需要一种可以支持动态数据的SSG，这时ISR出现了。Next.js在[v9.5中首先提出](https://nextjs.org/blog/next-9-5#stable-incremental-static-regeneration)了ISR的策略，从而允许我们在整个网站构建后，以页面为单位应用静态生成，而不必重新构建整个网站。使用ISR非常简单，我们只需要给`getStaticProps`函数的返回值中添加一个`revalidate`属性。该属性定义了我们数据的最大有效期，例如在下面的例子中，我们的数据最大有效期为10秒。值得注意的是，我们的页面并不是每十秒更新一次，而是，如果数据已经过期，那么在过期后的第一次请求时，我们仍然会得到过时的页面，但是Next.js会在后台触发一次该页面的重新渲染，在新页面渲染成功后，会在缓存中替代过时页面。在这之后，再次请求将会返回新页面。  
```js
export async function getStaticPaths() {
  const res = await fetch('https://userAPI/users')
  const users = await res.json()

  const paths = users.map((user) => ({
    params: {id: user.id.toString()},
  }))

  return {
    paths,
    fallback: 'blocking' // will server-render a page if doesn't exist.
  }
}

export async function getStaticProps({params}) {
  const res = await fetch(`https://userAPI/users/${params.id}`)
  const data = await res.json()

  return {
    props: {data},
    // Next.js will attempt to re-generate the page:
    // - When a request comes in
    // - At most once every 10 seconds
    revalidate: 10
  }
}
```
其实我们这里还有一行代码和之前不同，函数`getStaticPaths`中的`fallback`被修改为了`blocking`（也可以为`true`，效果略有不同），这是Next.js中使用ISR时另一个非常有用的特性。它的意义是，当我们触发一个对不存在页面的请求时，Next.js的回退策略。详细来说，如果我们定义为`blocking`，那么当我们请求一个不存在的页面（未被静态生成），Next.js会在第一次请求时服务器渲染该页面，未来的请求则将被从缓存中提供静态文件，从而提升后续用户的访问体验。当我们的网站体量很大并且拥有很多非核心页面时，我们可以在构建时只静态预渲染核心页面，加快我们的构建速度，其他的非核心页面使用SSR在被请求时渲染并缓存。  

## Summary ##  
在这篇文章中我首先简略介绍了四种常见的现代Web建站技术，并且介绍了我们如何在Next.js中应用这四种策略。虽然仍然有一些内容未涉及，例如Next.js在[v12中引入](https://nextjs.org/blog/next-12-2#on-demand-incremental-static-regeneration-stable)了On-Demand ISR，让我们可以手动清除特定页面的缓存。但是基本用法都有比较详细的介绍。下一篇中，我计划更详细地探讨每种策略的运行机制，并且比较它们的优缺点及使用场景。最后，如果有兴趣，Next.js给我们提供了一个比较四者效果的[demo](https://csr-ssr-ssg-ssr.vercel.app/)。