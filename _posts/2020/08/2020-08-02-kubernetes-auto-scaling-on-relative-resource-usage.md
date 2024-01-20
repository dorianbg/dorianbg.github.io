---
title: "Kubernetes auto-scaling on relative resource usage"
date: "2020-08-02"
categories: ["load-balancers"]
tags: ["kubernetes"]
---

I am working on auto-scaling in Kubernetes and I was seeing something weird in my system and only now I realised that auto-scaling based on relative metrics (like CPU usage) really has to be done on requests instead of limits since the requested resource usage is guaranteed but the area between a limit and the requests usage is not guaranteed and depends on Kubernetes scheduler, if there are overcommits and etc (described well in [this](https://www.magalix.com/blog/kubernetes-resource-requests-and-limits-101) article[)](https://www.magalix.com/blog/kubernetes-resource-requests-and-limits-101)

So if the CPU usage for the auto-scaler is calculated as:

```
sum(cpu_usage_per_container/cpu_limits_per_container)
```

then if you don't have enough resources on each node to satisfy all the limits, when you scale up the number of instances, you really only scale up the limits without necessarily giving the application a boost due to potential overcommits and scheduling/prioritisation complexities that brings.

In the end I changed the definition of the deployment resource I want to scale so that  limits = requests but the most important bit is to calculate CPU usage based on the requested resources and not limits.

```yaml
containers:
  - name: webapp
    image: ...
    imagePullPolicy: ...
    resources:
      # ensure limits are the same as requests !!
      # the scheduler doesn't guarantee that the limit resources will be honored
      limits:
        memory: "200Mi"
        cpu: "200m"
      requests:
        memory: "200Mi"
        cpu: "200m"
```
