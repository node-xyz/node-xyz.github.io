# An odyssey into XYZ: lifecycle of a single requests

This page is to review almost everything that you already know (or don't know) about the xyz system.

We will first begin with the initialization of the application, looking into configuration and bootstrap phases. Next we will look into all of the steps that are taken behind the scene after a single `.call(...)`. Don't get me wrong, it might seem a lot, but it's actually pretty simple and straight forward. Knowing these steps is very useful when it comes to debugging you application, or tweaking it. 

This documentation assumes that you are familiar with the **Getting Started** section.

## A loot at the final application

This is what we are going to have at the end of this tutorial: 

```
let fn = require('./../shared/mock.functions')
let XYZ = require('xyz-core')
let monitorBootstrap = require('xyz.monitor.basic.bootstrap').bootstrap

let basicAuthRcv = require('xyz.transport.auth.basic.receive')
let basicAuthSnd = require('xyz.transport.auth.basic.send')

var mathMs = new XYZ({
  selfConf: {
    allowJoin: true,
    name: 'MathMs',
    host: '127.0.0.1',
    port: 3333
  },
  systemConf: { nodes: []}
})

mathMs.bootstrap(monitorBootstrap, 9001)

mathMs.serviceRepository.transportClient.joinDispatchMiddlewareStack.register(0, basicAuthSnd)
mathMs.serviceRepository.transportServer.joinReceiveMiddlewareStack.register(0, basicAuthRcv)

mathMs.register('/math/decimal/mul', fn.mul)
mathMs.register('/math/decimal/neg', fn.neg)
mathMs.register('/math/decimal/sub', fn.sub)
mathMs.register('/blank', fn.blank)

console.log(mathMs)

```

The only thing that you have to assume is that `./../mock.functions.js` is a file containing some functions that will respond to requests appropriate to their name. As an example: 

```
module.exports = {
  mul: function mul (payload, xResponse) {
    xResponse.send(payload.x * payload.y)
  },
  ...
  
}
```

### Step 1: creating the xyz instance 

The first step is to create a new `XYZ` object. 

```
var mathMs = new XYZ({
  selfConf: {
    allowJoin: true,
    name: 'MathMs',
    host: '127.0.0.1',
    port: 3333
  },
  systemConf: { nodes: []}
})
```

