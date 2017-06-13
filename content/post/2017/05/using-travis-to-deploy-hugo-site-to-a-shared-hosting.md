+++
date = "2017-05-27T18:00:00+02:00"
tags = [ "github", "hugo", "travis", "ci" ]
title = "Using travis to deploy hugo site to a shared hosting"
+++

[GitHub Pages] are quite popular for hosting static sites built by site generators.
However, GitHub Pages have some limitations:

- no SSL/TLS for custom domains
- proper support of Jekyll sites only

One possible alternative is to use [GitLab Pages], which does not have this limitations.
Another possible alternative is to use [Travis CI] and deploy the site to some shared www hosting. Hetzner offers [some good plans], they also include a free-of-charge SSL certificate.

This blogpost describes how to deploy a static site hosted on [GitHub], built with [Hugo] and [Travis CI] and deployed via FTPS to Hetzner www space.

<!--more-->

# Preparation steps

1. [Install hugo](https://github.com/spf13/hugo/releases) on your local system, you might need it in order to build the site locally or test it
1. Create some GitHub repository for your site
1. [Enable Travis for this new repository](https://docs.travis-ci.com/user/getting-started/)

# Add the first content

Lets create a first commit in our new repository

```bash
# git clone or git init a new repository
# create a first commit
$ echo 'this is my hugo site' > README.md
$ git add README.md
$ git commit -m "first commit"
```

Add some hugo theme [as subtree](https://www.atlassian.com/blog/git/alternatives-to-git-submodule-git-subtree), we use the [hugo-future-imperfect](https://themes.gohugo.io/future-imperfect/) theme here:

```bash
$ mkdir themes
$ git subtree add -P themes/hugo-future-imperfect https://github.com/jpescador/hugo-future-imperfect.git master
```

Create a hugo configuration:

```bash
$ cat > config.toml <<EOF
languageCode = "en-us"
title = "Demo hugo site"
baseurl = "http://example.com"
theme = "hugo-future-imperfect"

[params]
    # Sets the meta tag description, usually reserved for the main page
    description          = "Some description"
    # This will appear on the top left of the navigation bar
    navbarTitle          = "Cool title"

[params.intro]
    header    = "This is a header"
    paragraph = "This is a paragraph"
    about     = "Lorem ipsum dolor sit amet..."

[params.postAmount]
    sidebar = 2

[[menu.main]]
    name = "Blog"
    url = "/blog"
    identifier = "fa fa-newspaper-o"
    weight = 1

[[menu.main]]
    name = "Categories"
    url = "/categories"
    weight = 2

[[menu.main]]
    name = "About"
    url = "/about"
    weight = 3
EOF
```

Add the first content:

```bash
$ hugo new blog/test.md
$ cat > content/blog/test.md <<EOF
+++
title = "Example blog post"
categories = ["test"]
description = "Description of example blogpost"
author = "author"
type = "post"
date = "2017-02-18"
+++

I'm just an example blog post

EOF
# run the internal hugo server and open a browser to see the site
$ hugo server
```

# Put your secrets to travis settings

After you enjoyed your first content with hugo, put the deployment credentials in Travis, in the settings of your project. This credentials contain the FTP data for deployment and Travis will configure them as environment variables in the CI jobs on the base repository (they are not visible to the forks).

![Travis Project Settings](travis-settings.png)

# Create travis configuration

Add deployment script `deploy.sh` to your repository:

```bash
#!/bin/sh
# deploy.sh
set -e

sudo apt-get install -y lftp

# deployment via ftp upload. Using FTPS for that
lftp -c "set ftps:initial-prot ''; set ftp:ssl-force true; set ftp:ssl-protect-data true; open ftp://$FTP_USER:$FTP_PASS@$FTP_HOST:21; mirror -eRv public .; quit;"
```

And add travis configuration `.travis.yml`:

```yaml
# .travis.yml
dist: trusty
sudo: required

install:
 - wget -O /tmp/hugo.deb https://github.com/spf13/hugo/releases/download/v0.21/hugo_0.21_Linux-64bit.deb
 - sudo dpkg -i /tmp/hugo.deb

script:
 - hugo

deploy:
  - provider: script
    script: ./deploy.sh
    skip_cleanup: true
    on:
      branch: master
```

In the above configuration we always build the site with hugo and do the deployment step only on master.

Optionally you can also add `.htaccess` with http-https redirection if you have already configured SSL for your site:

```text
# static/.htaccess
RewriteEngine on
# Redirect from http to https
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}/$1 [R=301,L]

# Add HSTS headers to the https connections
Header set Strict-Transport-Security "max-age=10886400; includeSubDomains" env=HTTPS
```

Now push everything to GitHub to the master branch and you should see Travis deploying your website.

[GitHub Pages]: https://pages.github.com/
[GitLab Pages]: https://pages.gitlab.io/
[Travis CI]: http://travis-ci.org/
[some good plans]: https://www.hetzner.de/ot/hosting/produktmatrix/webhosting-produktmatrix
[GitHub]: https://github.com/
[Hugo]: https://gohugo.io/
