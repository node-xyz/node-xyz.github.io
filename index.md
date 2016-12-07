---
layout: default
title: XYZ Homepage
---

## Node XYZ

Node XYZ is a NodeJS toolkit for creating microservice based distributed applications. The main aim of XYZ is to cover following aspects of microservice architecture:

  - **Service discovery and Communication**:
    XYZ provides an easy *Path based* identification for services that each host is exposing. This ensures flexible messaging infrastructure. Consequently, XYZ systems are ***Soft*** and can act as you wish when it comes to accepting new hosts to the system.

  - **Deployment and Development**:
    XYZ provides a handy command line interface, using which you can easily test your microservices locally, deploy them and monitor them.

  - **Infinite Scalability: Configurations** are the last key component to ensure that *XYZ will commensurate with whatever requirements your system might have*. That is to say, XYZ will not enforce any application dependent configuration, instead, it will provide only a default behavior (as a fallback) for it and enables you to easily override this default behavior.

---

## Watch the introduction video:

<iframe width="800" height="491" src="http://www.powtoon.com/embed/dRNJOFylWnr/" frameborder="0"></iframe>

## Documentations

The following documentations will help you get started with xyz, and understand it better:

  - **Getting started**: a minimal 5 step tutorial, covering installation, setting up and writing a simple microservices. This tutorial is consisted of:
    - [Hello XYZ](/documentations/getting-started#hello-xyz)
    - [Replicate your services](/documentations/getting-started#replicating-your-nodes-and-services)
    - [Service Discovery](/documentations/getting-started#service-discovery)
    - [Path based services](/documentations/getting-started#path-based-service-identification)
    - [Middlewares](/documentations/getting-started#middlewares)

  ---

  - **Advance Topics**[*work in progress*]:
    - An odyssey into XYZ: lifecycle of a single requests
    - XYZ Command line interface
    - Logging
    - Explaining how middlewares actually work
    - Best practices

  ---

  - **Wiki**: More technical details about the architecture and aims of xyz can be found in our [wiki page](https://github.com/node-xyz/xyz-core/wiki)


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
  - [`transport.global.send.logger`](https://github.com/node-xyz/xyz.transport.global.receive.logger) : logs as each request is being received

#### Basic Transport level authentication plugins
  - [`transport.auth.basic.send`](https://github.com/node-xyz/xyz.transport.auth.basic.send) : adds a naive password authentication header to each request being sent
  - [`transport.auth.basic.receive`](https://github.com/node-xyz/xyz.transport.auth.basic.receive) : checks a naive password authentication as each request is being received

#### Service Discovery plugins
  - [**D**][`service.send.first.find`](https://github.com/node-xyz/xyz.service.send.first.find) : first find strategy for service discovery
  - [`service.send.to.all`](https://github.com/node-xyz/xyz.service.send.to.all) : send to all strategy for service discovery

  > plugins marked by [**D**] are installed by default

---
