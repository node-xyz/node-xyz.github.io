---
layout: doc-page
title: Bootstrap functions
breadcrumb: bootstrap-function
---

Configuring an xyz instance can sometimes be annoying. Many nodes want to have the same configurations, same middlewares etc., but is it good to copy and paste a chunk of code in all of these microservices? I mean, think about it, Aside from `.register` which creates new endpoints for functions, and is dependent to the **business logic**, there is a good chance that most of the lines of the microservices that you are writing are the same among all of the microservices in that application/project. One of the ways to encapsulate these common steps in an isolated environment so that they can be reused as a **Bootstrap function**.

# Bootstrap functions


Bootstrap functions are... nothing in fact. They're just **_functions that take in the current xyz instance as argument and are free to make any modifications to it_**. This can include **exposing new functions**, listening to **events**, registering new **middlewares** etc.

# Case Study: Node Monitor
A complete example of this is the [Monitor Bootstrap function](https://github.com/node-xyz/xyz.monitor.basic.bootstrap). What this bootstrap function basically does is that it inserts two middlewares in the transport layer, which count the number of messages sent and received. Finally it will launch an express HTTP server to serve this information, alongside some additional data about the system, to a specific end point.

Let's see the big picture before going any further!

![](/assets/img/monitor.info.png)

As you see, the monitor bootstrap will automatically insert two new middlewares, indicated by green color in the image. These middlewares will measure the message rate **at transport level** and will report them to a new component, independent of `xyz-core`, an express server.

Keeping these information in mind, now we can see the code:

{% highlight javascript %}
function _basicMonitor (xyz, port = 5000) {
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
  let listener = app.listen(port, function () {
    logger.info(`monitor server started at port ${listener.address().port}`)
  })

  /*
  Register required middlewares
   */
  xyz.middlewares().transport.server('CALL')(xyz.id().port).register(0,
    monitorCallReceive)
  xyz.middlewares().transport.client('CALL').register(0,
    monitorCallSend)
}

module.exports = {
  // will be used by the user of this bootstrap function
  bootstrap: _basicMonitor,

  // will be used by the two middlewares inserted
  setSendLoad: (aLoad) => {
    load.snd = aLoad
  },
  setRcvLoad: (aLoad) => {
    load.rcv = aLoad
  }
}

{% endhighlight %}
 As you see, inside the bootstrap function, we manipulate a `xyz` parameter which is supposed to be passed into this function when it is being invoked. This parameter can be treated exactly as if it were an instance of the `XYZ` class. To simplify this even more:

Inside your normal code

{% highlight javascript %}
let ms = new XYZ({...})

ms.middlewres().register(0, _fooMw)

{% endhighlight %}

inside a bootstrap function:

{% highlight javascript %}
function bootstrapFoo(xyz) {
  xyz.middlewares().register(0, _fooMw)
}
{% endhighlight %}



It can also be seen that this simple monitor example will insert these middlewares to the default `CALL` route, located on the **primary server and port** of the node by default. This bootstrap function can be configured to work with multiple servers and outgoing routes, but that is not the our primary concern right now. You can see the repository of this bootstrap function for more information.

#### Notes

  - Every bootstrap module must return a function (the actual bootstrap function) that takes the `xyz` object as an argument. In this case, this bootstrap function exports `bootstrap: _basicMonitor` which will be later used to apply this bootstrap function.
  - Inside a bootstrap function, you are allowed to do almost any thing. In this case, we are creating a very simple express server to serve a single html file and a json API and we are registering two new middlewares inside the system.  
  - Theses two middlewares that we are exposing will later on use
    ```
    setSendLoad: (aLoad) => {
      load.snd = aLoad
    },
    setRcvLoad: (aLoad) => {
      load.rcv = aLoad
    }
    ```
    to set an estimate value for the load on the node. These values will then be served through the HTTP server.

Assuming that we have this bootstrap function, you could use it with:


{% highlight javascript %}
let ms = new XYZ({...})
let _basicMonitor = require('./_basicMonitor')

_basicMonitor(ms)
{% endhighlight %}

This would work, but we highly recommend you to use this method:

{% highlight javascript %}
let ms = new XYZ({...})
let _basicMonitor = require('./_basicMonitor')

ms.bootstrap(_basicMonitor)
{% endhighlight %}

This allows an xyz instance to keep track of its bootstrap functions and ensure their compatibility, if required. Aside form that, it can output its list of bootstrap functions in logs. You have already this when you use `console.log` on an instance of xyz. Take another look if you haven't seen the bootstrap section printed!

So, given that this bootstrap function is written once, you can always use it inside any of your microservices with just one (or two, including the `require` statement) line of code:

{% highlight javascript %}
let xyzMonitor = require('xyz.monitor.bootstrap').bootstrap
let mathMs = new XYZ({
  selfConf: {...},
  systemConf: { nodes: []}
})

mathMs.bootstrap(xyzMonitor, 7000)

{% endhighlight %}

After you set up this bootstrap function, you can view a nice monitoring page at your localhost (and on port `7000` in this case):

![example monitor app](/assets/img/monitor.example.png)

This bootstrap function is available on both npm and github. You can install and use it using:

```
npm install xyz.monitor.basic.bootstrap
```
