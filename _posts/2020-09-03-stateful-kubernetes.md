---
layout: post
title: "Stateful Kubernetes"
subtitle: "My initial thoughts on running stateful Kubernetes clusters"
date: 2020-09-03 00:30:00 -0400
background: '/img/posts/01.jpg'
---

## Disclaimer
I haven't spent much time researching the benefits of running stateful application servers, or stateful Kubernetes specifically. This post is being written to start conversations about the topic, so I'd be very happy to receive feedback or talk about this more (email: jeroen-rijks@hotmail.com). I've also added asterisks to areas that I'm not confident about, so treat those with an extra bit of caution.

## Stateful application servers
My colleagues and I were discussing [this article](https://mailchi.mp/railsspeed/one-heroku-feature-that-everyone-should-copy?e=1352295606), which talks about how Heroku restarts its instances every 24 hours, resetting their entire filesystem. By periodicically resetting a server's state, application design becomes incentivised to create stateless applications. By running statelessly, you can (among other things): 
- avoid having an irreproducible state that acts as a single point of failure
- scale horizontally without the complexity of replicating state across servers
- use spot instances to save money

## Stateful Kubernetes
This Heroku article got us thinking about stateful Kubernetes workloads. Stateful Kubernetes workloads seem to be a pretty common conversation topic, with Rancher Longhorn recently being donated to CNCF, articles benchmarking different tools, and podcast speakers that speak as though stateful Kubernetes is the "default" Kubernetes workload.

### Managing state
To form an opinion about running Kubernetes statefully, I first thought about how I would approach managing state in a "traditional" non-Dockerised stateful server\*:
- procure a dedicated volume for your server
- if I'm scaling horizontally, replicate this mounted volume between all of my server instances (could you mount a shared volume instead?\*)

I then thought about how this is done in Kubernetes:
- procure a dedicated volume for your server (e.g. persistentvolume)
- mount this volume into the application's pods

Requiring persistent storage in Kubernetes clusters introduces complications, especially since pods from different worker nodes may want to share volumes. This is where Rook, Longhorn & the rest come into play. From what I understand about these solutions\*, they offer Kubernetes-friendly wrappers around long-lived filesystems, but they still increase complexity, and don't all seem production-ready.

### Kubernetes's strengths
Next I thought about what makes Kubernetes so powerful. I've always thought that the main value of Kubernetes is its ability to break servers down into a super-scalable "pool of resources", giving me compute power in whatever form I want, running whatever process I want. The fact that lightweight containers can be run and scaled in/out so smoothly really makes me think that Kubernetes is best run statelessly. In turn, this makes me think that running Kubernetes with state seems "wasteful".

However, Kubernetes's value isn't just limited to the fact that it can act as a "pool of resources". I recently heard someone point out that there's a lot of value in the fact that Kubernetes abstracts compute power away from a physical machine. After you configure your worker, it no longer matters what you're running on, providing cross-vendor & cross-instance type flexibility.

Other benefits are 
- the fact that you can easily run additional workloads like cronjobs without needing new architectural components
- other processes (such as logging and monitoring agents) can be run on the same machine, to collect data from the machine that's being monitored or creating the logs
- other things like the extensibility of its APIs\*, & service meshes\*.

##  Going back to stateful Kubernetes
Initially, I didn't like the idea of stateful Kubernetes, because it seemed to me like its strength lies in its dynamic and scalable nature, which contrasts with the nature of stateful application servers. However, if you're deciding where to run your stateful application, you may decide that the other benefits that Kubernetes provides outweigh the storage complexity that it introduces.

I don't know enough about what drives people to design stateful application servers in 2020\*, so I can't decisively say that I dislike stateful application servers. However, having written this article, I can understand why people might prefer to run their stateful apps on Kubernetes, instead of running it on a more traditional server instance.
