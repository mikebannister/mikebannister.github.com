---
layout: post
title: "Create a Rails::Engine With RSpec and Capybara Tests"
date: 2011-10-03 23:21
comments: true
categories:
- Rails
- Testing
---

I'm very excited about the prominence of engines in Rails 3.1. As you may know Rails::Application inherits from Rails::Engine. So when you write an engine you get all the power of a full rails app in a tidy package that can be added, as a sub-app, to your main application. Rails 3.1 provides a generator for creating new engines but setting it up to work with RSpec and Capybara took a couple extra steps. What follows are the steps I took to get my engine ready to start driving out features with RSpec and Capybara.

### Recipe ###

Setup environment

    rvm gemset create squishable
    rvm gemset use squishable

Install Rails in the gemset so we can use the plugin generator

    gem install rails

Generate the engine

    rails plugin new squishable --mountable --skip-test-unit --dummy-path=spec/dummy
    cd squishable

Add project environment file

    echo 'rvm @squishable' > .rvmrc

Initial commit

    git init && git add . && git commit -am"generated engine"

Open `.gitignore`

    emacs .gitignore

Change references from `test/dummy` to `spec/dummy`

*Note: This is a bug, hopefully it gets [fixed](https://github.com/rails/rails/pull/3066)*

{% codeblock .gitignore %}

spec/dummy/db/*.sqlite3
spec/dummy/log/*.log
spec/dummy/tmp/

{% endcodeblock %}

Commit

    git commit -am"changed references from test/dummy to spec/dummy in gitignore"

Open `squishable.gemspec`

    emacs squishable.gemspec

Add Rails and testing dependencies

*Note: I had to pin rack to v1.3.3 to avoid strange warnings*

{% codeblock squishable.gemspec lang:ruby %}

s.add_development_dependency "rack", "1.3.3"
s.add_development_dependency "rspec-rails", "~> 2.6.1"
s.add_development_dependency "capybara", "~> 1.1.1"

{% endcodeblock %}

Install everything

    bundle update rack && bundle install

Install rspec config file and spec_helper

    rails g rspec:install

Open `spec_helper`

    emacs spec/spec_helper.rb

The spec helper is mostly broken in the context of an engine, replace with:

{% codeblock spec/spec_helper.rb lang:ruby %}

# Configure Rails Environment
ENV["RAILS_ENV"] = "test"
require File.expand_path("../dummy/config/environment.rb", __FILE__)
require 'rspec/rails'

Rails.backtrace_cleaner.remove_silencers!

# Load support files
Dir["#{File.dirname(__FILE__)}/support/**/*.rb"].each { |f| require f }

RSpec.configure do |config|
  config.use_transactional_fixtures = true
end

{% endcodeblock %}

Open engine's `Rakefile`

    emacs Rakefile

Add RSpec tasks and default task

{% codeblock Rakefile lang:ruby %}

require 'rspec/core/rake_task'

RSpec::Core::RakeTask.new(:spec)
task :default => :spec

{% endcodeblock %}

Commit

    git add . && git commit -m"added rspec"

### Setup RSpec/Capybara acceptance tests ###

Make directory for acceptance tests

    mkdir -p spec/acceptance

Open `acceptance_helper.rb`

    emacs spec/acceptance/acceptance_helper.rb

And add

{% codeblock spec/acceptance/acceptance_helper.rb lang:ruby %}

require 'spec_helper.rb'
require 'capybara/rspec'

{% endcodeblock %}

Commit

    git add . && git ci -am"installed capybara"

Open `squishes_spec.rb`

    emacs spec/acceptance/squishes_spec.rb

And add

{% codeblock spec/acceptance/squishes_spec.rb lang:ruby %}

require 'acceptance/acceptance_helper'

feature "Squishes", %q{
  In order to be better than my friends
  As a kid
  I want to create and manage my squishes
} do

  background do
    Squishable::Squish.create!(:squash => 'Squoshy Squish')
    Squishable::Squish.create!(:squash => 'Sploshy Squish')
  end

  scenario "View a list of squishes" do
    visit '/squishable/squishes'
    page.should have_content('Squoshy Squish')
    page.should have_content('Sploshy Squish')
  end
end

{% endcodeblock %}

Run specs to see them fail

    rake

Generate the code and migrate database to make the tests pass

*Note: The second call to db:migrate is needed in lieu of a rake:test:prepare equivalent for ab engine generated with `plugin new`*

    rails g scaffold Squishes squash:string
    rake db:migrate
    rake db:migrate RAILS_ENV=test

Run specs to see them pass

    rake

One last, but important, step is to change any controller you create in the engine to extend `::ApplicationController` rather than `ApplicationController::Base`. Among other things this will cause your engines views to be rendered in the main application's layout rather than the engine's.

Open `app/controllers/squishable/squishes_controller.rb`

    emacs app/controllers/squishable/squishes_controller.rb

Change the class definition:

{% codeblock app/controllers/squishable/squishes_controller.rbb lang:ruby %}

class SquishesController < ::ApplicationController

{% endcodeblock %}

Get rid of application layout in engine, we don't need it now that we're inheriting from the main app's controller.

    git rm -r app/views/layouts

Commit

    git add . && git ci -m"squishable scaffold" \
      -m$'rails g scaffold Squishes squash:string\nrake db:migrate\nremoved engine app layout'

See the resulting [repo on github](https://github.com/mikebannister/squishable).

### References ###

[Rails 3 In Action](http://www.manning.com/katz/) by Ryan Bigg and Yehuda Katz

[Rails::Engine documentation](https://github.com/rails/rails/blob/master/railties/lib/rails/engine.rb)

[Acceptance testing using Capybara's new RSpec DSL](http://jeffkreeftmeijer.com/2011/acceptance-testing-using-capybaras-new-rspec-dsl/) by Jeff Kreeftmeijer
