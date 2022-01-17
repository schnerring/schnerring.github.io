---
title: "Percent-encoding with the Hugo Built-in `urlquery` Function"
date: "2022-01-16T21:33:17+01:00"
draft: true
comments: false
socialShare: true
toc: false
cover:
  src: cover-pexels-cottonbro-7319070.jpg
---

I recently tried to _percent-encode_ (or _URL-encode_) strings in Hugo. Percent-encoding is commonly used to encode data in query strings of URLs because they must only contain ASCII characters. There already is a [Hugo built-in `querify` function to transform key-value pairs to query strings](https://gohugo.io/functions/querify/). But my requirement was to URL-encode a single string value and not build a query string from key-value pairs.

<!--more-->

My goal is to allow users of the Hugo theme to add social sharing buttons through the site configuration like this:

```toml
[params]
  [[params.socialShare]]
    iconSuite = "simple-icon"
    iconName = "facebook"
    formatString = "https://www.facebook.com/sharer.php?u={url}"
  [[params.socialShare]]
    iconSuite = "simple-icon"
    iconName = "reddit"
    formatString = "https://reddit.com/submit?url={url}&title={title}"
```

The `formatString` can contain the `{url}` and `{title}` placeholders. During build-time these placeholders are replaced with values from page-level variables. Here's a simplified version of what I'm trying to do:

<!-- prettier-ignore -->
```html
{{ $href := .formatString }}
{{ $href := replace $href "{url}" .Permalink }}
{{ $href := replace $href "{title}" .Title }}
```

The issue here is that `.Permalink` and `.Title` need to be percent-encoded. So I've been searching for a solution but couldn't find an elegant solution.

- https://discourse.gohugo.io/t/url-encoding-percent-encoding-with-hugo-solved/16546/13
- https://discourse.gohugo.io/t/is-there-a-url-encoder-function-in-hugo/2239/8

But I couldn't find an elegant solution.

[I discovered an undocumented built-in `urlquery` Hugo function](https://github.com/gohugoio/hugoDocs/issues/1627).
