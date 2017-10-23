---
title: Upgrade from Optics Agent
order: 11
---

We introduced the Engine proxy architecture to enable support for more GraphQL server languages and to unlock features like Error reporting.

If you're already using Optics, there's an easy upgrade from your current Optics integration to Engine - it's just a few lines of code in your `server.js` and an NPM package upgrade! If you are interested in using other languages, please see our other documentation pages.

This guide assumes you are starting with a Node.js app that has Optics already instrumented.

## Remove Optics agent integration

You can remove the code added to your Node `server.js` file to instrument Optics.

You no longer need to wrap your GraphQL schema in the Optics Agent instrumentation, so remove this line and remove your import of the `OpticsAgent`.

```javascript
// Remove your import of the OpticsAgent
// import OpticsAgent from 'optics-agent';

// Remove OPTICS_API_KEY reference

graphqlExpress((req) => {
  // Change this line back to this:
  // schema: OpticsAgent.instrumentSchema(schema)
  schema: schema

  // other options...
})
```

Remove the Optics Agent HTTP middleware:

```javascript
// Remove this line:
// expressServer.use(OpticsAgent.middleware());
```

Remove the `opticsContext` field from your GraphQL context.

```javascript
graphqlExpress((req) => {
  context: {
    // Remove the opticsContext field from your GraphQL context:
    // opticsContext: OpticsAgent.context(req),

    // other context fields...
  }
})
```

Remove the Optics Agent NPM package:

```javascript
npm remove optics-agent
```

## Install and configure Engine

We recommend that you use Apollo Server. It's a much simpler integration.

### Get an API key from Engine

Log in to [engine.apollographql.com](https://engine.apollographql.com) and click "Add Service" in the upper right-hand corner.

Name your endpoint and save your **API key**.

### Enable Apollo Tracing

**If using Apollo Server**

The only code change required is to add `tracing: true` to the options passed to the Apollo Server middleware function for your framework of choice. For example, for Express:

```javascript
app.use('/graphql', bodyParser.json(), graphqlExpress({
  schema,
  context: {},
  // Enable tracing:
  tracing: true,
}));
```

**If using Express-GraphQL**

Using Apollo Tracing with express-graphql requires more manual configuration. See [the README of `apollo-tracing-js`](https://github.com/apollographql/apollo-tracing-js#express-graphql) for details.

### Add Engine middleware code to your server

Import the Engine constructor from the apollo-engine NPM package.

```javascript
import { Engine } from 'apollo-engine';
```

Create a new Engine instance. Set the engine configuration through a JSON object.

```javascript
const engine = new Engine({ engineConfig: { apiKey: '<ENGINE_API_KEY>' } });

// Optionally, you can set these other configuration variables:
const engine = new Engine({
  engineConfig: {
    apiKey: '<ENGINE_API_KEY>',
    logcfg: {
      level: 'DEBUG'   // Engine Proxy logging level. DEBUG, INFO, WARN or ERROR
    }
  },
  graphqlPort: process.env.PORT || 8003,  // GraphQL port
  endpoint: '/graphql',                   // GraphQL endpoint suffix - '/graphql' by default
  dumpTraffic: true                       // Debug configuration that logs traffic between Proxy and GraphQL server
});
```

Add this line to your server code to start the Engine proxy, preferably not far from where you instantiated engine:

```javascript
engine.start();
```

Add the Engine middleware to your app's middleware so that your app can route requests through the Engine proxy. Since the Engine middleware is what sends requests to the Engine proxy, it's important that this is your app's _first middleware_.

The `apollo-engine` package supports the following middlewares:
1. `expressMiddleware` – use for Express servers.
2. `connectMiddleware` – use for Restify servers.
3. `instrumentHapiServer` – use for Hapi servers.
4. `koaMiddleware` – use for Koa servers.

In an Express server, adding the Engine middleware would look like this:

```javascript
app.use(engine.expressMiddleware());
// ...
// other middleware / handlers
// ...
```

## [Optional] Enabling Compression

Once instrumented, the tracing package will increase the size of GraphQL requests traveling between your GraphQL and the Engine proxy, because the requests will be augmented with additional tracing data.
Because of this, we recommend that you enable gzip compression in your GraphQL server – the added volume from the tracing format compresses well.

See [Node compression](/setup-node.html#\31 -Instrument-Node-Agent-with-Apollo-Tracing) for instructions.

## Run in localhost to test

1. Start the server in your localhost development environment.
2. Run your GraphQL queries and check that they provide the results you expect.
3. Check that they show up in your service report in the [Engine UI](http://engine.apollographql.com/).

## Deploy to staging and production

Deploy the Node server to staging and then production!
