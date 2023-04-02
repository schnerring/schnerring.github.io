---
title: Checking Out Code Committed Within the Same GitHub Actions Workflow Run
date: "2023-04-02T05:30:00+02:00"
comments: true
socialShare: true
toc: true
cover:
  src: cover-roman-synkevych-wX2L8L-fGeA-unsplash.jpg
  caption: Photo by [Roman Synkevych 🇺🇦](https://unsplash.com/@synkevych)
tags:
  - CI
  - Continuous Integration
  - DevOps
  - Git
  - GitHub Actions
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
          cat CHANGELOG.md
          # The changes are here 🎉🎉🎉
```

If you wanna know how it works and how I got there, continue reading!

## The Issue

When I recently tried to implement this, I came up with the following (example)
solution:

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
          cat CHANGELOG.md
          # The changes are missing 😕
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

In the `publish` job that follows, I check out the code again to inspect the
contents of our changelog file, only to find out that the changes are missing.

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

| `github.sha`                                      | `github.ref`                                                                                                                                                 |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| The commit SHA that triggered the workflow. [...] | The fully-formed ref of the branch [...] that triggered the workflow run. For workflows triggered by `push`, this is the branch [...] that was pushed. [...] |

Taking a look at the
[git-source-provider.ts](https://github.com/actions/checkout/blob/v3.5.0/src/git-source-provider.ts#L154-L197)
and the
[`getRefSpec` functions in ref-helper.ts](https://github.com/actions/checkout/blob/v3.5.0/src/ref-helper.ts#L66-L125)
shows that `actions/checkout` ensures that _exactly_ the commit specified in the
`github.sha` variable of the `github` context is fetched and checked out.

So now we know! Across the lifetime of a workflow run, `github.sha` never
changes. So by default, `actions/checkout` always checks out the commit that
triggered the workflow.

## Dirty Fix #1: `git pull`

One suggestion from the GitHub issue I linked to earlier is to perform a
`git pull` after checking out the repository like this:

```yml
# DIRTY CODE: KINDA WORKS!

publish:
  needs: update-changelog
  runs-on: ubuntu-latest
  steps:
    - name: Check Out the Repo Again
      uses: actions/checkout@v3

    - name: Pull Changes
      run: git pull origin main
      # OR git pull origin ${{ github.ref_name }}

    - name: Display CHANGELOG.md
      run: |
        cat CHANGELOG.md
        # The changes are here 🎉
```

This looks harmless enough, right? After checking out the code, we simply get
the changes we made by running `git pull`. Before we discuss why this might be a
bad idea, let's look at the second dirty fix that "kinda works".

## Dirty Fix #2: Use `ref: main`

Another recommendation given in the linked issue is to explicitly specify the
branch to check out via the `ref` parameter as follows:

```yml
# DIRTY CODE: KINDA WORKS!

publish:
  needs: update-changelog
  runs-on: ubuntu-latest
  steps:
    - name: Check Out the Repo Again
      uses: actions/checkout@v3
      with:
        ref: main
        # OR ${{ github.ref }}

    - name: Display CHANGELOG.md
      run: |
        cat CHANGELOG.md
        # The changes are here 🎉
```

Instead of checking out an exact commit, the action now checks out the latest
available commit on the branch specified in the `ref` parameter. The resulting
behavior is effectively identical to [Dirty Fix #1](#dirty-fix-1-git-pull).

## Why Is This a Bad Idea?

```goat
                               2. OOPSIE 🏇
                                     |
                                      '-------.
                                               |
                                               v

     *--------------------*--------------------*  main

     ^                    ^                    ^
     |                    |                    |
     |                    |                    |
github.sha      1. update-changelog       3. publish
```

The diagram illustrates a possible race condition that can occur when using the
dirty fixes above. In this context, an `OOPSIE` is another commit being pushed
_right between_ the execution of the `update-changelog` and `publish` jobs. This
means, that we could end up with a discrepancy between the changelog and what is
published. While this is more unlikely to happen when working on your own, it
can certainly happen on busy branches.

## The Solution

To mitigate this issue, we have to make sure that the `publish` job checks out
exactly the commit that was pushed by the `update-changelog` job.

To do that, we first define a `commit_hash` output on the `update-changelog`
job. It contains the value of the
[`commit_hash` output of the `stefanzweifel/git-auto-commit-action`](https://github.com/stefanzweifel/git-auto-commit-action/tree/v4.16.0#outputs):

```yml
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
```

I recommend using the `stefanzweifel/git-auto-commit-action`, but if you want to
do things manually, the `Commit and Push Changes` step would look similar to
this:

```yml
- name: Commit and Push Changes
  id: commit-and-push
  run: |
    git add CHANGELOG.md
    git config --local user.email "41898282+github-actions[bot]@users.noreply.github.com"
    git config --local user.name "github-actions[bot]"
    git commit -m "Update changelog"
    git push
    echo "commit_hash=$(git rev-parse HEAD)" >> $GITHUB_OUTPUT
```

In the following `publish` step, we can reference the `commit_hash` output like
this:

```yml
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
        cat CHANGELOG.md
        # The changes are here 🎉🎉🎉
```

This makes sure that we check out exactly what we committed and pushed in the
`update-changelog` job.

One final thing to note is that this puts the repository in a `detached HEAD`
state (not a bad thing).

I hope you enjoyed this article and that I could clear up a bit of the confusion
surrounding this topic. I would love to know what you think!
