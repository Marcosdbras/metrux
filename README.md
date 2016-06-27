# Metrux

An instrumentation library which persists the metrics on InfluxDB.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'metrux'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install metrux

## Configuration

You will need to create a config file on `config/metrux.yml`. See a
[sample](config/metrux.sample.yml) here.

Its configuration is based on ENV VARS and/or YAML, pretty similar to
`appsignal` or `newrelic` gems. ENV VARS will always override yaml configs.

### Configuration keys

| ENV VAR | yaml | Default | Description |
| ------- | ---- | ------- | ----------- |
| `METRUX_ACTIVE` | `active` | `false` | Whether it is active. Can be true or false |
| `METRUX_APP_NAME` | `app_name` |  | Your application's name (All metrics will be marked with this tag) |
| `METRUX_PERIODIC_GAUGE_INTERVAL` | `periodic_gauge_interval` | `60` | Interval that agent will execute all registered periodic gauges (in seconds) |
| `METRUX_LOG_FILE` | `log_file` | `STDOUT` | Log file path |
| `METRUX_LOG_LEVEL` | `log_level` | `info` | Log level |
| `METRUX_INFLUX_HOST` | `influx_host` |  | InfluxDB host - See: https://github.com/influxdata/influxdb-ruby#creating-a-client |
| `METRUX_INFLUX_PORT` | `influx_port` |  | InfluxDB port |
| `METRUX_INFLUX_DATABASE` | `influx_database` |  | InfluxDB database |
| `METRUX_INFLUX_USERNAME` | `influx_username` |  | InfluxDB username |
| `METRUX_INFLUX_ASYNC` | `influx_async` |  | InfluxDB async options - See: https://github.com/influxdata/influxdb-ruby#writing-data |

## Usage

Before you start sending your metrics to InfluxDB, is very important that you
read [InfluxDB Schema Design](https://docs.influxdata.com/influxdb/v0.13/concepts/schema_and_data_layout/#encouraged-schema-design)
for a better understanding of how to use `tags` and `fields`.

All writes will automatically include the tags `app_name` and `hostname`, unless
you pass another value.

### Write

Writes a point.

#### Single value/field

```ruby
key = 'my_awesome_key'
data =  1

# Options are not required
options = { tags: { something: 'a-string-value' }, precision: 's' }

Metrux.write(key, data, options)
```

Result:
```
name: my_awesome_key
--------------------
time                    app_name      hostname       something       uniq      value
1466604892000000000     Your appname  YOURHOSTNAME   a-string-value  ebb28331  1
```

#### Multi value/field

```ruby
key = 'my_awesome_key'
data =  { another_field: 1, value: 2 }

# Options are not required
options = { tags: { something: 'a-string-value' }, precision: 's' }

Metrux.write(key, data, options)
```

Result:
```
name: my_awesome_key
--------------------
time                    app_name      hostname       another_field  something       uniq      value
1466604892000000000     Your appname  YOURHOSTNAME   1              a-string-value  ebb28331  2
```

### Meter

Writes a meter (aka counter). All meters' key will have the prefix `meters/`.

#### Simple increment (value: 1)

```ruby
key = 'my_meter'

# Options are not required
options = { tags: { something: 'a-string-value' }, precision: 's' }

Metrux.meter(key, options)
```

Result:
```
name: meters/my_meter
---------------------
time                    app_name      hostname          something       uniq            value
1466604892000000000     Your appname  YOURHOSTNAME      a-string-value  4ce9827e        1
```


#### Different increment (value <> 1)

```ruby
key = 'my_meter'

value = 5

# Options are not required
options = {
  value: value, tags: { something: 'a-string-value' }, precision: 's'
}

Metrux.meter(key, options)
```

Result:
```
name: meters/my_meter
---------------------
time                    app_name      hostname          something       uniq            value
1466604892000000000     Your appname  YOURHOSTNAME      a-string-value  9930ea84        5
```

### Gauge

Writes a gauge (result of something). All gauges' key will have the prefix
`gauges/`.

#### Execute a block and save its result

```ruby
key = 'my_gauge'

# Options are not required
options = { tags: { something: 'a-string-value' }, precision: 's' }

Metrux.gauge(key, options) { 40 }
# => 40
```

Result:
```
name: gauges/my_gauge
---------------------
time                    app_name      hostname          something       uniq            value
1466604892000000000     Your appname  YOURHOSTNAME      a-string-value  f0e6e7da        40
```

The rule for multi value/field is the same of [write](#write).


#### Just save the result of a previously executed block

```ruby
key = 'my_gauge'

result = 42

# Options are not required
options = {
  result: result, tags: { something: 'a-string-value' }, precision: 's'
}

Metrux.gauge(key, options)
# => 42
```

Result:
```
name: gauges/my_gauge
---------------------
time                    app_name      hostname          something       uniq            value
1466604892000000000     Your appname  YOURHOSTNAME      a-string-value  75538b33        42
```

The rule for multi value/field is the same of [write](#write).

### Periodic gauge

Executes periodically a gauge (result of something) and writes it. All periodic
gauges' key will have the prefix `gauges/`.

The interval of each execution will depend on the configuration. See
[Configuration](#configuration).

**Remember that gauges must be lightweight and very fast to execute.**

```ruby
key = 'my_periodic_gauge'

# Options are not required
options = { tags: { something: 'a-string-value' }, precision: 's' }

Metrux.periodic_gauge(key, options) { Thread.list.count }
```

Result after having passed (interval * 1) seconds:
```
name: gauges/my_gauge
---------------------
time                    app_name      hostname          something       uniq            value
1466609741000000000     Your appname  YOURHOSTNAME      a-string-value  f0e6e7da        6
```

Result after having passed (interval * 2) seconds:
```
name: gauges/my_gauge
---------------------
time                    app_name      hostname          something       uniq            value
1466609741000000000     Your appname  YOURHOSTNAME      a-string-value  f0e6e7da        6
1466609746000000000     Your appname  YOURHOSTNAME      a-string-value  2eb7a01b        6
```

The rule for multi value/field is the same of [write](#write).

### Timer

Calculates and writes the time elapsed of something. All timers' key will have
the prefix `timers/`.

#### Execute a block and save its execution time

```ruby
key = 'my_timer'

# Options are not required
options = { tags: { something: 'a-string-value' }, precision: 's' }

Metrux.timer(key, options) { sleep(0.45); 40 }
# => 40
```

Result:
```
name: timers/my_timer
---------------------
time                    app_name      hostname          something       uniq            value
1466604892000000000     Your appname  YOURHOSTNAME      a-string-value  f0e6e7da        455
```

#### Just save the duration of a previously calculated block

```ruby
key = 'my_timer'

duration = 1342 # milliseconds

# Options are not required
options = {
  duration: duration, tags: { something: 'a-string-value' }, precision: 's'
}

Metrux.timer(key, options)
# => nil
```

Result:
```
name: timers/my_timer
---------------------
time                    app_name      hostname          something       uniq            value
1466604892000000000     Your appname  YOURHOSTNAME      a-string-value  f0e6e7da        1342
```

### Notice error

Meters errors.

```ruby
def do_something(a, b)
  raise(ArgumentError, 'Some message') if b = 0
rescue => e
  # Args are not required
  args = { a: a, b: b, uri: 'http://domain.tld/path' }

  Metrux.notice_error(e, args)

  raise
end

do_something(1, 0)
```

Result:
```
name: meters/errors
---------------------
time                 a  app_name       b  error           hostname       message         uniq      uri                       value
1466608927000000000  1  Your appname   0  ArgumentError   YOURHOSTNAME   Some message    a033161c  "http://domain.tld/path"  1
```

### Plugins

* `Metrux::Plugins::Thread` - Register a periodic gauge to count the amount of
  running threads. See [Metrux::Plugins::Thread](lib/metrux/plugins/thread.rb)

#### Registering

You need to register the plugins to have them working. It's also possible to
register your own plugins on `Metrux`:

```ruby
module Metrux
  module Plugins
    class MyAwesomePlugin
      def initialize(config, options); @config, @options = config, options; end

      def call
        Metrux.periodic_gauge('threads_count', @options) { Thread.list.count }
      end
    end
  end
end

plugin = Metrux::Plugins::MyAwesomePlugin
options = { my: { plugin: :options } }

Metrux.register(plugin, options) # => true
```

Or you can use a `Proc` instead of a class:

```ruby
options = { my: { plugin: :options } }

Metrux.register(options) do |config, options|
  Metrux.periodic_gauge('threads_count', options) { Thread.list.count }
end # => true
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `bin/rspec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bin/rake install`. To release a new version, update the version number in `version.rb`, and then run `bin/rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).

## Git tags

Don't forget to tag your work! After a merge request being accepted, run:

1. (git tag -a "x.x.x" -m "") to create the new tag.
2. (git push origin "x.x.x") to push the new tag to remote.

Follow the RubyGems conventions at http://docs.rubygems.org/read/chapter/7 to know how to increment the version number. Covered in more detail in http://semver.org/

## Pull requests acceptance

Don't forget to write tests for your changes. It's very important to maintain the codebase's sanity. Any pull request that doesn't have enough test coverage will be asked a revision.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
