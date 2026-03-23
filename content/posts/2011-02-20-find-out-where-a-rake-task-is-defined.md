---
title: Find out where a Rake task is defined
comments: true
date: 2011-02-20
slug: "find-out-where-a-rake-task-is-defined"
tags:
  - ruby
---

*TL;DR*
Rake > 0.8.7 has a handy `Rake::Task#locations` method that makes it
damn easy to know where a task is defined or enhanced.

During the last [LyonRb](http://lyonrb.fr) meetup we read a quite large
portion of Rake's source code. The idea was to find an easy way to determine
what file was defining/enhancing what task.

[Richard](http://twitter.com/richarkmichael) and I just opened Rake
0.8.7's source code (from our local machine's RVM directory) and started hacking a bit to have that feature...

<!--more-->

Two days after I can tell you that you won't need to hack anything
because it's actually already implemented in Rake's master since [May 19th
2009 by Jim Weirich](https://rubygems.org/gems/rake).

It wasn't obvious to us as [Rake 0.8.7](https://rubygems.org/gems/rake)
has been released on... May 15th 2009!


```
~  cd /tmp && git clone git://github.com/jimweirich/rake.git && cd rake && rake gem && gem install pkg/rake-0.8.99.5.gem
Cloning into rake...
remote: Counting objects: 5804, done.
remote: Compressing objects: 100% (2489/2489), done.
remote: Total 5804 (delta 3387), reused 5519 (delta 3180)
Receiving objects: 100% (5804/5804), 1.23 MiB | 613 KiB/s, done.
Resolving deltas: 100% (3387/3387), done.
RCov is not available
ERROR: unknown action 'gemsets'
Successfully installed rake-0.8.99.5
1 gem installed

/tmp  cat >> foo.rb << EOF
heredoc> desc 'foo'
heredoc> task :foo do; end
heredoc> desc 'foobar'
heredoc> task :foobar do; end
heredoc> EOF

/tmp  cat >> baz.rb << EOF
heredoc> desc 'baz'
heredoc> task :baz do; end
heredoc> task 'foobaz'
heredoc> task :foobar do; end
heredoc> EOF

/tmp  cat >> Rakefile << EOF
heredoc> load 'foo.rb'
heredoc> load 'baz.rb'
heredoc> EOF

/tmp  rake -T
rake baz      # baz
rake foo      # foo
rake foobar   # foobar

/tmp  rake --where
rake baz                            baz.rb:2:in `<top (required)>'
rake foo                            foo.rb:2:in `<top (required)>'
rake foobar                         foo.rb:4:in `<top (required)>'
rake foobar                         baz.rb:4:in `<top (required)>'
```
