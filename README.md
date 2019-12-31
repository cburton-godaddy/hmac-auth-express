# HMAC Auth Express

[![Build Status](https://travis-ci.org/connorjburton/hmac-auth-express.svg?branch=master)](https://travis-ci.org/connorjburton/hmac-auth-express)

## Description

This package provides [Express](https://expressjs.com/) middleware for HMAC authentication. Zero dependencies, timing safe, 100% code coverage, support for all node hash algorithms, replay attacks prevention.

## Installation

**yarn**

`yarn add hmac-auth-express`

**npm**

`npm install hmac-auth-express`

## Usage

#### Basic middleware registration

```javascript
const hmac = require('hmac-auth-express');

app.use('/api', hmac('secret'));
```

#### Advanced middleware registration

```javascript
const hmac = require('hmac-auth-express');

app.use('/api', hmac('secret', {
  algorithm: 'sha512',
  identifier: 'APP',
  header: 'authorization',
  maxInterval: 600
});
```

#### Function parameters

The function will throw `TypeError`'s if you provide incorrect parameters.

| Parameter  | Accepted Type  | Default  | Description  |
|---|---|---|---|
| `secret`  | *string*  | `undefined`  | Your hash secret  |
| `options.algorithm`  | *string*  | `sha256`  | Your hashing algorithim  |
| `options.identifier`  | *string*  | `HMAC`  | The start of your `options.header` should start with this  |
| `options.header`  | *string*  | `authentication`  | The header the HMAC is located, should always be lowercase (express lowercases headers)  |
| `options.maxInterval`  | *integer*  | `60 * 5`  | The amount of time you would like a request to be valid for, in seconds. See [time based protection against replay attacks](#replay-attacks) for more information  |

#### Error Handling

The middleware will pass an error to [express' error handler](http://expressjs.com/en/guide/error-handling.html#writing-error-handlers). You can use the provided `HMACAuthError`, or alternatively check the error by its code `ERR_HMAC_AUTH_INVALID`.

Example:

```javascript
const { HMACAuthError } = require('hmac-auth-express/src/errors');

// express' error handler
app.use((error, req, res, next) => {
  // check by error instance
  if (error instanceof HMACAuthError) {
    res.status(401).json({
      error: 'Invalid request',
      info: error.message
    })
  }

  // alternative: check by error code
  if (error.code === 'ERR_HMAC_AUTH_INVALID') {
    res.status(401).json({
      error: 'Invalid request',
      info: error.message
    })
  }

  else {
    // ... handle other errors
  }
})
```

## Structuring your HMAC header

Now you have configured your HMAC middleware, you need to structure your HMAC in the way the middleware expects.

#### What your HMAC should look like

This example uses the default `options.header` and `options.identifier`. These will be different if you override said defaults

`Authentication: HMAC 1573504737300:76251c6323fbf6355f23816a4c2e12edfd10672517104763ab1b10f078277f86`

#### Constructing the HMAC

`Authentication:` This is the header you send in the request that contains the HMAC. This is what the middleware will look for.

`HMAC` This is the identifier the middleware will look for, this is fine to be left as the default

`1573504737300` This is the UNIX timestamp of when the request was sent

`76251c6323fbf6355f23816a4c2e12edfd10672517104763ab1b10f078277f86` This is the HMAC digest, see [generating your HMAC digest](#generating-your-hmac-digest)

## Generating your HMAC digest

The HMAC signature is 4 parts (1 part optional) joined **without** a seperator. **UNIX TIMESTAMP**, **VERB**, **ROUTE** and optionally **MD5 JSON STRINGIFIED CONTENT HASH**

Below is an example request and how we would build that request's HMAC

```
POST http://www.domain.com/api/order HTTP/1.0
Authentication: HMAC 1573504737300:76251c6323fbf6355f23816a4c2e12edfd10672517104763ab1b10f078277f86
Content-Type: application/json
Date: Wed, 13 Nov 2019 22:06:01 GMT

{
  "foo": "bar"
}
```

```javascript
const crypto = require('crypto');

const hmac = crypto.createHmac('sha256', 'secret');
const time = Date.now().toString();

hmac.update(time);
hmac.update('POST');
hmac.update('/api/order');

const contentHash = crypto.createHash('md5');
contentHash.update(JSON.stringify({"foo": "bar"}));

hmac.update(contentHash.digest('hex'));

console.log(`HMAC ${time}:${hmac.digest('hex')}`);
```

## Replay attacks

The parameter `options.maxInterval` is the amount of time in seconds that a request is valid. We compare the unix timestamp sent in the HMAC header to the current time on the server. If the time difference is greater than `options.maxInterval` we reject the request.

The unix timestamp sent in the header is also included in the HMAC digest, this is to prevent someone replicating a request and just changing the unix timestamp to be in a valid range of `options.maxInterval`

## Credits

Reference article https://www.wolfe.id.au/2012/10/20/what-is-hmac-authentication-and-why-is-it-useful/
