---
title: "Types in Ruby"
date: 2019-04-20 10:24:04
comments: true
slug: "types-in-ruby"
tags:
  - ruby
---

Quick thoughts on [Sorbet][sorbet], that has been announced at [RubyKaigi
2018][rk18] and once again in the [2019 edition][rk19], the type-checker
for Ruby.

<!--more-->

I like types.  I've been working with dynamic languages for a little over a
decade, but it's my belief that types – when implemented properly – do bring
value and reduce cost.  But it's also my belief that not all type systems
are equal.  OCaml, Haskell, C++, Go, Java, and others have types, but they
certainly don't allow developers to reason the same way about the code they
write, and are – most of the time for mainstream languages – mostly slowing
them down rather than empowering them.

[Back in 2010 I released Gendarme][gendarme], a tool that would check for
pre-conditions and post-relations in your code.  The goal was the same:
have a tool that could check, at run-time with this implementation, for
invariants and raise errors/alert you if it found discrepancies:

```
class Foo
  precondition(0,"Argument 0 is a String") { |bar| bar.respond_to?(:to_str) }
  precondition(1,"Argument 1 is 10")       { |baz| baz == 10 }
  postrelation(0,"Result is an Integer")   { |res| res.is_a?(Integer)}
  def foo(bar,baz)
    ...
    "not an integer"
  end
end
```

This experiment didn't last long, Ruby (1.8?) was slow enough to not clutter
the VM with too many unnecessary runtime computations, and Matz' has also
historically been against the idea of trying out types with Ruby (fair
enough, he's the boss after all :-).) This motivated and accelerated my
learning of other languages and their type systems, so it's been a win-win
situation for me...


But then Stripe announced they were working on a type-checker for Ruby, and
as a lot of people expressed interest in the past for a type system (being
annotations, gradual typing, static typing, anything that would help getting
them), it quickly gets traction and press.

Here's how it does it:

```
# typed: true
class A
  extend T::Sig

  sig {params(x: Integer).returns(String)}
  def bar(x)
    x.to_s
  end
end
```
<small>(example from the documentation)</small>

A bank of custom signatures can be created to one's project.  Ruby core
contributors are likely to add signatures to the standard library, and gem
authors will also be able to add them to their gems... this is great!

A lot of things are happening with Ruby, and I can't wait to see other
features that Ruby 3.0 will bring.


[sorbet]: https://sorbet.org/docs/adopting
[rk18]: https://rubykaigi.org/2018
[rk19]: https://rubykaigi.org/2019
[gendarme]: https://github.com/franckverrot/gendarme