---
title: Talk to Gemcutter's API using XMPP/Google Talk
date: 2010-02-20
comments: true
slug: "talk-to-gemcutters-api-using-xmppgoogle-talk"
tags:
  - ruby
  - open-source
---

Working with Ruby in my day job made me try other exciting things, especially with Google App Engine. Programming in Python is not so bad, but I won't say I'm having fun remapping my brain onto the weirdosities of the language.

Anyhoo... Tonight I wanted to talk to Gemcutter, but not programmatically, I wanted to have it in my GTalk contact list and start having a little chat...


The very first step was to fire up a new application in App Engine web panel. Five seconds later, let's fired up a GVim and create (within a new directory) app.yml and gemtalker.py.

Let's start with the app.yml:
<script src="http://gist.github.com/309578.js?file=app.yml+to+enable+XMPP+in+AppEngine"></script>

~Mind the *inbound_services* declaration~

Now fill-in some code in your main python file:

<script src="http://gist.github.com/309577.js?file=Talk+to+your+App+Engine+application"></script>

As one would notice it's pretty easy to play with XMPP services and actually, we need to know very few things here (as we don't even handle user registration to secure application access, etc...):
* It's all in the request object;
* Replying is a no-brainer.

Now we must interact with Gemcutter's API to get some information (that was the purpose of this article actually...). I'm not a real Pythonista, so I didn't know how to make this more readable/simpler but please, don't hesitate to fork me hard on [Github](http://github.com/franckverrot/gemtalker).

<script src="http://gist.github.com/309585.js"></script>

I must admin I wanted to use method_missing but it's a... method missing in Python (I found a way to do it thought but that's gonna be for later on).

So let's wrap it up:

* Gemcutter's library: done;
* XMPP library: ready; 

So we're up for wiring it up and writing the logic behind the application.

I'll let read the draft implementation of all of this on [Github](http://github.com/franckverrot/gemtalker/) , but it's already up and running and you can invite gemtalker@appengine.com in Google Talk and use the "*info*" command.

What's mising:
* Activate the *AUTH* command and store api keys only for the current session
* Search a gem
* Modify webhooks, owners, etc... (requires the AUTH to be done)

Have fun!
