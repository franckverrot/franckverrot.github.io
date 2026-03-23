---
title: "Request Routing with Nomad and Consul"
date: 2019-03-16 11:20:16
comments: true
slug: "request-routing-with-nomad-and-consul"
tags:
  - go
  - infrastructure
---

Request Routing in the scheduler/container world is an ongoing challenge,
with a lot of different and competing solutions that tries to provide a solution.

Some solutions are built on top of others, some only support specific
schedulers, some operate only at the [L7][l7] layer, which doesn't make
things easy for Platform Engineers when it comes to adopting a specific
solution.

Building and operating Nomad/Consul clusters with the Platform Engineering
Team at &lt;&lt;work&gt;&gt; has been an interesting problem to solve. We
explored a few different solutions, and as we evolve in a highly-regulated
world (we must implement HIPAA, SOC2 Type II, and HiTrust), solutions that
weren't providing basic security (TLS everywhere, poor auditing, etc.)
haven't been considered.


<!--more-->


### Forewords: Build vs Buy

Choosing to operate Kubernetes or Nomad is a strategic choice. It requires
having an in-house team composed of talented "DevOps", sysadmins, developers
who will understand the challenges of maintaining a platform for the rest of
the engineering team. Most small-to-medium companies should then try to stick
to using commercial solutions like [Heroku][heroku], [GCP][gcp],
[Fargate/ECS][fargate], or other vendors who provide one-click deploys.

My advice would be that (most) early-stage "health tech" startups should work
on their value proposition rather than building a platform, and should
leverage existing providers before considering anything else.
Highly-regulated companies should try vendors like [Aptible][aptible], who
also seem to be trying to help passing security audits and "deploy
audit-ready apps and databases" but they might have no choice and jump the
sharks.  YMMV.


#### What features should one look for?

Most organizations should be looking for tools that solve the problem they have at hand but there are many, and some tools only implement a subset of all the requirements that could come to mind.  Our tool should probably:

  * embrace CI/CD patterns (blue/green deploys, canaries)
  * provide monitoring and metrics (Prometheus exporters and other tools one might be using/willing to use)
  * provide some sort of distributed tracing / observability
  * implement circuit breakers (retries, throttling, ...)
  * etc.


### Existing solutions

As stated earlier on, there are already many solutions out there:

  * [Envoy Proxy][envoy]
  * [HA Proxy][haproxy]
  * [Traefik][traefik]
  * [nginx Controller / nginx Plus][nginx] (commercial)
  * Custom L7 Lua/Ruby scripts with one's preferred reverse proxy

A note around "service meshes": the tools aforementioned are usually used to
implement what is called ["north-south" traffic][traffic_types], traffic from
and to one's data center. A service mesh is meant to implement ["east-west"
traffic][traffic_types], for one's services to talk to each other, and there are plenty of solutions that try to address this problem:

  * [Consul Connect][connect], from the HashiCorp stack
  * Envoy-based solutions:
    * [Istio][istio]
    * [Ambassador][ambassador]
  * Provider-focused solutions
    * [AWS App Mesh][app_mesh]
    * [Istio][istio] can run on both [GKE][gke_istio] and [AKS][aks_istio]


### Deployment Models

Timothy Perrett has written [a thorough blog post][timperrett] explaining a
lot of things about Nomad, Consul and how Envoy's discovery services are
working. This is a must read to get one up and running with the problem
space.

It depends largely on one's organization to decide what model they would want
to adopt, given that handling TLS should probably be a feature provided by the
platform developers will be operating on rather than something they will have
to maintain themselves. Some solutions like [Vault][vault] can help apps
acquiring their own certificates (the Go SDK even provides a background
worker for renewals), but it isn't always convenient for app developers to
deal with these types of concerns.


### Integration within an existing platform

Generic patterns have emerged, and recent solutions implemented a
segregation between their control plane and their data plan nodes:

  * control plane node: think of it as the brain responsible for the overall
    cluster configuration and its policies
  * data plane node: this is what does the actual job.

This segregation has been around in the networking world for more than a
decade, the RFC [Forwarding and Control Element Separation (ForCES)
Framework][rfc] explains how this all works.

```
+-----------------------+       +--------------------------+
|                       |       |                          |
|      Envoy Proxy      +<----->+   Custom Control Plane   |
|                       |       |                          |
+-----------------------+       +-------------+------------+
                                              |
                                              v
                                +-------------+------------+
                                |                          |
                                |  Consul service catalog  |
                                |  Nomad allocations       |
                                |  ...                     |
                                +--------------------------+

```

A tool like Envoy can be bootstrapped with a minimal configuration that would only know it will be configured later using one of the many service discovery strategies it implements:

  * Listeners Discovery Service aka `LDS`
  * Routes Discovery Service aka `RDS`
  * Virtual Hosts Discovery Service aka `VHDS`
  * Clusters Discovery Service aka `CDS`
  * Endpoints Discovery Service aka `EDS`
  * Secrets Discovery Service aka `SDS`

 A custom control plane would need to implement these to start talking to Envoy
 Proxy, the Envoy Proxy GitHub organization provides a [reference
 implementation][control_plane].

As a side-note, I also released [Raven, a Consul-based Control Plane for
Envoy Proxy][raven], which gets its information from Consul and can talk to Envoy
Proxy.  It is functional but only implements LDS and CDS for now.


### Afterword

Many solutions, and the space is continuously moving. A simple/quite standard
winning strategy is to list very early in the project what features and
requirements you would need, and categorize them:

  * `must have`: blockers, can't live without these features
  * `should have`: important features, not vitals
  * `could have`: won't be used immediately but would enable more in the future

I personally like Envoy Proxy's design, even if its documentation has been really challenging at first (it can get obsolete at times.)  nginx Controller seems promising too but I couldn't try it as it's a commercial product and I had no intention of getting in touch with their sales team for a trial version.

Large companies seem to be betting on Istio, but as it is based on [Envoy
Proxy, and it recently graduated at CNCF][cncf], I am not concerned about
transitioning from Istio to Envoy Proxy and vice-versa.



[aks_istio]: https://docs.microsoft.com/en-us/azure/aks/istio-install
[ambassador]: https://www.getambassador.io/
[app_mesh]: https://docs.aws.amazon.com/app-mesh/latest/userguide/what-is-app-mesh.html
[aptible]: https://www.aptible.com
[cncf]: https://www.cncf.io/projects/
[connect]: https://www.consul.io/docs/connect/index.html
[control_plane]: https://github.com/envoyproxy/go-control-plane
[envoy]: https://www.envoyproxy.io
[fargate]: https://aws.amazon.com/blogs/aws/aws-fargate/
[gke_istio]: https://cloud.google.com/istio/docs/istio-on-gke/overview
[haproxy]: http://www.haproxy.org
[heroku]: https://www.heroku.com
[istio]: https://istio.io
[l7]: https://en.wikipedia.org/wiki/OSI_model#Layer_7:_Application_Layer
[nginx]: https://www.nginx.com/products/
[raven]: https://github.com/franckverrot/raven
[rfc]: https://tools.ietf.org/html/rfc3746
[timperrett]: https://timperrett.com/2017/05/13/nomad-with-envoy-and-consul
[traefik]: https://traefik.io
[traffic_types]: https://blogs.technet.microsoft.com/tip_of_the_day/2016/06/29/tip-of-the-day-demystifying-software-defined-networking-terms-the-cloud-compass-sdn-data-flows/
[vault]: https://www.vaultproject.io