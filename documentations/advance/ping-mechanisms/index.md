---
layout: doc-page
title: Ping Mechanism
breadcrumb: ping-mechanisms
---


In this section we are going to discuss ping mechanism. We are going to understand what they are and what role they play in a xyz node.
We will avoid getting too technical by avoiding most of the syntax detail, since it is not likely for most developers to write their own ping mechanism. Therefore, we will stick to high level concepts and **discuss** rather than implement. We will use the [default ping]() as the simple example and sometimes we will refer to it for further clarity.

# Ping: The definition

The process that we call **_Ping_** in xyz has actually nothing to do with the [Ping network utility](https://en.wikipedia.org/wiki/Ping_(networking_utility)) that you probably have heard of already.

> The process of **identifying**, **exploring** and **checking the availability** of nodes in xyz is called ping.

The only common part of the two pings is that both of them are used for checking the availability. Sad fact: If I were to re-write xyz from scratch today, I wouldn't have name it Ping. Too late for that though. Moving On.

Ping Mechanisms should be encapsulated inside a bootstrap function. This will allow the developer to easily modify the type of the Ping to be used. Like other plugins of xyz (middlewares and bootstrap functions), xyz provides only the default ping in the simplest form built in and other, more sophisticated, pings are written as independent modules that you can import and use.  

Ping Mechanism is crucial for xyz's `ServiceRepository` to function properly. In other words, It wouldn't work without a proper ping mechanism.

Ping Mechanisms **must** include the current node too. XYZ does not distinguish between foreign and local nodes and all of them are treated the same. Consequently, if you run a single node with `selfConf.logLevel: debug` you see track of ping messages being sent to the local node. Furthermore, a node might not be able to send a message to itself in its first seconds of exisstance, since the ping mechanism hasn't yet explored it! This might seem a bit unusual, but it brings a valuable _generalization_ and _uniformity_ which can make your day much easier.

Without further explanation, let's jump into to duties of every ping mechanism and learn the details through it.

# Ping: The duties

### Identification

Every Ping Mechanism should identify other foreign nodes that are eligible for being part of the system. This is the most important step and the following ones (Exploring and Checking) are dependent on it. identifying nodes can usually happen through different ways. Let's discuss some of them

- `systemConf.nodes[]`: This is the simplest case of identifying the system. In fact, it lifts the burden of identification from Ping's shoulders and hands in a list of nodes that are deemed as identified so they can be carried to the next steps.

- Newly added nodes: This type of identification is is very important and is what your system probably needs. Let's face it, you can not (+ _should not_) hardcode the list of all nodes in the system inside a static list because:
  - impossible to get it right. One of the benefits of microservice architecture is that new nodes can join and old ones can leave. So a static list is destined to fail.
  - its hard and we're lazy.

  A ping mechanism **must** provide a way for new nodes to join the system. Furthermore, when a new node has been added, all other already existing nodes should be informed.

### Exploring

After a node has been identified, it should be explored. This means that a node X that now is aware of the existence of another node, say Y, should get informed of the services of Y. Exploring services are the most important aspect, but it is usually not enough. As you might have read in [Servers and Routes]() section, X should also learn about the servers that Y has and the routes of each server and its port. The results of Exploring should be stored inside a property of `ServiceRepository` object so that it can use it later in `sendStrategy` and other tasks.

These variables are:



### Checking

Given that X knows Y completely, it should not keep trusting it forever. This means that X should iteratively probe Y and check if it is still in good shape. This is because **changes in topology of services** is almost inevitable in microservices. It's actually the hole point of them, being able to add or compose services to reach new business requirements.

# Case study: The Default Ping

In order to make your knowledge of Ping more comprehensive and solid, let's see how the default ping performs these important tasks. Please keep in mind that this particular Ping might handle some of these tasks pretty naively. This is because _Default Ping_ is **just a default fallback** for cases where you do not provide a ping mechanism. It is also good for local development, because it handles most of the issues robustly, with a lot of overhead (like solving a problem with **brute force** algorithms. It will reach the optimum solution, but with some extra computation), so you don't have to deal with service discovery issues and can focus on **business logic**.

### Required Setup

The Default ping will create one new outgoing route named `PING`. It also creates a server route on the **default port** (aka. port of the first transport of the node) of the node, again, named `PING`. All of the messages will be sent and received through these routes and your `CALL` route is safe.

Default ping can be disabled by setting `selfConf.defaultBootstrap: false`. If you do so, you'll see that these routes will not be created.

### Exploring and Checking.

Given that node `X` knows (somehow) about the existence of nodes `Y` and `Z`, it will perform the following to keep track of them.

It will iteratively send a message through `PING` to each of them. Given that `X` has just sent a message to `Y`, in response it expects the following to be received:
  - The list of services that the `Y` is exposing via `.register()`.
  - The list of nodes that `Y` is aware of them, which is expected to be [`X`, `Y`, `Z`] in a normal day.
  - the list of routes and servers of `Y`.

in the second item of the list, if an ip:port is mentioned by `Y` that `X` is not aware of, `X` will add it to an internal variable named `joinCandidates` and ping it in the iteration. If it responds successfully, it will be permanently added as a new node. Note that all join and leaves should be informed to `ServiceRepository` by calling [`ServiceRepository.joinNode()`](/apidoc/ServiceRepository.html#joinNode) and [`ServiceRepository.kickNode()`](/apidoc/ServiceRepository.html#kickNode). Note that these methods will update `CONFIG.systemCong.nodes[]`, which should be used at the start of each iteration of Exploring and Checking to find the address of all nodes.

The values in the first and third item of the list will be stored in [`ServiceRepository.foreignNodes{}`](/apidoc/ServiceRepository.html#foreignNodes) and [`ServiceRepository.foreignRoutes{}`](/apidoc/ServiceRepository.html#foreignRoutes) respectively. The format of `foreignRoutes` is pretty simple, but `foreignNodes` isn't. This data structure will be discussed in [Send Strategies](/advance/send-strategies).


What happens if `X` pings `Y` and the message fails? `X` will tolerate this up to a number of failures. The default value is 10. This means that if a node fails to respond for 10 consecutive pings, it will be kicked. Will this message be broadcasted? **No**. Each node will find out about this independently by doing its own ping iteration.


### Identification and Join

default ping will treat all nodes mentioned in `systemConf.nodes[]` as known nodes and passes them directly to the _Exploring and Checking_ phase.

Joins can happen in two forms:

- A new node directly pings an internal node to join
- An known node mentions a new node's ip and port in response of a ping message, which was explained in the last section.

Let's further discuss the former one.

Recalling from the last section, assume that `X`, `Y` and `Z` are three nodes that know each other. `P` is a new node attempting to join. First, let's see what `P` would do.

As you might have already seen, we can pass `selfConf.seed[]` to the constructor of xyz. In [getting started](/documentations/getting-started) section we assumed that this key will magically be the gateway for us to join a system. Now let's see how.

Fun fact is that xyz-core itself does **nothing** with `selfConf.seed[]`. It is there merely for an arbitrary ping mechanism to use it. The default ping will sequencally send a **normal Ping message** to nodes in `selfConf.seed[]` until one of them responds. Since this is a normal message through `PING` route, recalling from the last section, the callee will respond with lots of information, including **list of all nodes in the system**. Hence P can ping all of them starting from the next iteration and will eventually (_soon_ is also a good alternative for _eventually_) learn the entire system

Assuming that `P` joins the system through Pinging `X`, let's see the same story from X's point of view.

`X` will receive a **normal ping message** from `P`. Things will soon become a bit **far from normal** since `X` will understand that `P` is not it its `systemConf.nodes[]` list. In other words, **`P` is a new node attempting to join**. This event will always cause a `warn` log:

`warn :: new node is pinging me. adding to joinCandidate list. address : X.X.X.X:P `


 `X` will add `P` to the `joinCandidates` list and will ping `P` in the next iteration. If `P` responds successfully, it will be permanently added. Will `X` broadcast a message to other nodes to inform them that `P` has joined? **No**. Because Default Ping has a **_no-auth_** policy be default and when `X` passed a list of all nodes to `P`, it has actually done the same as broadcasting. Why? because `P` will ping `Y` and `Z` (who yet do not know `P` exists). `Y` and `Z` will place `P` in `joinCandidates` list for an iteration and both of them will add it permanently afterwards. Once `P` is added, `P` will be chosen for ping messages and `Y` and `Z` can explore `P` too.
