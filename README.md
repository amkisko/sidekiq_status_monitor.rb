# SidekiqStatusMonitor

[![Gem Version](https://badge.fury.io/rb/sidekiq_status_monitor.svg)](https://rubygems.org/gems/sidekiq_status_monitor)
[![Total Downloads](https://img.shields.io/gem/dt/sidekiq_status_monitor?color=blue)](https://rubygems.org/gems/https://rubygems.org/gems/sidekiq_status_monitor)

---

SidekiqStatusMonitor offers a solution to add liveness probe for a Sidekiq instance deployed in Kubernetes.
This library can be used to check sidekiq health outside kubernetes.

**How?**

An additional HTTP server is started during Sidekiq worker initialisation and on each request validates that the Sidekiq worker metrics look healthy.

It reports either 200 or 500 HTTP code with additional JSON payload which includes metrics values.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'sidekiq_status_monitor'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install sidekiq_status_monitor

## Usage

SidekiqStatusMonitor will start when running `sidekiq` command.

Run `Sidekiq`:

```bash
bundle exec sidekiq
```

Bash example:

```bash
curl -v localhost:7433
```

Ruby example:

```ruby
uri = URI.parse("http://localhost:7433")
Net::HTTP.get_response(uri).body
```

**how to disable?**
You can disabled by setting `ENV` variable `DISABLE_SIDEKIQ_STATUS`
example:

```bash
DISABLE_SIDEKIQ_STATUS=true bundle exec sidekiq
```

### Kubernetes setup

Set `livenessProbe` and `readinessProbe` in your Kubernetes deployment

example with recommended setup:

#### Sidekiq < 6

```yaml
spec:
  containers:
    - name: my_app
      image: my_app:latest
      env:
        - name: RAILS_ENV
          value: production
      command:
        - bundle
        - exec
        - sidekiq
      ports:
        - containerPort: 7433
      livenessProbe:
        httpGet:
          path: /
          port: 7433
        initialDelaySeconds: 80 # app specific. Time your sidekiq takes to start processing.
        timeoutSeconds: 5 # can be much less
      readinessProbe:
        httpGet:
          path: /
          port: 7433
        initialDelaySeconds: 80 # app specific
        timeoutSeconds: 5 # can be much less
      lifecycle:
        preStop:
          exec:
            # SIGTERM triggers a quick exit; gracefully terminate instead
            command: ['bundle', 'exec', 'sidekiqctl', 'quiet']
  terminationGracePeriodSeconds: 60 # put your longest Job time here plus security time.
```

#### Sidekiq >= 6

Create file:

_kube/sidekiq_quiet_

```bash
#!/bin/bash

# Find Pid
SIDEKIQ_PID=$(ps aux | grep sidekiq | grep busy | awk '{ print $2 }')
# Send TSTP signal
kill -SIGTSTP $SIDEKIQ_PID
```

Make it executable:

```
$ chmod +x kube/sidekiq_quiet
```

Execute it in your deployment preStop:

```yaml
spec:
  containers:
    - name: my_app
      image: my_app:latest
      env:
        - name: RAILS_ENV
          value: production
      command:
        - bundle
        - exec
        - sidekiq
      ports:
        - containerPort: 7433
      livenessProbe:
        httpGet:
          path: /
          port: 7433
        initialDelaySeconds: 80 # app specific. Time your sidekiq takes to start processing.
        timeoutSeconds: 5 # can be much less
      readinessProbe:
        httpGet:
          path: /
          port: 7433
        initialDelaySeconds: 80 # app specific
        timeoutSeconds: 5 # can be much less
      lifecycle:
        preStop:
          exec:
            # SIGTERM triggers a quick exit; gracefully terminate instead
            command: ['kube/sidekiq_quiet']
  terminationGracePeriodSeconds: 60 # put your longest Job time here plus security time.
```

## Options

```ruby
SidekiqStatusMonitor.setup do |config|
  # ==> Server host
  # Host to bind the server.
  # Can also be set with the environment variable sidekiq_status_HOST.
  # default: 0.0.0.0
  #
  #   config.host = 0.0.0.0

  # ==> Server port
  # Port to bind the server.
  # Can also be set with the environment variable sidekiq_status_PORT.
  # default: 7433
  #
  #   config.port = 7433

  # ==> Shutdown callback
  # When sidekiq process is shutting down, you can perform some action, like cleaning up created queue
  # default: proc {}
  #
  #    config.shutdown_callback = proc do
  #      Sidekiq::Queue.all.find { |q| q.name == "#{config.queue_prefix}-#{SidekiqStatusMonitor.hostname}" }&.clear
  #    end

  # ==> Rack server
  # Web server used to serve an HTTP response, e.g. 'webrick', 'puma', 'thin', etc.
  # Can also be set with the environment variable sidekiq_status_SERVER.
  # default: 'webrick'
  #
  #    config.server = 'puma'
end
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake spec` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`.

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/arturictus/sidekiq_status_monitor. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](http://contributor-covenant.org) code of conduct.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
