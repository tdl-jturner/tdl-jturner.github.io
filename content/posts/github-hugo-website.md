+++
draft = false
date = 2019-08-25T16:44:58-07:00
title = "Hugo and GitHub Website"
+++

GitHub offers [GitHub Pages](https://pages.github.com/) for repositories, users, and organizations. I used this recently in conjunction with [Casquatch](https://tmobile.github.io/casquatch) to host a new [Hugo](https://gohugo.io/) based manual. Today, I decided to use the same technology to setup a personal profile page and what better way to start off the blog than discussing the creation of that blog.

## Prerequisites

The following technologies will be used

* [Git](https://git-scm.com/) - Client for source code repo
* [GitHub](https://github.com) - Source Code Repository
* [Hugo](https://gohugo.io/) - Static Website Framework
* [Coder Theme](https://github.com/luizdepra/hugo-coder/) - Theme for Hugo but this could easily be anything

Before starting, please make sure Git and Hugo are installed and functioning. If not, please refer to [Installing Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) and [Installing Hugo](https://gohugo.io/getting-started/installing/) respectively.

Also note that you must replace USERNAME with your GitHub username in the examples below.

## Creating a Hugo Website on GitHub Pages
First, create a new public repository at [GitHub](https://github.com/new) with the name USERNAME.github.io. This repository will be automatically tied and available at http://USERNAME.github.io when we are done.

Setup repository with an initial commit
```
echo "Initial Commit" >> README.md
git init
git add README.md
git commit -m "Initial Commit"
git remote add origin https://github.com/USERNAME/USERNAME.github.io.git
git push -u origin master
```

Setup source branch and worktree. This allows us to maintain two separate branches. Master will contain the generated website while source will be the Hugo source files. In addition, I use git worktree to set the public folder (where Hugo places the files) to be for the master branch.
```
git checkout -b source
git push --set-upstream origin source
git worktree add public master
```

Add the theme module
```
git submodule add https://github.com/luizdepra/hugo-coder.git themes/hugo-coder
```

Copy the theme's default config.toml. Modify it if desired.
```
cp themes/hugo-coder/exampleSite/config.toml .
```

Create a simple content folder with ```mkdir content```. (This is the only required folder).

Create a starter About with ```hugo new about.md``` and edit as desired. At minimum, you need to set draft = false.

Similarly blog posts can be created with ```hugo new posts/myfirstpost.md ```. In addition to setting draft = false, it will need a title to make it show up on the Blogs page.

Commit the new content with:
```
git add content
git commit -m "Placeholder content"
```

Test it locally with ```hugo serve```. View your new About and Blog from the links.

Generate and push the website with:
```
hugo -t hugo-coder
cd public
git add -f .
git commit -m "Updated From Source"
git push
```

View your new page at https://USERNAME.github.io

## Custom Domain
In addition, you can even configure this to use a [custom domain](https://help.github.com/en/articles/using-a-custom-domain-with-github-pages). Implementation varies by your chosen hosting provider but in general you would do the following steps.

In your DNS provider, add the following four A Records:
```
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

Wait for the existing records to time out. This can be tested with:
```
dig +noall +answer YOUR_WEBSITE.com
```

When this has processed, you will see the A records show up here and look like:
```
;YOUR_WEBSITE.com.
YOUR_WEBSITE.com.   3600  IN  A 185.199.108.153
YOUR_WEBSITE.com.   3600  IN  A 185.199.109.153
YOUR_WEBSITE.com.   3600  IN  A 185.199.110.153
YOUR_WEBSITE.com.   3600  IN  A 185.199.111.153
```

Update the CNAME in Github Settings by going to [https://github.com/USERNAME/USERNAME.github.io/settings] then scrolling down to "Custom domain" (Near the end) and set this to your desired domain. This creates a CNAME file containing the domain name to register your repository.

After a few minutes you should see your new hugo website showing up at YOUR_WEBSITE.com and https://USERNAME.github.io redirecting to YOUR_WEBSITE.com
