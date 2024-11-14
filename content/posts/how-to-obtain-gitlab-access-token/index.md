+++
date = '2024-11-01T00:00:00+02:00'
title = 'How to obtain Gitlab access token'
author = 'Genry Wood'
tags = ['gitlab', 'oauth', 'ruby']
+++

![Post Logo](./header.png)

Recently, i needed to access the list of repositories for specific users on GitLab. One crucial step in obtaining this access is acquiring an access token. In this article, i will explain how you can obtain a user's **`access token`**.

I would divide the whole process into three steps:

1. Creating an Application on GitLab
2. Receiving the Code
3. Obtaining an Access Token

## Creating an Application on GitLab

The process of creating an application on GitLab has the following steps:

1. On the left sidebar, select your avatar.
2. Select **Edit profile**.
3. Enter the name of your application and provide a **redirect URI**. This is the URL where the user will be redirected after successful authentication.
4. Specify the required **scopes**. In my case, the `api` scope was sufficient.
5. Disable the **Confidential** option. Since the authentication will occur via a web browser, ensure that the "`Confidential`" option is disabled.

These steps will be sufficient to create a GitLab application.

## Receiving the Code

To continue with the process, you need to obtain an `code`. For that, you need to send a GET request to `https://gitlab.com/oauth/authorize` with the following parameters:

1. **client_id** - The client ID you received when registering the application on GitLab.
2. **redirect_uri** – The redirect URI you specified during application registration.
3. **response_type** – Set this to `code`.
4. **scope** – The scope you specified during the application registration (e.g., `api`).

Once the user is authenticated, he will be redirected to the URI you previously specified. The redirect URL will contain the authorization `code` that we need.

## Obtaining an Access Token

Using the code obtained in the previous step, make a POST request to `https://gitlab.com/oauth/token`. In the body parameters, also include:

1. **client_id**
2. **grant_type** - Set this to `authorization_code`.
3. **redirect_uri**

Request example:
```ruby
require 'httparty'

client_id = 'YOUR_CLIENT_ID'
redirect_uri = 'YOUR_REDIRECT_URI'
code = 'AUTHORIZATION_CODE_RECEIVED'

response = HTTParty.post("https://gitlab.com/oauth/token", body: {
  client_id: client_id,
  code: code,
  grant_type: 'authorization_code',
  redirect_uri: redirect_uri
})
```
<br>
If the request is successful, you will receive a response containing the access token we've been working towards.

```json
{
 "access_token": "pofo3foqwd446309bd9362820ba8aed28aa506c71eedbe1c5c4f9dd350e54",
 "token_type": "bearer",
 "expires_in": 7200,
 "refresh_token": "8257e65c97202ed1726cf9571600918f3bffb2544b26e00a61df9897668c33a1",
 "created_at": 1731616569
}
```

> **Note:** Access token lives only for 2 hours!

