---
title: "Getting started with GitHub Actions"
date: 2020-07-05T12:00:00+01:00
---

![actions](https://cldup.com/oeFfVNoznL.png)

I want to share with you my first experience working with GitHub Actions. They're really neat and definitely worth your time if you're a fan of automation.

## Background

This weekend I was working on a small personal project: a [GitHub PR Comment resource for Concourse CI](https://github.com/mtharrison/github-pr-comment-resource). That isn't what this post is about really but first a little context.

The project I was building is a Go project. When built, it consists of 2 built binaries `check` and `in`. These are part of the standard interface for any Concourse resource. A resource should just be a Docker image where those 2 binaries exist in a specific directory. So unsurprisingly my project contains a `Dockerfile` to build said image.

I'll be honest and admit that I only really "got" and starting appreciating CI and build automation when I started working at my current gig. Probably because before that I was either writing personal projects or pretty much working within a non-technical team where I was the only one regularly writing code and so it made sense to run tests, produce builds and run deployments from my own machine. I was lucky enough not to run into the sort of issues that CI and centralised build systems solve.

I compare that to the aha moment I got when I first understood unit tests. Releasing code without unit tests is unthinkable to me now but (embarrassingly) I remember a point where they seemed like a frivolous and pointless luxury to me.

After working on a largish team for several years now, I can't imagine not having all the automated goodness that comes along with a well configured CI system. I'm talking about:

- Automated build/test checks on PR/commit
- Automated release of merge to master: docker images/npm packages build automatically

I can't bear the idea of having to run `docker build ...` on my own laptop anymore even for personal projects after having these tools at work. I want to focus on the code and let the machines do the grunt work for me.

A few years back I would have turned to TravisCI for this and that was my natural instinct here. However as I'm not that experienced with Go I thought I'd see what the Go community was doing. I saw the expected mix of CircleCI/Travis/Gitlab being used. But then I saw a few projects that were making use of GitHub actions.

I was aware of GitHub actions, I'd read the announcement via Hacker News and heard some mutterings at work but I'd filed it to the mental folder of things I should look at one undetermined day.

I thought perhaps this is a good time to get my feet wet as I only have fairly simple needs for now:

1. Test code on every PR against `master` and every push to `master`
2. When a new tag is pushed, test and then build and push the Docker image to docker hub

I'm a fan of Semantic Release, where even the tag step is redundant. Semantic Release will bump semver tags based on commit messages. However for now I'm happy with just tagging and pushing tags as I choose, provided the rest is all automated for me.

## Creating a GitHub Action

You add a new _workflow_ to your repository by creating a `yml` file under `.github/workflows`. You can think of a _workflow_ like a pipeline. It's a set of steps that run on response to some _event_ like a push to a branch, a new PR or many others.

There's a [ton of events you can trigger on](https://docs.github.com/en/actions/reference/events-that-trigger-workflows), which is probably going to be more exhaustive than anything you'll find in another CI system, being that Actions are a GitHub feature.

Your workflow runs in a _runner_. Think of this just like a linux container that is hosted by Github. The images have a [lot of common software](https://github.com/actions/virtual-environments/blob/master/images/linux/Ubuntu2004-README.md) installed that you might want to use (like `npm`, `git` etc).

Each step in the _workflow_ can either run a shell command, or can make use of an _app_. Apps are published by the community on [Github Marketplace](https://docs.github.com/en/actions/getting-started-with-github-actions/using-actions-from-github-marketplace), including some official apps and are prepackaged workflow behaviours.

### A workflow to test master branch and PRs

I want to make sure my master branch is always passing tests so I can display a nice green badge on the README, so contributors and users know that my project is in a good state.

This was really simple to set up, check out the annotated workflow yaml:

[_/.github/workflows/master.yml_](https://github.com/mtharrison/github-pr-comment-resource/blob/master/.github/workflows/master.yml)

```yaml
name: master                     # name of this workflow

on:
  push:
    branches: [master]           # only trigger on pushes to master

jobs:
  test:
    name: test
    runs-on: ubuntu-latest
    steps:
    - name: Set up Go 1.*
      uses: actions/setup-go@v2
      with:
        go-version: ^1.14

    - name: Check out code
      uses: actions/checkout@v2

    - name: Test code
      run: go test -v ./...
```

This workflow sets up my desired version of Go, checks out my code and then runs to the `go test` tool. Once this is committed to my repo any future matching events will trigger the Action. Providing feedback in all the usual places on Github.

![commit status](https://cldup.com/EvWOhUxeIQ.png)

You get the full workflow output in the GitHub UI similar to Travis. It's nice not having a separate place to go to check that out.

![workflow output](https://cldup.com/qSicJvI5A4.png)

You can easily grab a badge image to display in your README from the workflows screen.

![badge](https://cldup.com/c1Fn-7dnFN.png)

I also added another [workflow for PRs](https://github.com/mtharrison/github-pr-comment-resource/blob/master/.github/workflows/pr.yml) which is basically an exact duplication of the above with the `on` section only slightly different:

```yaml
on:
  pull_request:
    branches: [master]
```

I could have used both triggers in the same workflow but I like having a separate place to view PR builds. I'm not sure if there's a way to share steps between workflows but for now this but of duplication doesn't trouble me too much.

## A workflow to release Docker image

The other requirement I had is that I want a new tag to result in a Docker image to be built and pushed to the Docker registry and for the latest version to be displayed someone on the repo home page so anybody viewing knows which tag to use in their Concourse pipelines.

This was a bit more work than the master workflow but not by too much. I used an extra step to set some variables so they can be reused in other steps: namely the image name and the actual tag being released.

There's a funky way to set these variables by echo'ing a string that will be recognised by the runner:

```
echo ::set-output name=tag::$(echo ${GITHUB_REF#refs/*/})
```

It does feel a bit awkward but seems to work pretty well. The Docker cli is available within the Runner so I was able to just login to Docker hub using `docker login` and passing my username and password using credentials which are set inside your repository settings Secrets.

![repo secrets](https://cldup.com/-9ZSIo-KUu.png)

 Also I could just use `docker push` to push the image. Here's the entire workflow definition:

[_/.github/workflows/release.yml_](https://github.com/mtharrison/github-pr-comment-resource/blob/master/.github/workflows/release.yml)

```yaml
name: release

on:
  push:
    tags:
    - '*'

jobs:
  release:
    name: release
    runs-on: ubuntu-latest
    steps:
    - name: Set up Go 1.x
      uses: actions/setup-go@v2
      with:
        go-version: ^1.14
      id: go

    - name: Check out code
      uses: actions/checkout@v2

    - name: Test code
      run: go test -v ./...

    - name: Set some variables
      id: vars
      run: |
        echo ::set-output name=tag::$(echo ${GITHUB_REF#refs/*/})
        echo ::set-output name=image::mtharrison/github-pr-comment-resource

    - name: Login to DockerHub
      run: echo ${{ secrets.DOCKERHUB_PASSWORD }} | docker login -u ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin

    - name: Build the tagged Docker image
      run: docker build . -t ${{steps.vars.outputs.image}}:${{steps.vars.outputs.tag}}

    - name: Push the tagged Docker image
      run: docker push ${{steps.vars.outputs.image}}:${{steps.vars.outputs.tag}}

    - name: Create Release
      id: create_release
      uses: actions/create-release@v1
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      with:
        tag_name: ${{ github.ref }}
        release_name: Release ${{ github.ref }}
        draft: false
        prerelease: false
```

You'll notice that I used another _app_ `actions/create-release@v1` to publish a new release right at the end. This might seem like a pointless step because my release artifact is really the Docker image on Docker hub and I already have a tag. However, a broken build (due to failed image push maybe) will prevent this release from happening so the presence of a release is indication of the latest working version for which you can be sure to find a docker image.

![release](https://cldup.com/LCj1GWFMXm.png)

Also I can add a nice release badge to my README courtesy of the amazing badges at [shields.io](https://shields.io/):

![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/mtharrison/github-pr-comment-resource)

## Conclusion

I was impressed how straightforward and painless it was to migrate from my usual Travis workflow to using GitHub actions. The functionality, for my modest needs seems pretty much on par. The community potential via marketplace seems really promising too. The opportunity to monetise them seems like another positive step in making open source contributions financially rewarding.

I can see that GitHub actions may become the standard for small to medium open source projects.

I'm definitely going to dive deeper into them in the coming months and might even write a few apps of my own.

All this positivity aside, I still have a slight concern about the growing monopolising grip over open source that Microsoft seem to have and what this means for well established but less rich competitors in the market. However, I think the convenience of having automation built into GitHub is good news for the quality of open source and I'm all for anything that moves us forward to creating higher quality and more secure software!

## Resource

If you want to play with Actions I would recommend starting reading some of the materials here:

- [https://docs.github.com/en/actions](https://docs.github.com/en/actions)
- [https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions](https://docs.github.com/en/actions/reference/events-that-trigger-workflows)
- [https://docs.github.com/en/actions/reference/events-that-trigger-workflows](https://docs.github.com/en/actions/reference/events-that-trigger-workflows)
