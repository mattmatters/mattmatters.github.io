---
layout: post
title:  "Demystifying OAuth2"
date:   2017-10-01
description: 'OAuth2'
image: '/assets/img/jekyll.png'
categories:
- Authentication
tags:
- OAuth2
twitter_text: 'OAuth2'
author: 'Matt Lewis'
---

I used to shudder everytime I had to work with some sort of OAuth2 flow. There are so many moving parts. Every company's OAuth system is slightly different, there are a bunch of different pieces of authentication that need to be included and without a decent idea of what you are doing, it takes a while to set up.

I'm going to be using [curl](https://curl.haxx.se/) to demonstrate the nitty gritty of HTTP combined with Bearer tokens. Familiarity with it is not required, but it will certainly help.

There's also a small [Node.js](https://nodejs.org/en/) app that serves as a basic client endpoint for receiving authorization codes. Don't worry about its implementation, it is not important to this article.

First let's talk about what OAuth2 is.

## Understanding the OAuth2 Protocol

The [OAuth 2.0 Authorization Framework](https://oauth.net/2/) is at its core a framework for authorizing access to protected information utilizing an external provider.  Any account set up that let's you log in with a Facebook or Google account is using some variation of the [OAuth 2 specification](https://tools.ietf.org/html/rfc6749).

This can be used to streamline account management via filling out personal information, authenticating api usage, or authenticating a user via an external provider.

_Note that using a providers OAuth service as login authentication is offloading your login security to an external provider._

#### Common OAuth2 providers
+ [Facebook](https://developers.facebook.com/docs/facebook-login/)
+ [Google](https://developers.google.com/identity/protocols/OAuth2)
+ [Slack](https://api.slack.com/docs/oauth)

They all have slight differences, but overall they boil down to the same basic steps.


## Registering Your Application
The first step to setting of this flow is to register the application with the provider.

Usually most will ask for a basic overview of the **project**, **various credentials**, and **where
the server should redirect to on authorization**. Don't skimp on _any_ of this.  It's the basis of
the entire auth flow.

In our circumstance, we are using Google's OAuth2 service.

+ Create an app in the [Google Developer's Console](https://console.developers.google.com).
+ Select read-only access to Google Calender
+ Generate your credentials
+ the origin should be http://localhost:3000 and the redirect uri should be http://localhost:3000/oauthcallback
+ Store them somewhere secure

Your credentials should look something like this.
```json
{
  "web": {
    "client_id": "Your id will be here",
    "project_id": "The name you selected will appear here",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://accounts.google.com/o/oauth2/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_secret": "The secret is here",
    "redirect_uris": ["http://localhost:3000/oauthcallback"],
    "javascript_origins": ["http://localhost:3000"]
  }
}
```

#### Let's breakdown the important parts of this
+ **client_id**: The identifier for your application, we provide this for every authentication request made.
+ **client_secret**: Basically the password, we don't surface this information on the client side, but it will be used to obtain an access token by our backend
+ **auth_uri**: We send the user to this url from our app. They will authenticate with Google.
+ **redirect_uris**: Google will redirect the client to this url with an auth code parameter in the url.
+ **token_uri**: Our app takes the auth code, along with the client id and client secret and sends them to this url.


## Authentication Request
The first part of authorizing your app is to get the auth code.  This is a code generated by the OAuth2 provider after the client has sent over its credentials and the user has approved the access scopes.

Before we set up our endpoint, let's run the auth code request manually.

**This is what the initial auth request looks like**

```sh
$ curl -L "https://accounts.google.com/o/oauth2/v2/auth?scope=https://www.googleapis.com/auth/calendar&include_granted_scopes=true&redirect_uri=http://localhost:3000/oauthcallback&client_id=your_client_id&response_type=token"
```
Assuming the same scopes and setups were selected, the only thing of the request that needs to be changed is the client id.

It's simply a **GET** request! There isn't any crazy space technology involved. This can be done in any environment or language with an http client.

To make sense of the giant HTML blob, I recommend piping the output into some sort of command line HTML viewer.  I use [html2text](http://www.mbayer.de/html2text/). It's simply the authentication conset page for the user, all of our parameters are being stored for redirect.

**An important thing to note:** _providers have records over what scopes the client apps have been set to access, if a scope or endpoint is outside the registered app it will result in an error._

#### At this point the User would approve or deny the requested scopes
Instead of actually replicating this part from a curl request, let's use an endpoint.


## Starting and serving the endpoint
If you use Docker, just pull this container and run it over port 3000.

The container takes the client_id.json as an environment variable.

```sh
$ docker pull superpolkadance/basicoauth:latest
$ docker run -p 3000:3000 --env "$(<./client_id.json)" superpolkadance/basicoauth:latest
```

The endpoint is now being served, head to [http://localhost:3000](http://localhost:3000) and click log auth code.

It should redirect to Google, on approval it redirects back to the endpoint.

_Notice how the auth code was in the url?_


**Note:** _If you are not familiar with docker, you can still pull the [project](https://github.com/mattmatters/basic-oauth) and run it in your own enviroment._

## Exchange the Auth Code for a Token
Take the auth code, along with the apps credentials and send this form to the token provider.

```sh
$ curl -X "code=authCode&client_id=your_client_id&client_secret=your_client_secret&redirect_uri=http://localhost:3000/oauthcallback" https.google.com/oauth2/v4/token
```

_Hint: this will return a json, I recomend piping it into_ ``python -m json.tool``

It's simply a **POST** request this time! Granted it's a little weird that it's a form for the request, but the response is a json.

The response should appear looking somewhat like this.
```json
{
  "access_token":"Your token",
  "expires_in":3920,
  "token_type":"Bearer"
}
```

Token type is a signifier of how to set your authorization header.

```sh
curl -H "Authorization: Bearer yourToken" https://www.google.com/
```

Tadah! Data!

You've done it.


## Access Token
This was just one method to obtaining a token. The entire process for OAuth2 boils down to getting an access token, which is simply a short lived key to access specific properties of a user.

The above method relies only works when the client is online. The auth code process can be done periodically to prevent expiration during a session.  Once the client has authorized the app, the redirect does not require user input.

However some programs require access to data when the user is not _online_. This is where a refresh token comes in

## Refresh Token
A refresh token is a token the client application can use to obtain an access token without the user having to be onlined.  Like the other authorization process, we need to specify offline access in the permissions. The user will be informed of this and the auth code will instead yield a refresh token.  This type of token can only obtained access tokens for the scopes included in the auth process.

They only expire when a user revokes the permissions.

## Maintaining State Between Redirects

HTTP requests are stateless, and by default there is no way of proving or attaching any identifier to the OAuth2 handshake. However, this is solved by the state parameter. Simply add the state set as a string to the initial authorization code request. This state parameter will be passed back on redirect. Your client application is now being given back some form of state it generating on the initial reaquest.

This also solves the problem of verifiying the requests had been initiated by the proper client application. For example a security hash or id could be passed as a state and stored locally. On redirect it can be looked up to verify it originated from the target application.

# In Conclusion
This process is mostly just a walk through the authentication handshakes in HTTP requests.


### _Further Reading_

+ [HTTP Auth Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization)
+ [My blog](http://mattmatters.io) _thanks in advance_
+ [Example OAuth2 Endpoint Source Code](https://github.com/mattmatters/basic-oauth)