+++
title = "Hugo on GitHub Pages with Travis CI"
date = "2018-12-11T15:00:40+01:00"
tags = [ "github", "hugo", "travis", "ci" ]
+++

I [already wrote][Using travis to deploy hugo site to a shared hosting] about deploying [Hugo] websites to the shared hosting by using [Travis CI]. [GitHub Pages] support HTTPS for custom domains [from May 2018](https://blog.github.com/2018-05-01-github-pages-custom-domains-https/), so now it's worth to have a look at this topic again.

This blogpost describes the setup of deploying [Hugo] with [Travis CI] to the [GitHub Pages].

<!--more-->

# Preparation steps

1. [Install hugo](https://gohugo.io/getting-started/installing/) on your local system, you might need it in order to build the site locally or test it
1. Create a bot user account on GitHub (yes, they are [allowed](https://help.github.com/articles/differences-between-user-and-organization-accounts/), have a look to the Tips). This account will get the write/push rights to the repository with GitHub Pages content.

# Create repository or repositories

I assume you are going to setup a user or organisation page, so you would use a repository named like `<user/org name>.github.io` for your HTML content.

Starting from here there are two possible ways:

1. to use one repository for sources (hugo repository) and publishing (generated HTML)
1. to use two repositories: one for sources and another for publishing

Main differences between this two approaches:

- one or two repositories
- if your bot user would have write permissions to the source repository

I personally prefer the second approach, but some people like the first one because of 'as-few-repos-as-possible' perspective, which is also valid.

So, if you have chosen the first approach:

- please create the `<user/org name>.github.io` repository. Allocate two branches: `master` and `source` and configure the `source` branch as [protected branch](https://help.github.com/articles/configuring-protected-branches/)
- `master` branch will be used for hosting of generated HTML
- `source` branch will play a 'master' branch for the sources of your page
- Add your bot account as [collaborator] for the repository with write permissions

If you have chosen the second approach with two repositories:

- please create two repositories: `<user/org name>.github.io` and something like `homepage` (name doesn't really matter)
- Create `master` branches in both repositories
- Add your bot account as [collaborator] for the `<user/org name>.github.io` repository with write permissions

The collaborators page with write permissions should look like below:
![GitHub collaborators page](bot-account-credentials.png)

# Create content

Create hugo source content, commit and push it either to the `source` branch of your `<user/org name>github.io` repository or to the `master` branch of `homepage` repository. Please have a look to [my previous post](/post/2017/05/using-travis-to-deploy-hugo-site-to-a-shared-hosting/#add-the-first-content) for an example.

# Put your secrets to travis settings

In order to get the HTML content pushed to GitHub Pages, you have to [create a secret variable](https://docs.travis-ci.com/user/environment-variables/#defining-variables-in-repository-settings) in the Travis CI settings of your repository, which contains hugo source files. This variable should be called `GITHUB_AUTH_SECRET` and it should contain authentication string in the following syntax:

```text
https://<BOT USERNAME>:<BOT GITHUB PASSWORD>@github.com
```

It should look like this:

![Travis secret variables](travis-secret-settings.png)

# Create travis configuration

Add deployment script `deploy.sh` to your source repository and update the placeholders with your data. Please use the GitHub HTTPs repository URL of your publishing repository, There is no difference for single or two-repository approach.

```bash
#!/bin/bash

set -e

echo $GITHUB_AUTH_SECRET > ~/.git-credentials && chmod 0600 ~/.git-credentials
git config --global credential.helper store
git config --global user.email "<GITHUB LOGIN OF BOT>@users.noreply.github.com"
git config --global user.name "My cool bot"
git config --global push.default simple

rm -rf deployment
git clone -b master https://github.com/<GITHUB HTTPS PATH TO YOUR PUBLISHING REPO> deployment
rsync -av --delete --exclude ".git" public/ deployment
cd deployment
git add -A
# we need the || true, as sometimes you do not have any content changes
# and git woundn't commit and you don't want to break the CI because of that
git commit -m "rebuilding site on `date`, commit ${TRAVIS_COMMIT} and job ${TRAVIS_JOB_NUMBER}" || true
git push

cd ..
rm -rf deployment
```

Create Travis CI configuration: add following `.travis.yml` file to your source repository:

```yaml
---
install:
  - wget -O /tmp/hugo.deb https://github.com/gohugoio/hugo/releases/download/v0.52/hugo_0.52_Linux-64bit.deb
  - sudo dpkg -i /tmp/hugo.deb

script:
  - hugo

deploy:
  - provider: script
    script: ./deploy.sh
    skip_cleanup: true
    on:
      branch: source # or master if you are using the two-repository approach
```

In the above configuration we always build the site with hugo and do the deployment step only on master.

# Bonus: preview of PRs of your site

You can even get previews build from your Pull Requests, if you would click 'Details' you will be forwarded to a temporary web site with PR preview:

![Netlify preview](netlify.png)

This can be done with [netlify](https://www.netlify.com). See [this](https://github.com/dev-sec/dev-sec.github.io/blob/8a1fee630eeb281d0c1305010617daaae8dc02be/netlify.toml) as configuration example.

# See too

- [Using travis to deploy hugo site to a shared hosting]
- [Hugo - Host on GitHub](https://gohugo.io/hosting-and-deployment/hosting-on-github/)
- [Netlify docs](https://www.netlify.com/docs/)

[collaborator]: https://help.github.com/articles/inviting-collaborators-to-a-personal-repository/
[GitHub Pages]: https://pages.github.com/
[Hugo]: https://gohugo.io/
[Travis CI]: http://travis-ci.org/
[Using travis to deploy hugo site to a shared hosting]: /post/2017/05/using-travis-to-deploy-hugo-site-to-a-shared-hosting/
