# rayscenens.github.io

A website for racineennis.ca

## Prerequisites

Under ubuntu the following packages are required.
make
gcc
g++
zlib1g-dev

## Environment Setup

This website is built using the Jekyll static site generator.

You will need to install the Ruby devkit,

* on Windows: The easiest way is to use the [Ruby Installer](https://rubyinstaller.org).
* on Linux: `sudo apt install ruby-dev  libxslt-dev libxml2-dev zlib1g-dev`

Validate that ruby is installed with `ruby --version` the following command.

Install bundler (allow network access if prompted)

```PowerShell
gem install bundler
```

Install Jekyll and any dependencies.

```PowerShell
bundle install
```

To execute the Jekyll service.

```PowerShell
bundle exec jekyll serve --watch
```
