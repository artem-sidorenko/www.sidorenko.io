# homepage

This repository contains the source for the internet blog [www.sidorenko.io](https://www.sidorenko.io)

## Installation of hugo

In order to have a preview of posts, you should have hugo installed. There are two possible installation ways:

- you can download hugo from the [offical releases page](https://github.com/spf13/hugo/releases) (binaries and deb packages)
- you can use the hugo deb and rpm packages from [this OBS repo](https://software.opensuse.org//download.html?project=home%3Aartem_sidorenko&package=hugo)

## Adding new content

```bash
$ hugo new post/2017/01/testpost.md
```

## License

Content of this blog is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

## Vendored foreign code

This repository contains hugo theme [hugo-bootstrap-premium](https://github.com/appernetic/hugo-bootstrap-premium/). This theme is vendored via [git subtree](https://www.atlassian.com/blog/git/alternatives-to-git-submodule-git-subtree) in the folder `themes/bootstrap-premium` and is licensed under MIT license.

## Contributing

I just share the sources of my blog for the case somebody needs them. No collaboration via Pull Requests is expected, please do not open PRs. Feel free to open issues if you have any questions.
