---
title: "Replacing BEAM with Kubernetes"
date: "2020-06-23T04:50:48Z"
tags: ["programming", "erlang"]
---

I see this sentiment frequently while browsing Reddit, but I've yet to see an adequate reply explaining the major problem with it.
Most replies focus on OTP and supervisors, and while these things are wonderful in their own ways, it doesn't leave the reader understanding why the statement is misguided.
The most important point to remember is this: Kubernetes requires an inordinate amount of work to approximate the value provided by BEAM.

Kubernetes does not provide the correct primitive operations to easily replace BEAM; it is missing spawn, send, and receive operations at the proper abstraction level.
Objecting to the declaration of a missing spawn operation in Kubernetes is reasonable, it is, after all, a container orchestration tool and can therefore launch containers.
Yet what Kubernetes provides is coarse-grained, and frankly, not good enough to generally replace BEAM.
While Kubernetes makes you define Pods to launch containers to run programs, BEAM focuses on spawning single functions as isolated "processes" which have few means of sharing state other than message-passing via the send and receive primitives.
The orchestration functionality offered by Kubernetes ensures set quantities of programs are running; BEAM provides primitives to break those programs into smaller slices and to ensure they're available via supervisors.

You can certainly get to the same level of abstraction using Kubernetes by breaking your larger programs apart into smaller and smaller programs.
Eventually you will have created single-function programs which can be used as an entry-point for a container.
All that remains is to create Pod definitions for every single-function program so that Kubernetes can schedule your functions across the cluster.

Once you complete your Pod definitions you're not actually done this mammoth task.
Your containers are running single-function programs, but those functions still need to communicate to receive input and send output.
You could use shared disk volumes for reading and writing between containers, or make every single-function program send and receive messages using exposed ports.
Every single-function program you've created will now need some kind of socket connection and code which deals with communicating over that socket, be it an NFS mounted drive, HTTP requests, or some good, old, custom TCP-based protocol.

BEAM gives you send and receive support at the virtual machine level for your spawned, isolated functions, by providing each with a mailbox containing messages sent to it by other spawned functions.
With a little help from the runtime system, these messages can be sent and received across all the virtual machines in your cluster without any additional code.

So yes, you can definitely, eventually, replace BEAM with Kubernetes, but I don't think you'll like the look of your code afterwards.
