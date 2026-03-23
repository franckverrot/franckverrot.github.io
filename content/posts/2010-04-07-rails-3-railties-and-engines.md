---
title: Rails 3, Railties and Engines
date: 2010-04-07
comments: true
slug: "rails-3-railties-and-engines"
tags:
  - ruby
---

Rails 3 brings a lot of useful features. The main one, in my humble opinion, is the introduction of the classes Rails::Railtie and Rails::Engine.

They bring the modularity that made code more reusable and easily integrable in your current code base. They also prove that there no reason to say that Rails is not ready for the Enterprise. One quick tip for the party poopers (you know who you are ;)): [JRuby](:http://jruby.org/) makes things even easier for your development operations team members as it allows you to run your Ruby app (ie: Rails) within your favorite App Server (Websphere, Tomcat, you name them).

In this article, I will try to explain a little bit what are these new classes, how they were implemented and how to start using them.

If you want to see a live presentation of this, come and join us to the [RubyCamp Lyon 2010](http://www.rubyfrance.org/association/groupes/lyon/) on April 17th.


## The history:  Merb + Rails = Rails 3

If you're reading the gurus' blogs, you may already know that Rails have had a major [refactoring](http://www.engineyard.com/blog/2009/6-steps-to-refactoring-rails-for-mere-mortals/) and [decoupling](http://yehudakatz.com/2009/07/19/rails-3-the-great-decoupling/) phase done by the Rails Core Team (Yehuda Katz being the one doing the communication about it).

This gave birth to an AbstractController class, that reduced the tight-coupling between ActionController and ActionView, and lots of other cool stuff. But the real change comes down to the addition of the Railtie class. If you want to know more about how Rails 3 layers fit one onto another, I suggest your read [Jeremy McAnally's post "The Path to Rails 3 : Introduction"](http://omgbloglol.com/post/344792822/the-path-to-rails-3-introduction)

## Introducing Railties and Engines

<small>Note: To clear any future confusion, we can safely state that an Engine is a Railtie (line 89 of _rails3_beta_gem/lib/rails/engines.rb_ : **class Engine < Railtie**).</small>

The major components of Rails, from Action View to Active Record thru Action Mailer and others, have their own Railtie inherited from Rails::Railtie. Each Railtie has its own initializers being triggered by the overall Rails boot process, its own generators, tasks, ... It's then up to the Rails::Application object to coordinate the boot process and sequentially ask each Railtie to initialize itself. De facto, this removes the tight coupling that we used to see in the previous versions of Rails.

A Rails::Application is also a Railtie, but a bit more: an Engine. An Engine is an extension (a derived class) of the Railtie class. **Please do not think about Rails 3 Railtie as the old Rails 2.x engines plugin!** Rails 3 Engines are core aspects of Rails. The noticeable difference between a Railtie and an Engine (as far as I investigated) is that an Engine:

* comes with a predefined set of initializers (locales, metals, view paths, etc.)
* has its own _load_paths_, _eager_load_paths_ and _load_once_paths_ (scoped to the current Engine)

Engines make it easy for anyone to modify or extend, à la Merb, any part of a Rails application out of the box - from the ORM to the rendering system -  and for instance drop Active Record in favor of DataMapper or add an authentication system (like [Devise](http://github.com/plataformatec/devise)).


## Let's understand how it all works

Each Rails 3 Application has now a file called *config/application.rb*, that defines **MyAppName::Application**:

```ruby
require File.expand_path('../boot', __FILE__)
require 'rails/all'
Bundler.require(:default, Rails.env) if defined?(Bundler)

module MyAppName
    class Application < Rails::Application
        ....
    end
end
```

You can only have one instance of a **Rails::Application** at a time, it's a singleton and it will always point to **MyAppName::Application.instance**. If we now have a look at the code that defines **Rails::Application**, we can see this:

```ruby
require 'fileutils'
require 'rails/plugin'
require 'rails/engine'

class Application < Engine
    autoload :Bootstrap,      'rails/application/bootstrap'
    autoload :Configurable,   'rails/application/configurable'
    autoload :Configuration,  'rails/application/configuration'
    autoload :Finisher,       'rails/application/finisher'
    ... snip ....

    def initialize!
        run_initializers(self)
        self
    end
    ... snip ....

    def initializers
        initializers = Bootstrap.initializers_for(self)
        railties.all { |r| initializers += r.initializers }
        initializers += super
        initializers += Finisher.initializers_for(self)
        initializers
    end
    ... snip ....
end
```

Interesting! If you have a look at the **Rails::Application::Bootstrap** and **Rails::Application::Finisher** modules, they are including another module: **Rails::Initializable**, which is defining the **Initializer** class, finally defining the **run_initalizers** method we can see above.

So what does that mean at the end? Each of your Engine and Railtie will be able to hook into the boot process, right before the other initializers or right after. From the user (aka developer) perspective, it's easy to use:

```ruby
module MyModule
    class Railtie < Rails::Railtie
        # initialize before
        initializer "my_module.initialize_my_thingy", :before => "some_other_railtie.initialize_its_thingy" do
        ... some code
        end
        # initialize after
        initializer "my_module.initialize_my_other_thingy", :after => "some_other_railtie.initialize_its_other_thingy" do
        ... some code
        end
    end
end
```



### Creating an Engine

If you want your engine to work within your application, you obviously have to put in the load path of it. You can do this by putting it your code just like you would do for a plugin: in your *vendor/plugins* directory. The basic structure looks like:

```ruby
# vendor/plugins/my_gem/lib/engine.rb
module MyEngine
  class Engine < Rails::Engine
    engine_name :my_engine
  end
end
```

To use it in your Rails application, just update your _Gemfile_ and simply put:

```ruby
  gem "my_gem", :require => "my_engine"
```

By putting it in your Gemfile, and your Gemfile being used during the startup of your applicaiton, the initializers of your engine will automatically run, settings up the routes, loading the models, controllers, tasks, translations... that you may have defined in your Engine. Available paths are all pointing to a Rails-style folder name from the root folder of your engine (ie: *paths.app.views* will point to *app/views*):

  * app.[controllers, helpers, models, metals, views, observers, ...]
  * lib
  * lib.tasks
  * config.[initializers, locales, routes]


## To be continued...

You will find on [The Modest Rubyist](http://www.themodestrubyist.com/) very neat tutorials on how to create your Engine from the ground up.

I will continue with a new article that will be the base of the talk I will give during the [RubyCamp Lyon 2010](http://www.rubyfrance.org/association/groupes/lyon/) about enterprise Ruby-based software architecture.


