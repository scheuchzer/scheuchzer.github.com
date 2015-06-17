---
layout: post
title: "Running bower behind a firewall"
description: ""
category: "javascript"
tags: [Bower, JavaScript, GitHub, Https]
---
{% include JB/setup %}

[Bower]: http://bower.io/
[GitHub]: https://github.com/
[Git]: https://git-scm.com/
[SSH]: https://en.wikipedia.org/wiki/Secure_Shell

While working with [Bower] is nice, working with bower behind a company firewall is not that nice. It seems that by default [Bower] is trying to
download the dependencies from [GitHub] using SSH (or the `git://`) protocol. Unfortunately the [SSH] port is blocked by many firewalls.

Actually this is not a Bower problem but a [Git] problem. Bower uses Git to fetch the dependencies.

You can solve the problem by telling [Git] to use `https` instead of `git` url.

{% highlight bash %}
cd /project/dir
git config url."https://".insteadOf git://
bower install
{% endhighlight %}

This command solves the problem only for your current project.

You can even make this change globally, but I'm not sure if you really want to do that. For my personal projects I only use `git://` for cloning and I have two-factor authentication in place for `https` connections. So, this might break my setup but I've never tried so far.

{% highlight bash %}
git config --global url."https://".insteadOf git://
{% endhighlight %}
