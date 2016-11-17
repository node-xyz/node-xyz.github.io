---
layout: default
title: {{ site.name }}
---
## Node XYZ

Node XYZ is a NodeJS toolkit for creating microservice based distributed applications. It helps you:

  - Write microservices without worrying about all of the process communication overhead. Focus only on your business logic.
  - Dynamically join/remove nodes to your cluster of microservices.
  - Customize your system as much as possible. Authentication, logging, preprocess and post-processes are all possible via middlewares.
  - Deploy services --on the fly-- with structured path based identifier and call them within your service with different strategies.

---

## Install

XYZ has several packages and plugins. The one that should be solely included in your scripts is _xyz-core_

    npm install xyz-core


---

### Plugins

Current XYZ plugins are:

#### Transport Plugins for service call
  - [**D**]`transport.call.receive.event` : passes a call receive event to servicer repository
  - [**D**]`transport.call.send.export` : sends a call request out to the network

#### Transport Plugins for ping
  - [**D**]`transport.ping.receive.event` : passes a ping receive event to servicer repository
  - [**D**]`transport.ping.send.export` : sends a ping request out to the network

### General Transport Plugins
  - [`transport.global.receive.logger`](https://github.com/node-xyz/xyz.transport.global.receive.logger) : logs as each request is being sent
  - `transport.global.send.logger` : logs as each request is being received

#### Basic Transport level authentication plugins
  - [`transport.auth.basic.send`](https://github.com/node-xyz/xyz.transport.auth.basic.send) : adds a naive password authentication header to each request being sent
  - `transport.auth.basic.receive` : checks a naive password authentication as each request is being received

#### Service Discovery plugins
  - [**D**]`service.send.first.find` : first find strategy for service discovery
  - `service.send.to.all` : send to all strategy for service discovery

  > plugins marked by [**D**] are installed by default


---

## Getting started

The following sections will guide you through the basics of using xyz.

  - [Hello XYZ](/getting-started#hello-xyz)
  - [Replicate your services](/getting-started#replicating-your-nodes-and-services)
  -  Who is where?
  -  Path based services
  -  Middlewares
