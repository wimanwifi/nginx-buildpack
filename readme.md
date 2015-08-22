# Heroku Buildpack: NGINX

This is Belly's fork of  [ryandotsmith/nginx-buildpack](https://github.com/ryandotsmith/nginx-buildpack). Here's what is different:

* Moved the port variable configuration to a file that can be included in any nginx config file. This is helpful for breaking a large config file into multiple files and still being able to achieve a variable port number (necessary for Heroku). [9c1528a](https://github.com/bellycard/nginx-buildpack/commit/9c1528ae218b57fe40724ca87a61eb08e4046504)
* Upgraded nginx to version 1.9.4 & added openssl support. Belly's requirements dictate that our proxy support SSL, so we added it, also upgraded to the latest nginx, since we were recompiling it anyway. [8233318](https://github.com/bellycard/nginx-buildpack/commit/82333189afdf0bffd9fc8f6d5885f363968e6059), [0176e38](https://github.com/bellycard/nginx-buildpack/commit/0176e389cc97f4059d68102b6eb9c5bc0755953f), [bfacddd](https://github.com/bellycard/nginx-buildpack/commit/bfacddd641e5f77c7eb005be4a030dbe0c310911)
* Added `start-nginx-solo`. The original buildpack expected that you were proxying a request to some app running within the same dyno. This isn't necessary for Belly's use case, so the solo option was added to remove some of the moving parts. [f14698e](https://github.com/bellycard/nginx-buildpack/commit/f14698eb7435cd1ee009486ae0c28a1ca4dc5923), [b19c940](https://github.com/bellycard/nginx-buildpack/commit/b19c940202b61300dfec508a1f1664c6caccc13a)
* Added a configurable domain name option [b010d13](https://github.com/bellycard/nginx-buildpack/commit/b010d13ad54460364aa04e83023bea507648d648), [001259a](https://github.com/bellycard/nginx-buildpack/commit/001259a2f3d2322a2b37f9cb709da140c6411cdd)

## Recompile nginx

In order to recompile nginx, you need to build it on the target system that it will be running on. In this case we want to build it for `cedar` and `cedar-14`. Here's how to do it:

1. Make a local directory for the app that will be used to do the compiling and `git init` it:

  ```
  mkdir nginx-builder
  cd nginx-builder
  git init .
  ```

2. Next, create an app on Heroku for your target stack using the multi buildpack.

  ```
  heroku create --buildpack https://github.com/ddollar/heroku-buildpack-multi.git --stack cedar-14 (or cedar)
  ```

  ```
  Creating infinite-wave-4795... done, stack is cedar-14
  Buildpack set. Next release on infinite-wave-4795 will use https://github.com/ddollar/heroku-buildpack-multi.git.
  https://infinite-wave-4795.herokuapp.com/ | https://git.heroku.com/infinite-wave-4795.git
  Git remote heroku added
  ```

3. Add the `ruby` and `nginx` buildpacks to the `.buildpacks` file.

  ```
  echo 'https://codon-buildpacks.s3.amazonaws.com/buildpacks/heroku/ruby.tgz' >> .buildpacks
  echo 'https://github.com/bellycard/nginx-buildpack.git' >> .buildpacks
  ```

4. Add a Gemfile. (doesn't really matter what is in there, it just needs a Gemfile to boot)

  ```
  echo "source 'https://rubygems.org'" >> Gemfile
  echo "gem 'unicorn'" >> Gemfile
  bundle install
  ```


5. Create a `Procfile` in your local directory to tell the app what to do when it starts up. `build_nginx.sh` is the compiler script. So when the dyno boots it will just start to compile right away.

  ```
  echo 'web: scripts/build_nginx.sh' >> Procfile
  ```

6. Commit those files locally:

  ```
  git add Procfile Gemfile Gemfile.lock .buildpacks
  git commit -m "Added Procfile, Gemfile & buildpacks"
  ```

7. Start tailing the logs in a separate terminal and deploy the app:

  ```
  heroku logs --tail
  ```

  ```
  git push heroku master
  ```

8. If all goes well, the compiler will kick off and you will see output in the logs. This will take some time, so go grab some coffee.

9. Once the compiler is done the logs will just output a `.`. From there you can go to your browser and get the built binary.

  ```
  heorku open
  ```

  The compiled binary will be in `/nginx/sbin/nginx`.

10. Once you have the file, you can delete the temporary app created on Heroku.c


### -- original readme below --


# Heroku Buildpack: NGINX

Nginx-buildpack vendors NGINX inside a dyno and connects NGINX to an app server via UNIX domain sockets.

## Motivation

Some application servers (e.g. Ruby's Unicorn) halt progress when dealing with network I/O. Heroku's Cedar routing stack [buffers only the headers](https://devcenter.heroku.com/articles/http-routing#request-buffering) of inbound requests. (The Cedar router will buffer the headers and body of a response up to 1MB) Thus, the Heroku router engages the dyno during the entire body transfer –from the client to dyno. For applications servers with blocking I/O, the latency per request will be degraded by the content transfer. By using NGINX in front of the application server, we can eliminate a great deal of transfer time from the application server. In addition to making request body transfers more efficient, all other I/O should be improved since the application server need only communicate with a UNIX socket on localhost. Basically, for webservers that are not designed for efficient, non-blocking I/O, we will benefit from having NGINX to handle all I/O operations.

## Versions

* Buildpack Version: 0.4
* NGINX Version: 1.5.7

## Requirements

* Your webserver listens to the socket at `/tmp/nginx.socket`.
* You touch `/tmp/app-initialized` when you are ready for traffic.
* You can start your web server with a shell command.

## Features

* Unified NXNG/App Server logs.
* [L2met](https://github.com/ryandotsmith/l2met) friendly NGINX log format.
* [Heroku request ids](https://devcenter.heroku.com/articles/http-request-id) embedded in NGINX logs.
* Crashes dyno if NGINX or App server crashes. Safety first.
* Language/App Server agnostic.
* Customizable NGINX config.
* Application coordinated dyno starts.

### Logging

NGINX will output the following style of logs:

```
measure.nginx.service=0.007 request_id=e2c79e86b3260b9c703756ec93f8a66d
```

You can correlate this id with your Heroku router logs:

```
at=info method=GET path=/ host=salty-earth-7125.herokuapp.com request_id=e2c79e86b3260b9c703756ec93f8a66d fwd="67.180.77.184" dyno=web.1 connect=1ms service=8ms status=200 bytes=21
```

### Language/App Server Agnostic

Nginx-buildpack provides a command named `bin/start-nginx` this command takes another command as an argument. You must pass your app server's startup command to `start-nginx`.

For example, to get NGINX and Unicorn up and running:

```bash
$ cat Procfile
web: bin/start-nginx bundle exec unicorn -c config/unicorn.rb
```

### Setting the Worker Processes

You can configure NGINX's `worker_processes` directive via the
`NGINX_WORKERS` environment variable.

For example, to set your `NGINX_WORKERS` to 8 on a PX dyno:

```bash
$ heroku config:set NGINX_WORKERS=8
```

### Customizable NGINX Config

You can provide your own NGINX config by creating a file named `nginx.conf.erb` in the config directory of your app. Start by copying the buildpack's [default config file](https://github.com/ryandotsmith/nginx-buildpack/blob/master/config/nginx.conf.erb).

### Customizable NGINX Compile Options

See [scripts/build_nginx.sh](scripts/build_nginx.sh) for the build steps. Configuring is as easy as changing the "./configure" options.

### Application/Dyno coordination

The buildpack will not start NGINX until a file has been written to `/tmp/app-initialized`. Since NGINX binds to the dyno's $PORT and since the $PORT determines if the app can receive traffic, you can delay NGINX accepting traffic until your application is ready to handle it. The examples below show how/when you should write the file when working with Unicorn.

## Setup

Here are 2 setup examples. One example for a new app, another for an existing app. In both cases, we are working with ruby & unicorn. Keep in mind that this buildpack is not ruby specific.

### Existing App

Update Buildpacks
```bash
$ heroku config:set BUILDPACK_URL=https://github.com/ddollar/heroku-buildpack-multi.git
$ echo 'https://github.com/ryandotsmith/nginx-buildpack.git' >> .buildpacks
$ echo 'https://codon-buildpacks.s3.amazonaws.com/buildpacks/heroku/ruby.tgz' >> .buildpacks
$ git add .buildpacks
$ git commit -m 'Add multi-buildpack'
```
Update Procfile:
```
web: bin/start-nginx bundle exec unicorn -c config/unicorn.rb
```
```bash
$ git add Procfile
$ git commit -m 'Update procfile for NGINX buildpack'
```
Update Unicorn Config
```ruby
require 'fileutils'
listen '/tmp/nginx.socket'
before_fork do |server,worker|
  FileUtils.touch('/tmp/app-initialized')
end
```
```bash
$ git add config/unicorn.rb
$ git commit -m 'Update unicorn config to listen on NGINX socket.'
```
Deploy Changes
```bash
$ git push heroku master
```

### New App

```bash
$ mkdir myapp; cd myapp
$ git init
```

**Gemfile**
```ruby
source 'https://rubygems.org'
gem 'unicorn'
```

**config.ru**
```ruby
run Proc.new {[200,{'Content-Type' => 'text/plain'}, ["hello world"]]}
```

**config/unicorn.rb**
```ruby
require 'fileutils'
preload_app true
timeout 5
worker_processes 4
listen '/tmp/nginx.socket', backlog: 1024

before_fork do |server,worker|
  FileUtils.touch('/tmp/app-initialized')
end
```
Install Gems
```bash
$ bundle install
```
Create Procfile
```
web: bin/start-nginx bundle exec unicorn -c config/unicorn.rb
```
Create & Push Heroku App:
```bash
$ heroku create --buildpack https://github.com/ddollar/heroku-buildpack-multi.git
$ echo 'https://codon-buildpacks.s3.amazonaws.com/buildpacks/heroku/ruby.tgz' >> .buildpacks
$ echo 'https://github.com/ryandotsmith/nginx-buildpack.git' >> .buildpacks
$ git add .
$ git commit -am "init"
$ git push heroku master
$ heroku logs -t
```
Visit App
```
$ heroku open
```

## License
Copyright (c) 2013 Ryan R. Smith
Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:
The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
