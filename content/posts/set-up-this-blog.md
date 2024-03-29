---
title: "How To Set Up This Blog"
date: 2023-02-05T18:00:23-08:00
draft: false
type: post
showTableOfContents: true
tags: ["how-to", "hugo", "github", "pages", "blog", "markdown" ]
---

# Overview

This guide outlines the steps required to create a minimal blog like this one. This blog leverages [Github Pages](https://pages.github.com/) and [Hugo](https://gohugo.io/) to render [markdown](https://www.markdownguide.org/) pages as blog posts and apply the [Gokarna](https://github.com/526avijitgupta/gokarna) theme. 

The steps will begin with setting up an account on [github.com](https://github.com) and run through to deploying the site via a github workflow. I may create another post about adding the ability for your readers to comment on a post.

## Requirements

- Familiarity with using a terminal
- Ability to install `hugo` and `git` on your machine
- Text Editor (I may look at a adding a CMS, like [Netlify](https://www.netlifycms.org) in another post.)

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

Also consider choosing a [Hugo theme](https://themes.gohugo.io/). This site uses [Gokarna](https://github.com/526avijitgupta/gokarna).

## Add Git 

You already installed `git`. To track changes to our source code and make deploying to github pages easy, let's turn our project folder into a git repo.

1. From within your project folder, run `git init`.
2. Add a theme to your project `git submodule add https://github.com/526avijitgupta/gokarna.git themes/gokarna`.

## Configure the Theme

In addition to [Hugo configuration options](https://gohugo.io/documentation/), each theme may have its own theme-specific options. If you chose the Gokarna theme, they have [great documentation on their example site](https://main--gokarna-hugo.netlify.app/posts/theme-documentation-basics/). You can also checkout [the configuration file](https://github.com/radarsymphony/radarsymphony.github.io/blob/main/config/_default/config.toml) for this site.

# Develop the Site

Pages can be written in pure HTML (change file extension to .html) or the much lighter simpler [Markdown](https://www.markdownguide.org/). To start adding pages, you would: 

1. Run `hugo new posts/[post-name-here].md`.
2. Open `[project_root]/content/posts/[post-name-here].md` to start editing the page. 

When you create a new page with `hugo`, it will add some default "metadata" at the top. This metadata sets the title of the page, the publish date, the publish date, whether the current page is a "draft" (drafts are not published). Hugo and the theme you chose will also allow you to add more metadata. The metadata for this page is: 

```yml
title: "How To Set Up This Blog"
date: 2023-02-05T18:00:23-08:00
draft: false
type: post
showTableOfContents: true
tags: ["how-to", "hugo", "github", "pages", "blog", "markdown" ]
```

You can also create "pages" as a `type`, include a Table of Contents, and add tags in Gokarna. See [Gokarna docs](https://main--gokarna-hugo.netlify.app/posts/theme-documentation-advanced/#content-types) for more info.

To see how your page will look when it's published, launch Hugo's built-in server.

1. Navigate to the project folder,
2. Run `hugo server`.

Alternatively, I prefer running the `hugo server` with a few additional options. 

```
hugo server --environment staging --buildDrafts --navigateToChanged
```
You can find an explanation of these options and additional options for the server [here](https://gohugo.io/commands/hugo_server/#options) on the Hugo site.

When you are satisfied with your page, change the `draft` status to "false" and run `hugo` in your project root, which will build all of the static files for your site and put them in the "public" directory.

# Deploy the Site

To make your site live, you have a few options. The quick way is to copy the `./public` folder (with the files generated by running `hugo`) and initialize a new git repo for that folder. You can would then add your github-pages repo (created above). 

The other option is to set up a github workflow that builds the site when you push to the main branch and places the built files in a separate branch from which to serve the site.

I use the second method for this site.

## Setup Github Workflows

While I found several articles that outlined how to set up the workflow, the easiest to follow was [the method on the Hugo website](https://gohugo.io/hosting-and-deployment/hosting-on-github/#build-hugo-with-github-action). Their instructions use the github action created by [peaceiris](https://github.com/peaceiris/actions-gh-pages).

I recommend following that guide to set up the workflow. It is clear and avoids the need to set up additional github SECRETS as it automatically generates a token for you. My first "push" using that workflow did not serve the site correctly. I had forgotten to [make a `gh-pages` branch and set it as the branch to serve the site from](https://gohugo.io/hosting-and-deployment/hosting-on-github/#github-pages-setting). Once I set the correct branch and re-deployed, the site was published as expected.

You will also need to add your github repository as your "remote" if you haven't done so yet. You can by: 

1. Navigate to the project root of your site,
2. `git remote add origin https://github.com/username/username.github.io.git`
3. Confirm `git remove -v`

```
origin  https://github.com/username/username.github.io.git (fetch)
origin  https://github.com/username/username.github.io.git (push)
```

Then, to deploy the site:

1. Run `git commit -am "[your message about the current changes]"`
2. Run `git push`

That's it! That's all you need for a simple blog site. You should now be able to view your site at `https://username.github.io`. If you'd like to add a custom domain, I will give some pointers in the remaining sections.

# Add Custom Domain

In order to direct a custom domain to your github pages, you will need a few things:

- A custom domain
- Access to that domain's DNS records (I used cloudflare)
- Time (updating the DNS records can take some time)

## Configure Github

The first step is to [add a custom domain to the github repository](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site?platform=linux#configuring-a-subdomain) hosting your site. To do that:

1. Navigate to your repo's settings.
2. Under **Code and automation**, Select **Pages**
3. Enter your custom domain and click save.

I'd recommend starting with a subdomain like "blog.example.com" or "www.example.com" as not all DNS providers will allow you to direct the apex domain to github pages.

In addition to adding the subdomain to your repository's settings, you want to "verify" your domain. I found [Github's documentation](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages) on how to do this quite clear, so I'd recommend following that guide.

## Configure DNS

1. Login to your DNS provider.
2. Add a CNAME record that points to [sub].example.com
3. (optional) Temporarily set low TTL to see changes more quickly as you are setting things up.

## Changes to Site files

In addition to setting things up on Github and with your DNS provider, you need to adjust some files in your repo. 

### CNAME

You need to create a CNAME file that will be served at your project root. Hugo creates your project root dynamically, so we can't add that file directly to `./public`. Instead, 

1. Navigate to your project root.
2. If you don't have a `./static/` directory, make it `mkdir ./static`
3. Add a file called `CNAME` (no extension) under `./static/`
4. Inside that file, add the full custom domain you added to you Github pages settings.
5. Save.

### config.toml

You will also need to update the base URL for your site. To do so,

1. Open your `config.toml` file (e.g., `./config/_default/config.toml`)
2. Update any references to your site's URL.
3. Save.

In Gokarna, you just need to update `baseURL` to equal the custom domain.

Commit your changes and push. Depending on how long your workflow takes and how long your DNS TTL is, you may have to wait a several minutes to see the final result.

# Software Used
- https://github.com/git/git
- https://github.com/gohugoio/hugo
- https://favicon.io/favicon-generator/
- https://github.com/526avijitgupta/gokarna
- https://github.com/peaceiris/actions-gh-pages
- https://github.com/radarsymphony/radarsymphony.github.io (source for this site)

# Resources
- https://blog.hellohuigong.com/en/posts/how-to-build-personal-blog-with-github-pages-and-hugo/
- https://dev.to/importhuman/deploy-hugo-website-using-github-pages-1mc5
- https://blog.mattdaines.me/p/adding-a-custom-domain-to-your-hugo-site-on-github-pages/
- https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages
- https://gohugo.io/hosting-and-deployment/hosting-on-github/#use-a-custom-domain
