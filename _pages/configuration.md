---
layout: page
title: How to configure xyz nodes
permalink: /documentations/configuration
---

# configurations phases:

configurations are an important part of the xyz ecosystem, hence it is important for the developer to know how to use them and manipulate them.

XYZ configurations can be set on 3 phases:

  - [3] The default value: stored in [`Constants.js`](https://github.com/node-xyz/xyz-core/blob/master/src/Config/Constants.js#L26) file in `xyz-core`
  - [2] `selfConf` and `systemConf` at runtime
  - [1] via command line arguments

and each of them have priorities indicated by `[x]` in the list. This means that whatever you pass in as **command line argument** will override `selfConf` and `Constants.js`. Let's look at each them one by one:

### The default configuration: Constants.js

Recall from the two simple services that we worked with in the getting started section. Let's luch one of them but this time, we are not going to pass any configuration to it:

```
let xyz = require('xyz-core')
let mathMS = new xyz({
  selfConf: {},
  systemConf: {nodes: []}
})

// register any services

console.log(mathMS)

```

As you can see the system lunches perfectly with the default values.


### Runtime configurations: selfConf and systemConf

As you expect, whatever you place in `selfConf` and `systemConf` at runtime will override these values.

```
let mathMS = new xyz({
  selfConf: {
    port: 10000,
    name: 'from-self-conf'
  },
  systemConf: {nodes: []}
})

console.log(mathMS)

```

pay close attention to the values printed after `console.log(mathMS)` and see how `port` and `name` have changed.


### Command line arguments

One last way to change configurations is to use command line arguments. Normally, you would run an instance like:

```
$ node mathMs.js
```

But, we have seen in the getting-started section that you can actually override the port value of a node using:

```
$ node math.ms.js --xyz-port 5002
```

this is actually a global pattern for overriding the configurations. It goes like this: `--xyz-[CONFIG.KEY] [CONFIG.VALUE]`

As an example, you can change the name of a service via command line using:

```
$ node math.ms.js --xyz-name sth-from-cli
```

If you run a service like this, you'll see that:

```
____________________  GLOBAL ____________________
selfConfig:
  {
  "name": "sth-from-cli",
  "defaultSendStrategy": "xyz.service.send.first.find",
  "allowJoin": false,
  "logLevel": "info",
  ....

```

Keys with more depth can also be manipulated using a `.` :

```
$ node mathMs.js --xyz-cli.enable true

// analogous to selfConf: {
//    ...
//    cli: {enable : true }
//    ...  
```


A complete example could be as follows:

```
$ node mathMs.js --xyz-logLevel debug --xyz-seed 127.0.0.1:4000 --xyz-defaultSendStrategy xyz.service.send.to.all
```
