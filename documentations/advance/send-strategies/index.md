---
layout: doc-page
title: Send Strategies
breadcrumb: send-strategies
---

In this document, we're going to discuss **Send Strategies**. This particularly centers around different send Strategy middlewares. You will gain enough knowledge to be able to modify an existing middleware or even write a new one. Similar to other topics in our documentations, we will start by explaining high level concepts and then we will move into details by studying an existing send Strategy middleware.

# Send Strategy: The definition

> A middleware function takes a message as input and decides the target node or nodes of it is called a send strategy

Let's emphasize some points:

- Although not said explicitly, most send Strategies **should** make their decision based on **path of the target service**. As an example, [`sendToAll()`]() and [`firstFind()`]() will consider the path, whilst [`broadcastGlobal`]() does not.

- Since a send strategy is likely to make a decision base on path of the service, it is likely that it needs **a list of nodes and their services**. This concludes that the veracity of send strategies is dependent on [Ping Mechanism](/documentations/advance/ping-mechanisms), since Ping Mechanisms should keep track of other nodes and explore them.

- Send strategies will take a message, including its payload, servicePath and other information (basically everything that you pass to `.call()`) and should **invoke the Transport Layer**. They are free to call them once or more. As an example, [`firstFind()`]() will invoke the Transport layer exactly once, while [`broadcastGlobal`]() will invoke it multiple times.

> xyz's send strategies are based on **Service Paths**. This is the only place where xyz is independently introducing a new concept which is not part of the known microservice specification. Most literature about microservices suggest **pattern matching**, while xyz suggests **paths**.

In the next section we will see how these concepts are being applied in [`firstFind()`]().

# Case study: First Find

As you might already know, _First Find_ is a send strategy that matches the path that you provide in `.call()` which all possibilities and send the message to exactly one of the matches, namely the first one.

Send Strategies are **middlewares** in the service layer, hence they have the same parameter format as other middlewares. The definition of `firstFind` is:

{% highlight javascript %}
function firstFind (params, next, done, xyz) {
  let servicePath = params[0].servicePath
  let userPayload = params[0].payload
  let route = params[0].route
  let redirect = params[0].redirect

  let responseCallback = params[1]
}
{% endhighlight %}

where `params[]` is everything that you pass into `.call()`.

Other important variables that almost every send strategy function should use are:

{% highlight javascript %}
let foreignNodes = xyz.serviceRepository.foreignNodes
let transport = xyz.serviceRepository.transport
let Path = xyz.path
{% endhighlight %}

Where

- `foreignNodes` is the list of all nodes in the system and their service paths.
- `transport` is the Transport layer.
- `Path` is a utility object in xyz that provides some functions to match paths.

if we setup a simple node that sends a message to itself using `firstFind`, the `math.ms.js` for example,

{% highlight javascript %}
let XYZ = require('xyz-core')
let math = new XYZ({})

// register a dummy service
math.register('add', (payload, resp) => {
  resp.jsonify(payload.x + payload.y)
})

math.register('math/mul', (payload, resp) => {
  resp.jsonify(payload.x * payload.y)
})

setTimeout(() => {
  math.call({servicePath: 'add', payload: {x: 1, y: 7}}, (err, body) => {
    console.log(err, body)
  })
}, 2000)
{% endhighlight %}


and log these variables (you can find this function in `xyz-core/src/Service/Middlewares/service.first.find.js`)

{% highlight javascript %}
// first.find.js
let foreignNodes = xyz.serviceRepository.foreignNodes
let transport = xyz.serviceRepository.transport
let Path = xyz.path

console.log(foreignNodes)
console.log(Path)
{% endhighlight %}

you see:

{% highlight bash %}
// console.log(foreignNodes)
{
  "127.0.0.1:4000":{
    "":{                         /
      "add":{},                  /add
      "math":{                   /math
        "mul":{}                 /math/mul
      }
    }
  }
}
{% endhighlight %}

which is in the `serializedTree` format. This format is used by xyz's `Path` object to match function paths. The `""` at the begining means `"/"` and as we go deeper, you should be able to see how this object depicts `/add` and `/math/mul`.

In the `Path`, we see these functions.
{% highlight bash %}
// console.log(Path)
{ validate: [Function: validate],
  format: [Function: format],
  merge: [Function: merge],
  match: [Function: match],
  getTokes: [Function: getTokes] }
{% endhighlight %}

The most important function in `Path` is `match()`. This function takes a **Path String** and one **serializedTree** and output all of the possible matches. Let's add another log to the poor `firstFind` and see how this works.

{% highlight javascript %}
console.log('foreignNodes', JSON.stringify(foreignNodes, 4))
console.log('servicePath', servicePath)
console.log(Path.match(servicePath, foreignNodes['127.0.0.1:4000']))
{% endhighlight %}

The output is

{% highlight bash %}
[ '/add' ]
{% endhighlight %}

Which makes sense because we sent a message to `add`. `Path` class can also handle wildcards. Since we have a function registered at `/math/mul`, we can send a message to `/math/*`.

Change this line and test again

{% highlight javascript %}
setTimeout(() => {
  math.call({servicePath: '/math/*', payload: {x: 1, y: 7}}, (err, body) => {
    console.log(err, body)
  })
}, 2000)
{% endhighlight %}

and the log should be

{% highlight bash %}
foreignNodes {"127.0.0.1:4000":{"":{"add":{},"math":{"mul":{}}}}}
servicePath /math/*
[ '/math/mul' ]
{% endhighlight %}

Which means that `Path.match()` resolved `math/*` to `/math/mul`. Of course, you can use your own matching function too.

Let's goo back to the original `firstFind` and stop adding logs to it. This function will iterate over all `foreignNodes` and tries to match `servicePath` to one of them

{% highlight javascript %}
for (let node in foreignNodes) {
    matches = Path.match(servicePath, foreignNodes[node])
    if (matches.length) {
      ...
      // invoke the transport layer
      // return immediately and do not continue
      done()
      return
  }
{% endhighlight %}

and if no match is find after checking all of the nodes in `foreignNodes`,

{% highlight javascript %}
logger.warn(`Sending a message to ${servicePath} from first find strategy failed (Local Response)`)
  if (responseCallback) {
    responseCallback(http.STATUS_CODES[404], null, null)
    done()
    return
  }
{% endhighlight %}

which is something that you probably have seen at least once!

In fact, you will see it if you remove the `setTimeout()` in `math.ms.js`. Since the ping Mechanism takes a few milliseconds to explore the nodes, `add` or `math/mul` are unknown at the very beginign of execution and the output will be:

{% highlight bash %}
[node-xyz-init@127.0.0.1:4000] warn :: Sending a message to /math/* from first find strategy failed (Local Response)
Not Found null
{% endhighlight %}


# Built-in Strategies

xyz provides the following send strategies Built-in. All of them can be imported from `xyz-core/src/Service/Middleware/*`.

- **_service.broadcast.global_**: sends the message to all nodes, regardless of the path.
- **_service.broadcast.local_**: sends the message to all nodes with the same `selfConf.host`. It is useful for message queueing or local load balancing.
- **_service.send.to.all_**: sends the message to all nodes that have a service with a path that matches.
- **_service.first.find_**: sends the message the first node that have a service with a path that matches.
- **_service.send.to.target_**: sends the message to a given ip:port. It will ignore the path. should be used like this: `ms.call({..., sendStrategy: sendToTarget('X.Y.Z.P')})`. It can not be used as `selfConf.defaultSendStrategy`.
