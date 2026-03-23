---
title: "Benchmarking Ruby's `method_missing` vs `define_method`"
date: 2015-07-12 23:45:00
categories:
  - ruby
  - performance
  - cruby
slug: "benchmarking-ruby-method-missing-and-define-method"
tags:
  - ruby
  - performance
---

After recently [benchmarking `PostgreSQL`][benchmarking-pg] to find out if some
of the techniques we used were efficient, I decided to look at the usage of a
controversed Ruby feature: `method_missing`.

<!--more-->

# Introduction

As I was recently working on a slow part of a codebase, which started to
degrade the overall experience of the rapplication, I decided to go ahead
and conduct an investigation to understand where the problem was.

The usual plan to perform that type of work is the following

1. Setup some probes (this will be the source of another article on the matter)
2. Get measure out of those probs on real-life scenarios
3. Quantify their impact in terms of I/O, CPU, ...

At the end a list of components to be fixed can be established, and it's up to
the team to address the top X of them.

# Why benchmarking

I quickly noticed what I called a case of “cascading `method_missing`s”. Most
seasoned developers know a few things about that method:

1. It should be avoided as it leads to producing complicated codebases
2. It is slow

(Arguably, 1/ is both a matter of taste and discipline so there's no real
data behind that point.)

The alternative is to use `define_method` instead, and a combination of those
two techniques if of course a viable solution: we discover the method's
behavior with `method_missing`, and immediately use `define_method` to bind
a new method to the receiver.

We are expecting `define_method` to be faster (as once it's defined, the method
can be called directly), but it means we will have to list all the methods to
be created beforehand.

The convenience of `method_missing` could outweight its cost in one's specific
situation but without data, it's just guess work. So let's measure!

# Defining methods

## `method_missing`

For those new to `method_missing`, its usage is simple: define some behavior by
parsing the parameters it's being given, and decide what do to.

```ruby
class Foo
  def method_missing(method_name, *arguments, &block)
    # do something with the parameters
    puts "I could have implemented #{method_name}"
  end
end

foo = Foo.new
foo.do_something # => this will print the message defined above
```

Other methods and objects can even ask beforehand the receiving object if it
supports that method call using `respond_to_missing?` but this method won't be
covered here.

## `define_method`

`define_method` on another hand is equivalent to defining a method in the body
definition of a class:

```ruby
class Foo
  def bar; end
end
```

is exactly equivalent to writing

```ruby
class Foo
  define_method :bar do
  end
end
```

(Just like `Foo = Class.new do...end` is equivalent to `class Foo; end`

# Designing the benchmarking infrastructure

Our investigation will try to understand when one method is faster than the
other, based on a range of pre-defined number of method calls.

We will use the `benchmark` standard lib and its `bm` method so first, we
should build our test infrastructure.

## `method_missing`

Here's the code replicating the “cascading `method_missing`s” case:

```ruby
module Converted
  def method_missing(method_sym, *arguments, &block)
    if method_sym.to_s =~ /^(.+)_converted$/
      self.send($1).to_f
    else
      super
    end
  end
end

module Value
  def method_missing(method_sym, *arguments, &block)
    if method_sym.to_s =~ /^(.+)_value$/
      self.send($1) * 6.55957
    else
      super
    end
  end
end

class MethodMissed
  include Converted
  include Value

  def some_attribute; 42; end
end
```

Pretty scary... I know.

## `define_method`

Now onto the alternative implementation using `define_method`

```ruby
module DefineMethod
  %i(some_attribute).each do |meth|
    converted = define_method :"#{meth}_converted" do
      send(meth).to_f
    end

    define_method :"#{converted}_value" do
      send(meth) * 6.55957
    end
  end
end

class MethodDefined
  include DefineMethod

  def some_attribute; 42; end
end
```

Now only it is more concise, but it also makes the intent clearer: we are
defining custom methods, and we know which ones.

## Defining the benchmark itself

So now let's define a range of number of method calls, and run our benchmarks!

```ruby
Benchmark.bm(7) do |x|
  [10, 100, 1_000, 10_000, 100_000, 1_000_000].each do |n|
    val = sprintf("%7d", n)
    x.report("#{val} method_missing - new object     ") do
      n.times { MethodMissed.new.some_attribute_converted_value }
    end
    x.report("#{val} define_method  - new object     ") do
      n.times { MethodDefined.new.some_attribute_converted_value }
    end
    x.report("#{val} method_missing - existing object") do
      obj = MethodMissed.new
      n.times { obj.some_attribute_converted_value }
    end
    x.report("#{val} define_method  - existing object") do
      obj = MethodDefined.new
      n.times { obj.some_attribute_converted_value }
    end
  end
end
```

(The full benchmark can be found under my [GitHub profile][gh])

# Results

## Running the benchmarks

Running the benchmarks gives this output:

                                              user     system      total        real
         10 method_missing - new object       0.000000   0.000000   0.000000 (  0.000074)
         10 define_method  - new object       0.000000   0.000000   0.000000 (  0.000008)
         10 method_missing - existing object  0.000000   0.000000   0.000000 (  0.000054)
         10 define_method  - existing object  0.000000   0.000000   0.000000 (  0.000005)
        100 method_missing - new object       0.000000   0.000000   0.000000 (  0.000418)
        100 define_method  - new object       0.000000   0.000000   0.000000 (  0.000032)
        100 method_missing - existing object  0.000000   0.000000   0.000000 (  0.000420)
        100 define_method  - existing object  0.000000   0.000000   0.000000 (  0.000023)
       1000 method_missing - new object       0.000000   0.000000   0.000000 (  0.003902)
       1000 define_method  - new object       0.000000   0.000000   0.000000 (  0.000330)
       1000 method_missing - existing object  0.000000   0.000000   0.000000 (  0.004180)
       1000 define_method  - existing object  0.000000   0.000000   0.000000 (  0.000211)
      10000 method_missing - new object       0.050000   0.000000   0.050000 (  0.050558)
      10000 define_method  - new object       0.010000   0.000000   0.010000 (  0.003782)
      10000 method_missing - existing object  0.040000   0.010000   0.050000 (  0.047005)
      10000 define_method  - existing object  0.000000   0.000000   0.000000 (  0.002178)
     100000 method_missing - new object       0.480000   0.000000   0.480000 (  0.483377)
     100000 define_method  - new object       0.040000   0.000000   0.040000 (  0.035107)
     100000 method_missing - existing object  0.430000   0.000000   0.430000 (  0.439768)
     100000 define_method  - existing object  0.020000   0.000000   0.020000 (  0.021825)
    1000000 method_missing - new object       4.600000   0.030000   4.630000 (  4.653720)
    1000000 define_method  - new object       0.340000   0.000000   0.340000 (  0.355784)
    1000000 method_missing - existing object  4.400000   0.020000   4.420000 (  4.430967)
    1000000 define_method  - existing object  0.230000   0.010000   0.240000 (  0.238588)

Both methods seem linear, let's see some graph.

## Plotting the results

![Plotting the results](https://raw.githubusercontent.com/franckverrot/franckverrot.github.io/master/images/perf-ruby-method-missing-vs-define-method.png "Plotting the results")

# Conclusion

As expected, `define_method` is faster, and from at least one order of magnitude.
A mix of both `define_method` and `method_missing` could have made the
`method_missing` strategy a bit faster, but we would still have some logic to
be executed in it so the difference wouldn't be that big (and this is left as
an exercise to the reader to perform that investigation :-D).


[benchmarking-pg]: http://franck.verrot.fr/blog/2015/07/12/benchmarking-postgresql-select-query-planning-and-performance-on-columns-aggregates/
[gh]: https://github.com/franckverrot
