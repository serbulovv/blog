+++
date = '2026-10-28T12:44:00+02:00'
title = 'Uploading files to S3 with presigned URLs in a Rails API'
author = 'Genry Wood'
tags = ['aws', 's3', 'rails', 'api', 'uploads', 'presigned-url']
+++

![Post Logo](./header.png)

An API endpoint that accepts a file in the request body has to receive every byte of it. A Rails worker stays busy for the whole upload, a slow mobile connection keeps it busy longer, and the file then travels a second time from the server to S3. A presigned URL removes the middle step. The API signs a request that is valid for a short time, the client sends the file straight to S3, and Rails only handles two small JSON requests: one to get the URL and one to say that the upload is done.

## The flow

1. The client asks the API for an upload URL and states the content type of the file.
2. The API checks the request, generates an object key and returns a presigned URL together with that key.
3. The client sends the file to the URL with `PUT`. S3 stores it.
4. The client sends the key to the API, which checks that the object exists and attaches it to a record.

S3 never calls the API in this flow, and the API never sees the file. Whatever the API needs to know about the object it asks S3 afterwards. 

## Setup

All the AWS classes in this article come from the `aws-sdk-s3` gem:

```ruby
gem 'aws-sdk-s3'
```

`Aws::S3::Presigner` creates presigned URLs for `GET`, `PUT` and other operations, `Aws::S3::Bucket` creates presigned `POST` forms, and `Aws::S3::Client` sends ordinary requests to S3, which is used later to check an uploaded object. When the presigner is created without arguments, it builds its own `Aws::S3::Client`. The client takes credentials and the region from the standard AWS configuration: the environment variables `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_REGION`, a shared config file, or an IAM role of the server. The name of the bucket is read from the `S3_BUCKET` environment variable in the examples below.

Signing is a local computation with the credentials. Generating a URL does not send a request to S3, so it is fast and does not depend on the network.

## Generating the URL

The URL is created with `Aws::S3::Presigner`. It accepts the name of an S3 operation and its parameters, and `expires_in` sets the lifetime in seconds. The default is 15 minutes and the maximum is one week, so a value of a few minutes is a deliberate choice.

The signing logic lives in a service object, so that the controller only passes parameters in and renders the result:

```ruby
# frozen_string_literal: true

# app/services/uploads/presign.rb
module Uploads
  class Presign
    ALLOWED_TYPES = %w[image/jpeg image/png image/webp].freeze
    URL_TTL = 300

    Result = Struct.new(:url, :key, :headers, keyword_init: true)

    class UnsupportedContentType < StandardError; end

    def initialize(user:, content_type:)
      @user = user
      @content_type = content_type
    end

    def call
      raise UnsupportedContentType unless ALLOWED_TYPES.include?(@content_type)

      key = "uploads/#{@user.id}/#{SecureRandom.uuid}"

      Result.new(url: presigned_url(key), key: key, headers: { 'Content-Type' => @content_type })
    end

    private

    def presigned_url(key)
      Aws::S3::Presigner.new.presigned_url(
        :put_object,
        bucket: ENV.fetch('S3_BUCKET'),
        key: key,
        content_type: @content_type,
        expires_in: URL_TTL
      )
    end
  end
end
```

```ruby
# frozen_string_literal: true

# app/controllers/uploads_controller.rb
class UploadsController < ApiController
  before_action :authenticate!

  rescue_from Uploads::Presign::UnsupportedContentType, with: :unsupported_content_type

  def create
    upload = Uploads::Presign.new(user: current_user, content_type: params.require(:content_type)).call

    render json: upload.to_h, status: :created
  end

  private

  def unsupported_content_type
    head :unprocessable_entity
  end
end
```

Three decisions in the service matter for security.

- The key is generated on the server. A client that chooses its own key can overwrite someone else's object, because S3 replaces an existing object with the same key. The user id in the prefix also gives a way to check ownership later.
- The content type is part of the signature. The upload must be sent with exactly this `Content-Type`, otherwise S3 rejects it. The list of allowed types is checked before signing.
- The URL is limited by the permissions of whoever created it. The credentials of the application should allow `s3:PutObject` only for the `uploads/` prefix of this bucket, so that even a leaked URL cannot reach anything else.

## Uploading from the client

The client sends the file as the body of a single `PUT` request with the header from the response. Nothing else is required, and S3 returns an empty response on success:

```bash
curl -X PUT -H "Content-Type: image/png" -T avatar.png "<url from the response>"
```

In a browser the same request is a `fetch` call:

```javascript
await fetch(url, { method: 'PUT', headers: { 'Content-Type': contentType }, body: file });
```

A browser sends this request to another origin, so the bucket needs a CORS rule that allows the `PUT` method and the `Content-Type` header from the origin of the application. Without it the upload works from `curl` and fails in the browser.

When S3 answers `SignatureDoesNotMatch`, the usual causes are the same four: the `Content-Type` of the upload differs from the one that was signed, the URL has expired, the clock of the signing server is out of sync, or the request goes to a different region than the bucket. The URL also has to be used exactly as it was generated, without modifying the query string.

## Confirming the upload

A successful `PUT` proves nothing to the API by itself, since the client could also send a key without uploading anything. When the client reports the key, the API verifies it before use:

```ruby
# frozen_string_literal: true

# app/services/uploads/verify.rb
module Uploads
  class Verify
    MAX_SIZE = 10 * 1024 * 1024

    def initialize(user:, key:)
      @user = user
      @key = key
    end

    def call
      return false unless @key.start_with?("uploads/#{@user.id}/")

      object = Aws::S3::Client.new.head_object(bucket: ENV.fetch('S3_BUCKET'), key: @key)
      object.content_length <= MAX_SIZE
    rescue Aws::S3::Errors::NotFound
      false
    end
  end
end
```

The prefix check ties the key to the user who requested it. `head_object` returns the metadata of the object without downloading it, and a missing object raises `NotFound`. Only after this check does the key get stored on a record.

## Limiting the file size

The check above has a weakness. A signed `PUT` URL cannot restrict the size of the file, so an oversized object is already in the bucket when the API finds out. If this is a concern, the alternative is a presigned `POST`. Its policy can state a range for the size and a prefix for the content type, and S3 rejects anything outside them during the upload:

It fits into the same structure as a second service, and the controller calls it in the same way:

```ruby
# frozen_string_literal: true

# app/services/uploads/presign_post.rb
module Uploads
  class PresignPost
    MAX_SIZE = 10 * 1024 * 1024
    URL_TTL = 5.minutes

    Result = Struct.new(:url, :fields, :key, keyword_init: true)

    def initialize(user:)
      @user = user
    end

    def call
      key = "uploads/#{@user.id}/#{SecureRandom.uuid}"

      post = Aws::S3::Bucket.new(ENV.fetch('S3_BUCKET')).presigned_post(
        key: key,
        content_length_range: 1..MAX_SIZE,
        content_type_starts_with: 'image/',
        signature_expiration: URL_TTL.from_now
      )

      Result.new(url: post.url, fields: post.fields, key: key)
    end
  end
end
```

The client builds a multipart form from the returned fields, appends the file as the last field under the name `file`, and sends it to the URL with `POST`:

```javascript
const form = new FormData();
Object.entries(fields).forEach(([name, value]) => form.append(name, value));
form.append('file', file);

await fetch(url, { method: 'POST', body: form });
```

The extra step on the client is small, and `POST` is the variant to prefer whenever the size of uploads has to be enforced and not merely checked.

## Summary

A presigned URL lets the client send a file directly to S3 while the API only issues a short-lived, narrowly signed permission and later verifies the result, which is why the key must be generated by the server, the content type signed, the credentials limited to one prefix, and a presigned `POST` used when the file size has to be enforced.