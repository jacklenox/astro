---
layout: ~/layouts/MainLayout.astro
title: API Reference
---

## `Astro` global

The `Astro` global is available in all contexts in `.astro` files. It has the following functions:

### `Astro.fetchContent()`

`Astro.fetchContent()` is a way to load local `*.md` files into your static site setup. You can either use this on its own, or within [Astro Collections](/core-concepts/collections).

```jsx
// ./src/components/my-component.astro
---
const data = Astro.fetchContent('../pages/post/*.md'); // returns an array of posts that live at ./src/pages/post/*.md
---

<div>
{data.slice(0, 3).map((post) => (
  <article>
    <h1>{post.title}</h1>
    <p>{post.description}</p>
    <a href={post.url}>Read more</a>
  </article>
))}
</div>
```

`.fetchContent()` only takes one parameter: a relative URL glob of which local files you'd like to import. Currently only `*.md` files are supported. It's synchronous, and returns an array of items of type:

```js
{
   /** frontmatter from the post.. example frontmatter:
    title: '',
    tag: '',
    date: '',
    image: '',
    author: '',
    description: '',
   **/
    astro: {
      headers: [],  // an array of h1...h6 elements in the markdown file
      source: ''    // raw source of the markdown file
      html: ''      // rendered HTML of the markdown file
    },
    url: '' // the rendered path
  }[]
```

### `Astro.request`

`Astro.request` returns an object with the following properties:

| Name           | Type  | Description                                     |
| :------------- | :---- | :---------------------------------------------- |
| `url`          | `URL` | The URL of the request being rendered.          |
| `canonicalURL` | `URL` | [Canonical URL][canonical] of the current page. |

⚠️ Temporary restriction: this is only accessible in top-level pages and not in sub-components.

### `Astro.site`

`Astro.site` returns a `URL` made from `buildOptions.site` in your Astro config. If undefined, this will return a URL generated from `localhost`.

## `getStaticPaths()`

If a page uses dynamic params in the filename, that component will need to export a `getStaticPaths()` function. 

This function is required because Astro is a static site builder. That means that your entire site is built ahead of time. If Astro doesn't know to generate a page at build time, your users won't see it when they visit your site.

```jsx
---
export async function getStaticPaths() {
  return [
    { params: { ... } },
    { params: { ... } },
    { params: { ... } },
    // ...
  ];
}
---
<!-- Your HTML template here. -->
```

The `getStaticPaths()` function should returns an array of objects to determine which paths will be pre-rendered by Astro. 

⚠️ The `getStaticPaths()` function executes in its own isolated scope once, before any page loads. Therefore you can't reference anything from its parent scope, other than file imports. The compiler will warn if you break this requirement.

### `params`

The `params` key of every returned object tells Astro what routes to build. The returned params must map back to the dynamic parameters and rest parameters defined in your component filepath.

`params` are encoded into the URL, so only strings are supported as values. The value for each `params` object must match the parameters used in the page name. 

For example, suppose that you have a page at `src/pages/posts/[id].astro`. If you export `getStaticPaths` from this page and return the following for paths:

```js
---
export async function getStaticPaths() {
  return [
    { params: { id: '1' } },
    { params: { id: '2' } }
  ];
}
const {id} = Astro.request.params;
---
<body><h1>{id}</h1></body>
```

Then Astro will statically generate `posts/1` and `posts/2` at build time.

### Data Passing with `props`

To pass additional data to each generated page, you can also set a `props` value on every returned path object. Unlike `params`, `props` are not encoded into the URL and so aren't limited to only strings.

For example, suppose that you generate pages based off of data fetched from a remote API. You can pass the full data object to the page component inside of `getStaticPaths`:

```js
---
export async function getStaticPaths() {
  const data = await fetch('...').then(response => response.json());
  return data.map((post) => {
    return { 
      params: { id: post.id }, 
      props: { post } };
  });
}
const {id} = Astro.request.params;
const {post} = Astro.props;
---
<body><h1>{id}: {post.name}</h1></body>
```

Then Astro will statically generate `posts/1` and `posts/2` at build time using the page component in `pages/posts/[id].astro`. The page can reference this data using `Astro.props`:

### `paginate()` 

Pagination is a common use-case for websites that Astro natively supports via the `paginate()` function. 

`paginate()` will automatically generate the array to return from `getStaticPaths()` that creates one URL for every page of the paginated collection. The page number will be passed as a param, and the page data will be passed as a `page` prop.

```js
export async function getStaticPaths({paginate}) {
  const allPosts = Astro.fetchContent('../post/*.md');
  const sortedPosts = allPosts.sort((a, b) => new Date(b.date) - new Date(a.date));
  // Return a paginated collection of paths for all posts
  return paginate(sortedPosts, {pageSize: 10});
}
// The page prop has everything that you need to render a single page in the collection.
const {page} = Astro.props;
```

`paginate()` assumes a file name of `[page].astro` or `[...page].astro`. The `page` param becomes the page number in your URL:
- `/posts/[page].astro` would generate the URLs `/posts/1`, `/posts/2`, `/posts/3`, etc. 
- `/posts/[...page].astro` would generate the URLs `/posts`, `/posts/2`, `/posts/3`, etc. 

You must use a rest parameter (the `...` syntax) to build the first page without a page number (`/posts` instead of `/posts/1`). This is needed because non-rest params cannot be optional, and so `/posts` is not a valid match for `/posts/[page].astro` 

You can can customize pagination to use any param name that you'd like, by passing a `param` option in the second argument of the `paginate()` function:

```js
// Example: src/pages/posts/[...myCustomParam].astro
paginate(sortedPosts, {pageSize: 10, param: 'myCustomParam'});
```

#### The pagination `page` prop

Pagination will pass a `page` prop to every rendered page that represents a single page of data in the paginated collection. This includes the data that you've paginated (`page.data`) as well as metadata for the page (`page.url`, `page.start`, `page.end`, `page.total`, etc). This metadata is useful for for things like a "Next Page" button or a "Showing 1-10 of 100" message.

| Name                      |         Type          | Description                                                                                                                       |
| :------------------------ | :-------------------: | :-------------------------------------------------------------------------------------------------------------------------------- |
| `page.data`         |        `Array`        | Array of data returned from `data()` for the current page.                                                                        |
| `page.start`        |       `number`        | Index of first item on current page, starting at `0` (e.g. if `pageSize: 25`, this would be `0` on page 1, `25` on page 2, etc.). |
| `page.end`          |       `number`        | Index of last item on current page.                                                                                               |
| `page.total`        |       `number`        | The total number of items across all pages.                                                                                       |
| `page.page.current` |       `number`        | The current page number, starting with `1`.                                                                                       |
| `page.page.size`    |       `number`        | How many items per-page.                                                                                                          |
| `page.page.last`    |       `number`        | The total number of pages.                                                                                                        |
| `page.url.current`  |       `string`        | Get the URL of the current page (useful for canonical URLs)                                                                       |
| `page.url.prev`     | `string \| undefined` | Get the URL of the previous page (will be `undefined` if on page 1).                                                              |
| `page.url.next`     | `string \| undefined` | Get the URL of the next page (will be `undefined` if no more pages).                                                              |

### `rss()`

RSS feeds are another common use-case that Astro supports natively. Call the `rss()` function to generate an `/rss.xml` feed for your project using the same data that you loaded for this page.

```js
export async function getStaticPaths({rss}) {
  const allPosts = Astro.fetchContent('../post/*.md');
  const sortedPosts = allPosts.sort((a, b) => new Date(b.date) - new Date(a.date));
  // Generate an RSS feed from this collection
  rss({
    title: 'Don’s Blog',
    description: 'An example blog on Astro',
    customData: `<language>en-us</language>`,
    items: sortedPosts.map(item => ({
      title: item.title,
      description: item.description,
      link: item.url,
      pubDate: item.date,
    })),
  });
  // Return a paginated collection of paths for all posts
  return [...];
}
```

## `import.meta`

> In this section we use `[dot]` to mean `.`. This is because of a bug in our build engine that is rewriting `import[dot]meta[dot]env` if we use `.` instead of `[dot]`.

All ESM modules include a `import.meta` property. Astro adds `import[dot]meta[dot]env` through [Snowpack](https://www.snowpack.dev/).

**`import[dot]meta[dot]env[dot]SSR`** can be used to know when rendering on the server. Sometimes you might want different logic, for example a component that should only be rendered in the client:

```jsx
import { h } from 'preact';

export default function () {
  // Note: rewrite "[dot]" to "." for this to to work in your project.
  return import[dot]meta[dot]env[dot]SSR ? <div class="spinner"></div> : <FancyComponent />;
}
```

[canonical]: https://en.wikipedia.org/wiki/Canonical_link_element
