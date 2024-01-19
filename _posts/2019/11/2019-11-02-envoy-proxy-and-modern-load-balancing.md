---
title: "Envoy proxy and modern load balancing"
date: "2019-11-02"
categories: 
  - "software-engineering"
---

I spent a bit of  time looking into alternative load balancing solutions to Nginx/HA Proxy. It seems Envoy is by far the leader in this space and the product seems really good.

Probably the best article on envoys features is [here](https://blog.envoyproxy.io/introduction-to-modern-network-load-balancing-and-proxying-a57f6ff80236). Envoy has good support for basic load balancing [algorithms](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers): -  Weighted round robin - Weighted least request - Random And a few extra algorithms but not that different really from Nginx/HA Proxy: - Ring hash - Maglev Of other features, Nginx/Envoy/HA Proxy are mainly the same. Another [article](https://blog.getambassador.io/envoy-vs-nginx-vs-haproxy-why-the-open-source-ambassador-api-gateway-chose-envoy-23826aed79ef) compared them and had mainly the same conclusion as I have below.

An advantage of Envoy is that it can be configured dynamically over a REST API if you build a management server to which you talk over a REST API. But this feature does not seem very mature and I couldn't find many implementations.

Another advantage I can see of Envoy over nginx is better observability/metrics and health checks. You can get metrics of how many requests [have been sent](https://www.envoyproxy.io/docs/envoy/latest/operations/admin) to a specific host. The "GET /clusters" includes all discovered upstream hosts and seems to be able to give per host statistics on total requests and etc. which could be fairly useful for monitoring.

HA Proxy also has solid monitoring, observability and health checks. Nginx really doesn't offer any of these outside of Nginx plus or external modules you have to build Nginx with which makes a deployment more difficult.

The service mesh concept (eg. Istio built on top of Envoy) looks interesting as it allows you to do monitor everything going on in a distributed system including getting great metrics and tracing but it still feels like a concept that hasn't really been adopted widely or proven itself yet. Probably best to wait on it for now and get it to mature.
