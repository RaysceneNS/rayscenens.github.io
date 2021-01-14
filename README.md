# rayscenens.github.io

A website for racineennis.ca

## Environment Setup

This website is built using the Jekyll static site generator.

You will need to install the Ruby devkit,

* on Windows: The easiest way is to use the [Ruby Installer](https://rubyinstaller.org).
* on Linux: `sudo apt install make gcc g++ zlib1g-dev ruby-dev libxslt-dev libxml2-dev`

Validate that ruby is installed with `ruby --version` the following command.

Install bundler (allow network access if prompted)

```bash
gem install bundler
```

Install Jekyll and any dependencies.

```bash
bundle install
```

To execute the Jekyll service.

```bash
bundle exec jekyll serve --watch
```
