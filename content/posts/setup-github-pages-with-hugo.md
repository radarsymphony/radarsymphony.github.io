---
title: "Setup Hugo Site With Github Pages"
date: 2023-01-29T10:00:23-08:00
draft: true
type: post
showTableOfContents: true
---

# Overview

This guide outlines the steps required to create a minimal blog like this one. This blog leverges [Github Pages](https://pages.github.com/) and [Hugo](https://gohugo.io/) to render markdown pages as blog posts and apply the [Gokarna]() theme. 

I will begin with setting up an account on github.com and run through to deploying the site via a github workflow.

## Requirements

- Familiarity with CLI
- Ability to install `hugo` and `git`
- Text Editor (I might look at a CMS in another post)

# Github Setup

1. [Login to github.com](https://github.com/login) or [Create a new account](https://github.com/signup).
2. Create a new repository. _(In the upper right corner, click the **'+'** > **New repository**.)_
3. Name the repository **[your_username].github.io** (e.g., _tombombadil.github.io_).
4. Verify repository is **Public**.
5. Click **Create repository**.

# Software

Before you can deploy your site, you will need to install `hugo` and `git`.

## Install Hugo

The installation instructions differ depending on the OS you are using. Please refer to Hugo's [official documentation](https://gohugo.com.cn/getting-started/installing/) for how to install `hugo`.

Your process might look something like:
1. Open a terminal.
2. Install the package with `sudo pacman -S hugo` or `sudo apt-get install hugo`. 
3. Confirm `hugo` is installed `hugo version`.

There is a [Docker image](https://hub.docker.com/r/klakegg/hugo/) available, if you'd prefer to add an alias to your `.bashrc`:

```
alias hugo='docker run --rm -it -v $(pwd):/src klakegg/hugo:latest shell'
```

## Install Git

The installation instructions differ depending on the OS you are using. Please refer to Git's [official documentation](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) for how to install `git`.

Your process might look something like:
1. Open a terminal.
2. Install the package with `sudo pacman -S git` or `sudo apt-get install git`. 
3. Confirm `git` is installed `git --version`.

# Setup Local Project

1. Open a terminal.
2. Navigate to the folder that will store your blog's files, e.g., `${HOME}/src/github.com/`
3. Run `hugo new site [your-site-name]`.
4. Navigate into your project `cd !!`.
5. Run `hugo new posts/hello-world.md` to create a new blog post.

Also consider choosing a [Hugo theme](https://themes.gohugo.io/). This site uses using [Gokarna](https://github.com/526avijitgupta/gokarna).

## Git (Version Control)

You already installed `git`. To track changes to our source code and make deploying to github pages easy, let's turn our project folder into a git repo.

1. From within your project folder, run `git init`.
2. Add a theme to your project `git submodule add https://github.com/526avijitgupta/gokarna.git themes/gokarna`.

## Hugo Theme (Submodules)

# Developing Site
To launch Hugo's built-in server, navigate to the project folder and run:

```
hugo server 
```

Alternatively, I prefer running the `hugo server` with a few additonal options. 

```
hugo server --environment staging --buildDrafts --navigateToChanged
```
You can find an explanation of these options and additional options for the server [here](https://gohugo.io/commands/hugo_server/#options).

# Github Workflow

# Deploying Site

# Setup Custom Domain

# Resources
- [favicon.io](https://favicon.io/favicon-generator/)
- https://github.com/526avijitgupta/gokarna
- https://blog.hellohuigong.com/en/posts/how-to-build-personal-blog-with-github-pages-and-hugo/
