+++
date = '2026-03-24T00:00:00+02:00'
title = 'Ruby Sahha Integration'
author = 'Genry Wood'
tags = ['api', 'wrapper', 'ruby', 'health', 'sahha', 'healtkit']
+++

![Post Logo](header.png)

## What is Sahha?
Sahha is an app for tracking your health, sleep, activity, and more — [docs.sahha.ai.](Docs)
Put simply, Sahha is a powerful API and SDK for behavioral data analysis. It acts as a "bridge" between raw data from smartphones or wearable devices (Apple Health, Google Fit) and a deep understanding of a person's psychological and physical state.
Instead of just showing you step count graphs, Sahha uses sophisticated machine learning models to detect patterns in behavior. It turns passive data into insights about mental health, stress levels, and overall wellbeing.

The architecture is pretty simple. I did the same thing [in this article]({{< ref "posts/my-first-api-wrapper/index.md" >}}) for the API wrapper and followed the same approaches.

## How can you integrate Sahha
The most obvious first step is to get an API key for the sandbox. The sandbox gives you API access for 30 days.
Next, you need to choose how you'll receive data:
* _Using webhooks_ (in my opinion, the best approach)
* _Using polling_ (what I chose for my own needs, though I don't think it's the ideal option)

So let's start with what I actually implemented - polling. Why? I need to deliver the freshest possible data from Sahha right when the app launches. This isn't achievable with webhooks alone (mostly because of Apple's ecosystem). Apple HealthKit only returns health data when the user is actively using an app that syncs it - so if a user hasn't opened the app in a long time, there won't be fresh data available the instant they open it. With polling, we can show the user in real time that their data is loading (a similar effect could be achieved with sockets + webhooks). Here's how it works and what's needed for it.


### Architecture:

**configuration.rb** - stores credentials, current environment (sandbox/production), and the cached account token with its expiry.
```ruby
module Sahha
  class Configuration
    attr_accessor :client_id, :client_secret, :environment,
                  :account_token, :account_token_expires_at

    SANDBOX_URL = "https://sandbox-api.sahha.ai"
    PRODUCTION_URL = "https://api.sahha.ai"

    def initialize(client_id:, client_secret:, environment: :sandbox)
      @client_id = client_id
      @client_secret = client_secret
      @environment = environment
    end

    def base_url
      environment == :production ? PRODUCTION_URL : SANDBOX_URL
    end

    def account_token_valid?
      account_token && account_token_expires_at && Time.now < account_token_expires_at
    end
  end
end
```

**connection.rb** - a thin Faraday wrapper that builds requests with the correct base URL, headers, and auth token.
```ruby
require "faraday"
require "json"

module Sahha
  class Connection
    TOKEN_TYPE = "account"
    
    def initialize(configuration)
      @configuration = configuration
    end

    def get(path, params: {}, token: nil, token_type: TOKEN_TYPE)
      request(:get, path, params: params, token: token, token_type: token_type)
    end

    def post(path, body: {}, token: nil, token_type: TOKEN_TYPE)
      request(:post, path, body: body, token: token, token_type: token_type)
    end

    private

    def request(method, path, params: {}, body: {}, token: nil, token_type: TOKEN_TYPE)
      connection.public_send(method) do |req|
        req.url path
        req.params = params if params.any?
        req.headers["Content-Type"] = "application/json"
        req.headers["Authorization"] = "#{token_type} #{token}" if token
        req.body = body.to_json if body.any?
      end
    end

    def connection
      @connection ||= Faraday.new(url: @configuration.base_url)
    end
  end
end
```

**error_handler.rb** - maps HTTP status codes (200/201/204/400/401) to explicit exception classes so calling code can rescue specific failure modes instead of parsing raw responses.
```ruby
module Sahha
  class ApiError < StandardError; end
  class UnauthorizedError < ApiError; end
  class BadRequestError < ApiError; end
  class NoContentError < ApiError; end

  class ErrorHandler
    def self.handle(response)
      case response.status
      when 200, 201
        response
      when 204
        raise NoContentError, "No data available for this query yet"
      when 400
        raise BadRequestError, "Invalid parameters or externalId already in use"
      when 401
        raise UnauthorizedError, "Missing, invalid, or expired token"
      else
        raise ApiError, "Unexpected response: #{response.status} #{response.body}"
      end
    end
  end
end
```


**api.rb** - maps Sahha's actual endpoints (account token, profile, scores, refresh token) to connection calls, passing every response through the error handler.
```ruby
module Sahha
  class Api
    def initialize(connection)
      @connection = connection
    end

    def account_token(client_id, client_secret)
      response = @connection.post(
        "/api/v1/oauth/account/token",
        body: { clientId: client_id, clientSecret: client_secret }
      )
      ErrorHandler.handle(response)
    end

    def refresh_profile_token(refresh_token)
      response = @connection.post(
        "/api/v1/oauth/profile/refreshToken",
        body: { refreshToken: refresh_token }
      )
      ErrorHandler.handle(response)
    end

    def profile(external_id, account_token)
      response = @connection.get(
        "/api/v1/account/profile/#{external_id}",
        token: account_token
      )
      ErrorHandler.handle(response)
    end

    def scores(external_id, account_token, types: [])
      response = @connection.get(
        "/api/v1/profile/score/#{external_id}",
        params: { types: types.join(",") },
        token: account_token
      )
      ErrorHandler.handle(response)
    end
  end
end
```


**client.rb** - the public interface I actually use; it refreshes the account token before each call so we never have to think about token expiration.
```ruby
module Sahha
  class Client
    def initialize(client_id:, client_secret:, environment: :sandbox)
      @configuration = Configuration.new(
        client_id: client_id,
        client_secret: client_secret,
        environment: environment
      )
      @connection = Connection.new(@configuration)
      @api = Api.new(@connection)
    end

    def profile(external_id)
      ensure_account_token!
      @api.profile(external_id, @configuration.account_token)
    end

    def scores(external_id, types: ["activity", "sleep"])
      ensure_account_token!
      @api.scores(external_id, @configuration.account_token, types: types)
    end

    private

    def ensure_account_token!
      return if @configuration.account_token_valid?

      response = @api.account_token(@configuration.client_id, @configuration.client_secret)
      data = JSON.parse(response.body)
      @configuration.account_token = data["accountToken"]
      @configuration.account_token_expires_at = Time.now + 24 * 60 * 60
    end
  end
end
```

## Webhooks

The alternative to polling is webhooks - and honestly, for most integrations, it's the better choice (do not follow my polling flow xD). Instead of your app repeatedly asking Sahha if there anything new, Sahha pushes data to your server the moment an event happens like a new score is calculated or a biomarker updates. No wasted requests, no delay between the event and your server knowing about it - pretty simple.

Here's how it works. You register an endpoint URL in your Sahha dashboard. When something relevant happens for one of your profiles, Sahha sends a POST request to that URL.

Unfortunately I won't provide webhook processing but i would like to notice important think - do now forget to validate ur webhook. Sahha provides X-Signature, X-External-Id and X-Event-Type header in terms to validate them.

## Wrapping up

So that's basically my journey with Sahha so far. I won't pretend it was all smooth - figuring out the polling vs webhooks trade-off took me longer than I'd like to admit, and I definitely wasted a couple of evenings staring at HealthKit's quirks before it clicked why webhooks alone wouldn't cut it for my case.

But honestly? Once it clicked, it clicked. Having a service that turns raw sensor noise into something like "this person is more stressed than usual" or "their sleep quality dropped this week" - without me having to build any of the ML behind it feels like a genuine shortcut ( but do not forget about Sahha price :) )
