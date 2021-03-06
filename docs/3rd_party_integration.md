---
layout: docs
title: 3rd-party Integration
prev_section: changelog
next_section: communication
permalink: /3rd_party_integration/
---

NoBrainer can be extended to support other functionnality and play nicely with
other gems. The following describes such gems.

If you'd like to create a gem, please use the naming convention to prefix
your gem with `nobrainer-`.

## Devise

[nobrainer-devise](https://github.com/nviennot/nobrainer-devise) is an adapter
to make [Devise](https://github.com/plataformatec/devise/) work nicely with
NoBrainer.

Include in your Gemfile:

{% highlight ruby %}
gem 'nobrainer-devise'
{% endhighlight %}

Use devise as usual.

If devise does not work, make sure that your `config/initializers/devise.rb` file
contains `require 'devise/orm/nobrainer'`.

## Awesome Print

[awesome_print](https://github.com/michaeldv/awesome_print) pretty prints objects.
NoBrainer support is built in.
