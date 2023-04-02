---
title: "Percent-encoding with the Hugo `urlquery` Function"
date: "2022-01-16T21:33:17+01:00"
comments: true
socialShare: true
toc: false
cover:
  src: cover-pexels-cottonbro-7319070.jpg
  caption: Photo by [cottonbro studio](https://www.pexels.com/@cottonbro/)
tags:
  - hugo
---

I recently tried to _percent-encode_ (or _URL-encode_) strings in Hugo.
Percent-encoding is used to encode data in query strings of URLs because they
must only contain ASCII characters. There is a
[Hugo built-in `querify` function to transform key-value pairs to query strings](https://gohugo.io/functions/querify/).
But I required to URL-encode a single string value and not build a query string
from key-value pairs.

<!--more-->

My goal is to allow users of the Hugo theme to add social media sharing buttons
through the site configuration like this:

```toml
[params]
  [[params.socialShare]]
    formatString = "https://www.facebook.com/sharer.php?u={url}"
  [[params.socialShare]]
    formatString = "https://reddit.com/submit?url={url}&title={title}"
```

The `formatString` contains `{url}` and `{title}` placeholders. The following is
a simplified sharing button implementation using the format strings where
placeholders are substituted with page-level variable values during Hugo
build-time:

<!-- prettier-ignore -->
```html
{{ $href := .formatString }}
{{ $href := replace $href "{url}" .Permalink }}
{{ $href := replace $href "{title}" .Title }}

<a href="{{ $href | safeURL }}">Share Me!</a>
```

The issue here is that `.Permalink` and `.Title` must be percent-encoded. As
I've searched the web on how to do this with Hugo, I only found unsolved forum
posts:

- [About htmlEscape: transform https into https%3A%2F%2F](https://discourse.gohugo.io/t/about-htmlescape-transform-https-into-https-3a-2f-2f/36116)
- [URL encoding (percent encoding) with Hugo?](https://discourse.gohugo.io/t/url-encoding-percent-encoding-with-hugo-solved/16546)
- [Is there a url encoder function in Hugo?](https://discourse.gohugo.io/t/is-there-a-url-encoder-function-in-hugo/2239)

I couldn't believe this was an unsolved mystery and assumed I was just too dumb
to read the docs. Hugo was written in Go, so I looked into how to do
percent-encoding in Go: with the `url.QueryEscape()` function from the `net/url`
package.

Searching the Hugo code base for `url.QueryEscape`
[leads to exactly one result](https://github.com/gohugoio/hugo/blob/3d5dbdcb1a11b059fc2f93ed6fadb9009bf72673/tpl/internal/go_templates/texttemplate/funcs.go#L738-L742).
It turns out
[I discovered an undocumented, built-in Hugo function called `urlquery`](https://github.com/gohugoio/hugoDocs/issues/1627).
The following:

```html
{{ urlquery "https://schnerring.net" }}
```

...returns the result: `https%3A%2F%2Fschnerring.net`. Here is the fixed version
of the sharing button:

<!-- prettier-ignore -->
```html
{{ $href := .formatString }}
{{ $href := replace $href "{url}" (urlquery .Permalink) }}
{{ $href := replace $href "{title}" (urlquery .Title) }}

<a href="{{ $href | safeURL }}">Share Me!</a>
```

[The full social sharing button implementation is available on my GitHub](https://github.com/schnerring/hugo-theme-gruvbox/blob/main/layouts/partials/social-share.html).
