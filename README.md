# ServiceWorker::Rails

[![Build Status](https://travis-ci.org/rossta/serviceworker-rails.svg?branch=master)](https://travis-ci.org/rossta/serviceworker-rails)

Use [Service Worker](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) with the Rails asset pipeline.

## Why?

The Rails asset pipeline makes a number of assumptions about what's best for deploying JavaScript, including asset digest fingerprints and long-lived cache headers - mostly to increase "cacheability". Rails also assumes a single parent directory, `/public/assets`, to make it easier to look up the file path for a given asset.

Service worker assets must play by different rules. Consider these behaviors:

* Service workers may only be active from within the scope from which they are
served. So if you try to register a service worker from a Rails asset pipeline
path, like `/assets/serviceworker-abcd1234.js`, it will only be able to interact
with requests and responses within `/assets/`<em>**</em>. This is not what we want.

* [MDN states](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API#Download_install_and_activate) browsers check for updated service worker scripts in the background every 24 hours (possibly less). Rails developers wouldn't be able to take advantage of this feature since the fingerprint strategy means assets at a given url are immutable. Beside fingerprintings, the `Cache-Control` headers used for static files served from Rails also work against browser's treatment of service workers.

We want Sprockets to compile service worker JavaScript from ES6/7, CoffeeScript, ERB, etc. but must remove the caching and scoping mechanisms offered by Rails asset pipeline defaults. This is where `serviceworker-rails` comes in.

*Check out the [blog post](https://rossta.net/blog/service-worker-on-rails.html)
for more background.*

### Demo

See various examples of using Service Workers in the demo Rails app, [Service Worker Rails Sandbox](https://serviceworker-rails.herokuapp.com/). The [source code](https://github.com/rossta/serviceworker-rails-sandbox) is also on GitHub.


## Features

* Maps service worker endpoints to Rails assets
* Adds appropriate response headers to service workers
* Renders compiled source in production and development

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'serviceworker-rails'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install serviceworker-rails

## Usage

To use `serviceworker-rails` in a Rails app, install the gem as above. When
`serviceworker-rails` is required, it will insert a middleware into the Rails
middleware stack. You'll want to configure it by mapping serviceworker routes to
Sprockets JavaScript assets, like the example below, in `application.rb`.

```ruby
# config/application.rb

Rails.application.configure do
  config.serviceworker.routes.draw do
    # maps to asset named 'serviceworker.js' implicitly
    match "/serviceworker.js"

    # map to a named asset explicitly
    match "/proxied-serviceworker.js" => "nested/asset/serviceworker.js"
    match "/nested/serviceworker.js" => "another/serviceworker.js"

    # capture named path segments and interpolate to asset name
    match "/captures/*segments/serviceworker.js" => "%{segments}/serviceworker.js"

    # capture named parameter and interpolate to asset name
    match "/parameter/:id/serviceworker.js" => "project/%{id}/serviceworker.js"

    # insert custom headers
    match "/header-serviceworker.js" => "another/serviceworker.js",
      headers: { "X-Resource-Header" => "A resource" }

    # anonymous glob exposes `paths` variable for interpolation
    match "/*/serviceworker.js" => "%{paths}/serviceworker.js"
  end
end
```

`Serviceworker::Rails` will insert a `Cache-Control` header to instruct browsers
not to cache your serviceworkers by default. You can customize the headers for all service worker routes if you'd like,
such as adding the experimental [`Service-Worker-Allowed`](https://slightlyoff.github.io/ServiceWorker/spec/service_worker/#service-worker-allowed) header to set the allowed scope.

```ruby
config.serviceworker.headers["Service-Worker-Allowed"] = "/"
config.serviceworker.headers["X-Custom-Header"] = "foobar"
```

### Precompilation

For use in production, instruct Sprockets to precompile service worker scripts separately from `application.js`, as in the following example:

```ruby
# config/initializers/assets.rb

Rails.application.configure do
  config.assets.precompile += %w[
    serviceworker.js
  ]
end
```

### Tutorial

Not sure how to start? This section is for you.

Let's add a `ServiceWorker` to cache some of your JavaScript and CSS assets. We'll assume you already have a Rails application using the asset pipeline built on Sprockets.

#### Setup

Add `serviceworker-rails` to your `Gemfile` [as described above](#installation) and run `$ bundle install`.

Create a JavaScript file called `app/assets/javascripts/serviceworker.js.erb`:

```javascript
// app/assets/javascripts/serviceworker.js.erb
console.log('[Service Worker] Hello world!');

self.addEventListener('install', function onInstall(event) {
  event.waitUntil(
    caches.open('cached-assets').then(function prefill(cache) {
      return cache.addAll([
        '<%= asset_path "application.js" %>',
        '<%= asset_path "application.css" %>',
        '<%= asset_path "admin.css" %>',
        // you get the idea ...
      ]);
    })
  );
});
```

You'll need to register the service worker with a companion script in your main page JavaScript, like `application.js`. You can use the following:

```javascript
// app/assets/application.js
// rest of your js ...

if (navigator.serviceWorker) {
  navigator.serviceWorker.register('/serviceworker.js', { scope: './' })
    .then(function(reg) {
      console.log('[Page] Service worker registered!');
    });
}
```

Add a snippet of Ruby in `config/application.rb` as show below. This can also go in a new initializer file like `config/initializers/serviceworker.rb`.

```ruby
# config/application.rb

Rails.application.configure do
  config.serviceworker.routes.draw do
    match "/serviceworker.js"
  end
end
```

At this point, restart your Rails app and reload a page in your app in Chrome or Firefox. Using dev tools, you should be able to determine.

1. The page requests a service worker at `/serviceworker.js`
2. The Rails app responds to the request by compiling and rendering the file in `app/assets/javascripts/serviceworker.js.erb`.
3. The console displays messages from the page and the service worker
4. The application JavaScript and CSS assets are added to the browser's request/response [Cache](https://developer.mozilla.org/en-US/docs/Web/API/Cache).

#### Using the cache

So far so good? At this point, all we've done is pre-fetched assets and added them to the cache, but we're not doing anything with them yet.

Now, we can use the service worker to intercept requests and either serve them from the cache if they exist there or fallback to the network response otherwise. In most cases, we can expect responses coming from the local cache to be much faster than those coming from the network.

(...more coming soon, WIP)

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `bin/rake` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/[USERNAME]/serviceworker-rails. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](http://contributor-covenant.org) code of conduct.

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).

