---
title:  "Designing multi-tenant systems"
date: 2021-01-31
categories:
  - blog
tags:
  - multi-tenant
  - design
  - architecture
---

# Multi-Tenancy

Hi there! In my current company, I have been working on a product that is used by multiple B2B customers. This is not your internet scale B2C application, but customized SaaS solutions for enterprises. It takes weeks and months to sign up a new customer, and involves proof of concepts, showcasing case studies, customization and new development. But the resulting customized application is not a stand alone solution for that customer, but is fed back into our own SaaS product. In this and the next few posts, I will go over the different aspects to consider when creating a multi-tenant system. I hope this post will breathe life into the title of this blog.

---

From Wikipedia:

The term *software multitenancy* refers to a software architecture in which a single instance of software runs on a server and serves multiple tenants. A *tenant* is a group of users who share a common access with specific privileges to the software instance. With a multitenant architecture, a software application is designed to provide every tenant a dedicated share of the instance - including its data, configuration, user management, tenant individual functionality and non-functional properties. *Multitenancy* contrasts with *multi-instance architectures*, where separate software instances operate on behalf of different tenants.

---

Let's consider two companies (tenants) who are our customers - Ravenclaw and Hufflepuff. Both want to use our wand matching software *MatchWand*, but their requirements are not exactly the same.

One important business decision is whether to sell the software as-a-service or as-a-solution. In the former, we will try to develop *MatchWand* as a product, with variations for different tenants in it. In the solution approach, we will typically have a generic codebase, which will be forked once for Ravenclaw, and again for Hufflepuff.

> What I specify as solution and product is very specific to how companies (like where I work) consider software. These are overloaded term, like services. If it is a solution, it is only for one customer. A product, on the other hand, is a *solution* for multiple customers, and hence highly *SaaS-able*.

A solution approach typically ends up as a multi-instance deployment (or could even be deployed at the customer site), and there isn't any complication there. Once the tenant-specific code base diverges from the generic codebase sufficiently, it is difficult to merge it back and create a software product out of it.

The product approach is more interesting. We typically start off developing *MatchWand* for one client, Ravenclaw. One codebase. It is deployed for Ravenclaw. Another client Hufflepuff now comes in and wants to license the software. The second client is always the trickiest and most critical. If you decide to take short-cuts here, you might end up with unmaintainable code (more on this later). We spend enough time refactoring, so that *MatchWand* remains as a single codebase, and can be deployed to Hufflepuff also.

There are a few approaches to scaling this to more clients:

| Option | Description | Pros | Cons |
| -------| ----------- | ---- | ---- |
| Static Switching  | At compile time (rare) OR deployment time (common), decide the tenant with a config | Simpler architecture |Forces you to do multi-instance deployment |
| Dynamic Switching | At request execution time, decide the tenant and execute corresponding code | Very flexible, simplifies operations and paves way for cost optimization | Increased overall complexity |

The static approach enforces you to take a multi-instance deployment, but the advantage is that you can have different versions of the software running for different tenants. While this is not a good thing from the point of view of the engineering and ops teams, it is sometimes unavoidable, especially in a B2B setting. The dynamic switching approach could also handle multi-instance deployment. You could deploy *MatchWand* for one tenant or multiple tenants, mix and match multi-instance and multitenant deployments. Let's dig deeper into this.

How would Ravenclaw or Hufflepuff access *MatchWand*? As a webapp, it would be either via `https://ravenclaw.matchwand.com` and `https://hufflepuff.matchwand.com` or a generic `https://matchwand.com`. Like always, myriads of options exist to implement this behind the scenes, two of which could be like this..

![Deployment2](/assets/images/deployment-2.jpg)

![Deployment1](/assets/images/deployment-1.jpg)

Say you decide to use kubernetes, you could choose separate namespace for each tenant on same cluster (in which case, it is multi-instance from the software perspective, while your hardware, k8s versions, etc. are all shared), or you could have separate clusters altogether. You could also have one single namespace for both tenants (in a true multi-tenant way).

Imagine a completely shared deployment with compute and data stores (databases, message queues, storages etc.) all being the same for multiple tenants. This is the way to go for B2C software, because typically clients can sign up on their own. B2B is way different. A client might be concerned about data being stored in (or outside of) a particular country. They might ask that their data not be shared with other clients, at least logically.

So as you can see, there are *varying degrees of multi-tenancy*. From completely isolated software and hardware deployments to fully shared models. As always, it's something in between that typically ends up being the most practical one.

![Deployment3](/assets/images/deployment-3.jpg)

Sharing only compute among tenants is a great way towards multi-tenancy, because you gain all the good things like simplicity of operations and cost reduction. How fast you achieve this depends on how your software is written.

**Stateless services** are key to moving towards this goal. Is there a shared state in your service? Static or global variables? Singletons? In-memory cache perhaps? When the same service is supposed to handle requests from multiple tenants, you cannot have *any* shared state, unless you make it *tenant-aware*.

What about object pools or connection pools? If you have shared compute, but separate databases, your connection pools also have to be tenant aware.

**Distinguishing the tenant for each request** is paramount when compute is shared. We can distinguish the tenant for each request using some middleware, where we use either the host name or the logged-in user id to determine the tenant. You then use it for figuring out the context for each execution.

**Functional Testing and Automation** is important when you go for multi-tenancy. You cannot afford to perform manual testing on *each* tenant for each change.

Another important aspect of sharing compute is to perform **load testing** to find correct autoscaling options. Not only do you need to load test multiple tenants concurrently, you also need to set resource requests and limits in k8s appropriately. See my [blog post](../_posts/2021-10-01-kubernetes-settings-resource-constraints.md) on how to find out resource constraints.

Well, that's it for today, I'll expand on each of these areas in detail in the upcoming posts.

---
