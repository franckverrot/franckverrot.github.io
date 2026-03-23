---
title: "Hijacking gem commands"
date: 2015-04-21 08:00:00
comments: true
slug: "hijacking-gem-commands"
tags:
  - ruby
---

Rubygems has been made extensible by the usage of plugins. Any gem that provides a `lib/rubygems_plugin.rb` file
will be discovered by the `gem` infrastructure, whether it is loaded by your application or not.

Adding new commands is fairly easy, here's how we could replace existing ones (like `install`), using only
Rubygems' public API.

<!--more-->

Writing Rubygems plugins is fairly easy, as mentioned by the paragraph in the [offical guide][plugins]:

    As of RubyGems 1.3.2, RubyGems will load plugins installed in gems or $LOAD_PATH. Plugins
    must be named ‘rubygems_plugin’ (.rb, .so, etc) and placed at the root of your gem’s #require_path.
    Plugins are discovered via Gem::find_files then loaded. Take care when implementing a plugin as
    your plugin file may be loaded multiple times if multiple versions of your gem are installed.


Registering a new command requires two operations:

1. Subclassing `Gem::Command`

      # Subclass the regular class
      class Gem::Commands::Foo < Gem::Commands::Command
        def initialize
          # This will be run any time the `gem` command is executed
          super
        end
        def execute
          # This will be run only when `gem install` is executed
          super
        end
      end

2. Calling `Gem::CommandManager.instance.register_command` with the symbol representing that class

Trying to replace a built-in command isn't as straightforward though. Built-in commands [are listed within the
`Gem::CommandManager` class][builtins]. When that class will be initialized [it will]:

1. Call `register_command` with the correct symbol (here `:foo`)
2. Fallback on the private method [`load_and_instantiate`][l_and_i]:

        def load_and_instantiate(command_name)          # command_name = :foo
          command_name = command_name.to_s              # command_name = foo
          const_name = command_name.capitalize.
              gsub(/_(.)/) { $1.upcase } << "Command"   # const_name = FooCommand
          #...
          begin
            begin
              require "rubygems/commands/#{command_name}_command" # Try requiring... and fail
            rescue LoadError => e
              load_error = e
            end
            Gem::Commands.const_get(const_name).new # Instantiate a `Gem::Commands::FooCommand` object
          rescue Exception => e
            # ...
          end
        end

If we redefined the behavior of the `InstallCommand` and were to call `register_command :install` again,
the `require "rubygems/commands/#{command_name}_command"` call would load the command located within the
`rubygems` gem [1] and not our new command.

This means that hijacking a built-in Rubygems commands requires to:

1. Instantiate the hijacked command (we don't want to redefine it all) : `Gem::CommandManager.instance[:install]`
2. Subclass it : `class Gem::Commands::Install2Command < Gem::Commands::InstallCommand... end`
3. Unregister the old one : `Gem::CommandManager.instance.unregister_command :install`
4. Register the new one: `Gem::CommandManager.instance.register_command :install2`

`install2` ? Yes. [Any shorter command names will take precedence over longer ones][shortcuts]:

* `in` means `install`
* `instal` means `install`

In our case, we called `unregister_command` on `install`. So anything beginning with `install` will do.

Here's the [full program], to put in `lib/rubygems_plugin.rb`:

    require 'rubygems/command_manager'
     
    ## Load the initial `Gem::Commands::InstallCommand` class
    Gem::CommandManager.instance[:install]
     
    # Drop it from the list of builtin commands
    Gem::CommandManager.instance.unregister_command :install
     
    # Subclass the regular class
    class Gem::Commands::Install2Command < Gem::Commands::InstallCommand
      def initialize
        puts "This will be run any time the `gem` command is executed"
        super
      end
      def execute
        puts "This will be run only when `gem install` is executed"
        super
      end
    end
     
    # Profit!!!1!
    Gem::CommandManager.instance.register_command :install2


Here's the results:

    ➜ gem install activevalidators
    This will be run any time the `gem` command is executed
    This will be run only when `gem install` is executed
    Successfully installed activevalidators-3.3.0
    Parsing documentation for activevalidators-3.3.0
    Done installing documentation for activevalidators after 0 seconds
    1 gem installed


[plugins]: http://guides.rubygems.org/plugins/
[builtins]: https://github.com/rubygems/rubygems/blob/800f2e63bc6174b5b4dea5528110b09d89fe3dd1/lib/rubygems/command_manager.rb#L36
[l_and_i]: https://github.com/rubygems/rubygems/blob/800f2e63bc6174b5b4dea5528110b09d89fe3dd1/lib/rubygems/command_manager.rb#L197-L215
[1]: We could actually play with the `$LOAD_PATH` to make this work
[shortcuts]: https://github.com/rubygems/rubygems/blob/800f2e63bc6174b5b4dea5528110b09d89fe3dd1/lib/rubygems/command_manager.rb#L172-L193
[fullprogram]: https://gist.github.com/franckverrot/0d45aaade63bd513cd1f
