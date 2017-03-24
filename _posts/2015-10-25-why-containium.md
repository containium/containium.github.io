---
layout:	post
title:	"Why Containium?"
image:  reactor_core.jpg
imgpos: bottom
date:	2015-10-25
author: danny
tags:	containium reactor clojure boxure
---

> #### TL;DR
>
> In summary, we have created a application server for running multiple Clojure applications -- the *Containium* -- on top of several core services -- the *Systems* -- providing some important and well sought-after features:
>
> - Run **multiple** Clojure applications in **one process**
>
> - **Isolate** these applications from each other, but **share** the same **Clojure runtime and core** namespaces, meaning efficient use of resources and **blazingly fast starting** of the applications
>
> - Provide **shared services** to the applications, such as **HTTP** connectivity, **distributed session management**, connection pools and other resources, i.e. **batteries included**
>
> - Let the applications **communicate by Clojure data**, without serialization and deserialization
>
> - **Hot-swap** the applications within **milliseconds**

## The end of an Age (2012)

Three years ago, we set out to build a very ambitious product with only a small team of developers. Because of the large scale of the project and the number of resources we had, we were forced to pick tools that allowed us to go fast, yet deliver quality and scalability. From past experience, we knew that Clojure would fit the job for many of the project's services we needed to develop. Thanks to power of the language, the opinionated contructs and restrictions, and the fantastic ecosystem, we were indeed able to make huge leaps. One piece was missing for us though in this ecosystem, and -- as we will see soon -- partly in Clojure itself: How are we to deploy our services in an ops friendly way?

We first set out to use Immutant for this. Note that Immutant "The Deuce" 2 did not exist back then. Immutant was still very much in flux, and had some time-consuming issues. This, together with the unsure future of Immutant, not requiring any of the JBoss/JEE "legacy" it brought with it, and the huge memory footprint of it with only a couple of Clojure apps deployed, we decided to create something more suitable for our needs, which we thought would be more Clojuresque, more resource efficient - both CPU as well as memory wise -- and library oriented.

While unsure if it would work, we had a simple core goal: have a couple of base sysems, such as a ring server, a database connection pool and a distributed session store, which can then be used by hot-swappable Clojure apps, _using only one Clojure runtime_ per JVM instance. To test whether this goal was feasable, we set out to create the [boxure][boxure] library.

## Boxure

Boxure is a library that loads a Clojure project -- a JAR or a local directory with a `project.clj` file -- into a special classloader. It makes sure the classloader has all the dependencies and source paths from the Clojure project in its classpath. The boxure classloader works with the concept of _"isolates"_. Whenever the classloader is asked to load a class, it first checks (by name) whether it should be loaded in isolation or not. When it is listed to be isolated, the boxure classloader tries to load the class using its own classpath, and will _not_ try the parent classloader at all. If the class is not listed to be isolated, normal classloading kicks in, i.e. trying the parent classloader first. The boxure classloader has a default list of classes that must be loaded in isolation, but when creating the classloader, one can add more to this list.

Here's the thing: we identified the stateful parts of the Clojure runtime, and isolated those classes, and the rest -- such as all the datastructures -- is _not_ isolated. This, together with some trickery with dynamic classes, thread bindings, thread locals, weak references and Clojure agents, gave us true horizontally isolated and completely unloadable Clojure-laden classloaders. And this while using a single instance of most of the Clojure runtime, also enabling sharing datastructures between the classloaders directly -- no serializing needed. By surrounding this with some helper functions to evaluate forms within a boxure classloader, we had our eureka!

One slight downside though (or so we thought): we had to make some small changes to the Clojure runtime classes. Because the stateful Clojure runtime classes now live in a child classloader with regard to where the rest of the runtime classes live, those "child" classes cannot access _package private_ fields and methods of "parent" classes. And there are lots of those fields and methods in the Clojure runtime. Alas, we had to fix those. Our solution now required to run on our own version of the Clojure runtime. Luckily this altered version only needs to be used in the base classloader; all the apps can use the official version.

What we thought of as a downside, actually turned out inspire us to see how deep the rabbit hole would go. Since we were now changing the Clojure runtime, boxure and its runtime evolved to isolate less and less. The namespace handling in the Clojure runtime has been changed, in such a way that they are loaded on a per-classloader basis. Or in other words, stateful _per app_. Even the Clojure core namespaces are loaded only once and shared with the classloader as soon as they are created, while still allowing `alter-var-root`-ing them per classloader. We call this _"injections"_, which are optional and can be user defined.

The essence of our work was now done. Next, build an application server around this.

## The Reactor, Systems and Containium

With boxure, we have made a sort of _containers_ for Clojure applications. We call these services/applications the _Containium_. The Containium is only one part of a deployment though. A full deployment consists of three parts: the Reactor, the Systems and the Containium itself (the apps).

The Reactor is responsible for managing Systems and managing the Containium. The Reactor is mostly just a process combining various libraries to fulfill its goal. It uses the boxure library for Clojure application isolation, the custom Clojure runtime discussed before, and management/ops functionality around them for deploying and swapping Containium.

The Systems are libraries that are loaded and started in the base classloader of the Reactor. Each of them implements a lifecycle protocol, in order to be started and stopped. This is similar to how the well known [component][component] library works, but that did not exist at the time. Any System may depend on other Systems, which is why they mostly implement a protocol for their functionality as well; i.e. their implementation can be switched with another implementation. For instance, there are multiple ring server Systems, such as for Jetty 9 and Http-Kit, yet they implement the same protocol.

An important goal was to implement the Systems as libraries. This way, they can be used outside the Reactor as well. This is especially important for out-of-container unit testing, but also has the added benefit that it is trivial to create a custom Reactor, or have a Containium run standalone, not using boxure at all.

And finally, the Containium itself. Each app that wants to be run as a Containium needs to declare some information about itself. For example, to be run as a Containium requires that the application must have lifecycle functions, similar to the Systems, and must therefore declare where these functions can be found. It is also through these lifecycle functions that the Containium gains access to the Systems. Furthermore, it may also declare where a ring handler can be found after it has been started, optionally supplying a context-path and/or filtering on hostname.
Containium can be undeployed, freeing resources. A huge benefit that the Reactor provides here, is that a newer version of a Containium can be deployed, while the older version is still running. Only after the newer version has been fully deployed, the new ring handler is used. We refer to this as swapping Containium, and because the Systems just keep running and a shared Clojure runtime is used, deploying or swapping Containium is ridiculously fast!

## Wrapping up

Now you know why Containium and its ecosystem has been created. We have been using Containium in production for over a year now, swapping newer versions of our services to our development server on every push. We think we have made a big leap towards fast and efficient deployment of Clojure applications and services.
That being said, we have open sourced our current codebase just as is. In the coming time we will be evaluating every line of code, and polish and enhance it to make the Reactor and the Systems more generic, robust and developer/operator friendly wherever possible. For now, have a look at [boxure][boxure], the [reactor and systems][reactorsystems] or an [example][cassandra-file-api] Containium, and expect more posts explaining the Reactor, the Systems and Containium in more detail.

[boxure]: https://github.com/containium/boxure
[component]: https://github.com/stuartsierra/component
[cassandra-file-api]: https://github.com/containium/cassandra-file-api
[reactorsystems]: https://github.com/containium/containium
