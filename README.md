hapi-raven
==========

[![Build Status](https://travis-ci.org/pagerinc/hapi-raven.svg)](https://travis-ci.org/pagerinc/hapi-raven) [![Code Climate](https://codeclimate.com/repos/55a3d323695680658f00ad6d/badges/4b4d0c077d87c3af2491/gpa.svg)](https://codeclimate.com/repos/55a3d323695680658f00ad6d/feed)

A Hapi plugin for sending exceptions to Sentry through Raven.

This is a fork of @bendrucker's `hapi-raven` package tweaked to better adjust to Pager's requirements.

## Setup

Options:

* **`dsn`**: Your Sentry DSN (required)
* **`client`**: An options object that will be passed directly to the client as its second argument (optional)
* **`patchGlobal`**: Boolean value that will trigger raven's `patchGlobal` functionality (optional)

Note that DSN configuration using `process.env` is not supported. If you wish to replicate the [default environment variable behavior](https://github.com/getsentry/raven-node/blob/master/lib/client.js#L21), you'll need to supply the value directly:

```js
server.register({
  register: require('hapi-raven'),
  options: {
    dsn: process.env.SENTRY_DSN,
    patchGlobal: true,
    captureUnhandledRejections: true
  }
});
```

## Usage

Once you register the plugin on a server, logging will happen automatically.

The plugin listens for [`'request-error'` events](http://hapijs.com/api#server-events) which are emitted any time `reply` is called with an error where `err.isBoom === false`. Note that the `'request-error'` event is emitted for all thrown exceptions and passed errors that are not Boom errors. Transforming an error at an extension point (e.g. `'onPostHandler'` or `'onPreResponse'`) into a Boom error will not prevent the event from being emitted on response.

--------------

#### Boom Non-500 Errors are Not Logged

```js
server.route({
  method: 'GET',
  path: '/',
  handler: function (request, reply) {
    reply(Hapi.error.notFound());
  }
});

server.inject('/', function (response) {
  // nothing was logged
});
```

#### 500 Errors are Logged

```js
server.route({
  method: 'GET',
  path: '/throw',
  handler: function (request, reply) {
    throw new Error();
  }
});

server.inject('/throw', function (response) {
  // thrown error is logged to Sentry
});
```

```js
server.route({
  method: 'GET',
  path: '/reply',
  handler: function (request, reply) {
    reply(new Error());
  }
});

server.inject('/throw', function (response) {
  // passed error is logged to Sentry
});
```

-------------------------

For convenience, hapi-raven [exposes](http://hapijs.com/api#pluginexposekey-value) the `node-raven` client on your server as `server.plugins.raven.client`. If you want to capture errors other than those raised by `'request-error'`, you can use the client directly inside an [`'onPreResponse'`](http://hapijs.com/api#error-transformation) extension point.

### Example: Capture all 404 errors
```js
server.ext('onPreResponse', function (request, reply) {
  if (request.isBoom && request.response.statusCode === 404) {
    server.plugins.raven.client.captureError(request.response);
  }
});
```
