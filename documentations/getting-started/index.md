---
layout: doc-page
title: Getting Started with Node XYZ
breadcrumb: getting-started
---

# Hello XYZ

This brief tutorial will help you fathom the concepts of XYZ, and help you get started with it.

The only dependency of this tutorial is `xyz-core`, which has only one logging dependency itself. You can install it  in your woking directory using:

```
$ npm install xyz-core --save
```

The code and testcases of this documentation is available [here](https://github.com/node-xyz/xyz.example.getting.started)

Let's get this tutorial started!

### Baby Microservice

Assume that we only have two services that will communicate with each other. Suppose that each of these services (aka. Node modules) have another API exposed two end users. That is not our business at the moment, since XYZ will divide **the matter of end user** and the **matter of system**. This means that although you can have end user calling the APIs created in this tutorial, their main goal is to serve other nodes inside your own system. In this section, our goal is to focus on what is important to our own system.

The -- dummy -- services are as follows:

```javascript
// math.ms.js
// a very fancy math module with super fast calculation
function add(x, y) {
  return x + y
}

function mul(x, y) {
  return x * y
}

// string.ms.js
// some ultra fast string manipulations
function up(str) {
  return s.toUpperCase()
}
function down(str) {
  return s.toLowerCase()
}
```

Note that these functionalities are, of course, way to small and naive to be exposed as a standalone service, even in a microservice architecture. But we will stick to these services, **Math** and **String** in some of our tutorial to keep things simple.

Now, assume that these services need to call each other in order to share their services, or more realistic, in order to do something for the end-user. This is the first place that XYZ will come to action.

In order to have two xyz instances communicate with each other, they need to have a system, a **shared** system. Information about the system is filled to the xyz instance using a `config` parameter. The config parameter has information such as local `port`/`ip`/`service name` and a list of nodes who live inside the system. Note that a node can join the system without being in the static list (more on this later), this is just a basic example!

Our two services are going to be deployed with the following topology:

- math.ms will be hosted on port 4000 of the local machine
- string.ms will be hosted on port 5000 of the local machine

Therefore, we need to pass the address of the other node to each of the nodes. This will be placed inside `systemConf.nodes`.

Let's look at the `math.ms.js` first:

{% highlight javascript %}
// math.ms.js
// a very fancy math module with super fast calculation
let XYZ = require('xyz-core')

let mathMS = new XYZ({
  selfConf: {
    // will indicate the name of the sevice in logs
    name: 'math.ms',
    // default hostname
    host: '127.0.0.1'
  },
  systemConf: {
    // list of other addresses
    nodes: ['127.0.0.1:5000']
  }
})

// expose these function over the system

mathMS.register('mul', (payload, response) => {
  response.jsonify(payload.x * payload.y)
})

mathMS.register('add', (payload, response) => {
  response.jsonify(payload.x + payload.y)
})
{% endhighlight %}

Strange thing is that we did not tell `math.ms` to place itself on port 4000. Why?

> Each node will load itself with one HTTP server on port 4000 by default. You can read more about thr default values in [Configuration]() section and more about Servers in [Server Management]() section

And the `string.ms` is fairly similar. We expose some functions relating to string objects. The most important difference is that now we must explicitly tell `string.ms` to listen on port 5000. This can be done using the `selfConf.transport[]` array.

{% highlight javascript %}
//string.ms.js
let XYZ = require('xyz-core')

let stringMS = new XYZ({
  selfConf: {
    name: 'string.ms',
    host: '127.0.0.1',
    transport: [{type: 'HTTP', port: 5000}]
  },
  systemConf: {
    nodes: ['127.0.0.1:4000']
  }
})
stringMS.register('up', (payload, response) => {
  response.jsonify(payload.toUpperCase())
})
stringMS.register('down', (payload, response) => {
  response.jsonify(payload.toLowerCase())
})
{% endhighlight %}

Now, if you run both `math.ms.js` and `string.ms.js`, you'll notice that... well, nothing is yet happening because we didn't write any code for anything to happen! We will add some code in a minute to have one service calling the other one.

> I see an error: `[math.ms@127.0.0.1:4000] error :: Ping Error :: 127.0.0.1:5000 has been out of reach for x pings :: ... `

This is natural. XYZ has a built in and configurable service discovery mechanism out of the box. You can disable it of course, but it will work by default. This is error is because each node will iteratively check other node's health by pinging it and the reason you are seeing this is because `string.ms` was not able to respond to a ping message. This error should go away as you run `string.ms`

> **notes about the default ping**: XYZ offers a few ping strategies, explained in a different section ([Ping Types]()). The [default ping]() is a naive, yet reliable approach which is easy and appropriate to use in test environmetns. Using the default ping, each node will probe other nodes and wait for a response. If a node is out of reach for 10 pings, it will be removed from the `systemConf.nodes[]`. The default interval for pinging is 2000ms.

Moving away from all that ping stuff. Now, thanks to the service discovery mechanism, our services are virtually binded to one another and they can call the services that each of them is exposing using `.register()`. The good part of `.register()` is that the callee is transparent to the location of this function (or at least, has the option to be) and can call it very easily. Let's see how:

Add the following to `string.ms.js`:

{% highlight javascript %}
setInterval(() => {
  // .call will invoke a foreign (or even local) service exposed using .register
  // servicePath must be identical in caller and callee
  stringMS.call({servicePath: 'mul', payload: {x: 2, y: 5}}, (err, body, res) => {
    console.log(`response of mul => ${body} [err ${err}]`)
  })
}, 2000)

{% endhighlight %}


You should now see an output like:
`response of mul => 10 [err null]`

#### A Note on terminology.

Before going any further, it is important to be on the same page regarding terminology.

- <span class='spacing'> SERVICE </span>: each function -- or literally any piece of code -- exposed via `.register` is a **_service_**.
- <span class='spacing'> NODE </span>: Each _Node Process_, having a number of _services_ exposed using `xyz-core` and `.register()` is called a **node**.


#### What's next?

In this example we hardcoded the list of service, so that they can communicate with each other. This is... just about ok. It is not awesome. What would be awesome is to have nodes joining the system dynamically. Yes, that's what we're going to do in the Next Section. Aside from being way less _Cool_, having a fixed list of nodes in a [micro]service based architecture is almost useless. There is a pretty good chance of having _new nodes_, or seeing _existing nodes crash or restart_. We can not achieve a **_proper service discovery_** using a fixed list of nodes.

Furthermore, you can even test this flaw in the current system. If you kill `math.ms`, you'll seee immediatly that the response of `.call()` changes to `null`. And the sad part is that if you do not re-lunch `math.ms` before 10 ping cycles is over, `string.ms` will remove it from its `systemConf` (literrly, it will forget about `math.ms` and remove all of its information. like a sad break-up) and there is no coming back from this. (just kidding. there is a way called `seed nodes` and we will get to it soon)

# Replicating Your Nodes and Services

In the end of the last section, we reached a conclusion:

It would have been much more interesting, if we could add nodes dynamically. Good news: we can.

XYZ nodes can join a system, given that they have a [list of] seed node[s]. Seed nodes are the entry point to the system. They should check the authority of the incoming node, if required, and pass the list of nodes available inside the system to it, or any other authentication certificate, again, if required. The fact that there are a lots of ' _if required_ ' phrases in the last statement is that all of these steps (like any authentication or certificate exchange) are optional and the developer can choose to have them. In the most **_Wildcard_-ish** scenario, all nodes are open to accpeing join requests and any other node can join them only by knowing their IP address.

This **Join Mechanism** is a subset of the **Ping Mechanism**. In other words, every ping mechanism should implement how and when join requests should be accepted. Since the [_default ping_]() is pretty naive and focuses of test and developement phases, it does not **enforce** any authentication. Albeit, we will see in the end of this tutorial that we can modify this behaviour with some small changes.

Let's assume that both the string.ms and _math.ms_ in the previous section do not know each other beforehand. That is to say, their `systemConf.nodes[]` is empty:

{% highlight javascript %}
let stringMS = new XYZ({
  selfConf: {
    name: 'string.ms',
    host: '127.0.0.1',
    transport: [{type: 'HTTP', port: 5000}]
  },
  systemConf: {nodes: []}
})
{% endhighlight %}

and

{% highlight javascript %}
let mathMS = new XYZ({
  selfConf: {
    name: 'math.ms',
    host: '127.0.0.1'
  },
  systemConf: {nodes: []}
})
{% endhighlight %}




> Note that you can omit putting the empty `nodes: []` in your config file. In fact, you can initialize youre system using `new XYZ({})`. See the [Configuration]() section for more information about the default valuse.

Furthermore, let's assume that the `math.ms` is the constant node in the system and `string.ms`'s are going to come and go. Go ahead and run the `math.ms.js` alone, you will see that nothing happens. It's ok. math.ms does nothing.

Now run the `string.ms.js`. you'll see:

```
[2017-3-6 16:11:30][string.ms@127.0.0.1:5000] warn :: Sending a message to /mul from first find strategy failed (Local Response)
response of mul => null [err Not Found]
```

The [first find strategy]() is a default behavior embedded inside node XYZ for finding nodes to send messages to. In more advanced sections of this documentation, you'll learn that the `sendStrategy` is actuallya middleware in `xyz-core`. This enables you to change any function inside the middleware at runtime. More on this later!

The main point is that **the message failed**. Obviously because string.ms does not know any math.ms at this point, so it does not actually send the message to the network and instead, it will locally respond to it with en error.

Change the `string.ms.js` to the following, specifying a new **seed node** for it.

{% highlight javascript lineno %}
let stringMS = new XYZ({
  selfConf: {
    name: 'string.ms',
    host: '127.0.0.1',
    // new parameter
    seed: ['127.0.0.1:4000'],
    transport: [{type: 'HTTP', port: 5000}]
  },
  systemConf: {nodes: []}
})
{% endhighlight %}

Note that you do not need to chnage `math.ms.js` at all. Joining is a part of Ping and pings a er configured using a different approach  (if you are curious, using [Bootstrap Functions]()), not using the xyz constructor.

Now, the two nodes have no previous knowledge of each other, yet they can join and work with each other. Go ahead and run the math.ms, next run the string.ms in a separate terminal and see the results. As you see, the output is still the same. You might also see a `info` log indicating that join has been successfully approved.

in `string.ms.js`:

```bash
[string.ms@127.0.0.1:5000] info :: JOIN PING ACCEPTED. response : ...
```

in `math.ms.js`:
```
[math.ms@127.0.0.1:4000] info :: A new node {127.0.0.1:5000} added to systemConf
```

This get's even more interesting when we add more nodes. You can add more string.ms nodes and see that all of them will eventually find the math.ms and can connect with it. To be more accurate, the do not just find `math.ms`, they find all of the nodes in the system, including the previously deployed `string.ms`s.

To test this, there is only one small problem. All string.ms ss are configured to use one port. Hopefully, XYZ provides a way to override any configuration, for debugging purposes mainly. We'll cover this later (see [Configuration] for more detail), but, for now, you can use the `--xyz-transport.0.port` command line argument to override the port. Run many many more StringMS instances with:

{% highlight bash %}
$ node string.ms.js --xyz-transport.0.port 5001
$ node string.ms.js --xyz-transport.0.port 5002
$ node string.ms.js --xyz-transport.0.port 5003
...
{% endhighlight %}

> The `--xyz` prefix is backdoor for overriding **any** value in `selfConf`. Keeping this in mind, I think you can find the relation between `--xyz-transport.0.port 5001` and `selfConf : {transport: [{type: 'HTTP', port:  5001}]}`

All of your logs should indicate that all `string.ms` nodes are calling `mul` successfully and all nodes should be logged `joined` in `math.ms`.

A question might come to mind at this point. We have many sender nodes, and only just one receiver node in this example. So... it is kinda obvious how each request gets routed. In other words, a `string.ms` does not have that much of an option!

Assume that we have two receiver nodes (math.ms) and a dozen sender nodes, how could we decide on one of the math.ms nodes?

You might argue that this is a matter that xyz should handle. Indeed, we do handle it. it's just that we are very keen to keep you involved whit what's happening under the hood, so that you can change it when required.

In order to have different message routing ways, xyz supports two mechanisms:

  1. Service Discovery middlewares
  2. Path based service identification

The upcoming sections will describe these concepts.

Before going into the next section, you might argue that why is this so important? In a microservice-based system, it is crucially important!

If you think about the system as a whole, you come to realize that many many types of messages might be sent. One node might want to broadcast a message to **all other nodes**, or to a subset of them. A node might want to apprise just a few database nodes about a record update, and many many more events that might happen!

# Service Discovery

As mentioned in the previous section, the process of choosing a destination node when making a call is rather difficult and obscure. XYZ provides a configuration for each of the nodes, so that each node can actually choose a strategy for sending a message. Note that each node can have its own `sendStrategy`.

The good thing about the `sendStrategy` is that it is a **plugin** (aka. middleware). these middlewares are completely modifiable. That is to say, you can write your own strategy and patch it into your system.


Although one of the reasons for having send strategies as a plugin is to keep the main repository minimal ,`xyz-core` has a few fundamental send strategies built in:

- <span class='spacing'> [xyz.first.find]() </span> . **_Note that this is the default sendStrategy_**.
- <span class='spacing'> [xyz.send.to.all]() </span>
- <span class='spacing'> [xyz.send.to.target]() </span>
- <span class='spacing'> [xyz.broadcast.local]() </span>
- <span class='spacing'> [xyz.broadcast.global]() </span>

In this tutorial we are only interested in the first two.

We will avoid going into the code of these middlewares(although they are fairly simple and you can probably get a feeling by seeing the source code), but it is crucially important to understand exactly what they are doing.

The `sendStrategy` middleware has the following main duty:

> it must receive all of the parameters that you pass to `.call()`, with some additional information that `xyz-core` injects to it, and pass this message request to the `Transport` layer.

(You will soon understand clearly what the **Transport layer** is. For now you can assume that it handles network messages.)

In other words, (at least using `xyz.first.find`) you never call a service by specifying its exact `ip:port`. **This information will be discovered using sendStrategy**. (awkward side note: `xyz.send.to.target` is a sendStrategy middleware that accepts the ip and port of the destination from you and redirects the message exactly to that node, regardless of any other information)


In order to investigate the last sentence better, let's verbally describe how `xyz.first.find` does its duty:

1. After you send a message using `.call()`, the send strategy function will be invoked. this function has access to all of the parameters of `.call()` and some additional data.
2. One of these additional data, is list of functions exposed by **all of the nodes in the system**. This information is persisted throughout the system during ping calls.
3. Assuming that you called a service with `{servicePath: 'foo'}`, `xyz.first.find` will then search this list and find all of the nodes that have enlisted `foo` as on of their functions.
4. Finally it will invoke the Transport layer **with the ip and port of the first node** in the list that can response to this message. (and the rest of the job will be carried by the Transport layer middlewares).

The `send.to.all` works in similar manner, but it will send the message to **all of the nodes that have `foo` as an exposed service**.


XYZ is configured to work with _first find_ by default. if you wish to change this, a new key named **`defaultSendStrategy`** should be added to the **selfConf** key passed to xyz constructor. Built in middlewares can be imported from `xyz-core/src/Service/Middlewares/[...]`. Let's set `send.to.all` function for `string.ms.js`:

{% highlight javascript %}
let stringMS = new XYZ({
  selfConf: {
    name: 'string.ms',
    host: '127.0.0.1',
    seed: ['127.0.0.1:4000'],
    defaultSendStrategy: require('xyz-core/src/Service/Middleware/service.send.to.all'),
    transport: [{type: 'HTTP', port: 5000}]
  },
  systemConf: {nodes: []}
})
{% endhighlight %}

Again, the code of `math.ms` does not have to change at all. In fact, math.ms does not care about any of this, since it's sitting in the receiving side of this scenario.

Before going any further, you can see at this very point that `send.to.all` has had some effetcs!. run one math and one string service and see the difference in the response. Since `send.to.all` assumes multiple messages are being sent, it also returns an object of responses in return:

```bash
response of mul => {"127.0.0.1:4000:/mul":[null,10]} [err null]
```

So if we run three math.ms instances, `send.to.all` finds three target nodes and we expect to receive an object with three keys in return, each indicating the response of that node!

Let's run 3 math services, and a dozen of string services. The configurations are similar to the previous section. `string.ms` has _send to all_ as sendStrategy and `127.0.0.1:4000` as seed node.

You can run your services in the following order:

{% highlight bash %}
//default math at port 4000
$ node math.ms.js

//default string at port 5000
$ node string.ms.js

// more string on different ports
$ node string.ms.js --xyz-transport.0.port 5001
$ node string.ms.js --xyz-transport.0.port 5002

{% endhighlight %}

at this point, each string will have receive a response like:

```bash
response of mul => {"127.0.0.1:4000:/mul":[null,10]} [err null]
```

As you add more `math.ms` to the system, all information regarding join event will be propagated to the system and string.ms will receive more and more response!

{% highlight bash %}
//note that these new instances must have a seed node,
//otherwise they can not find the rest of the system!

$ node math.ms.js --xyz-transport.0.port 3999 --xyz-seed 127.0.0.1:4000
$ node math.ms.js --xyz-transport.0.port 3998 --xyz-seed 127.0.0.1:4000
$ node math.ms.js --xyz-transport.0.port 3997 --xyz-seed 127.0.0.1:4000

{% endhighlight %}

Similar to how we can override the port number, a single seed node can also be added by command line argument using the `--xyz` prefix.

At this point, you should expect to see:

```bash
response of mul => {
  "127.0.0.1:4000:/mul":[null,10],
  "127.0.0.1:3998:/mul":[null,10],
  "127.0.0.1:3997:/mul":[null,10],
  "127.0.0.1:3999:/mul":[null,10]
}
```

in each array response, the first index is the error and the second one is the actual response.

In fact, if you close one of the math.ms nodes, you'll see the response to be:

Assuming tht you kill `math.ms@127.0.0.1:3998`:

```bash
response of mul => {
  "127.0.0.1:4000:/mul":[null,10],
  "127.0.0.1:3998:/mul":[
    {"code":"ECONNREFUSED","errno":"ECONNREFUSED","syscall":"connect","address":"127.0.0.1","port":5001},
    null
  ],
  "127.0.0.1:3997:/mul":[null,10],
  "127.0.0.1:3999:/mul":[null,10]
}
```

> Note that each xyz node will kick out-of-reach nodes after a certain threshold. You can learn more about it in the [Ping Types]().

As you see, the first parameter is the error and the second parameter is the response. This happens because each node will not immediately assume that an other node is offline after one failure, so it will keep sending the message to it. If you wait for 10 ping, you will see that all nodes will eventually update their `systemConf` and will not send any message to this node.

Another interesting send strategy that you might want to implement and test is majority function. You can change the `send.to.all` in a way that it will send the message to all accpetable nodes, will wait for all responses, and will return with a single result, if all if the responses from different services were the same. If not, it will return an Error. Who knows, maybe you actually use this Microservice to do floating point calculations. Each machine handles floating point differently and the responses might actually differ!

Note that you can also pass the send strategy ***per call***. This will override whatever send strategy you have already chosen in `selfConf`:


{%highlight javascript%}
let sendToAll = require('xyz-core/src/Service/Middleware/service.send.to.all')
let firstFind = require('xyz-core/src/Service/Middleware/service.first.find')
setInterval(() => {
  stringMS.call({servicePath: 'mul', payload: {x: 2, y: 5}, sendStrategy: sendToAll}, (err, body, res) => {
    console.log(`response of mul => ${JSON.stringify(body)} [err ${err}]`)
  })

  stringMS.call({servicePath: 'mul', payload: {x: 2, y: 5}, sendStrategy: firstFind}, (err, body, res) => {
    console.log(`response of mul => ${JSON.stringify(body)} [err ${err}]`)
  })
}, 1000)

{%endhighlight%}

# Path based service identification

Aside from Service discovery middlewares, xyz provides yet another way to make service invocation even more flexible. It's called **Path Base** service identification. That is to say, each service (aka. function) that you expose, will be exposed on a certain path on that process. So far we have used a plain string as the service name. The truth is that those were paths too! XYZ has been adding a single `/` to each of them. Recall one of the logs printed from the previous section. Look at it again:

```
  "127.0.0.1:4000:/mul":[null,10]}
```

As you see, the actual **identifier of the service** is actually: **[IP]:[PORT]:/[PATH]**.

Recalling from the last section, The `[IP]:[PORT]` portion of this identifier must be discovered and decided upon in the **sending side**, the caller. Once the IP and PORT has been resolved using a send strategy, the message will be sent to that node. The last part of the address, the **:/[PATH]**, will be used at the **receiving side**, which help the callee decide whihc of its exposed functions to invoke.

Of course, this does not separate these two identifiers entirely. As an example, in `first.find`, the sender must be aware of the **PATH** of services exposed by other nodes, while `send.to.target` will send a message to a specific **IP:PORT** regardless of the **PATH**.

As a naive example, let's assume that our math service is super accurate and can add floating point numbers up to thousands digits of accuracy. But we don't want to waste resource on simple integer calculations. Hence, we will divide our services into to section:

```
decimal/add
decimal/mul

float/add
float/mul
```

Applying this turns out to be very easy!

Change the first parameter, the _Path_, of each function is math.ms:

```
mathMS.register('decimal/mul', (payload, response) => {
  response.send(payload.x * payload.y)
})

mathMS.register('decimal/add', (payload, response) => {
  response.send(payload.x + payload.y)
})

mathMS.register('float/mul', (payload, response) => {
  // let's add the float casting, just fore sake of our example!
  response.send(parseFloat(payload.x * payload.y))
})

mathMS.register('float/add', (payload, response) => {
  response.send(parseFloat(payload.x + payload.y))
})
```

If you run the stringMS now, it will get only `null`. This is natural because it is looking for `/mul`, while it only knows `/decimal/mul`.

Next, change the call path in stringMS:

```
setInterval(() => {
  stringMS.call({servicePath: '/decimal/mul', payload: {x: 2, y: 5}}, (err, body, res) => {
    console.log(`my fellow service responded with ${JSON.stringify(body)}`)
  })
}, 2000)
```

As you see, now we get the same result.

The main purpose of paths is to allow:

  1. better grouping and organization among services.
  2. allow more meaningful names and identification
  3. allow multiple calls at once -*Wildcard*-:

Let's see all of these in action! We will use the same configuration like the previous section.

First, let's have string.ms make two message calls:

{% highlight javascript %}
setInterval(() => {
  stringMS.call({
    servicePath: '/decimal/mul',
    payload: {x: 2, y: 5},
    sendStrategy: firstFind
  },
  (err, body, res) => {
    console.log(`/decimal/mul [firstFind] => ${body} [err ${err}]`)
  })

  stringMS.call({
    servicePath: '/decimal/*',
    payload: {x: 2, y: 5},
    sendStrategy: sendToAll
  },
  (err, body, res) => {
    console.log(`/decimal/* [sendToAll]=> ${JSON.stringify(body)} [err ${err}]`)
  })
}, 1000)
{% endhighlight %}

Before testing this, let's try and understand how it is going to work:

The first call is lucid. we try to find `/decimal/mul` in a node using `firstFind` which will eventually find one node that can respond to this.

But in the second case, you should know somethings beforehand. Specifically talking about `sendToAll`, this sendStrategy will take into consider both the **IP:PORT** and **PATH**. In other words, it will first resolve the path, if it contains `*`. Next it will iterate over nodes and find those who can respond to a specific path.

Taking this into account, in the second case, `/decimal/*` will first translate into [`decimal/mul`, `decimal/add`]. Next, all possible target nodes for each of these paths will be resolved and the message will be sent to all of them.

With these explanations in mind, the output of the previous  double call is:

```
/decimal/mul [firstFind] => 10 [err null]
/decimal/* [sendToAll]=> {
  "127.0.0.1:4000:/decimal/mul":[null,10],
  "127.0.0.1:4000:/decimal/add":[null,7]}
```

> Note that this example is working only because we are using `service.send.to.all` as send strategy is second call. `first.find` would simple call either one of `/decimal/add` or `/decimal/mul` in only one of possible nodes

Go ahead and try things like `*/mul` or even `/*/*`.

It would be helpful to mix the this section and the last one. Go ahead and run a few math.ms nodes and a single string.ms node with `sendToAll` as sendStrategy and see the output.


#### Note on valid Paths and wildcards

Currently, XYZ path parser only supports characters and wildcards in paths. That is to say, when creating a new service, the valid regex is:

```
/^(\/([a-zA-Z]|[1-9]|\*)+)*$/
```

and when calling a service, you can only call definite paths or full-wildcards. `abc/*/abc` is valid, albeit `abc/ab*/abc` is not. This is a work in progress and will change in the upcoming version.

# Middlewares

Middlewares are nothing new to you. You have already seen them. `send.to.all` was a simple middleware in xyz system. In this section we are going to get into the details of middleware and use them to develop an authentication mechanisms.

### Middlewares and Layers in XYZ

So far, our system has been plain open! No authentication. That is not good. The good news is that XYZ does not provide any authentication mechanisms by default. You heard me, **No authentication mechanisms**. As opposed to application-dependent steps like authentication which are not compulsory in xyz, **middlewares** are. That is to say, whatever happens, requests will go through a middleware in xyz.

In order to understand the place and organization of middleware in xyz, you should get a bit more familiar with it.

> XYZ consists of 3 main layers. `xyz-core`, `Service-Repository` and `Transport`. The pipeline that will communicate a message from one layer to another is a **middleware**.

### Middleware Stacks

Middlewares are implemented using the [GenericMiddlewareHandler](/apidoc/generic.middleware.handler.html) Class in XYZ. this class will register an array of functions and passes an object (aka. parameters) through this array of middlewares.

```
MiddlewareStack : [Function, Function, Function, ... , Function]
```

As an example, Whenever a request is received and parsed in Transport Layer, a middleware function will pass it to the Service repository layer. This action will happen in the last function placed in the last index of this middleware in Transport layer.

This enables us to prepend / append more functions to this array and perform any type of pre/post process on every request. After seeing the syntax structure of each middleware in the next section, we will actually use this to add a generic authentication middleware to our system.


### Middleware Stacks in XYZ

The overall structure of xyz, in the simplest form can be explained like this:

![xyz-arch]({{site.baseurl}}/assets/img/arch-c.png)

Although it is good for you to have a glimpse of some components like multiple servers and routes, this image is exhustively complicated for this tutorial. Let's relax this into something simpler:

![xyz-arch-simple]({{site.baseurl}}/assets/img/arch-s.png)

> Due to the fact that critical actions such as passing a request object to the upper/lower layer or passing a request out to the HTTP layer happen in middleware, it is crucially important to be very careful with them. Specifically, some middlewares are placed in xyz by **default**. They should remain in their place, unless you have a good reason to remove them.

### Middleware syntax

As mentioned above, each middleware is nothin but a function. In order to examine the parameters of this function, let's write a bare-bone middleware.

The key structure of a middleware is as follows:

  - A middleware should be exported as a function
  - The function will take three parameters:
    - `params`: an array of parameters values passed to the middleware function.
    - `next`: function that will ignore the rest of the execution of this middleware and invoke the next function in the stack.
    - `end`: will end of the execution of the entire stack.
    - `xyz`: a reference to the current xyz object. This is useful because all of the information required, such as configurations, Service-Repository layer etc. can be accesses from this object.

Keeping this ideas in mind, let's write a simple logger middleware:

```
let dummyLogger = function(params, next, end) {
  console.log('i was called! now what?')
  console.log(params[0])
  console.log(params[1])
  next()
}
module.exports = dummyLogger;
```

This is the minimum structure required for each middleware.

Next important step is to know what is the content of `params`.

Each middleware in xyz has specific params structure. As an example, The transport layer call dispatch middleware has the following parameters according to the [source code](https://github.com/node-xyz/xyz-core/blob/master/src/Transport/HTTP/http.client.js#L35): `[requestConfig, callResponseCallback]`. That means that the first index is a variable containing request information and the second parameter is the callback function for this call.

> Remember how you call something like `someMS.call({servicePath: 'somePath'}, ()=> {})`? The second argument to this call is the actual callback function that will be passed to the Transport layer, and you have access to it inside the middleware. This is because the last index of this middleware stack is a function [that will wrap this request in an http object and pass it to the network](). Therefore, it needs the callback function to invoke it when the response is back. Note that this middleware is embedded in xyz and it is not on a separate repository, unlike other middlewares.

Next, let us see how we can insert this dummy logger to the system. The XYZ interface provides methods for inserting a function to a specific index of a specific MiddlewareStack. As an example, we can insert this middleware in StringMS (continuing from the previous section):

```
// StringMS
stringMS.middlewares().transport.callDispatch.register(0, require('./dummy.logger'))

// stringMS.register() ...
```

note that in this function, the first parameter is the index in middleware stack. This call will insert the new function to the first index and will push all other functions to the next index. If you want to remove a specific middleware, [use](https://node-xyz.github.io/apidoc/GenericMiddlewareHandler.html)`.remove(idx)`.

Check your logs after running the system in any desired configuration, one StringMS and one MathMS for sake of simplicity:

```
[Function: bound ]
i was called! now what?
{ hostname: '127.0.0.1',
  port: '3333',
  path: '/call',
  method: 'POST',
  json: { userPayload: { x: 2, y: 5 }, service: '/float/mul' } }
[Function: bound ]
my fellow service responded with {"127.0.0.1:3333:/decimal/mul":[null,10],"127.0.0.1:3333:/float/mul":[null,10]}
```

As you see, our new middleware was invoked before the response was sent.

The parameters are also now clear. Note that except the payload, all of these parameters will be used in the last middleware to create and send a HTTP request. Altering them will indeed have consequences. Albeit, not all alterations are harmful. As an example, consider the following scenarios:

  - Applying a pre/post-processing to all request
  - logging systems
  - validation (note that by calling `end()`, the request will be dismissed)
  - reusability of middlewares
  - more decoupling: means that not only that your application logic is separated from your xyz service code, your configurations, which are implemented in the middlewares are also decoupled from the xyz service code.

  Note that the `json` key is a generic place to put data in it. But, only the `userPayload` will be available to the receiver.

  - last but not least! authentication. Although this is analogous to validation, we are going to mention it separately since we are going to implement one in the next section:

### An authentication middleware:

Suppose that your system uses a very simple  authentication system, kind of a shared-secret mechanism. You want authentication to happen in Transport layer. This is generally better because the Transport layer can drop unauthorized requests immediately and the Service-Repository layer, which is more critical, won't be bothered to check them twice.

We are going to place two middlewares in **Transport layer** for this purpose, namely in **CallReceiveMiddlewareStack** and **CallDispatchMiddlewareStack**.

The code of these middlewares should be fairly understandable to you now. The sender side is fairly (perhaps more than *fairly*) simple:

```
const SECRET = 'SECRET'

let authSend = function (params, next, end) {
  params[0].json.authorization = SECRET
  next()
}

module.exports = authSend;
```

The only point to notice in the receiving side is that the params are a bit different. This is natureal because the middleware stack that will be invoked is directly getting its values and params from **Node HTTP module**. Hence, the params are similar to node networking and are as follows:

  - **params[0]** the request object. Similiar to node's [IncomingMessage]()
  - **params[1]** the response object. The wrapper used to send a response to the sender.
  - **params[2]** body. The payload of the request. Note that this is the entire `json` key which was available in the sending side. Later, another middleware will only pass the userPayload section of this object the service repository and the service repository will pass it to the user. Thus, the authorization key that we added in the last section is entirely hidden from the application code.

Hence, the receiving side of the middleware is similar to this:

```
const SECRET = 'SECRET'

let authReceive = function (params, next, end) {
  let authorization = params[2].authorization

  if (authorization === SECRET) {
    console.log('auth accepted')
    next()
  }else {
    console.log('auth failed')
    // it's better to also close the request immediately
    params[0].destroy()
    end()
  }
}

module.exports = authReceive

```

Similarly, we can insert these middlewares into the system using:

```
// stringMS
stringMS.middlewares().transport.callDispatch.register(0, require('./auth.send'))
stringMS.middlewares().transport.callReceive.register(0, require('./auth.receive'))

// mathMS
mathMS.middlewares().transport.callDispatch.register(0, require('./auth.send'))
mathMS.middlewares().transport.callReceive.register(0, require('./auth.receive'))
```

Again, note that what you add to the `json` key is not available to the application layer. That is to say, you do not need to explicitly delete it from the object. Add a log to the receiving function and and double check this.


### a pretty-print utility:

XYZ overrides the default `console.log` behavior. So when you print an xyz instance, such as `stringMS`, you'll get something like this:

```
selfConfig:
  {
  "name": "stringMS",
  "defaultSendStrategy": "xyz.service.send.first.find",
  "allowJoin": false,
  "logLevel": "info",
  "seed": [
    "127.0.0.1:3333"
  ],
  "port": 3334,
  "host": "127.0.0.1",
  "intervals": {
    "reconnect": 2500
  },
  "defaultBootstrap": true,
  "cli": {
    "enable": false,
    "stdio": "console"
  }
}
systemConf:
  {
  "nodes": [
    "127.0.0.1:3334"
  ]
}
____________________  SERVICE REPOSITORY ____________________

Middlewares:
  callDispatchMiddlewareStack || firstFind[0]
Services:
  anonymousFN @ /up
  anonymousFN @ /down

____________________  TRANSPORT LAYER ____________________
Transport Client:
  Middlewares:
  callDispatchMiddlewareStack || authSend[0] -> callDispatchExport[1]
  pingDispatchMiddlewareStack || pingDispatchExport[0]
  joinDispatchMiddlewareStack || joinExport[0]

Transport Server:
  Middlewares:
  callReceiveMiddlewareStack || authReceive[0] -> passToRepo[1]
  pingReceiveMiddlewareStack || passToRepo[0]
  joinReceiveMiddlewareStack || joinAcceptAll[0]
```

You can clearly see the stack of middlewares and how they are places.
