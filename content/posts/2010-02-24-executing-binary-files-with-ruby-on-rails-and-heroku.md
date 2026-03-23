---
title: Executing binary files with Ruby on Rails and Heroku
date: 2010-02-24
slug: "executing-binary-files-with-ruby-on-rails-and-heroku"
tags:
  - ruby
---

One would easily wonder why in hell someone else would want to do that, but it's actually often because you are forced to.

In this post, I'll explain how to proceed.

<!--more-->

That's not the kind of thing you usually do and it's actually during the implementation of an online boutique that made me hit the issue. 

####1. A matter of time in the beginning...

Some banks have unobtrusive systems. They give you a very detailed specification of how to build the form that will allow your visitor to connect to the payment gateway and also, how to verify after the payment processing whether or not the payment has been successful.

Some others require to execute a binary file (with a couple of special parameters and files that must be relative to the Rails' app root) that will return the actual HTML form.

####2. ... and a matter of hidden technical debt at the end.

As one would imagine it's easier to really on the second option, but it also means that *your hosting platform is now tight-coupled to your application* and in the cloud-oriented future that is ours, it seems like you're building your own time bomb but you don't really know when it's gonna blow out.

In any case, you're still gonna need to actually execute a program so the underlying hosting plateform must be known and your binary must run on it. 

####3. The right choice.

In fact the choice is not entirely yours when you rely on someone else's binary, but hopefully some operating systems are more or less recognized as standards: Debian-based GNU/Linux distributions, Red Hat's, and Windows.

Saying that everybody can just go for Heroku would be misleading, but their stack is 100% business standard and unmodified (understand: if it's working in-house, it's working in the cloud).

####4. Executing a binary in Ruby

That was actually the reason we're here. If you made that far it's only because you wanted to run one, so let's get going. To effectively call the program you can use different methods as:

<script src="http://gist.github.com/315103.js?file=executing+a+binary+on+heroku+part+2.rb"></script>

####5. Executing a binary in a Ruby application

I really prefer the @Open3.popen3@ call cause it gives access to the well-known standard Unix I/O. But they all work and you can pick any of these given your own requirements.

First of all, create a directory at the root of your Ruby application called @bin@ and add it to your git repository:

<script src="http://gist.github.com/315087.js?file=executing+a+binary+on+heroku+part+1.sh"></script>

Let me emphasize the last sentence: *you must put in the @bin@ directory at the root of your application and make it executable*.

You already know how to call your executable (or you should read again the beginning of the post :)). So let's embed it in your model but I won't cover that topic as it's quite framework-related and  we're not here to talk about frameworks.

####6. Wrap up

* The @bin@ directory at the root of your application.
* The executable flags have to be set up.
* Any of the described way of executing a binary will work, pick your favorite.
