---
layout: doc-page
title: Getting Started with Node XYZ
breadcrumb: getting-started
---

#### Contents

- [Hello XYZ](/documentations/getting-started#hello-xyz)
- [Replicate your services](/documentations/getting-started#replicating-your-nodes-and-services)
- [Service Discovery](/documentations/getting-started#service-discovery)
- [Path based services](/documentations/getting-started#path-based-service-identification)
- [Middlewares](/documentations/getting-started#middlewares)

# Hello XYZ

This brief tutorial will help you fathom the concepts of XYZ, and help you get started with it.

### Baby Microservice

Assume that we only have two services that will communicate with each other. Suppose that each of these services (aka Node modules) have another API exposed two end users. That is not our business at the moment, since as mentioned before, XYZ will divide the matter of end user and the matter of system. Now, our goal is to focus on what is important to our own system.

The -- dummy services are as follows:
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
    // some ultra hard string manipulations
    function up(str) {
      return s.toUpperCase()
    }
    function down(str) {
      return s.toLowerCase()
    }
```
Now, assume that these services need to call each other in order to share their services. This is the first place that XYZ will come to action.

In order to have two xyz instances communicate with each other, they need to have a system, a **shared** system. Information about the system is filled to the xyz instance using a config parameter. The config parameter has information such as local port/ip/serviceName and a list of nodes who live inside the system. Note that a node can join the system without being in the static list, this is just a basic example!

    // math.ms.js
    // a very fancy math module with super fast calculation
    let xyz = require('xyz-core')
    let mathMS = new xyz({
      selfConf: {
        name: 'MathMS',
        host: '127.0.0.1',
        port: 3333
      },
      systemConf: {
        nodes: ['127.0.0.1:3334']
      }
    })

    // expose these function over the system
    mathMS.register('mul', (payload, response) => {
      response.send(payload.x * payload.y)
    })

    // expose these function over the system
    mathMS.register('add', (payload, response) => {
      response.send(payload.x + payload.y)
    })

    // string.ms.js
    // some ultra hard string manipulations
    let xyz = require('xyz-core')
    let stringMS = new xyz({
      selfConf: {
        name: 'stringMS',
        host: '127.0.0.1',
        port: 3334
      },
      systemConf: {
        nodes: ['127.0.0.1:3333']
      }
    })

    stringMS.register('up', (payload, response) => {
      response.send(payload.toUpperCase())
    })
    stringMS.register('down', (payload, response) => {
      response.send(payload.toLowerCase())
    })

> The entire process of initializing a system and passing all of the parameters seems too much of a duplicate, and tedious. We agree! check out the best practices section#common.js

And that's it! Now, our services are virtually binded to one another and they can call the service that each of them exposed using `.register`. The `nodes: [...]` key in **systemConf** will declare the list of other nodes that we can connect to. Obviously, they can be local or remote IPs.

#### A Note on terminology.

Words and phrases in XYZ system are explained in Wiki page. Just to recap, each function exposed via `.register` is a _service_.
Each **Node Process**, having a number of services is called a **node**.

---

Back to our little system. Now, one of these system can call a service in the other system using the `.call`

    //stringMS
    // note that these callback styles are inherited from node's native http module.
    // `resp` is the IncomingMessage instance etc.
    // https://nodejs.org/api/http.html#http_class_http_incomingmessage

    stringMS.call({servicePath: 'mul', payload: {x: 2, y:5}}, (err, body, res) => {
      console.log(`my fellow service responded with ${body}`)
    })


You should now see an output like:
`my fellow service responded with 10`

You can find the code of this section [here](https://github.com/node-xyz/xyz-core/tree/master/examples/examples_getting_started/helloxyz).

### What's next?

In this example we hardcoded the list of service, so that they can communicate with each other. This is... just about ok. It is not awesome. What would be awesome is to have nodes joining the system dynamically. Yes, that's what we're going to do in the Next Section.

----

# Replicating Your Nodes and Services

It would have been much more interesting, if we could add nodes dynamically. Good news: we can.

XYZ nodes can join a system, given that they have a [list of] seed node(s). Seed nodes are the entry point to the system. They should check the authority of the incoming node, if required, and pass the list of nodes available inside the system to it, or any other authentication certificate, again, if required. The fact that there are a lots of ' _if required_ ' phrases in the statement is that all of these steps are optional and the developer can choose to have them. In the most **_Wildcard_ish** scenario, all nodes are public and anyone can join them.

Let's assume that both the _StringMS_ and _MathMS_ in the previous section do not know each other. That is to say, their list of microservice/nodes in _systemConf_ is empty:

    // string.ms.js
    let xyz = require('xyz-core')
    let stringMS = new xyz({
      selfConf: {
        name: 'stringMS',
        host: '127.0.0.1',
        port: 3334
      },
      systemConf: {
        nodes: []
      }
    })

    // math.ms.js
    let xyz = require('xyz-core')
    let mathMS = new xyz({
      selfConf: {
        name: 'MathMS',
        host: '127.0.0.1',
        port: 3333
      },
      systemConf: {
        nodes: []
      }
    })

> Note that you can omit putting the empty `nodes: []` in your config file. XYZ will populate each key in the config object according to a template. You can see it [here](/apidoc/Constants.html).

Furthermore, let's assume that the MathMS is the constant node in the system and stringMSs are going to come and go. Go ahead and run the mathMS alone, you will see that nothing happens.

Now run the stringMS. you'll see:

`warn :: Sending a message to /mul from first find strategy failed (Local Response)
my fellow service responded with null`

The [first find strategy](https://github.com/node-xyz/xyz.service.send.first.find) is a default behavior embedded inside node XYZ for finding nodes to send messages to. We'll see more on this later. The main point is that the message failed. Obviously because StringMS does not know any mathMS at this point!

Change the stringMS to the following, specifying a new **seed node** for it. Also add a filed in mathMS that will indicate that this node is allowed to admit new nodes:

    // string.ms.js
    let xyz = require('xyz-core')
    let stringMS = new xyz({
      selfConf: {
        name: 'stringMS',
        host: '127.0.0.1',
        port: 3334,
        seed: ['127.0.0.1:3333']
      },
      systemConf: {
        nodes: []
      }
    })

    // math.ms.js
    let xyz = require('xyz-core')
    let mathMS = new xyz({
      selfConf: {
        allowJoin: true,
        name: 'MathMS',
        host: '127.0.0.1',
        port: 3333
      },
      systemConf: {
        nodes: []
      }
    })

> note that when you want to use a node as seed, that node MUST have `allowJoin` key set to true.

Now, the two nodes have no knowledge of each other, yet they can join and work with each other. Go ahead and run the MathMS, next run the stringMS in a separate terminal and see the results. As you see, the output is still the same.

This get's even more interesting when we add more nodes. You can add more stringMS nodes and see that all of them will eventually find the MathMS and can connect with it. There is only one small problem. All stringMSs are configured to use one port. Hopefully, XYZ provides a way to override any configuration, for debugging purposes mainly. We'll cover this later, but, for now, you can use the `--xyz-port` command line argument to override the port. Run many many more StringMS instances with:

    $ node math.ms.js --xyz-port 5000
    $ node math.ms.js --xyz-port 5001
    $ node math.ms.js --xyz-port 5002
    ...

A question might come to mind at this point. We have many sender nodes, and only just one receiver node in this example, so... it is kind obvious how each request gets routed. Assume that we have two receiver nodes (mathMS) and a dozen sender nodes, how could we decide on one of the mathMSs?

You might argue that this is a matter that xyz should handle. Indeed, we do handle it. it's just that we are very keen to make you familiarized whit what's happening under the hood, so that you can change it when required.

In order to have different message routing ways, xyz supports two mechanisms:

  1. Service Discovery middlewares
  2. Path based service identification

The upcoming sections will describe these concepts.

Before going into the next section, you might argue that why is this so important? In a microservice-based system, it is crucially important!

If you think about the system as a whole, you come to realize that many many types of messages might be sent. One node might want to broadcast a message to **all other nodes**, or to a subset of them. A node might want to apprise just a few database nodes about a record update, and many many more events that might happen!

If you are not convinced about this, I highly recommend you to read the [Wiki](https://github.com/node-xyz/xyz-core/wiki/) page of xyz-core, specially the one about [microservices](https://github.com/node-xyz/xyz-core/wiki/Microservices%3F-What%3F).

You can find the code of this section [here](https://github.com/node-xyz/xyz-core/tree/master/examples/examples_getting_started/replicating)

# Service Discovery

As mentioned in the previous section, the process of choosing a destination node when making a call is rather difficult and obscure. XYZ provides a configuration for each of the nodes, so that each node can actually choose a strategy for sending a message.

The good thing about these plugins is that they are completely modifiable. That is to say, you can write your own strategy and patch it into your system.

XYZ currently has two trivial approaches implemented as plugins. Let's look at one of them [here](https://github.com/node-xyz/xyz.service.send.first.find/blob/master/call.middleware.first.find.js). Let's look at a pseudo-code:

```javascript
let http = require('http')

function firstFind (params, next, done, xyz) {
  let servicePath = params[0],
    userPayload = params[1],
    responseCallback = params[2]

    foreignNodes = xyz.serviceRepository.foreignNodes,
    transportClient = xyz.serviceRepository.transportClient


  let logger = xyz.logger
  let wrapper = xyz.Util.wrapper
  let Path = xyz.path

  let serviceTokens = servicePath.split('/')

  for (let node in foreignNodes) {
    matches = Path.match(servicePath, foreignNodes[node])
    logger.silly(`FIRST FIND :: determined matches ${matches} in node ${node} for ${servicePath}`)
    if (matches.length) {
      logger.verbose(`${wrapper('bold', 'FIRST FIND')} :: determined node for service ${wrapper('bold', servicePath)} by first find strategy : ${wrapper('bold', node + ':' + matches[0]) }`)
      transportClient.send(matches[0], node , userPayload, responseCallback)
      done()
      return
    }
  }

  // if no node matched
  logger.warn(`Sending a message to ${servicePath} from first find strategy failed (Local Response)`)
  if (responseCallback) {
    responseCallback(http.STATUS_CODES[404], null, null)
    done()
    return
  }
}

module.exports = firstFind
```

We really don't want to go too deep inside this code at the time, but you can get a general feeling about it that a `Path` object will eventually find a list of nodes that can response to a certain `servicePath` (like `/mul`) and calls the first one:

`transportClient.send(matches[0], node , userPayload, responseCallback)`

whenever you call `someMs.call({servicePath:'foo'} , ()=>{} )` in your code, somewhere along the path, this function will be called and it will find one destination node and invokes a transport client with it. Transport client is the actual module that will make the HTTP/TCP call to destination node.

> This structure, a function with parameters passed as array, with `next` and `done` callbacks is actually the **middleware** signature of XYZ. We'll learn soon about middleware in this tutorial.

> Change your string ms to call an invalid service, say, `.call({servicePath: 'blahblah'}, () => {})`. Now look at the warning logs. Now look at the end of this plugin :
`logger.warn("Sending a message to ${servicePath} from first find strategy failed (Local Response)")`
Are you seeing what I am seeing too?

XYZ is configured to work with this module by default. if you wish to change them, a new key named **`defaultSendStrategy`** should be added to the **selfConf** key passed to xyz constructor.

In order to test this feature, let's use another send strategy, **send.to.all**. You can find the plugin [here](https://github.com/node-xyz/xyz.service.send.to.all).

First, include this plugin inside your working directory:

```
npm install xyz.service.send.to.all
```

Next, we are going to run three math services, and a dozen of string services. The configurations are similar to the previous section. We only have to configure string.ms to use `xyz.service.send.to.all` :

```
// math.ms.js
// same as before


// string.ms.js
let sendToAll = require('xyz.service.send.to.all')

let stringMS = new xyz({
  defaultSendStrategy: sendToAll,
  selfConf: {
    name: 'stringMS',
    host: '127.0.0.1',
    port: 3334,
    seed: ['127.0.0.1:13333']
  },
  systemConf: {
    node: []
  }
})

...

// our responses are going to be from few possible nodes, hence they are objects from now on!
// use JSON.stringify instead.
setInterval(() => {
  stringMS.call({servicePath:'mul', payload: {x: 2, y: 5}}, (err, body, res) => {
    console.log(`my fellow service responded with ${JSON.stringify(body)}`)
  })
}, 2000)

```

> Note that you can either `require()` the `xyz.service.send.to.all` or you can pass it as string: `defaultSendStrategy: 'xyz.service.send.to.all'`

First, run the math ms and string ms like before. The string ms will join the math ms. First point is that now, our responses are different, it has been indicated the this values has been responded form which node.

```
my fellow service responded with {"127.0.0.1:3333:/mul":[null,10]}
```

in each array response, the first index is the error and the second one is the actual response.

Now go ahead and run a new math.ms. An old response happens here again. We need the new instance of Math ms to have a seed node, yet, we didn't include it in it's configuration.

Similar to how we can override the port number, a single seed node can also be added by command line argument. Run the new math Ms :

```
node math.ms.js --xyz-port 5000 --xyz-seed 127.0.0.1:3333
node math.ms.js --xyz-port 5001 --xyz-seed 127.0.0.1:3333
```

after just a few seconds, the log of string ms should change to

```
my fellow service responded with {
    "127.0.0.1:3333:/mul":[null,10],
    "127.0.0.1:5000:/mul":[null,10],
    "127.0.0.1:5001:/mul":[null,10]
}
```

In fact, if you close one of the mathMSs, you'll see the response to be:

```
my fellow service responded with {
  "127.0.0.1:5001:/mul":[
    {"code":"ECONNREFUSED","errno":"ECONNREFUSED","syscall":"connect","address":"127.0.0.1","port":5001},
    null
  ],
  "127.0.0.1:3333:/mul":[null,10],
  "127.0.0.1:5000:/mul":[null,10]}
```

> Note that each xyz node will kick out-of-reach nodes after a certain threshold. This is configured in the ping.default.bootstrap function. You can learn more about it in the section regarding Ping.

As you see, the first parameter is the error and the second parameter is the response. This happens because each node will not immediately assume that another node is offline after one failure. The process and parameters of node failure will be discussed later.

So ... are you not liking the way that send to all is functioning? I mean, who would respond with an array like that? This is actually what we hoped for!  this plugin is just a test plugin that we used in our development. You can easily change it to whatever you like.

For example, you can modify [this line](https://github.com/node-xyz/xyz.service.send.to.all/blob/master/call.send.to.all.js#L31) to get rid of the annoying errors or...

Another interesting send strategy that you might want to implement and test is majority function. You can change the `send.to.all` in a way that it will send the message to all nodes, will wait for all responses, and will return with a single result, if all if the responses from different services were the same. If not, it will return an Error. Who knows, maybe you actually use this Microservice to do floating point calculations. Each machine handles floating point differently and the responses might actually differ!

You can find the code of this section [here](https://github.com/node-xyz/xyz-core/tree/master/examples/examples_getting_started/service-discovery).

Note that you can also pass the send strategy ***per call***. This will override whatever send strategy you have already chosen in `selfConf`:

```
stringMS.call({servicePath: 'mul', payload: {x: 2, y: 5}, sendStrategy: sendToAll}, (err, body, res) => {
    console.log(`my fellwo service reponded with ${body}`)
})
```


# Path based service identification

Aside from Service discovery procedures, xyz provides another way to make service invocation even more flexible. It's called **Path Base** service identification. That is to say, each service (aka. function) that you expose, will be exposed on a certain path on that process. So far we have used a plain string as the service name. The truth is that those were paths too! XYZ has been adding a single `/` to each of them. Recall one of the logs printed from the previous section. Look at it again:

```
  "127.0.0.1:5000:/mul":[null,10]}
```

As you see, the actual identifier of the service is actually: **[IP]:[PORT]:/[PATH]**.

As a naive example, let's assume that our math service is super accurate and can add floating point numbers up to thousands of decimal points. But we don't want to waste resource on simple integer calculations. Hence, we will divide our services into to section:

```
decimal/add
decimal/mul

float/add
float/add
```

Applying this turns out to be very easy!

Change the address of each function is mathMS:

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
  3. allow grouping call:

That's right. You can also use the combination of `xyz.call.send.to.all` plugin and paths to call multiple services:

```
setInterval(() => {
  stringMS.call({servicePath: '/decimal/*', payloa: {x: 2, y: 5}}, (err, body, res) => {
    console.log(`my fellow service responded with ${JSON.stringify(body)}`)
  })
}, 2000)
```

Now our response is:

```
my fellow service responded with {
  "127.0.0.1:3333:/decimal/mul":[null,10],
  "127.0.0.1:3333:/decimal/add":[null,7]
}
```

> Note that this example is working only because we are using `xyz.call.send.to.all` as send strategy is StringMS.

Go ahead and try things like `*/mul` or even `/*/*`.


#### Note on valid Paths and wildcards

Currently, XYZ path parser only supports characters and wildcards in paths. That is to say, when creating a new service, the valid regex is:

```
/^(\/([a-zA-Z]|[1-9]|\*)+)*$/
```

and when calling a service, you can only call definite paths or full-wildcards. `abc/*/abc` is valid, albeit `abc/ab*/abc` is not. This is a work in progress and will change in the upcoming version.

# Middlewares

Middlewares are nothing new to you. You have already seen them. `xyz.service.send.to.all` was a simple middleware in xyz system. In this section we are going to get into the details of middleware and use them to develop an authentication mechanisms.

### Middlewares and Layers in XYZ

So far, our system has been plain open! No authentication. That is not good. The good news is that XYZ does not provide any authentication mechanisms by default. You heard me, **No authentication mechanisms**. As opposed to application-dependent steps like authentication which are not compulsory in xyz, middlewares are. That is to say, whatever happens, requests will go through a middleware in xyz.

In order to understand the place and organization of middleware in xyz, you should get a bit more familiar with it. A brief summery of the architecture of xyz is proivded in the [wiki page of xyz-core](https://github.com/node-xyz/xyz-core/wiki/XYZ-in-a-nutshell#the-close-up-picture). before going any further, read it carefully.

As you return to this page, you should have a superficial knowledge about how xyz middlewares works and where they are. Just to recap:

> XYZ consists of 3 main layers. `xyz-core`, `Service-Repository` and `Transport`. The pipeline that will communicate a message from one layer to another is a **middleware**.

Furthermore,

> Three main middlewares exist in xyz. Two of which are invoked when a call is being sent and one when a call is being received:
  - Service Level CallDispatchMiddlewareStack
  - Transport Layer CallDispatchMiddlewareStack
  - Transport Layer CallReceiveMiddlewareStack

As a final recap:

![Overall structure of XYZ](https://raw.githubusercontent.com/node-xyz/xyz-core/master/media/mw.png)

> Bad news is that there are a few more middlewares inside a xyz system. But you don't need to know about them now :)).

### Middleware Stacks

Middlewares are implemented using the [GenericMiddlewareHandler](/apidoc/generic.middleware.handler.html) Class in XYZ. this class will register an array of functions and passes an object (aka. parameters) through this array of middlewares.

```
MiddlewareStack : [Function, Function, Function, ... , Function]
```

As an example, Whenever a request is received and parsed in Transport Layer, a middleware function will pass it to the Service repository layer. This action will happen in the last function placed in the last index of CallReceiveMiddlewareStack in Transport layer.

> Due to the fact that critical actions such as passing a request object to the upper/lower layer or passing a request out to the HTTP layer happen in middleware, it is crucially important to be very careful with them. Specifically, some middlewares are placed in xyz by **default**. They should remain in their place, unless you have a good reason to remove them.

As an example, as mentioned above, the middleware placed between Transport layer and Service-Repository layer has such initial state:

```
// Initial state of CallReceiveMiddlewareStack
[Funciton():passToTransportLayer]
```

This enables us to prepend more functions to this array and perform any type of process. After seeing the syntax structure of each middleware in the next section, we will actually use this to add a generic authentication middleware to our system.

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
