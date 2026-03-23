---
title: Introducing Live-Note appspot com
date: 2010-03-04
comments: true
slug: "introducing-live-note-appspot-com"
tags:
  - meta
---

I was about to call this article "Gemcutter Webhooks on Google Wave (and Google App Engine) part 2" but then I realized that it was no more about that I wanted to focus on, but more on the Wave part (sorry Rubyists friends, but I had fun with Python (as long as I don't try to do metaprogramming Python is nice to play with :)).

Google released the Google Wave Robot API v2 (hurray). In that major revision, they are introducing the Active Robot API that makes it possible for robots (i.e. GAE-baked applications) to interact with Waves. In the previous version, your robot was being notified each time a wave (or wavelet, or blip) was modified or when a participant was added to the wave, but now, your application can actually be active and contact Wave on its own.

In the first part of this series of articles, I was demonstrating how to build a GAE-baked application and how to subscribe to a web hook (being a Rubyist I was taking the webhooks from Gemcutter / Rubygems.org). I went only half-way as I wanted to actually see the result _inside_ Google Wave. But then I think to my self, what a wonderful <del>world</del> think it would be to do something actually useful in my everyday life instead of just demonstrating mix of technologies (even if it's neat to be able to make applications talk to each others).

So today, it's gonna be about a 12-hour-design application: **Live-Note**.

<!--more-->
When I first created "Gemhooker":http://gemhooker.appspot.com, GAE was kinda new for me and I was learning all about it. Even understanding how to plug request handlers within a Google Robot application was hard. But after a lot of investigations, I was able to create in no-time an app able to understand 3 over the 5 communication mediums: email, chat and web app forms. The only missing forms of communications are voice and video.


And this app, is **Live-Note**. I tried to use "280slides":http://www.280slides.com for this:

<iframe width="400" height="328" src="http://280slides.com/Viewer/?user=34211&name=Live-Note" style="border: 1px solid black; margin: 0; padding: 0;"></iframe>

It's still in beta, and is likely to stay as it is for a while because I won't have anytime to update it in the future weeks/months, but you can already work with it. I tried to do something nice using IUI, that emulates the iPhone GUI, but I gotta admit I can improve that a little bit :)

To wrap up:

* First time use: http://live-note.appspot.com
* Then Chat with *live-note@appspot.com*
* Or Mail *live-note@live-note.appspotmail.com*
* Or Use the "web app":http://live-note.appspot.com


And of course, feel free to give your feedback!

**Edit: I fixed the chat username and email... sorry about that :)**
