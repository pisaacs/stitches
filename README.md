You'll be in stitches at how easy it is to get your API service up and running!

![build status](https://travis-ci.org/stitchfix/resque-brain.svg?branch=master)

This gem provides:

* Rails plugin to handle api key and versioned requests
* Generator to get you set up and validate your API before you write any code
* Spec helpers to use when building your API

## To install

Add to your `Gemfile`:

```ruby
gem 'stitches'
```

Then of course:

```
bundle install
```

If your app is brand new or you don't already have rspec or Apitome, be sure to run those generators and set them up first. You can skip this if you've done either of these before:

```
rails generate rspec:install
rails generate apitome:install
```

Then, run the Stitches generator:

```
rails g stitches:api
```

Run database migrations (do `rake db:create` first if you haven't yet set up a database)

```
rake db:migrate
```


## Test the install

In your app run:

```
rake generate_api_key[some_name_you_pick]
```

Then follow the instructions it outputs.

If you are doing this locally and get an error like the following:

> WEBrick::HTTPStatus::LengthRequired

that's because WEBrick doesn't want an empty body. Throw on a `-d ''` to the end of the command.

## Rails Plugin

Just using this gem will get you:

* Middleware that requires an api key in the request.  See `Stitches::ApiKey`.
* Middleware that requires a versioned JSON mime type.  We aren't using version numbers in the URLs but putting versions in the `Accept` header.  See `Stitches::ValidMimeType`.
* Monkeypatch ActiveSupport's `TimeWithZone` to use ISO8601 UTC as the encoding for all timestamps.  This ensures that every JSON response is using the same timezone and can be parsed properly with JavaScript.
* `Error` and `Errors` objects for creating error reponses that will be rendered properly in API responses.  These can work with exceptions, ActiveRecord objects, or whatever you like.

The plugin works in conjunction with other pieces of this gem set up by using the generator.

### A word on errors

The error format generated by `Errors` and friends is designed for two purposes:

* provide a "programmer key" upon which logic can be based for each error
* provide a message to explain the error to the programmer and, in times of desperation, the user

The idea is that the caller should not have made a call that generates an error, although this isn't always 100% possible to avoid.  As such, the error messages that come back are not as "rich" as you might get
from ActiveRecord.  APIs aren't intended to be used for validating user input.  The messages are aimed at a programmer trying to understand why a call failed.

## Generator

When you run the generator via `rails g stitches:api`, you'll get a lot of handy setup done for you:

* Two "ping" controllers that do nothing but respond to requests.  This are useful when creating an API client or when validating
a deployed instance, as they will succeed when all the moving parts are working, but won't actually hit a database or depend on
any particular business logic.
* Routes configured using `Stitches::ApiVersionConstraint` to route to your ping controllers based on the version in
the `Accept` header.  This allows your api client to fully validate that it's using the versioning system properly.
* An `ApiClient` model and migration for your database.  This additionally sets up UUID support in Postgres.
* An `api_spec.rb` spec that will spin up your application and check that all the RESTful/HTTP stuff is working as designed.
* A means to write API documentation via [rspec_api_documentation](https://github.com/zipmark/rspec_api_documentation), and serve it nicely via [apitome](https://github.com/modeset/apitome).  You write tests using a slightly modified DSL that allows you to document the API.  The produced documentation shows the headers and wire formats with your test data.

Once this is done, you can (and possibly should) deploy your app to validate everything's working before you get too into your
business logic.

Once you start writing your app, there's a few spec helpers available.

## Generator-provided tests

The generator will provide some tests for your app to test a base level of functionality. To use them, add the Capybara gem to your app:

```
gem 'capybara'
```

And in your `spec_helper.rb`:

```ruby
require 'capybara/rails'
```

## Spec Helpers

```ruby
# in your spec_helper.rb
require 'stitches/spec'
```

### `have_api_error` 

This can be used in a feature spec on a response object to check that the error format is in our "canonical" format, which is as
follows:

```json
{
  "errors": [
    {
      "code": "some_code",
      "message": "some readable message"
    },
    {
      "code": "some_other_code",
      "message": "some other readable message"
    }
    # etc.
  ]
}
```

To assert this in your specs:

```ruby
expect(response).to have_api_error(status: 404,
                                   code: "not_found",
                                   message: "the exact error message")

# regexps also work
expect(response).to have_api_error(status: 404,
                                   code: "name_invalid",
                                   message: /can't be blank/)
```

Omitting `status` will use a default of 422, which is Rails' default for validation errors.

### `be_iso8601_utc_encoded`

This can be used to ensure that your JSON-encoded timestamps meet the recommended format of IS8601 UTC-encoded strings.

```ruby
expect(json["created_at"]).to be_iso8601_utc_encoded
```

### `TestHeaders`

This class is useful when creating headers in feature specs that will conform to what's needed by the API you are building.
The generated `api_spec.rb` is a complete example, but here's a sampling:

```ruby
# Create headers for a version 2 request
headers = TestHeaders.new(version: 2)
post "/api/ping", {}.to_json, headers.headers

# Create headers that have the wrong mime type in them
headers = TestHeaders.new(mime_type: "application/foobar")
post "/api/ping", {}.to_json, headers.headers
```

### `ApiClients` module

Mixing this in provides the method `api_client` that will return the one api client.  This is because fixtures have been so flaky
in our tests, and factories don't always work for stuff like this.  Also, why require the use of factories?

## Configuration

You'll see in the initializer that you can configure aspects of the library globally.  `Stitches::Configuration` documents the available
options.

## Avoiding auto-magical Set-up

Instead of requiring `stitches`, you can avoid the railtie and configure things yourself:

```
# Gemfile
gem 'stitches', require: false

# config/initializers/stitches.rb
require 'stitches_norailtie'

Stitches.configure do |config|
  config.whatever = :foo
end

# config/application.rb
config.app_middleware.use "Stitches::ApiKey", except: %r{/super-secret}
config.app_middleware.use "Stitches::ValidMimeType", except: %r{/super-secret}
# or whatever you want to do
```

## Developing

Although `Stitches.configuration` is global, do not depend directly on that in your logic.  Instead, allow all classes to receive a
configuration object in their constructor.  This makes the classes easier to deal with and change, without incurring much of a real cost to development.
Glaobl symbols suck, but are convienient.  This is how you make the most of it.


---

Provided with love by your friends at [Stitch Fix Engineering](http://technology.stitchfix.com)

![stitches](https://s3.amazonaws.com/stitchfix-stitches/stitches.png)
