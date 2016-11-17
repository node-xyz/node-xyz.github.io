---
layout: page
title: Getting Started with Node XYZ
permalink: /getting-started
---

# Hello XYZ

This brief tutorial will help you fathom the concepts of XYZ, and help you get started with it.

### Baby Microservice

Assume that we only have two services that will communicate with each other. Suppose that each of these services (aka Node modules) have another API exposed two end users. That is not our business at the moment, since as mentioned before, XYZ will divide the matter of end user and the matter of system. Now, our goal is to focus on what is important to our own system.

The -- dummy services are as follows:

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

Now, assume that these services need to call each other in order to share their services. This is the first place that XYZ will come to action.

In order to have two xyz instances communicate with each other, they need to have a system, a shared system. Information about the system is filled to the xyz instance using a config parameter. The config parameter has information such as local port/it/serviceName and a list of nodes who live inside the system. Note that a node can join the system without being in the static list, this is just a basic example!

    // math.ms.js
    // a very fancy math module with super fast calculation
    let xyz = require('xyz-core').xyz
    let mathMS = new xyz({
      selfConf: {
        name: 'MathMS',
        host: '127.0.0.1',
        port: 3333
      },
      systemConf: {
        microservices: [{
            host: '127.0.0.1',
            port: 3334
          }]
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
    let xyz = require('xyz-core').xyz
    let stringMS = new xyz({
      selfConf: {
        name: 'stringMS',
        host: '127.0.0.1',
        port: 3334
      },
      systemConf: {
        microservices: [{
            host: '127.0.0.1',
            port: 3333
          }]
      }
    })

    stringMS.register('up', (payload, response) => {
      response.send(payload.toUpperCase())
    })
    stringMS.register('down', (payload, response) => {
      response.send(payload.toLowerCase())
    })

> The entire process of initializing a system and passing all of the parameters seems too much of a duplicate. We agree! check out the best practices section#common.js

And that's it! Now, our services are virtually binded to one another and they can call the service that each of them exposed using `.register`. The `microservices: [...]` key in **systemConf** will declare the list of other nodes that we can connect to. Obviously, they can be local or remote IPs.

#### A Note on terminology.

Words and phrases in XYZ system are explained in Wiki page. Just to recap, each function exposed via `.register` is a _service_.  
Each **Node Process**, having a number of services is called a **node** (or in some places a _microservice_, but its just temporary).

Back to our little system. Now, one of these system can call a service in the other system using the `.call`

    //stringMS
    // note that these callback styles are inherited from node's native http module.
    // `resp` is the IncomingMessage instance etc.
    // https://nodejs.org/api/http.html#http_class_http_incomingmessage

    stringMS.call('mul', {x: 2, y:5}, (err, body, res) => {
      console.log(`my fellow service responded with ${body}`)
    })


You should now see an output like:  
`my fellow service responded with 10`

### What's next?

In this example we hardcoded the list of service, so that they can communicate with each other. This is... just about ok. It is not awesome. What would be awesome is to have nodes joining the system dynamically. Yes, that's what we're going to do in the Next Section.

----

# Replicating Your Nodes and Services

It would have been much more interesting, if we could add nodes dynamically. Good news: we can.

XYZ nodes can join a system, given that they have a [list of] seed node(s). Seed nodes are the entry point to the system. They should check the authority of the incoming node, if required, and pass the list of nodes available inside the system to it, or any other authentication certificate, again, if required. The fact that there are a lots of ' _if required_ ' phrases in the statement is that all of these steps are optional and the developer can choose to have them. In the most _Wildcard_is scenario, all nodes are public and anyone can join them.

Let's assume that both the _StringMS_ and _MathMS_ in the previous section do not know each other. That is to say, their list of microservice/nodes in _systemConf_ is empty:

    // string.ms.js
    let xyz = require('xyz-core').xyz
    let stringMS = new xyz({
      selfConf: {
        name: 'stringMS',
        host: '127.0.0.1',
        port: 3334
      },
      systemConf: {
        microservices: []
      }
    })

    // math.ms.js
    let xyz = require('xyz-core').xyz
    let mathMS = new xyz({
      selfConf: {
        name: 'MathMS',
        host: '127.0.0.1',
        port: 3333
      },
      systemConf: {
        microservices: []
      }
    })

Furthermore, let's assume that the MathMS is the constant node in the system and stringMSs are going to come and go. Go ahead and run the mathMS alone, you will see that nothing happens.

Now run the stringMS. you'll se that:  
`warn :: Sending a message to /mul from first find strategy failed (Local Response)  
my fellow service responded with null`

The [first find strategy](https://github.com/node-xyz/xyz.service.send.first.find) is a default behavior embedded inside node XYZ for finding nodes to send messages to. We'll see more on this later. The main point is that the message failed. Obviously because StringMS does not know any mathMS at this point!

Change the stringMS to the following, specifying a new seed node for it. Also add a filed in mathMS that will indicate that this node is allowed to admit new nodes:

    // string.ms.js
    let xyz = require('xyz-core').xyz
    let stringMS = new xyz({
      selfConf: {
        name: 'stringMS',
        host: '127.0.0.1',
        port: 3334,
        seed: [{host: '127.0.0.1', port: 3333}]
      },
      systemConf: {
        microservices: []
      }
    })

    // math.ms.js
    let xyz = require('xyz-core').xyz
    let mathMS = new xyz({
      selfConf: {
        allowJoin: true,
        name: 'MathMS',
        host: '127.0.0.1',
        port: 3333
      },
      systemConf: {
        microservices: []
      }
    })

Now, the two nodes have no knowledge of each other, yet they can join and work with each other. Go ahead and run the MathMS, next run the stringMS in a separate terminal and see the results. As you see, the output is still the same.

This get's even more interesting when we add more nodes. You can add more stringMS nodes and see that all of them will eventually find the MathMS and can connect with them. There is only one small problem. All stringMSs are configured to use one port. Hopefully, XYZ provides a way to override any configuration, for debugging purposes mainly. We'll cover this later, but, for now, you can use the `--xyzport` command line argument to override the port. Run many many more StringMS instances with:  

    $ node math.ms.js --xyzport 5000
    $ node math.ms.js --xyzport 5001
    $ node math.ms.js --xyzport 5002
    ...

If you are more interested to see how nodes actually work with each other, see the ()[Service Discovery and Ping] section.

A question might come to mind at this point. We have many sender nodes, and only just one receiver node in this example, so... it is kind obvious how each request gets routed. Assume that we have two receiver nodes (mathMS) and a dozen sender nodes, how could we decide on one of the mathMSs?

You might argue that this is a matter that xyz should handle. Indeed, we do handle it. it's just that we are very keen to make you familiarized whit what's happening under the hood, so that you can change it when required.

In order to have different message routing ways, xyz supports to mechanisms:

  1. Service Discovery middlewares
  2. Path based service identification

The upcoming sections will describe these concepts.


Before going into the next section, you might argue that why is this so important? In a microservice-based system, it is crucially important!

If you think about the system as a whole, you come to realize that many many types of messages might be sent. One node might want to broadcast a message to **all other nodes**, or to a subset of them. A node might want to apprise just a few database nodes about a record update, and many many more events that might happen!

If you are not convinced about this, I highlt recommend you to read the [Wiki](https://github.com/node-xyz/xyz-core/wiki/) page of xyz-core, specially the one about [microservices](https://github.com/node-xyz/xyz-core/wiki/Microservices%3F-What%3F).

[further explanation]

# Service Discovery middlewares



# Path based service identification
