[![Build Status](https://travis-ci.org/bessey/brillo.svg?branch=master)](https://travis-ci.org/bessey/brillo)
[![Gem Version](https://badge.fury.io/rb/brillo.svg)](https://badge.fury.io/rb/brillo)

# Brillo

Brillo is a Rails database scrubber and loader, useful for making lightweight copies of your production database for development machines, with sensitive information obfuscated. Most configuration is done through YAML: Specify the models that you want to back up, what associations you want with them, and what fields should be obfuscated (and how).

Once that is done, dropping your local DB and replacing it with the latest scrubbed copy is as easy as `rake db:load`.

Under the hood we use [Polo](https://github.com/IFTTT/polo) to explore the classes and associations you specify in brillo.yml, obfuscated fields as configured.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'brillo'
# We currently rely on an unreleased version of Polo
gem 'polo',   github: 'IFTTT/polo'
```

Generate a starter `brillo.yml` file and `config/initializers/brillo.rb` with

```bash
$ rails g brillo_config
```

If you're using Capistrano, add Brillo's tasks to your Capfile:

```ruby
# Capfile
require 'capistrano/brillo'
```

Lastly, since the scrubber is pretty resource intensive you may wish to ensure it runs on separate hardware from your app servers:

```ruby
# config/deploy.rb
set :brillo_role, :my_batch_role
```

## Usage

Here's an example `brillo.yml` for IMDB:

```yaml
name: imdb            # Namespace the scrubbed file will occupy in S3
explore:
  user:               # Name of ActiveRecord class in snake_case
    tactic: all       # Scrubbing tactic to use (see Brillo:TACTICS for choices)
    associations:     # Associations to include in the scrub (ALL associated records included)
      - comments
  movie:
    tactic: latest    # The latest tactic explores the most recent 1,000 records
    associations:
      - actors
      - ratings
  admin/note:         # Corresponds to the Admin::Note class
    tactic: all
obfuscations:         #
  user.name: name     # Scrub user.name with the "name" scrubber (see Brillo::SCRUBBERS for choices)
  user.phone: phone
  user.email: email
```

In order to communicate with S3, Brillo expects `AWS_ACCESS_KEY` and `AWS_SECRET_KEY` to be set in the environment. It uses [Tim Kay's AWS cli](http://timkay.com/aws/) to communicate with AWS.

### Loading a database in development

```bash
$ rake db:load
```

### Loading a database on a stage

```bash
$ cap staging db:load
```

### Adding scrub tactics and obfuscations

If the built in record selection tactics aren't enough for you, or you need a custom obfuscation strategy, you can add them via the initializer. They are available in the YAML config like any other strategy.

```ruby
# config/initializers/brillo.rb

Brillo.configure do |config|
  config.add_tactic :oldest, -> (klass) { klass.order(created_at: :desc).limit(1000) }

  config.add_obfuscation :remove_ls, -> (field) {
    field.gsub(/l/, "X")
  }

  # If you need the context of the entire record being obfuscated, it is available in the second argument
  config.add_obfuscation :phone_with_id, -> (field, instance) {
    (555_000_0000 + instance.id).to_s
  }
end

```

## To Do

- Support S3 transfer via the usual AWS CLI
- Support alternative transfer mechanisms
