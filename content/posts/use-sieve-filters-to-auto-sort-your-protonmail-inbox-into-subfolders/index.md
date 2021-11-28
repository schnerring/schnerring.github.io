---
title: "Use Sieve Filters to Auto-Sort Your ProtonMail Inbox into Subfolders"
date: 2021-06-16T20:32:21+02:00
cover:
  src: "img/cover.svg"
comments: true
tags:
  - protonmail
  - sieve
aliases:
  - /posts/use-sieve-filters-to-auto-sort-your-protonmail-inbox-into-subfolders
---

[Sieve](<https://en.wikipedia.org/wiki/Sieve_(mail_filtering_language)>) is a programming language used for email filtering. Today, I show you how I automatically sort my ProtonMail inbox into folders and subfolders using [custom sieve filters](https://protonmail.com/support/knowledge-base/sieve-advanced-custom-filters/). My setup uses the [catch-all feature](https://protonmail.com/support/knowledge-base/catch-all/) requiring at least a [ProtonMail Professional subscription](https://protonmail.com/pricing) and a [properly configured custom domain](https://protonmail.com/support/categories/custom-domains/).

<!--more-->

The ProtonMail Bridge has been supporting subfolders for a while now. With the [release of the re-designed ProtonMail for web](https://protonmail.com/blog/new-protonmail-announcement/), subfolders are now officially supported. So now is a great time to share my setup with you guys!

## The Goal

I only want emails from unknown senders to land in my inbox, e.g., non-automated messages from people directly contacting me or emails from unknown websites. Sieve filters sort emails from well-known senders into folders and subfolders, like webshops or social media for which I have signed up.

I use root-level folders to categorize my email. Subfolders are for specific websites belonging to a category:

```text
Inbox/
Webshops/
├─ Amazon/
├─ Ex Libris/
Entertainment/
├─ Netflix/
├─ Spotify/
```

Here, `Webshops` and `Entertainment` are the categories, and `Amazon` and `Spotify` are specific websites. For each website, I use a separate email address that maps to a pre-created folder. The generic structure of the email address I use to sign up for websites is the following:

```text
category+website@schnerring.net
```

For the example folder structure above, the email-to-folder mapping looks like this:

- `webshops+amazon@schnerring.net` &harr; `Webshops/Amazon/`
- `webshops+exlibris@schnerring.net` &harr; `Webshops/Ex Libris/`
- `entertainment+netflix@schnerring.net` &harr; `Entertainment/Netflix/`
- `entertainment+spotify@schnerring.net` &harr; `Entertainment/Spotify/`
- `*@schnerring.net` &harr; `Inbox/` (**catch-all**)

## The Solution

You can add custom sieve filters in the ProtonMail web client by navigating to `Settings` &rarr; `Filters` &rarr; `Add sieve filter`. The following snippet meets all of my requirements:

```sieve
require ["include", "variables", "fileinto", "envelope"];

if envelope :localpart :matches "To" "*+*" {
  set :lower "category" "${1}";
  set :lower "website"  "${2}";
}
else {
  return;
}

if string :is "${website}" "exlibris" {
  fileinto "${category}/Ex Libris";
} else {
  fileinto "${category}";
  fileinto "${category}/${website}";
}
```

The `require` command in the first line is used to load extensions that provide functionality, like `fileinto` to file messages into folders or `variables` to declare variables.

Next, the `if`-conditional checks the `:localpart`, the part before the `@` symbol, of the `To` address. If it matches the pattern `*+*` (`category+website`):

- the value of the `category` variable is set to the match left of `+` symbol (`${1}`)
- the value of the `website` variable is set to the match right of the `+` symbol (`${2}`).

If `:localpart` does not match the pattern (`else`), the sieve filter exits (`return`).

Next, the actual sorting happens. Let's look into the `else` case first:

```sieve
} else {
  fileinto "${category}";
  fileinto "${category}/${website}";
}
```

Most of the magic happens here. The email is first moved to the `category` folder and then moved to the more specific `website` folder. This way, if the `website` subfolder does not (yet) exist, the email is at least moved into the root-level `category` folder. I find this useful when only ordering at a webshop _once_ via guest checkout instead of signing up. This way, I don't need to bother creating an extra subfolder that will only ever contain one or two emails.

Lastly, let's cover the exception for the `exlibris` webshop in the `if`-part:

```sieve
if string :is "${website}" "exlibris" {
  fileinto "${category}/Ex Libris";
}
```

It is just a cosmetic exception. When I register at a webshop with `webshops+exlibris@schnerring.net` but want to map it to a folder called `Ex Libris`, an explicit mapping is required. Note that `fileinto` is not case-sensitive, so it is valid if the email address is `webshops+amazon@schnerring.net`, but the folder name is `Webshops/Amazon/`. It also is easily extensible by adding more `elseif` conditions.

For more details on advanced custom sieve filtering, check the [official ProtonMail documentation](https://protonmail.com/support/knowledge-base/sieve-advanced-custom-filters/).
