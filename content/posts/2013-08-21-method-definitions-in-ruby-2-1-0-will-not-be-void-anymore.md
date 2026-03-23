---
title: "Method definitions in Ruby 2.1.0 will not be void anymore"
date: 2013-08-21 08:00:00
slug: "method-definitions-in-ruby-2-1-0-will-not-be-void-anymore"
tags:
  - ruby
---

Ruby 2.1.0 will be released at the end of the year and among other features, a change done in the parser caught my eyes...

<!--more-->

This hack, showed as an example for the real implementation, in the [related ticket][redmine-3753] does something interesting:

    Index: vm.c
    ===================================================================
    --- vm.c(リビジョン 29124)
    +++ vm.c(作業コピー)
    @@    -1893,7 +1893,7 @@ m_core_define_method(VALUE self, VALUE c
         REWIND_CFP({
     vm_define_method(GET_THREAD(), cbase, SYM2ID(sym), iseqval, 0, rb_vm_cref());
         });
    -    return Qnil;
    +    return sym;
     }

     static VALUE
    @@ -1902,7 +1902,7 @@ m_core_define_singleton_method(VALUE sel
         REWIND_CFP({
     vm_define_methodethod(GET_THREAD(), cbase, SYM2ID(sym), iseqval, 1, rb_vm_cref());
         });
    -    return Qnil;
    +    return sym;
     }


&nbsp;


Instead of returning nothing (`Qnil`) after a method definition, the proposal is to return the method's name symbol.

So before [revision 42337][rev42337] with Ruby 2.0.0:

    >> RUBY_VERSION
    => "2.0.0"
    >> def foo; end
    => nil


&nbsp;


And tomorrow with Ruby 2.1.0:

    >> RUBY_VERSION
    => "2.1.0"
    >> def foo; end
    => :foo


&nbsp;


If you ever tried Rubinius, you might have noticed that it returns a `CompiledMethod` object:

    def foo; end
    => #<Rubinius::CompiledMethod foo file=(irb)>


&nbsp;


In my opinion the behavior of Rubinius makes more sense but I don't how hard it would be to mimic that in MRI.

[redmine-3753]:https://bugs.ruby-lang.org/issues/3753
[rev42337]:https://bugs.ruby-lang.org/projects/ruby-trunk/repository/revisions/42337/diff
