+++
title = 'Github Pages'
date = 2023-11-11T11:45:33-04:00
tags = [
    "tools",
    "ci/cd",
    "github",
]
+++

In an attempt to blog more I'm going to use hugo and have it setup to use github pages that get updated when the repo is updated. Pretty straight forward stuff. So let's just quickly documention setup so we don't forget... again
<!--more-->
## Local setup

`choco install hugo-extended`
Super simple

## Create Site

The [quick-start](https://gohugo.io/getting-started/quick-start) page will be you best bet for up-to-date install info but so you don't need to nav away

```bash
hugo new site quickstart
cd quickstart
git init
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
echo "theme = 'ananke'" >> hugo.toml
hugo server
```

## Create Post

`hugo new content posts/my-first-post.md`

## Better Post in PWSH

```pwsh
function Post([string]$Name){
    hugo new content posts/$Name.md
}                                                                                                                        
```

Can you can just `post new-post`

## Make post ready to publish

Remove `draft:true` from the header data of in the `.md` file

## Config Site

Edit hugo.toml

```toml
baseURL = 'https://openbracket/'
languageCode = 'en-us'
title = 'Openbracket'
theme = 'ananke'
```

## Config git for remote

In the root of your site dir

`git remote add origin git@github.com:you_awesome_site/here.git`

## Setup .github workflow

Check [here](https://gohugo.io/hosting-and-deployment/hosting-on-github/) for hugo's docs.

but...

In guthub go to Settings > Pages.

Change `Build and Deployment` `Source` to `Github Actions`.

Add [.gitflow/workflows/hugo.yaml](https://github.com/bardic/obsite/blob/main/.github/workflows/hugo.yaml) to your repo.

## PUSH

Now do a push of all those changes.  You sshould see on your github repo page under `Actions` that a job is running. Once its done you github pages should be populated

## New Posts

Using the pwsh alias above it's as easy as `post new-idea` and pushing to your repo
