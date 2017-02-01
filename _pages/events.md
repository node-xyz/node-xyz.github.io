---
layout: page
title: Getting Started with Node XYZ
permalink: /documentations/events
---

# Events

Due to the fact that xyz provides middleware handlers for most of the tasks and requests, most of the customized functionality can be implemented using them. Though, some events can also become very handy.

With the current release, xyz-core only emits events from the `ServiceRepository` layer. This means that you can listen to them like:

```
aMicroServ = new XYZ({...})

aMicroServ.serviceRepository.on('event', ()=>{})
```

The current emitted events are as follows:

  - `'request:receive'`: emitted whenever a request (analogous to `.call()` from another service) is received. Parameters: an object with keys:
    - body: payload received with the request

  - `'request:send'`: emitted whenever a request (analogous to `.call()` from another service) is being sent. Parameters: an object with keys:
    - opt: options passed to the `.call()`

  - `'cluster:join'`: emitted whenever a new node's join request has been received.  Parameters: an object with keys:
    - body: payload received with the join request


Note that these events are received whenever ***Service Repository*** is informed of these events. This means that the Transport layer's middlewares might drop a request/join request due to any reason and in these scenarios, the service repository will not be informed, hence these events will not be called.

A good way to use these events is through bootstrap functions. As an example, assume that your application wants a specific behavior whenever a node join the cluster and you want to implement this using events, not [JoinDispatch and JoinReceive](https://github.com/node-xyz/xyz-core/blob/master/xyz.js#L120) middlewares.

You could do something like:

```
function clusterListenerBootstrap (xyz) {
  xyz.serviceRepository.on('cluster:join', (data) => {
    console.log(`joined cluster from event listener`)
    ...
    ...
  })
}

module.exports = clusterListenerBootstrap


// in you main app

let aMicroServ = new XYZ({...})
aMicroServ.bootstrap(clusterListenerBootstrap)

```


## Some events under the hood

If you think that these two events are not enough, which is not hard to imagine!, you can also consider listening to xyz's internal events.

XYZ is made up of two underlying layer, Service layer and Transport Layer. These two layers communicate with each other using events, and since you have access to them using `xyz.serviceRepository.transportClient` and `xyz.serviceRepository.transportServer`, you can also listen to these events.

As an example, you might have noticed that one of the default middlewares in transportServer's `callReceiveMiddlewareStack` is something named [call.receive.event.middleware](https://github.com/node-xyz/xyz-core/blob/master/src/Transport/Middlewares/call/call.receive.event.middleware.js). The only thing that this middleware does is to emit an event on the transportServer so that the service Repository can listen to it and receive it.

```
_transport.emit(
    CONSTANTS.events.REQUEST, {
      userPayload: body.userPayload,
      serviceName: body.service
    },
    response)
```

If you want to listen to it, you can do something like:

```
let aMicroServ = new XYZ({...})
let CONSTANTS  = aMicroServ.CONSTANTS
// more info on constant names of events:
// https://github.com/node-xyz/xyz-core/blob/master/src/Config/Constants.js
aMicroServ.serviceRepository.transportClient.on(CONSTANTS.events.REQUEST, (data) => {
    ...
    ...
  })
```
