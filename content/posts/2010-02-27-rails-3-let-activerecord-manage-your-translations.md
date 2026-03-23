---
title: Rails 3 Let ActiveRecord Manage Your Translations
date: 2010-02-27
slug: "rails-3-let-activerecord-manage-your-translations"
tags:
  - ruby
---

With the previous versions of Rails we have the choice between storing the translations into a YAML file (one per language) and standard Ruby Hashes.

Bringing the ActiveRecord backend to light, the I18n gem allows us now to manage all our translations in a regular database.

The I18n gem (required to run Rails 3) has been released in late December and is providing all you need to store your translations into a database. A neat thing is the I18n::Backend::Base module, which makes it easy to start writing a new backend in no time.

Today we'll just configure our Rails 3 application to start storing our translations in our ActiveRecord-baked database.

First of all, let's create a database table that will be used for the translations. Fire up a new migration and fill-in the blanks in *self.up* with:

    create_table :translations do |t|
      t.string :locale
      t.string :key
      t.text   :value
      t.text   :interpolations
      t.boolean :is_proc, :default => false
    end



This table is required by I18n, but its structure obviously evolve for your own needs. You should now also create a model in *app/models/translation.rb*:

    class Translation < ActiveRecord::Base
    end



Now you can migrate (*rake db:migrate*), open up your *config/application.rb* file and modify it to make it looks like this:

    [...]
    module YourApplicationName
      class Application < Rails::Application
      [...]
        I18n.backend = I18n::Backend::ActiveRecord.new
      [...]
    end



Voilà! To start creating some translations you can open a Rails console and :

    Translation.create!(:locale => "en", :key => "activerecord.models.user", :value => "my user")



*[Edit: Feb 28th]* : Cédric and Nicolas got it right: you're gonna run into a lot of troubles if you hit the database each and every time you need a translation. A solution to that: caching. How to proceed? Just again your *config/application.rb* and add:

    [...]
    I18n.backend = I18n::Backend::ActiveRecord.new
    I18n::Backend::ActiveRecord.send(:include, I18n::Backend::Cache)
    I18n.cache_store = ActiveSupport::Cache.lookup_store(:memory_store) # or whatever store you prefer


Ain't it great? :)
