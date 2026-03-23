---
title: "Looking forward to Ruby's future"
date: 2018-12-31 22:45:00
categories:
  - ruby
slug: "looking-forward-to-ruby-s-future"
tags:
  - ruby
---

Ruby became a more stable and mature language over the years. Some would say
innovation slowed down (and I was probably one of them), but I still
appreciate writing Ruby on a daily basis on the job. In this blog post I will
succinctly explain why Ruby is still a valid choice for writing new
(web and non-web) applications, the challenges of maintaining large code
bases and what I look forward in the coming years.

<!--more-->

# Ruby, a valid choice for writing new applications

The Ruby ecosystem grew quickly over the past 10 years. Most of its success
comes from the advent of [Ruby on Rails](rails), but some other libraries and
frameworks – which have been created to propose alternatives to Rails – also
contributed to Ruby's growth.

[Sinatra](sinatra) came up with a very lightweight and powerful DSL:

```ruby
# example.rb
# Run `gem install sinatra` first, and then run `ruby example.rb
require 'sinatra'
get '/hello' do
  'world'
end
```

And a large population of developers developed APIs with it.

[Merb](merb) also gained momentum (along with [Datamapper](datamapper)) to
provide more flexibility to a Rails which, at that time (circa 2008), wasn't
as modular and extensible as it is nowadays.

But Ruby isn't only great for writing web apps but is also well equipped for
all sorts of apps. [Chef](chef), a configuration management tool, makes use
of Ruby as its frontend/client tool, leveraging Ruby's force: writing
Embedded DSLs.

```ruby
execute 'some command' do
  command 'some_command'
  not_if { some_condition }
end
```


The most recent versions of Ruby (2.5 and 2.6) have brought many improvements
and new features that make developers lives easier:

* Distribution of `bundler` (the package manager) by default
* Function composition with the `>>` operator
* Endless ranges `(1..)`
* A Just In Time compiler to improve performance
* `yield_self`
* And many more...


# Some challenges

Maintaining large scale code bases has been, in my experience, challenging at
times, a blessing at others. The emphasis on Test-Driven Development allowed
a generation of developers to write battle-tested code without having to
convince their peers that it was the right thing to do. The "Ruby community"
at large made it so simple and evangelized so many software engineers,
product managers and engineering managers that it became the only true way
for developing software.

But writing the right tests, and designing software architecture right,
quickly became at the expense of being able to refactor code easily.
Extracting classes and methods in new classes or modules should be safe as
upper layers of testing (controller tests and up) ensure that changes that
are being made aren't breaking anything, but it still requires a thorough
"Find and Replace" that is mostly manual and error-prone (even the best IDEs
like RubyMine or Visual Studio Code do not provide a 100% bullet-proof way of
doing this).

Static typing has gained popularity in the past year or so (2017-2018), and
some research and actual programming languages showed it brought a net
benefit (in some circumstances. For example, Elm came with a very
stripped-down/simple type system that a lot of people liked – and a lot
disliked too). Some companies decided to invest research and development
time, like Stripe, and are funding the development of gradual typing system
for Ruby called [Sorbet][sorbet] which are supposed to "helps developers understand
code better, write code with more confidence, and detect+prevent significant
classes of bugs."


# Looking forward to Ruby's future

All-in-all, Ruby has been more than a programming language for a lot of
people: it was a life-style change: saying no to the at-the-time tyranny of
large XML files, providing simple workflows for tasks that should be simple,
and opinionated convention-over-configuration-driven frameworks which sped up
their adoption for standard applications.

Ruby is trying to get back on track when it comes to performance and the
addition of a Just In Time compiler in Ruby 2.6.0 that came out late 2018 is
the sign there are tons of efforts to make it better for compute-intensive
workloads.

Embedding Ruby is still hard as it requires spawning a brand new interpreter
(entire VM in a separate process) as it doesn't support running multiple VMs
at once (the Ruby core teams seems to be calling this "MVM"), but
[mruby][mruby] provides a solid alternative: lightweight, fast, composable
architecture. [Holycorn][holycorn] is an example of using `mruby` to
implement PostgreSQL's "Foreign Data Wrappers". Check out `mruby` if you
haven't, it's great!

And last but not least, the community is still vibrant and will continue
being vibrant. Ruby on Rails' latest releases prove that innovation is still
happening, and that Ruby's versatility is here to stay.



[rails]: https://rubyonrails.org
[sinatra]: http://sinatrarb.com/
[merb]: https://en.wikipedia.org/wiki/Merb
[datamapper]: https://datamapper.org/
[chef]: https://en.wikipedia.org/wiki/Chef_(software)
[sorbet]: https://sorbet.run/
[mruby]: https://mruby.org
[holycorn]: https://github.com/franckverrot/holycorn