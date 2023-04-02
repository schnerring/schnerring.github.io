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

When using [GitHub Actions](https://docs.github.com/en/actions), we often
include steps that use Git to check out the code of the repository, make some
changes to the code, and then commit these changes back to the repo. Additional
steps subsequently act upon these changes within the same workflow run. For
example, appending auto-generated release notes to a `CHANGELOG.md` file and
committing it to the repo, to later bundle the changelog with release artifacts.
Another example is using a workflow that periodically uses a linter to clean up
the code base. Simple enough, right?

<!--more-->

## TL;DR

If you're just interested in the solution, I recommend using the following
snippet:

```yml
jobs:
  update-changelog:
    runs-on: ubuntu-latest
    outputs:
      commit_hash: ${{ steps.commit-and-push.outputs.commit_hash }}
    steps:
      - name: Check Out the Repo
        uses: actions/checkout@v3

      - name: Update CHANGELOG.md
        run: echo "Added changes on $(date)" >> CHANGELOG.md

      - name: Commit and Push Changes
        id: commit-and-push
        uses: stefanzweifel/git-auto-commit-action@v4

  publish:
    needs: update-changelog
    runs-on: ubuntu-latest
    steps:
      - name: Check Out the Repo Again
        uses: actions/checkout@v3
        with:
          ref: ${{ needs.update-changelog.outputs.commit_hash }}

      - name: Display CHANGELOG.md
        run: |
          # The changes are here 🎉
          cat CHANGELOG.md
```

If you wanna know how it works and how I got there, continue reading!

## The Issue

When recently trying to implement this, I came up with the following
(simplified) solution:

```yml
# BAD CODE: DON'T USE!

jobs:
  update-changelog:
    runs-on: ubuntu-latest
    steps:
      - name: Check Out the Repo
        uses: actions/checkout@v3

      - name: Update CHANGELOG.md
        run: echo "Added changes on $(date)" >> CHANGELOG.md

      - name: Commit and Push
        uses: stefanzweifel/git-auto-commit-action@v4

  publish:
    needs: update-changelog
    runs-on: ubuntu-latest
    steps:
      - name: Check Out the Repo Again
        uses: actions/checkout@v3

      - name: Display CHANGELOG.md
        run: |
          # The changes are missing 😕
          cat CHANGELOG.md
```

As you can see, there are two jobs: `update-changelog` and `publish`. By
default, `jobs` always run in parallel. To ensure publishing only happens
_after_ updating the changelog, we set `publish.needs` to `update-changelog`.

The `update-changelog` job checks out the code using the
[`actions/checkout`](https://github.com/actions/checkout) action, appends a
message containing the current date to the `CHANGELOG.md` file, and then uses
the
[`stefanzweifel/git-auto-commit-action`](https://github.com/stefanzweifel/git-auto-commit-action)
action to commit and push the code to the repository.

In the `publish` job that follows, we check out the code again and inspect the
contents of our changelog file, only to find out that the changes we made are
missing.

It's not the first time that I have fallen into this trap, and since you are
reading this, likely you have too. We are not alone! I came across a
[popular GitHub issue from the `actions/checkout` action repository](https://github.com/actions/checkout/issues/439)
where people share the same pain.

## Why Does This Happen?

To understand what's going on, I did a little digging in the `actions/checkout`
code base. In the following, I will only discuss workflows triggered by `push`
events to branches. For events with other origins, such as pull requests or
pushing tags, please consult the documentation and make appropriate adjustments.

First, let's have a look at the
[docs of the `ref` parameter of the action](https://github.com/actions/checkout/tree/v3.5.0#usage):

> The branch, tag or SHA to checkout. When checking out the repository that
> triggered a workflow, _this defaults to the reference or SHA_ for that event.

So if the `ref` parameter is left unspecified, the action will use the
`github.ref` and `github.sha` values from the
[`github` context](https://docs.github.com/en/actions/learn-github-actions/contexts#github-context).
You can find the code for this logic in
[input-helper.ts](https://github.com/actions/checkout/blob/v3.5.0/src/input-helper.ts#L60-L79).

| `github.sha`                                                                                                                     | `github.ref`                                                                                                                                                             |
| -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| The commit SHA that triggered the workflow. The value of this commit SHA depends on the event that triggered the workflow. [...] | The fully-formed ref of the _**branch or tag that triggered the workflow run**_. For workflows triggered by `push`, this is the branch or tag ref that was pushed. [...] |

Taking a look at the
[git-source-provider.ts](https://github.com/actions/checkout/blob/v3.5.0/src/git-source-provider.ts#L154-L197)
and the
[`getRefSpec` functions in ref-helper.ts](https://github.com/actions/checkout/blob/v3.5.0/src/ref-helper.ts#L66-L125)
shows that `actions/checkout` ensures that _exactly_ the commit specified in the
`github.sha` variable of the `github` context is fetched and checked out.

So now we know! Across the lifetime of a workflow run, `github.sha` never
changes. So by default, `actions/checkout` always checks out the commit that
triggered the workflow.
