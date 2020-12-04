---
layout: post
title: Jekyll to Setup Github Pages
subtitle: Setup your own blog 
header-style: text
author: Hanke
tags: [Github, Gem, Jekyll]
---

This is the track for local setup for the github pages.

Before that suppose you already have the github.io local github repository.
For Jekyll case , please refer the [guideline](https://github.com/poole/poole#usage).

### Prepare Env
#### Install Gem

#### Install Bundle
```bash
sudo gem install bundler
bundle init
sudo gem install -n /usr/local/bin/ jekyll
```

#### Error Met
```bash
  Dependency Error: Yikes! It looks like you don't have jekyll-paginate or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. If you've run Jekyll with `bundle exec`, ensure that you have included the jekyll-paginate gem in your Gemfile as well. The full error message from Ruby is: 'cannot load such file -- jekyll-paginate' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/!
                    ------------------------------------------------
      Jekyll 4.1.1   Please append `--trace` to the `serve` command
                     for any additional information or backtrace.

```

```bash
  Dependency Error: Yikes! It looks like you don't have jekyll-gist or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. If you've run Jekyll with `bundle exec`, ensure that you have included the jekyll-gist gem in your Gemfile as well. The full error message from Ruby is: 'cannot load such file -- jekyll-gist' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/!
                    ------------------------------------------------
      Jekyll 4.1.1   Please append `--trace` to the `serve` command
                     for any additional information or backtrace.
                    ------------------------------------------------

## Install Dependencies
```bash
gem install jekyll jekyll-gist jekyll-sitemap jekyll-seo-tag
```

#### Edit the Local Gemfile
```bash
gem "jekyll", "~> 4.1.1"
gem "jekyll-paginate"
gem "jekyll-gist"
```

### Select Theme
[Hux Theme](https://github.com/Huxpro/huxpro.github.io)

Clone the code and modify the config according to the readme [instructions](https://github.com/Huxpro/huxpro.github.io/blob/master/_doc/Manual.md)

#### Run Locally
```bash
bundle exec jekyll serve
```

#### Reference
[github jekyll integration guide](https://docs.github.com/cn/free-pro-team@latest/github/working-with-github-pages/testing-your-github-pages-site-locally-with-jekyll)

-----

Want to see something else added? <a href="https://hanke.github.io/">Open an issue.</a>
