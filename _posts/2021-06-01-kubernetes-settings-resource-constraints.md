---
title:  "Kubernetes: Setting Resource Constraints on Pods"
date: 2021-01-07
categories:
  - blog
tags:
  - kubernetes
  - vertical pod autoscaler
  - resource constraints
---

![Hope](/assets/images/hope.jpg)

Happy new year folks! Its finally 2021, and I wish you a healthy and safe year ahead. The world needs hope and more good wishes. Stay strong!

I'm going to write about my experiments with tweaking resource constraints for an application that has been deployed in kubernetes. This post will help you find the right values for requests and limits for your pods in a large system. It assumes that you know what kubernetes is, and how pods and deployments work.

Do let me know your thoughts in the comments section, and all feedback is welcome!

## Why resource constraints?

When you have multiple services deployed in a k8s cluster (microservices or not), you will need to set the CPU and Memory resource values. You may have started off without specifying these, or given some random values like 500m and 500MiB, as these are optional fields. But eventually, before going to production, you need to find the right values. Let's see why, and how.

### What are resource constraints?

Requests: These are the minimum amount of resources the pod is guaranteed to get, and is allocated while the pod is scheduled.
Limits: These are the maximum resource values permissible to be used by the pod. If CPU usage of pod exceeds this, it is throttled, whereas excess memory usage will get the pod terminated.

Let's take a simple example, say you have one node with 8 cores, and you have 10 pods running, each requesting 500m (millicores), then all the pods will get scheduled (request = 0.5 * 10 = 5 cores; you have total 8 cores available). However, if the limits are a much higher value, say 1000m, then total limits becomes 10 cores (1 * 10), but only 8 cores are available. In this case, it is clear that not all pods can use the max of 1000m at the same time. This is called overcommiting. If you describe a node, you will see this:

`kubectl describe nodes`

```
..
..
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource                       Requests           Limits
  --------                       --------           ------
  cpu                            7499m (95%)        27800m (355%)
  memory                         12101121920 (91%)  39547844864 (298%)
  ephemeral-storage              0 (0%)             0 (0%)
  hugepages-1Gi                  0 (0%)             0 (0%)
  hugepages-2Mi                  0 (0%)             0 (0%)
  attachable-volumes-azure-disk  0                  0
..
..
```

The problem with overcommitting is that at load, your pod may crash or not respond as it does not have enough resources.

The concept of Quality of Service or QoS also comes from this. If you set requests = limits, then it is called Guaranteed QoS, because the above scenario will never happen. If the pod is scheduled, it is guaranteed to get that resource, and it doesn't need anything more.

If requests are less than limits (or no limits are set), then it is called a Burstable QoS, when there is a need, the pod can use more. By setting lower requests, you can fit more pods into the node, thus reducing number of nodes needed, and hence cost. It might be ok as long as you are overcommitted to a small extent, say 110%-120%, but not the crazy values I have above like 355%. This is generally helpful if you know that not all pods will hit their peaks at the same time.

There is a third type of QoS which is called BestEffort, that is when you have neither requests nor limits set. Don't use this except during experimentation

If you describe a pod with `kubectl describe po _podname_`, you will see the QoS.

Note that, there are lots of system pods which also need to be factored in your calculation. Not manually of course, just that, the simplistic calculation above of 10 user pods is not sufficient.

You can read more about requests and limits in [this article](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-resource-requests-and-limits).

Now that we have seen what resource contraints are and why we need to specify them, let's take a look at how we can find the ideal values for our services.

## How to find the 'right' resource constraints

### The manual way

When I had less number of application services, I used to manually find the resource constraints of each service by performing a load test on each of them. The process goes something like this:

The first important thing is to have one service handle similar load only – does not really have to be a microservice, but it should not be a monolith either. You should be able to do simple math like, if one pod can handle 100 operations/second, 10 pods can handle 1000 operations/sec.

1. Do not set any requests or limits on the pods. Do load testing on one particular pod/service (1 replica), and increase requests/sec gradually. Find the point at which the response grows exponentially, or the point at which the response time is more than your SLA. You will typically see that after a particular load, the response time jumps (or pod starts crashing)
2. Now we know the breaking point - something like _upto 250 requests/sec, the response time of each request is under 2 seconds_. Now we need to find the CPU and memory at which this condition can be met. Start with smaller values like 0.5 CPU, 250MiB etc. for limits (requests can be same) and find the point beyond which increased resources has no effect on overall throughput. This is done just to optimize the cpu and memory for different services (we could start with higher numbers and it would work but would be suboptimal).
   1. Run a similar load test now, but at constant load – say, 250 requests / second (or a little less, say 200) continuously for some time, say a minute or so. Keep the cpu and memory to low values as per your service needs. You should see the response time increasing in the same way after a point – but now the reason would be not enough cpu.
   2. Repeat the above but with a higher CPU value, such that you get the original throughput, say 200 or 250 requests/sec.

Now you have found out the capacity / limit of a single pod. Adding Horizontal Pod AutoScaler (HPA) is easy. If you are expecting maximum 1000 requests/sec, set minimum pods to 1 or 2 and maximum to 5 in HPA, and you will be able to handle load as efficiently as possible. You can test again with HPA enabled and handle 1000 requests/sec easily.

Finally, do a load test which includes a ramp up, plateau, and a ramp down, to see if HPA works properly.

The above manual way is adapted from [this blog post](https://opensource.com/article/18/12/optimizing-kubernetes-resource-allocation-production).

### VPA

Well, that was 2019. We expect simpler things in 2021, given how 2020 turned out.

While the above manual way works for a small number of services, it is cumbersome as the system grows. Also, it might not be possible to load test a service in isolation. We need a more automated way to do these tests, and to do them regularly (as you add new features, the behaviour of these services will change, these tests might need to be repeated every few months or so).

I recently came across the [AutoScaler project in kubernetes](https://github.com/kubernetes/autoscaler). This is a set of kubernetes objects that can find the ideal cpu and memory values for each pod, based on history. It is called Vertical Pod AutoScaler (VPA), and it has a 'Recommender' mode, which doesn't change the pod resource values, and that is what we want here.

High level logic: Run load tests and tweak the number of replicas manually such that there are no failures. Once you have the optimal number of replicas that can handle the load, find the recommended resource constraints from VPA.

Setup:

1. Install VPA and Goldilocks
  
  ```bash
  helm repo add fairwinds-stable https://charts.fairwinds.com/stable
  helm repo update
  helm install vpa fairwinds-stable/vpa --namespace vpa --create-namespace
  helm install goldilocks fairwinds-stable/goldilocks --namespace goldilocks --create-namespace
  ```

2. Mark the namespace you want VPA to monitor
`kubectl label ns loadtest-env-01 goldilocks.fairwinds.com/enabled=true`

3. Visualize the recommendations after the load test here:
`kubectl -n goldilocks port-forward svc/goldilocks-dashboard 8080:80`

Running the load tests:

* Set some request values (say 50m and 100MiB) and no limits. The request that VPA recommends is the most important value, you can tweak the limit as needed for your QoS. Also, by setting no `limit`, we are testing the limits of the pod.
* Disable HPA - VPA and HPA don't work together.
* Run your load test by manually keeping the number of replicas for all services to a number where there are no failures. For example, for one service, I had to keep 10 replicas for the load test to succeed, anything fewer would cause application failures as some pod(s) will not be able to handle the load.
* Once you have a working load test, find the replicas and the VPA recommended request values. If the requests are small enough you may stop here. But if you get large requests values, then I suggest to increase the replicas a little bit more and see if the requests come down. Reason is that I prefere horizontal scaling over vertical, even in pods. See [below](#prefer-horizontal-scaling)
  * In my example above, say for 10 replicas I got CPU required as 0.5 cores. Assume this is for a load test that generates 1000 req/sec. I would rerun the test with 15 replicas, so that the avg load on a pod reduces from 100reqs/sec to 67 req/sec. If this reduces my cpu requirement from 0.5 cores, I'll go with 15 replicas and not 10. Or repeat the test to get pod sizes that you want.
* Tweaking number of replicas for this test is based on knowledge of your application and some assumptions about the most used services. The important point here is to be able to finish your load test without any failures by making sure there are enough pods available.

Min replicas = I always keep this to at least 2, even on a light service.
Max replicas = this is what we have experimentally found out above.
Resource values = this is what VPA gives us. You can tweak it and set on your deployment pod spec.

Some points to note:

* When you run multiple iterations of tests, I used to redeploy the app into different namespaces each time to make sure the metrics are not skewed by previous runs. I'm not sure if there is a better way.
* Once you have found good values for requests, and limits, turn off VPA, and re-run your load test with HPA enabled, and ensure that there are no failures and your SLAs are met.
* Finally, install this recommender on your final environments and keep monitoring for any drastic changes.

## Prefer horizontal scaling

I prefer horizontal scaling of pods over vertical scaling for multiple reasons

* Vertical scaling restarts the pod
* Better chance of getting scheduled - Smaller pods can be fit into available space in VMs more easily.
* Less pods need to run when usage is low

If the application will work the same way with 'one pod with 500m CPU' or 'two pods with CPU 250m', it is better to choose the latter. The pod has better distribution possibility now, can be scaled down and resources released when not used. Obviously, you cannot take this idea to the extremes and give very less resources to the pod. My rule of thumb is:

* I generally start with 50m/100MiB for requests. Even if VPA recommends lesser values, I don't go below this.
* If VPA recommends small updates, say 75m, or 175MiB, I generally keep limits to be 2-3x of the recommended request, if I want a Burstable QoS.
* Generally, if it is an orchestrator service, or a pure IO based service, load tests don't affect them much in terms of CPU/Memory.
* If the service has any CPU work, I tend to lean towards more 'Guaranteed' QoS.

Another consideration is NodeJS services vs .NET services. Having smaller ndoejs pods is a very simple way to get multiple processes to serve your requests - much better than one big pod serving requests, especially if they also do some CPU bound work.

## Relation between CPU and memory

One very important aspect is to find the relation between CPU and Memory in these load tests. As long as they are linear (mostly the case), it is easy to pin the constraint values by measuring one of them (typically CPU) and changing the other appropriately. If for some reason they are skewed, then you need to profile them separately, and this would become complex.

## Special cases

In the application I'm working on, there are these special "engine" sevices, which are pure C# algorithms, complex stuff. Each request could run for minutes, have a high degree of multi-threadedness in them. Such special cases might need different strategies.
