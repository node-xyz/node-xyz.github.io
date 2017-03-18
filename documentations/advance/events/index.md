---
layout: doc-page
title: Getting Started with Node XYZ
breadcrumb: events
---

# Default Events

Due to the fact that xyz provides middleware handlers for most of the tasks and requests, most of the customized functionality can be implemented using them. Though, some events can also become very handy.

With the current release, xyz-core only emits events from the `ServiceRepository` layer. This means that you can listen to them like:

```
aMicroServ = new XYZ({...})

aMicroServ.serviceRepository.on('event', ()=>{})
```

The current emitted events are as follows:

  - `'message:receive'`: emitted whenever a request (analogous to `.call()` from another service) is received. Parameters: an object with keys:
    - body: payload received with the request

  - `'message:send'`: emitted whenever a request (analogous to `.call()`) is being sent. Parameters: an object with keys:
    - opt: options passed to the `.call()`


Note that these events are received whenever ***Service Repository*** is informed of these events. This means that the Transport layer's middlewares might drop a request due to any reason and in these scenarios, the service repository will not be informed, hence these events will not be fired.

A good way to use these events is through bootstrap functions. As an example, assume that your application wants a specific logging behavior whenever a message is being received, for security reasons for example.

You could do something like:

{% highlight javascript %}
function _topSecretMessageLogger (xyz) {
  xyz.serviceRepository.on('message:receive', (data) => {
    topSecretTunnel.write('message received!')
    ...
  })
}

module.exports = _topSecretMessageLogger

{% endhighlight %}

And in you main app
{% highlight javascript%}

let aMicroServ = new XYZ({...})
aMicroServ.bootstrap(require('./_topSecretMessageLogger'))

{% endhighlight %}

# Events under the hood

If you think that these two events are not enough, which is not hard to argue!, you can also consider listening to xyz's internal events.

XYZ is made up of two underlying layer, Service layer and Transport Layer. These two layers communicate with each other using events, and since you have access to them using `xyz.serviceRepository.transport`, you can also listen to these events.

As an example, you might have noticed that one of the default middlewares in transportServer's `call.receive.mw` is something named **[_httpMessageEvent](https://github.com/node-xyz/xyz-core/blob/master/src/Transport/Middlewares/call/http.receive.event.js)**. The only thing that this middleware does is to emit an event on the transport layer so that the service Repository can listen to it and receive it.

{% highlight javascript %}
_transport.emit(
    CONSTANTS.events.MESSAGE, {
      userPayload: body.userPayload,
      serviceName: body.service
    },
    response)
{% endhighlight %}

If you want to listen to it, you can do something like:

{% highlight javascript %}
let aMicroServ = new XYZ({...})
let CONSTANTS  = aMicroServ.CONSTANTS
// more info on constant names of events:
// https://github.com/node-xyz/xyz-core/blob/master/src/Config/Constants.js

// will listen to messages emitted from a server on port 4000
aMicroServ.serviceRepository.transport.servers[4000].on(CONSTANTS.events.MESSAGE, (data) => {
    ...
    ...
  })
{% endhighlight %}

## This is not the end!

We also encourage all plugins and middlewares to use this design pattern efficiently, when required. This means that one bootstrap function might actually extent the number of event that the `serviceRepository` can emit and add an event to it. As an example, since each bootstrap function has access to `xyz`, it has access to `xyz.serviceRepository`, hence it can `xyz.serviceRepository.emit(...)` and your code listen to it like any other event.
