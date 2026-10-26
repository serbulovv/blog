+++
date = '2026-10-26T17:34:00+02:00'
title = 'JWT authentication in a Rails API'
author = 'Genry Wood'
tags = ['api', 'authentication', 'jwt', 'ruby', 'rails']
+++

![Post Logo](./header.png)

A browser session works because the server remembers it. After login the server stores a session, hands the browser an identifier in a cookie, and looks that identifier up on every request. An API consumed by mobile applications and other services has a harder time with this arrangement: cookies are not a natural fit for those clients, and every request needs a shared store that all servers can reach.

A signed token removes the lookup. The server gives the client a piece of data that states who the client is, together with a signature that only the server can produce. On the next request the server checks the signature and trusts the statement, without consulting any storage. JSON Web Token (JWT) is a standard format for such a token.

This article builds JWT authentication for a Rails API using only Ruby's standard library and a few lines of Rails, with no gem that hides the mechanism. The goal is to show what a token contains, what makes it trustworthy, and which decisions around it matter for security. Production code often uses a maintained library for the same job, since a library covers more algorithms and edge cases, but the library does the same three things shown here: encode, sign, verify.

## What a token looks like

A JWT is a string of three parts separated by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsImlhdCI6MTc5MjAwMDAwMCwiZXhwIjoxNzkyMDAwOTAwfQ.gIlk8jAtSFziXPMHn5HKhOIC7ceoNp8O4cbmnfcxGZc
```

The first two parts are JSON objects encoded with base64url, a variant of base64 that is safe to use in URLs and headers. Decoding them shows what they hold:

```
header:  {"alg":"HS256","typ":"JWT"}
payload: {"sub":"42","iat":1792000000,"exp":1792000900}
```

The header names the signing algorithm. The payload holds claims, which are statements about the token. Three of them are used in this article:

- `sub` (subject): who the token is about, here the user id.
- `iat` (issued at): when the token was created, in Unix seconds.
- `exp` (expiration): the moment after which the token must be rejected.

The third part is the signature. Base64url is an encoding and not encryption, so anyone who holds a token can read the header and the payload. The payload must not contain anything that should stay secret. What the signature provides is integrity: nobody can change the payload without the change being detected.

## How the signature works

The signature is an HMAC, a keyed hash. The server takes the encoded header and payload joined by a dot, and computes HMAC-SHA256 over that string with a secret that only the server knows. The result is encoded with base64url and appended to the token.

To verify a token, the server repeats the computation on the first two parts and compares the result with the third. If an attacker changes `"sub":"42"` to `"sub":"1"`, the payload changes, the expected signature changes, and the attacker cannot compute the new one without the secret. The token is rejected.

The secret therefore has to be long and random, and it must stay on the server. A value generated with `SecureRandom.hex(32)` and kept in Rails credentials meets both requirements:

```ruby
Rails.application.credentials.jwt_secret
```

## Encoding a token

The encoder lives in a small module. It builds the header and claims, encodes them, signs the result, and joins the three parts:

```ruby
# frozen_string_literal: true

# app/lib/auth/jwt.rb
require 'openssl'
require 'base64'
require 'json'

module Auth
  module Jwt
    class InvalidToken < StandardError; end
    class ExpiredToken < InvalidToken; end

    ALGORITHM = 'HS256'
    HEADER = { alg: ALGORITHM, typ: 'JWT' }.freeze

    module_function

    def encode(payload, secret:, expires_in: 900, now: Time.now.to_i)
      claims = payload.merge(iat: now, exp: now + expires_in)
      signing_input = "#{segment(HEADER)}.#{segment(claims)}"

      "#{signing_input}.#{sign(signing_input, secret)}"
    end

    def segment(data)
      Base64.urlsafe_encode64(JSON.generate(data), padding: false)
    end

    def sign(input, secret)
      Base64.urlsafe_encode64(OpenSSL::HMAC.digest('SHA256', secret, input), padding: false)
    end
  end
end
```

The default lifetime is 900 seconds, fifteen minutes. The `now` argument exists so that the time can be fixed when reproducing an example. In normal use it is not passed.

Calling `Auth::Jwt.encode({ sub: '42' }, secret: 'example-secret-do-not-use', now: 1_792_000_000)` returns the token shown at the beginning of this article.

## Verifying a token

Verification is the part that carries the security. It runs several checks in a fixed order, and a failure at any step rejects the token:

```ruby
module Auth
  module Jwt
    module_function

    def decode(token, secret:, now: Time.now.to_i)
      header_part, payload_part, signature = split(token)

      header = parse(header_part)
      raise InvalidToken, 'unsupported algorithm' unless header['alg'] == ALGORITHM

      expected = sign("#{header_part}.#{payload_part}", secret)
      unless ActiveSupport::SecurityUtils.secure_compare(expected, signature)
        raise InvalidToken, 'invalid signature'
      end

      payload = parse(payload_part)
      raise InvalidToken, 'missing exp' unless payload['exp'].is_a?(Integer)
      raise ExpiredToken, 'token expired' if payload['exp'] <= now

      payload
    end

    def split(token)
      parts = token.to_s.split('.', -1)
      raise InvalidToken, 'malformed token' unless parts.size == 3 && parts.none?(&:empty?)

      parts
    end

    def parse(part)
      JSON.parse(Base64.urlsafe_decode64(part))
    rescue ArgumentError, JSON::ParserError
      raise InvalidToken, 'malformed token'
    end
  end
end
```

1. **Structure.** A token that does not have exactly three non-empty parts is rejected before anything is decoded.
2. **Algorithm.** The header is attacker-controlled input, because the client sends the whole token. The server must not obey the `alg` field. It accepts one algorithm, HS256, and rejects the rest. Early JWT libraries accepted the value `none`, which means "unsigned", and a forged token with that header passed verification.
3. **Signature.** The expected signature is recomputed and compared with the one in the token. The comparison uses `secure_compare`, which takes the same time regardless of where the strings differ. A plain `==` stops at the first differing byte, and the difference in timing can in principle be measured to guess a signature byte by byte.
4. **Claims.** Only now is the payload parsed and its content trusted. A token without a numeric `exp` is rejected, because a token that never expires is not acceptable. A token whose `exp` is not in the future raises `ExpiredToken`.

The behavior on invalid input, run against the token above with the same secret:

```
decode(token, now: iat + 10)      # => {"sub"=>"42", "iat"=>1792000000, "exp"=>1792000900}
decode(token, now: iat + 900)     # Auth::Jwt::ExpiredToken: token expired
decode(token, secret: 'wrong')    # Auth::Jwt::InvalidToken: invalid signature
decode(token_with_sub_1)          # Auth::Jwt::InvalidToken: invalid signature
decode(token_with_alg_none)       # Auth::Jwt::InvalidToken: unsupported algorithm
decode('abc')                     # Auth::Jwt::InvalidToken: malformed token
```

`token_with_sub_1` is the original token with its payload replaced by one that has `"sub":"1"`, and the old signature left in place.

## Issuing tokens at login

Login is an ordinary request that exchanges credentials for a token. The password check is delegated to `has_secure_password`, and the controller returns a token only when it passes:

```ruby
# frozen_string_literal: true

# app/controllers/sessions_controller.rb
class SessionsController < ApiController
  def create
    user = User.find_by(email: params[:email].to_s.downcase)

    if user&.authenticate(params[:password])
      render json: Auth::Tokens.issue(user)
    else
      head :unauthorized
    end
  end
end
```

The response is the same whether the email is unknown or the password is wrong, so the endpoint does not reveal which accounts exist.

A second module keeps the token settings in one place and builds the response:

```ruby
# frozen_string_literal: true

# app/lib/auth/tokens.rb
module Auth
  module Tokens
    ACCESS_TTL = 15 * 60

    module_function

    def issue(user)
      {
        access_token: Jwt.encode({ sub: user.id.to_s }, secret: secret, expires_in: ACCESS_TTL),
        refresh_token: RefreshToken.issue_for(user),
        expires_in: ACCESS_TTL
      }
    end

    def user_from(token)
      payload = Jwt.decode(token, secret: secret)

      User.find_by(id: payload['sub'])
    rescue Jwt::InvalidToken
      nil
    end

    def secret
      Rails.application.credentials.jwt_secret
    end
  end
end
```

## Accepting tokens on requests

The client sends the access token in the `Authorization` header with the `Bearer` scheme. The base controller extracts it and asks `Auth::Tokens` for the user:

```ruby
# frozen_string_literal: true

# app/controllers/api_controller.rb
class ApiController < ActionController::API
  def current_user
    return @current_user if defined?(@current_user)

    @current_user = Auth::Tokens.user_from(bearer_token)
  end

  private

  def authenticate!
    head :unauthorized unless current_user
  end

  def bearer_token
    scheme, token = request.headers['Authorization'].to_s.split

    token if scheme == 'Bearer'
  end
end
```

Controllers that need a user opt in with a `before_action`:

```ruby
# frozen_string_literal: true

# app/controllers/posts_controller.rb
class PostsController < ApiController
  before_action :authenticate!

  def index
    render json: current_user.posts
  end
end
```

Every failure mode, whether a missing header, a wrong scheme, a bad signature or an expired token, ends in the same place: `user_from` returns `nil` and the response is `401 Unauthorized`. The client does not learn why the token was refused, and does not need to. An expired token has one consequence for the client, which is to obtain a new one.

No database table is consulted to validate the token. The only query on a request is the user lookup that most endpoints need anyway.

## Expiration and refresh

A token cannot be taken back once it is issued, since the server keeps no record of it. The only limit on the damage from a stolen token is its lifetime. This is why the access token is short, fifteen minutes in the example.

Asking the user for a password every fifteen minutes is not workable, so a second token handles renewal. A refresh token differs from the access token in two ways. It lives much longer, thirty days here. And it is not a JWT: it is a random string that the server stores and looks up. This is where the server keeps state on purpose, because the refresh token is the part that must be revocable.

```ruby
# frozen_string_literal: true

# db/migrate/XXXXXXXXXXXXXX_create_refresh_tokens.rb
class CreateRefreshTokens < ActiveRecord::Migration[7.1]
  def change
    create_table :refresh_tokens do |t|
      t.references :user, null: false, foreign_key: true
      t.string :token_digest, null: false, index: { unique: true }
      t.datetime :expires_at, null: false

      t.timestamps
    end
  end
end
```

```ruby
# frozen_string_literal: true

# app/models/refresh_token.rb
class RefreshToken < ApplicationRecord
  TTL = 30.days

  belongs_to :user

  def self.issue_for(user)
    raw = SecureRandom.urlsafe_base64(48)
    create!(user: user, token_digest: digest(raw), expires_at: TTL.from_now)

    raw
  end

  def self.consume(raw)
    transaction do
      record = lock.where('expires_at > ?', Time.current).find_by(token_digest: digest(raw.to_s))
      next unless record

      record.destroy!
      record.user
    end
  end

  def self.digest(raw)
    OpenSSL::Digest::SHA256.hexdigest(raw)
  end
end
```

The table stores a SHA-256 digest of the token and not the token itself. If the database leaks, the digests cannot be used as tokens. A plain fast hash is enough here because the token is 48 random bytes, unlike a password, which a person chooses and which therefore needs a slow hash.

`consume` finds the record, deletes it and returns the user, all under a row lock. A refresh token can be used once. The client receives a new pair of tokens in exchange, so a copied refresh token stops working as soon as the legitimate client uses its own copy, and two simultaneous requests with the same token cannot both succeed.

The endpoint that exchanges it:

```ruby
# frozen_string_literal: true

# app/controllers/refreshes_controller.rb
class RefreshesController < ApiController
  def create
    user = RefreshToken.consume(params[:refresh_token])

    if user
      render json: Auth::Tokens.issue(user)
    else
      head :unauthorized
    end
  end
end
```

## Logout and revocation

For logout the server deletes the user's refresh tokens:

```ruby
current_user.refresh_tokens.delete_all
```

After this the client cannot obtain new access tokens. The access token that is already in circulation remains valid until its `exp`, at most fifteen minutes. This is the cost of the design, and for many applications it is acceptable.

When it is not, for example after a password change or when an account is blocked, an access token has to be invalidated immediately. A cheap way to do it is a version number. The user record gets an integer column `token_version`, the version is placed in the token as a claim, and the check compares the two:

```ruby
# in Auth::Tokens.issue
Jwt.encode({ sub: user.id.to_s, ver: user.token_version }, secret: secret, expires_in: ACCESS_TTL)

# in Auth::Tokens.user_from
user = User.find_by(id: payload['sub'])
user if user && user.token_version == payload['ver']
```

Incrementing `token_version` makes every token issued before that moment fail the comparison. The user is loaded on each request anyway, so the check adds no query.

## Common mistakes

- **Sensitive data in the payload.** The payload is readable by anyone who has the token. It should hold identifiers and claims about the token, and no passwords, personal data or internal permissions that should not be disclosed.
- **Trusting the header.** The algorithm is fixed on the server. Code that reads `alg` from the token and picks the verification method accordingly lets the sender choose how its own token is checked.
- **Comparing signatures with `==`.** The comparison should take the same time for every input, which `secure_compare` guarantees.
- **A weak or shared secret.** A short or guessable secret can be found by brute force from a single token, after which anyone can sign tokens for any user. The secret should be random, at least 32 bytes, and different in each environment. Changing it invalidates all issued tokens.
- **A long access token lifetime.** Hours or days of validity turn a leaked token into a long-lived credential. The lifetime should match the time the application can tolerate after a theft.
- **Missing `exp`.** A token without an expiration claim never stops working. The verifier above rejects it.
- **Plain HTTP.** Tokens are bearer credentials: whoever holds one is the user. They must only travel over HTTPS.

## Summary

A JWT is a signed statement that the server issues after login and verifies on every request without a lookup, which works only if the verifier fixes the algorithm, compares signatures in constant time and checks expiration, and which stays safe in practice by pairing a short-lived access token with a stored, single-use refresh token that the server can revoke.