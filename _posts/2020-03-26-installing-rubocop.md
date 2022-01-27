---
layout: post
title: "Installing Rubocop shouldn't be this difficult"
date: 2020-03-26
categories: rubocop, ruby, jaro_winkler, docker
github_comments_issueid: 2
---
Last time I needed to install rubocop in a container I hit the jaro_winkler install problem:

```(bash)
An error occurred while installing jaro_winkler (1.5.4), and Bundler cannot
continue.
Make sure that `gem install jaro_winkler -v '1.5.4' --source
'https://rubygems.org/'` succeeds before bundling.

In Gemfile:
  rubocop was resolved to 0.80.1, which depends on
    jaro_winkler
```

and this one:

```(bash)
ERROR:  Error installing rubocop:
ERROR: Failed to build gem native extension.

    current directory: /usr/lib/ruby/gems/2.6.0/gems/jaro_winkler-1.5.4/ext/jaro_winkler
/usr/bin/ruby -I /usr/lib/ruby/2.6.0 -r ./siteconf20200326-7-nrebfa.rb extconf.rb
mkmf.rb can\'t find header files for ruby at /usr/lib/ruby/include/ruby.h

You might have to install separate package for the ruby development
environment, ruby-dev or ruby-devel for example.

extconf failed, exit code 1
```

I took the easy route out (after googling for way longer than I should have) - use a Ruby base image. This time I didn't want to do that. No real reason, just being stubborn.

Rubocop has a dependency on jaro_winkler that won't be removed until support for ruby 2.3 is removed.

As I couldn't find the solution to the problem anywhere on the interwebs or on the Rubocop or Jaro_winkler github pages, but after a bit of trial and error.....

In addition to ruby, the following dependencies need to be installed, in no particular order:

* ruby-etc
* ruby-dev
* make
* gcc
* libc-dev

And finally...

```(bash)
Building native extensions. This could take a while...
Successfully installed jaro_winkler-1.5.4
Successfully installed parallel-1.19.1
Successfully installed ast-2.4.0
Successfully installed parser-2.7.0.5
Successfully installed rainbow-3.0.0
Successfully installed ruby-progressbar-1.10.1
Successfully installed unicode-display_width-1.6.1
Successfully installed rubocop-0.80.1
```
