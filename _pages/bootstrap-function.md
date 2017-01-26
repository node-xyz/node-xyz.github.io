---
layout: page
title: Bootstrap function
permalink: /documentations/bootstrap-function
---

# Bootstrap functions

Configuring an xyz instance can sometimes be annoying. Many nodes want to have the same configurations, same middlewares etc, but is is good to copy and paste a chunk of code in all of these microservices? I mean, think about it, Aside from `.register` which creates new endpoints for functions, there is a good chance that most of the lines of the microservices that you are writing are the same among all of the microservices in that application/project. One of the ways to encapsulate these common steps in an isolated environment so that they can be reused is **Bootstrap function**.

Bootstrap functions are... nothing in fact. They're just functions that take in the current xyz instance as argument and are free to make any modifications to it. This can include `exposing new functions`, listening to `events`, registering new `middlewares` etc.

A complete example of this is the [Monitor Bootstrap function](https://github.com/node-xyz/xyz.monitor.basic.bootstrap). Let's have a closer look at this bootstrap function:

```
let express = require('express')
let monitorCallReceive = require('./middleware/xyz.monitor.call.receive.middleware')
let monitorCallSend = require('./middleware/xyz.monitor.call.send.middleware')
let load = { snd: 0, rcv: 0}
let logger

function basicMonitorBootstrap (xyz, port = 5000) {
  /*
  Initialize a simple express server
   */
  logger = xyz.logger
  var app = express()
  app.use(express.static(__dirname + '/lib'))

  app.get('/', function (req, res) {
    res.sendFile(__dirname + '/index.html')
  })

  app.get('/all', function (req, res) {
    res.json({load: load, inspectJSON: xyz.inspectJSON(), inspect: xyz.inspect()})
  })
  app.listen(port, function () {
    logger.info(`monitor server started at port ${port}`)
  })

  /*
  Register required middlewares
   */
  xyz.serviceRepository.transportServer.callReceiveMiddlewareStack.register(0,
    monitorCallReceive)
  xyz.serviceRepository.transportClient.callDispatchMiddlewareStack.register(0,
    monitorCallSend)
}

module.exports = {
  bootstrap: basicMonitorBootstrap,
  setSendLoad: (aLoad) => {
    load.snd = aLoad
  },
  setRcvLoad: (aLoad) => {
    load.rcv = aLoad
  }
}
```

  - It can be seen that every bootstrap module must return a function (the actual bootstrap function) that takes the `xyz` object as an argument. In this case, this bootstrap function exports `bootstrap: basicMonitorBootstrap` which will be later used to apply this bootstrap function.
  - Inside a bootstrap function, you are allowed to do almost any thing. In this case, we are creating a very simple express server to serve a single html file and a json API and we are registering two new middlewares inside the system.  
  - Note that theses two middlewares that we are exposing will later on used  
    ```
    setSendLoad: (aLoad) => {
      load.snd = aLoad
    },
    setRcvLoad: (aLoad) => {
      load.rcv = aLoad
    }
    ```
    to set an estimate value for the load on the node. These values will then be served through the HTTP server.

So, given that this bootstrap function is written once, you can always use it inside any of your microservices with just one (or two, including the `require` statement) line of code:

```
let xyzMonitor = require('./../xyz.monitor.basic.bootstrap')
let mathMs = new XYZ({
  selfConf: {...},
  systemConf: { nodes: []}
})

xyzMonitor.bootstrap(mathMs, 7000)

```

There is complete example of this [here](https://github.com/node-xyz/xyz.monitor.basic.bootstrap/tree/master/example). After you set up this bootstrap function, you can view a nice monitoring page at your localhost (and on port `7000` in this case):

![example monitor app](/assets/monitor.png)
