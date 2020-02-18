<h1 align="center">
  🚀 Graceful Server 🐢
</h1>

<p align="center">
  <a href="#">
    <img src="https://img.shields.io/github/package-json/v/gquittet/graceful-server?style=flat" alt="GitHub package.json version">
  </a>
  <a href="https://standardjs.com" target="_blank" rel="noopener">
    <img src="https://img.shields.io/badge/code_style-standard-brightgreen.svg?style=flat" alt="JavaScript Style Guide">
  </a>
  <a href="https://www.npmjs.com/package/@gquittet/graceful-server" target="_blank" rel="noopener">
    <img src="https://img.shields.io/npm/types/@gquittet/graceful-server" alt="npm type definitions">
  </a>
  <a href="https://www.npmjs.com/package/@gquittet/graceful-server" target="_blank" rel="noopener">
    <img src="https://img.shields.io/npm/l/@gquittet/graceful-server" alt="License">
  </a>
  <a href="https://www.npmjs.com/package/@gquittet/graceful-server" target="_blank" rel="noopener">
    <img src="https://img.shields.io/npm/dw/@gquittet/graceful-server" alt="npmjs download">
  </a>
  <a href="https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=JN3XLTQCX3NR8&source=url" target="_blank" rel="noopener">
    <img src="https://img.shields.io/badge/Donate-PayPal-green.svg" alt="Donate with PayPal">
  </a>
</p>

<p align="center">
  Tiny (~5k), dependency-free Node.JS library to make your API more graceful
</p>

---

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
  - [NPM](#npm)
  - [Yarn](#yarn)
- [Example](#example)
  - [ExpressJS](#expressjs)
  - [HTTP Server](#http-server)
- [Options](#options)
- [Integration with Docker](#integration-with-docker)
  - [HEALTH CHECK in Dockerfile](#health-check-in-dockerfile)
  - [Content of *healthcheck.js*](#content-of-healthcheckjs)
  - [Example of Dockerfile](#example-of-dockerfile)
- [Integration with Kubernetes](#integration-with-kubernetes)
- [Thanks](#thanks)
- [Donate](#donate)

## Features

✔ It's listening system events to gracefully close your API on interruption.

✔ It facilitates the disconnect of data sources on shutdown.

✔ It facilitates the use of liveness and readiness.

✔ It manages the connections of your API.

## Requirements

✔ NodeJS >= 8

## Installation

### NPM

```
npm install --save @gquittet/graceful-server
```

### Yarn

```
yarn add @gquittet/graceful-server
```

## Example

### ExpressJS

```javascript
const express = require('express')
const http = require('http')
const GracefulServer = require('@gquittet/graceful-server')
const { connectToDb, closeDbConnection } = require('./db')
const app = express()

app.get('/test', (_, res) => {
  return res.send({ uptime: process.uptime() | 0 })
})

const server = http.createServer(app)
const gracefulServer = GracefulServer(server, { closePromises: [closeDbConnection] })

server.listen(8080, async () => {
  await connectToDb()
  gracefulServer.setReady()
})
```

### HTTP Server

```javascript
import http from 'http'
import url from 'url'
import GracefulServer from '@gquittet/graceful-server'
import { connectToDb, closeDbConnection } from './db'

const server = http.createServer((req, res) => {
  if (req.url === '/test' && req.method === 'GET') {
    res.statusCode = 200
    res.setHeader('Content-Type', 'application/json')
    return res.end(JSON.stringify({ uptime: process.uptime() | 0 }))
  }
  res.statusCode = 404
  return res.end()
})

const gracefulServer = GracefulServer(server, { closePromises: [closeDbConnection] })

gracefulServer.on(GracefulServer.READY, () => {
  console.log('Server is ready')
})

gracefulServer.on(GracefulServer.SHUTTING_DOWN, () => {
  console.log('Server is shutting down')
})

gracefulServer.on(GracefulServer.SHUTDOWN, error => {
  console.log('Server is down because of', error.message)
})

server.listen(8080, async () => {
  await connectToDb()
  gracefulServer.setReady()
})
```

## Options

| Name              |            Type            | Default |                                                      Description |
| ----------------- | :------------------------: | :-----: | ---------------------------------------------------------------: |
| closePromises     | (() => Promise<unknown>)[] |   []    |                    The functions to run when the API is stopping |
| timeout           |           number           |  1000   | The time in milliseconds to wait before shutting down the server |
| livenessEndpoint  |           string           |  /live  |                                            The liveness endpoint |
| readinessEndpoint |           string           | /ready  |                                           The readiness endpoint |

## Integration with Docker

### HEALTH CHECK in Dockerfile

```Dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s CMD ["node healthcheck.js"]
```

### Content of *healthcheck.js*

```javascript
const http = require('http')

const options = {
  timeout: 2000,
  host: 'localhost',
  port: 8080,
  path: '/live'
}

const request = http.request(options, res => {
  console.info('STATUS:', res.statusCode)
  process.exitCode = res.statusCode === 200 ? 0 : 1
  process.exit()
})

request.on('error', err => {
  console.error('ERROR', err)
  process.exit(1)
})

request.end()
```


### Example of Dockerfile

```Dockerfile
FROM node:12-slim as base
ENV NODE_ENV=production
ENV TINI_VERSION=v0.18.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
RUN chmod +x /tini && \
  mkdir -p /node_app/app && \
  chown -R node:node /node_app
WORKDIR /node_app
USER node
COPY --chown=node:node package.json package-lock*.json ./
RUN npm ci && \
  npm cache clean --force
WORKDIR /node_app/app

FROM base as source
COPY --chown=node:node . .

FROM source as dev
ENV NODE_ENV=development
ENV PATH=/node_app/node_modules/.bin:$PATH
RUN npm install --only=development --prefix /node_app
CMD ["nodemon", "--inspect=0.0.0.0:9229"]

FROM source as test
ENV NODE_ENV=development
ENV PATH=/node_app/node_modules/.bin:$PATH
COPY --from=dev /node_app/node_modules /node_app/node_modules
RUN npm run lint
ENV NODE_ENV=test
RUN npm test
CMD ["npm", "test"]

FROM test as audit
RUN npm audit --audit-level critical
USER root
ADD https://get.aquasec.com/microscanner /
RUN chmod +x /microscanner && \
    /microscanner your_token --continue-on-failure

FROM source as buildProd
ENV PATH=/node_app/node_modules/.bin:$PATH
COPY --from=dev /node_app/node_modules /node_app/node_modules
RUN npm run build

FROM source as prod
COPY --from=buildProd --chown=node:node /node_app/app/build ./build
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s CMD ["node healthcheck.js"]
ENTRYPOINT ["/tini", "--"]
CMD ["node", "./build/src/main.js"]
```

## Integration with Kubernetes

```yml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  failureThreshold: 1
  initialDelaySeconds: 5
  periodSeconds: 5
  successThreshold: 1
  timeoutSeconds: 5
livenessProbe:
  httpGet:
    path: /live
    port: 8080
  failureThreshold: 3
  initialDelaySeconds: 10
  # Allow sufficient amount of time (90 seconds = periodSeconds * failureThreshold)
  # for the registered shutdown handlers to run to completion.
  periodSeconds: 30
  successThreshold: 1
  # Setting a very low timeout value (e.g. 1 second) can cause false-positive
  # checks and service interruption.
  timeoutSeconds: 5

# As per Kubernetes documentation (https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#when-should-you-use-a-startup-probe),
# startup probe should point to the same endpoint as the liveness probe.
#
# Startup probe is only needed when container is taking longer to start than
# `initialDelaySeconds + failureThreshold × periodSeconds` of the liveness probe.
startupProbe:
  httpGet:
    path: /live
    port: 9000
  failureThreshold: 3
  initialDelaySeconds: 10
  periodSeconds: 30
  successThreshold: 1
  timeoutSeconds: 5
```

## Thanks

★ [Terminus](https://github.com/godaddy/terminus)

★ [Lightship](https://github.com/gajus/lightship)

★ [Bret Fisher](https://github.com/BretFisher)

★ [Node HTTP documentation](https://nodejs.org/api/http.html)

## Donate

[![Donate](https://img.shields.io/badge/Donate-PayPal-green.svg)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=JN3XLTQCX3NR8&source=url)

If you like my job, don't hesite to contribute to this project! ❤️