---
title: Checking Out Code Committed Within the Same GitHub Actions Workflow Run
date: "2023-03-30T19:43:55+02:00"
draft: true
comments: true
socialShare: true
toc: false
cover:
  src: roman-synkevych-wX2L8L-fGeA-unsplash.jpg
  caption: Photo by [Roman Synkevych 🇺🇦](https://unsplash.com/@synkevych)
---

[GitHub Actions](https://docs.github.com/en/actions) workflows often include
steps that use Git to checkout out the code of the repository, make some changes
to the code, and then commit these changes back to the repo. Additional steps
subsequently act upon these changes within the same workflow run. For example,
appending auto-generated release notes to a `CHANGELOG.md` file and committing
it to the repo, to later bundle the changelog with release artifacts. Another
example is using a workflow that periodically uses a linter to clean up the code
base.

<!--more-->
