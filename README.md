# Orchparty

[![Build Status](https://travis-ci.org/jannishuebl/orchparty.svg?branch=master)](https://travis-ci.org/jannishuebl/orchparty)
[![Gem Version](https://badge.fury.io/rb/orchparty.svg)](https://badge.fury.io/rb/orchparty)

Write your own orchestration config with a Ruby DSL that allows you to have mixins, imports and variables.

```ruby
import "../logging.rb"

application 'my-cool-app' do
  variables do
    var port: 8080
  end

  service "api" do
    mix "logging.syslog"
    image "my-cool-image:latest"
    commad -> { "bundle exec rails s -b 0.0.0.0 -p #{port}" }
  end

end
```

## Why the hell?

### 1. Powerfull Ruby DSL as a YAML replacement

YAML is great for configuration, it has a clean syntax that is
readable for humans as well as machines. In addition it suites for the most of the
configurations. Furthermore YAML supports features like referencing,
inheritence and multiline Strings. 

In our company we have a microservice architecture with multiple applications,
which is consisting of multiple services (container types).
After a while we have realized that our orchestration configuration were growing continuously and got
hell complex, which determine global changes like replacing our logging
infrastucture. This changes were quite painfull, because we need to touch every single service.

Some main features for us:

  1. Mixin support
  1. Import from different files
  1. Using variables in imported configs e.g. Stack/Service names


Some of the features are already included in YAML, unfortunately we are not able to use it, because of the complexity.


### 2. Use Ruby instead of Templating engines

Most of the orchestration frameworks are using a derivative of docker-compose.yml.
But most of the users realized that yml is not enough for complex
orchestration.

So most of the framework teams started to allow templating engines in the
docker-compose configuration.

But why keep going with a data serialization language when we want to program
our own configuration?

### 3. Have one config for multiple orchestration frameworks

How much effort is it to get a application running on an
orchestration frameworks? Actually we are glad about finding a prebuild docker-compose file
which can be modified by us e.g. for kontena.io, but after modifying kontena.io we
have to redo nearly all the work for rancher, kubernets etc.

It would be really nice, if people starting to write an opensource application config like
orchparty and we just simply compile the config for all popular orchestration frameworks.

## Installation

Setup a Ruby Enviroment with Ruby 2.2 or higher is necessary for the intallation.

Install from rubygems.org

    $ gem install orchparty

Maybe for the future it is possible to run the gem in a docker container, so no local Ruby Environment is needed.

## Usage

See the commandline usage instrucution by running:

    $ orchparty help

## DSL spec

So let us start an example! Let us implement a beautiful app called [app_perf](https://github.com/randy-girard/app_perf) with orchparty. This App is the opensource replacement for New Relic!

### Applications

app_perf needs the following components:
- postgres for data storage 
- redis as queuing system
- web handler as receiver for all metrics 
- worker for processing the metrics and inserting them to the postgres db.

```ruby
application "app_perf" do

  service "web" do
    image "blasterpal/app_perf"
    command "bundle exec rails s -p 5000"
    expose 5000
    links do 
      link "redis"
      link "postgres"
    end
  end

  service "worker" do
    image "blasterpal/app_perf"
    command "bundle exec sidekiq"
    links do 
      link "redis"
      link "postgres"
    end
  end

  service "redis" do
    image "redis"
  end

  service "postgres" do
    image "postgres"
  end

end
```


### Applevel Mixins

For using external services like RDS from AWS we do not want to ship postgres in a production setup.

```ruby
mixin "app_perf" do

  service "web" do
    image "blasterpal/app_perf"
    command "bundle exec rails s -p 5000"
    expose 5000
    links do 
      link "redis"
    end
  end

  service "worker" do
    image "blasterpal/app_perf"
    command "bundle exec sidekiq"
    links do 
      link "redis"
    end
  end

  service "redis" do
    image "redis"
  end

end

application 'app_perf-dev' do
  mix "app_perf"

  service "web" do
    links do 
      link "postgres"
    end
  end

  service "worker" do
    links do 
      link "postgres"
    end
  end

  service "postgres" do
    image "postgres"
  end

end

application 'app_perf-prod' do
  mix "app_perf"

  service "web" do
    environment do
      env POSTGRES_HOST: "rds-domain.amazon.com"
    end
  end

  service "worker" do
    environment do
      env POSTGRES_HOST: "rds-domain.amazon.com"
    end
  end

end
```

### Service level Mixin

But we might also mixin a logging config in production.

```ruby
mixin "logging" do

  service "syslog" do
    logging do 
      conf driver: "syslog"
      options do
        opt syslog-address: "tcp://192.168.0.42:123"
      end
    end
  end

end


application 'app_perf-prod' do
  mix "app_perf"

  service "web" do
    mix "logging.syslog"
    environment do
      env POSTGRES_HOST: "rds-domain.amazon.com"
    end
  end

  service "worker" do
    mix "logging.fluentd"
    environment do
      env POSTGRES_HOST: "rds-domain.amazon.com"
    end
  end

end
```

### Commonblock

Using the all-block for adding configs to all services in one application. Of course the mix "logging.syslog" and environment variables will also added to the redis and postgres service.

```ruby
application 'app_perf-prod' do
  mix "app_perf"

  all do
    mix "logging.syslog"
    environment do
      env POSTGRES_HOST: "rds-domain.amazon.com"
    end
  end

  service "web" do
  end

  service "worker" do
  end

end
```

### Variables

You want to use variables right? Because "DRY" ;) well you can: 

```ruby
application "app_perf" do
  variables do
    var image: "blasterpal/app_perf"
  end

  service "web" do
    variables do
      # service local variables
    end
    image -> { image }
    command -> { "bundle exec rails s -p #{ service.expose }" }
    expose 5000
    links do 
      link "redis"
      link "postgres"
    end
  end

  service "worker" do
    image -> { image }
    command "bundle exec sidekiq"
    links do 
      link "redis"
      link "postgres"
    end
  end

  service "redis" do
    image "redis"
  end

  service "postgres" do
    image "postgres"
  end

end
```

special variables:

  1. service:
    - service.name
  1. application:
    - application.name

### Import

Above we assumed that everything is written in one file. If you do not want to, just use the import feature.

```ruby
import "../logging.rb"
import "./app_perf.rb"

application 'app_perf-prod' do
  mix "app_perf"

  all do
    mix "logging.syslog"
    environment do
      env POSTGRES_HOST: "rds-domain.amazon.com"
    end
  end

end
```


## Development

After checking out the repo:
1. run `bin/setup` to install dependencies
2. run `rake spec` to run the tests
You can also run `bin/console` for an interactive prompt that will allow you to make some experiments.

To install this gem onto your local machine, run `bundle exec rake install`.

To release a new version:
1. update the version number in `version.rb`
2. run `bundle exec rake release` which will create a git tag for the version
3. push git commits and tags and additionally push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/jannishuebl/orchparty.
This project is intended to be a safe, welcoming space for collaboration,
and contributors are expected to adhere to the
[Contributor Covenant](http://contributor-covenant.org) code of conduct.


## License

The gem is available as opensource project under the terms of the [GNU Lesser General Public License v3.0](http://www.gnu.de/documents/lgpl-3.0.en.html).
