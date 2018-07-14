# rayscenens.github.io

A website for racineennis.ca

## Environment Setup

This website is built using the Jekyll static site generator.

You will need to install the Ruby devkit on windows. The easiest way is to use the [Ruby Installer](https://rubyinstaller.org).

Validate that ruby is installed with the following command.

```PowerShell
ruby --version
```

Install bundler (allow network access if prompted)

```PowerShell
gem install bundler
```

Install jekyll and any dependencies.

```PowerShell
bundle install
```

To execute the jekyll service.

```PowerShell
bundle exec jekyll serve -watch
```
