---
title: Setup for Node Servers
order: 1
---

If you are using servers other than Node - please choose the appropriate guide on the left.

**Supported Node servers:** [Apollo Server](https://github.com/apollographql/apollo-server) (Express, Hapi, Koa, Restify, and Lambda); [Express-GraphQL](https://github.com/graphql/express-graphql)

To get started with Engine, you will need to:

1. Instrument your server with the Apollo Tracing NPM package.
2. Configure the Engine proxy. You have two options here:
  - Use the Engine proxy sidecar NPM package **or**
  - Deploy the Engine proxy in a standalone docker container.
3. Send requests to your service – you're all set up!

<h2 id="node-with-apollo-tracing" title="1. Enable Apollo Tracing">1. Enable Apollo Tracing</h2>

You will need to instrument your Node server with a tracing package that follows the [Apollo Tracing](https://github.com/apollographql/apollo-tracing) format. Engine relies on receiving data in this format to create its performance telemetry reports.

If you are using Apollo Server, the only code change required is to add  `tracing: true` to the options passed to the Apollo Server middleware function for your framework of choice. For example, for Express:

```javascript
app.use('/graphql', bodyParser.json(), graphqlExpress({
  schema,
  context: {},
  tracing: true,
}));
```

If you are using Express-GraphQL, follow the instructions [on GitHub](https://github.com/apollographql/apollo-tracing-js#express-graphql).

<h3 id="enabling-compression" title="Enabling compression">Enabling compression (optional)</h3>

Once instrumented, the tracing package will increase the size of GraphQL requests traveling between your GraphQL and the Engine proxy, because the requests will be augmented with additional tracing data.

Because of this, we recommend that you enable gzip compression in your GraphQL server – the added volume from the tracing format compresses well.

#### Express

Install the compression middleware (https://github.com/expressjs/compression) package into your app with `npm install compression --save` and add it to your server's middleware stack as follows:

```javascript
var compression = require('compression')
app.use(compression());
```

#### Koa

Install the compress middleware (https://github.com/koajs/compress) package into your app with `npm install koa-compress --save` and add it to your server's middleware stack as follows:

```javascript
var compress = require('koa-compress')
app.use(compress())
```

#### Hapi

Hapi comes with support for compression enabled by default, unless it has been configured with `compression: false`.

<h2 id="configure-proxy" title="2. Configure the proxy">2. Configure the proxy</h2>

There are two options for configuring and deploying the Engine proxy with Node servers. You can either install Engine's JavaScript sidecar package from NPM or run a standalone docker container.

<h3 id="sidecar-package" title="Sidecar package">Option 1: Sidecar package</h3>

This option involves adding an NPM package to your server that will run an Engine proxy in the same container as your server. We provide an NPM package that includes a pre-built copy of the Engine proxy. It spawns an Engine process side-by-side with your GraphQL server process and incoming GraphQL operations are routed through the Engine proxy and then sent to your server.

**Step 1. Install the NPM package.**

```bash
npm install --save apollo-engine
```

**Step 2: Create a new Engine instance**

Import the `Engine` constructor from the `apollo-engine` NPM package.

```javascript
import { Engine } from 'apollo-engine';
```

When you instantiate Engine, you have two options for referencing configuration fields: A JavaScript object with options, or a path to a JSON file with those options.

```javascript
// Option 1: JSON Object
const engine = new Engine({ engineConfig: { apiKey: <ENGINE_API_KEY> } });

// Option 2: Config.json
const engine = new Engine({ engineConfig: 'path/to/config.json' });
```

You can get your `ENGINE_API_KEY` by creating a service in the [Engine UI](http://engine.apollographql.com/). You will need to log in and click "Add Service" to recieve your API key.

**Step 3: Set optional configuration**

```javascript
const engine = new Engine({
  engineConfig: {
    apiKey: engineApiKey,
    logcfg: {
      level: 'DEBUG'   // Engine Proxy logging level. DEBUG, INFO, WARN or ERROR
    }
  },

  // GraphQL port
  graphqlPort: process.env.PORT || 8003,

  // GraphQL endpoint suffix - '/graphql' by default
  endpoint: '/graphql',

  // Debug configuration that logs traffic between Proxy and GraphQL server
  dumpTraffic: true
});
```

**Step 4: Start Engine**

Add this line to your server code to start the Engine proxy, preferably not far from where you instantiated `engine`:
```javascript
engine.start();
```
It does not matter when you call `engine.start()` in your server file, but the earlier Engine is started the better. Your server will start normally and handle requests without the Engine proxy until it has fully started and is ready.

**Step 5: Invoke your Node.js middleware function**

Add the Engine middleware to your app's middleware so that your app can route requests through the Engine proxy. Since the Engine middleware is what sends requests to the Engine proxy, it's important that this is your app's _first middleware_.

The `apollo-engine` package supports the following middlewares:
1. `expressMiddleware` – use for Express servers.
2. `connectMiddleware` – use for Connect or Restify servers.
3. `instrumentHapiServer` – use for Hapi servers.
4. `koaMiddleware` – use for Koa servers.

In an Express server, adding the Engine middleware would look like this:
```javascript
app.use(engine.expressMiddleware());

// ...
// other middleware / handlers
// ...
```

<h3 id="standalone-docker-container" title="Docker container">Option 2: Standalone Docker container</h3>

This option involves running a standalone docker container that contains the Engine proxy process and is hosted and managed separately from your Node server.

**Step 1. Create the proxy's config.json**

The proxy uses a JSON object to get configuration information. If the configuration is passed the path to your file, that file will be watched for changes. Changes will cause the proxy to adopt the new configuration without downtime. The file should look like this:

```
{
  "apiKey": "<ENGINE_API_KEY>",
  "logcfg": {
    "level": "INFO"
  },
  "origins": [
    {
      "url": "http://localhost:3000/graphql"
    }
  ],
  "frontends": [
    {
      "host": "127.0.0.1",
      "port": 3001,
      "endpoint": "/graphql"
    }
  ]
}
```

You can get your `ENGINE_API_KEY` by creating a service on http://engine.apollographql.com/. You will need to log in and click "Add Service" to recieve your API key.

Here are the configuration options available:

1. `apiKey`: The API key for the Engine service you want to report data to.
2. `logcfg.level` : Logging level for the proxy. Supported values are `DEBUG`, `INFO`, `WARN`, `ERROR`.
3. `origin.url` : The URL for your GraphQL server.
4. `frontend.host` : The hostname the proxy should be available on.
5. `frontend.port` : The port the proxy should bind to.
6. `frontend.endpoint` : The path for the proxy's GraphQL server . This is usually `/graphql`.

**Step 2. Run the proxy container**

The Engine proxy is a Docker image that you will deploy and manage separate from your server.

If you have a working [Docker installation](https://docs.docker.com/engine/installation/), type the following lines in your shell (variables replaced with the correct values for your environment) to run the Engine proxy:
```
engine_config_path=/path/to/engine.json
proxy_frontend_port=3001
docker run --env "ENGINE_CONFIG=$(cat "${engine_config_path}")" \
  -p "${proxy_frontend_port}:${proxy_frontend_port}" \
  gcr.io/mdg-public/engine:2017.10-362-g6ef7c72c
```

It does not matter where you choose to deploy and manage your Engine proxy. We run our own on Amazon's [EC2 Container Service](https://aws.amazon.com/ecs/).

We recognize that almost every team using Engine has a slightly different deployment environment, and encourage you to [contact us](mailto: support@apollodata.com) with feedback or for help if you encounter problems running the Engine proxy.

<h2 id="view-metrics-in-engine" title="3. View metrics">3. View metrics in Engine</h2>

Once your server is set up, navigate your new Engine service in the [Engine UI](https://engine.apollographql.com). Start sending requests to your Node server to start seeing information appear!
