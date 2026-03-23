---
title: Gemcutter Webhooks on Google Wave (and Google App Engine) part 1
date: 2010-01-27
slug: "gemcutter-webhooks-on-google-wave-and-google-app-engine-part-1"
tags:
  - ruby
  - open-source
---

<p>So what are Gemcutter webhooks? It&#8217;s a way to automatically notice an online app (say gemhooker.appspot.com for instance) about the activities of your favorite (or all) Gemcutter&#8217;s gems...</p>

<p>As mentioned in the <a href="http://gemcutter.org/pages/gem_docs#webhook">documentation</a>, let&#8217;s install gemcutter&#8217;s gem if not already done:</p> 
<script src="http://gist.github.com/271511.js"></script><p>Now this is done, let&#8217;s get it started and fire up a new Python&#8217;s powered <span class="caps">GAE</span> app (gemhooker.py):<br /> 
<script src="http://gist.github.com/271504.js"></script></p> 
<p>If you deploy this app (with appcfg.py update gemhooker-clone-you-just-initialized), you won&#8217;t see anything so let&#8217;s ask Gemcutter to add a webhook that will notice our app everytime someone pushes changes onto Gemcutter! I&#8217;m subscribing to every event of all the existing gems:</p> 
<script src="http://gist.github.com/271507.js"></script><p>You can see the result on <a href="http://gemhooker.appspot.com">gemhooker.appspot.com</a>.</p> 
<p><a href="http://twitter.com/qrush">Nick Quanranto</a> has done it (and it looks good, really good job!) using Ruby and deployed it on <a href="http://gemwhisperer.heroku.com">Heroku</a>.</p> 
<p>I wish I could have done this with Ruby too but as I need to make it work with Wave (and it seems like a <span class="caps">PITA</span> to make Rails 2.x work on AppEngine), I&#8217;ll suffer till the end of this project :)</p> 
<p>In part 2, I&#8217;ll explain how to make a Wave Gadget.</p> 
