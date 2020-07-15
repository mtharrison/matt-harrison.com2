---
title: "Automating this Hugo Blog with GitHub Actions"
date: 2020-07-14T00:00:00+01:00
---

This Blog is built with the wonderful [Hugo](https://gohugo.io/) static site generator. It is hosted on [GitHub pages](https://pages.github.com/) too and uses the [Soho theme](https://themes.gohugo.io//theme/soho/about/) - that's a lot of great stuff for free.

GitHub pages hosts a static site straight off your repo, either via a specific `docs` folder on `master` branch or a dedicated `gh-pages` branch. To build the static directory Hugo just has a simple `hugo` command to run. Up till now I would run that `hugo` command after making and changes and then commit and push the changes to my blog repo, simple right?

I've been looking into [GitHub actions](https://github.com/features/actions) recently and doing quite a bit of devopsy stuff at [work](https://github.com/features/actions) so I thought this might be another ripe opportunity to take advantage of automation to make my life just that little bit easier.

An added advantage to letting GitHub handle this is that I can then write or edit blog posts on the go via the GitHub UI, when I don't easily have access to my laptop and usual build environment.

When I push to `master` of my blog repo, I want a workflow to do the following for me:

1. Build latest static site built with a specific version of Hugo
2. Commit to `gh-pages` branch (if there are changes)
3. Push to remote `gh-pages` branch (if there are changes)

I want to use a specific version of Hugo so I can be sure I tested my blog on this same version locally, rather than just pull in say the latest version from `apt-get`. I want to use the `gh-pages` branch approach rather than serving the site from a directory on `master` because I don't want to have to do a `git pull` everytime I make any changes to fetch the latest automatic commits.

Here's the full workflow that I came up with:

_[.github/workflows/build.yml](https://github.com/mtharrison/matt-harrison.com2/blob/master/.github/workflows/build.yml)_

```yml
name: build

on:
  push:
    branches: [master]

jobs:
  build:
    name: Build Hugo Site
    runs-on: ubuntu-latest
    steps:
      - name: Install Hugo
        env:
          HUGO_VERSION: 0.74.1
        run: |
          mkdir ~/hugo
          cd ~/hugo
          curl -L "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_Linux-64bit.tar.gz" --output hugo.tar.gz
          tar -xvzf hugo.tar.gz
          sudo mv hugo /usr/local/bin

      - name: Checkout master branch
        uses: actions/checkout@v2
        with:
          ref: master
          path: main
          submodules: true

      - name: Checkout gh-pages branch
        uses: actions/checkout@v2
        with:
          ref: gh-pages
          path: gh-pages

      - name: Hugo Build
        run: cd main && hugo

      - name: Copy files
        run: cp -rf main/public/* gh-pages/

      - name: Commit changes
        run: |
          cd gh-pages

          git config --local user.email "actions@github.com"
          git config --local user.name "GitHub Action"

          git add -A .

          if git diff-index --quiet HEAD --; then
            echo "No changes..."
          else
            git commit -m "[CI] build hugo static site"
            git push
          fi
```
I think the workflow is pretty simple to understand, however I'll go through what each part does.

The following part defines the name of the workflow and also how it should be triggered. In this case I only want my `build` workflow to be triggered when a push happens on the `master` branch.

```yml
name: build

on:
  push:
    branches: [master]
```
The following step installs the Hugo binary by pulling a specific version from GitHub releases. I've used an environment variable to define the version that I'm interested in. When new releases come out, I will first test them locally before bumping this. In practice I don't change this too often. `run` just runs a shell script that you pass as a string.
```yml
    steps:
      - name: Install Hugo
        env:
          HUGO_VERSION: 0.74.1
        run: |
          mkdir ~/hugo
          cd ~/hugo
          curl -L "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_Linux-64bit.tar.gz" --output hugo.tar.gz
          tar -xvzf hugo.tar.gz
          sudo mv hugo /usr/local/bin
```
Next up I use a prebaked Action to checkout both the `master` (with raw posts) and the `gh-pages` (generated static site) branches. Additionally, I checkout submodules for the `master` branch as my theme is included via a Git submodule. I check these 2 branches out side-by-side into 2 different directories.
```yml
    steps:

      ...

      - name: Checkout master branch
        uses: actions/checkout@v2
        with:
          ref: master
          path: main
          submodules: true

      - name: Checkout gh-pages branch
        uses: actions/checkout@v2
        with:
          ref: gh-pages
          path: gh-pages
```
Next I want to actually build the static site. This is as easy as calling `hugo` in the directory containing the `master` branch. Then I'll copy the generated files over to the checked out `gh-pages` branch dir.
```yml
    steps:

      ...

      - name: Hugo Build
        run: cd main && hugo

      - name: Copy files
        run: cp -rf main/public/* gh-pages/
```
In this final step, I set up some config so we can actually change things in `git`. Next inside the `gh-pages` dir I check if anything has actually changed (we're comparing what was checked out from the remote `gh-pages` branch versus what we just generated from `master`). If nothing changed, there's nothing to do. Otherwise we'll make a new commit and push the changes.
```yml
    steps:

      ...

      - name: Commit changes
        run: |
          cd gh-pages

          git config --local user.email "actions@github.com"
          git config --local user.name "GitHub Action"

          git add -A .

          if git diff-index --quiet HEAD --; then
            echo "No changes..."
          else
            git commit -m "[CI] build hugo static site"
            git push
          fi
```
Once the changes are pushed up to `gh-pages` GitHub will take care of deploying the new static files to my site here.

You may wonder if the `git push` that happens within the workflow will trigger another build and result in an infinite loop. Thankfully the GitHub folks have thought about that and any [events triggered from your workflow will not in turn triggers new workflows](https://docs.github.com/en/actions/configuring-and-managing-workflows/authenticating-with-the-github_token#using-the-github_token-in-a-workflow).

So in summary I can just add new posts to my blog as `.md` files, push them to `master` and GitHub will take care of building my static site and redeploying it all within a matter of seconds. And all happening automatically via GitHub Actions and GitHub Pages 🚀

![magic](https://cldup.com/xKUxlqHIQc.png)
