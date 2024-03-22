---
title: "Hugo for Beginners"
date: 2018-08-31T19:04:23+10:00
draft: false
---

I wanted to put together some notes on how to set up a [Github Pages](https://pages.github.com/) blog using the [Hugo](https://gohugo.io/) static pages publishing application as when I went through this process myself, I found it less than intuitive.

Github Pages is a great way to spin up a personal blog, product page or any other type of site that doesn't require stateful application logic to be involved.

The defacto static page generator is considered to be Jekyll, which is a versatile, and extensible framework. However, being a fan of Golang I decided to go with the more gopher-friendly option, Hugo.

# Using Github Pages

The way Github Pages works is to use a repo with the format:

`<username/org>.github.io`

In effect this means that a repo at something like the the following address is created:

`https://github.com/<username/org>/<username/org>.github.io`

For example, my username is `wilvk` and so my Github Pages repository is:

`wilvk/wilvk.github.io`

with the repository address:

`https://github.com/wilvk/wilvk.github.io`

The corresponding static pages are delivered from the address:

https://wilvk.github.io

If you have your own domain name, this can be integrated into Github Pages by going to the Settings section of the repository.

# An overview

The way I set up my blog was to have one repository for the static pages to be delivered by Github Pages, and another for generating the static pages using Hugo. I have a script for generating the site and pushing the changes to the Github Pages site so I don't have to do this manually every time.


![hugo build diagram](http://me.wvk.au/img/hugo_blog.svg)


We can see from the diagram that the build clones the `wilvk/wilvk.github.io` repository inside the `wilvk/blog-hugo` repository to the `./public` path. It then builds the full Hugo site into the `./public` path and pushes the changes back to the `wilvk/wilvk.github.io` repository.

# An automated process

I place the following script in the base of the `wilvk/blog-hugo` repository and run it to push the changes to the site:

```bash
#!/bin/bash

set -x

rm -rf ./public
git clone https://github.com/wilvk/wilvk.github.io public
hugo
cd ./public
git add .
git commit -m "build release"
git push --set-upstream origin master
cd ..
rm -rf ./public
```

The `set -x` allows me to see any errors that may occur during the build process. Using different themes or various settings can cause Hugo to fail the build process and not push new changes to the static website repository `wilvk.github.io`. Showing the full output enables inspection of this process for any errors.

Hugo can also be run with verbose output with `hugo --verbose`, or even `hugo --debug` for greater levels of detail in the build process if required.

The static site is built to the path `./public` and so the first thing the script does is delete the `./public` path.

Then the `wilvk/wilvk.github.io` repository is cloned into the base of the `blog-hugo` repository.

Running `hugo` builds all the static files to the `./public` path.

The new static website is then 'pushed' back up to the repository `wilvk.github.io` and the `./public` path deleted so that is is not accidentally committed to the `blog-hugo` repository. It may be useful sometimes to comment this line out if investigating why a build didn't complete but most of the time it is useful to have it delete the path.

This way there is delineage between the build process and the resultant artefact, the static website.

# Configuring config.toml

Hugo has built-in support for [Google Analytics](https://analytics.google.com/analytics/web/#/) so you can see pageviews over time, users currently viewing your site and other relevant details about your site.

Hugo also has support for [Disqus](https://disqus.com/) so that users can comment on your blogs and you don't have to store the comments in an application - allowing your website to remain stateless.

Google Analytics is free from Google and Disqus is also a free service. With IDs from both these services, you can turn your Github Pages into a proper blog.

The following is how I have set up my `config.toml` file:

```
baseurl = "http://me.wvk.au"
title = "Will's Blog"
copyright = "Copyright &copy; 2018 - Willem van Ketwich"
canonifyurls = true
theme = "kiera"

## kiera theme settings
paginate = 3
summaryLength = 30
enableEmoji = true
pygmentsCodeFences = true

## Hugo Built-in Features
disqusShortname = "wilvk-github-io"
googleAnalytics = "UA-122351489-2"
enableRobotsTXT = true

builddrafts = false
languageCode = "en-US"

[author]
    name = "Willem van Ketwich"
    github = "wilvk"
    linkedin = "willvk"
    twitter = "wilvk"

[params]
    tagline = "Just a dude that does stuff.. mostly with computers."

## Main Menu
[[menu.main]]
    name = "blog"
    weight = 10
    identifier = "blog"
    url = "/posts/"
[[menu.main]]
    name = "about"
    identifier = "about"
    weight = 20
    url = "/about/"
```

I'll just point out a few things about the config file above. 

- Your Google Analytics ID will go in the variable `googleAnalytics`.
- Your Disqus shortname ID will go in the variable `disqusShortname`.

I am also using a custom domain name and so this is set in the variable `baseurl`. The variable `cannonifyurls` needs to be set to `true` to enable correct redirects to your custom domain name.

# Skinning/theming your site

If you are just starting out with Hugo (or even if you're not), you may want to change the theme for your site.

To do this, the usual procedure is to:

- Clone the theme into the `./themes` path.
- Edit the `config.toml` file and set the theme to the name of the directory cloned into the `./themes` path.

Some themes require specific config settings in the `config.toml` file. They will usually tell you in the `README.md` on the specifics of this.

# Onwards and upwards

These are just a few things I have found while setting up a Hugo blog. For more info, I have found the [Hugo docs](https://gohugo.io/getting-started/usage/) are also quite useful.

Happy blogging!
