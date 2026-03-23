---
title: Fatal GCJ GNU Compiler for Java is not supported by JRuby
date: 2010-03-22
comments: true
slug: "fatal-gcj-gnu-compiler-for-java-is-not-supported-by-jruby"
tags:
  - ruby
---

If you ever encountered this when installing JRuby on Ubuntu:

`Fatal: GCJ (GNU Compiler for Java) is not supported by JRuby.`

Doing 

`export JAVA_HOME=/usr/lib/jvm/java-6-sun/jre `

could do the trick, given you've installed sun-java6's packages on your machine.

I hope it'll help.

<!--more-->

If you ever encountered this when installing JRuby on Ubuntu:


`Fatal: GCJ (GNU Compiler for Java) is not supported by JRuby.`


Doing 


`export JAVA_HOME=/usr/lib/jvm/java-6-sun/jre`


could do the trick, given you've installed sun-java6's packages on your machine.
I hope it'll help.
