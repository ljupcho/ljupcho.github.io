---
layout: post
title: "Scaling worker environment with k8s"
date: 2023-09-27 21:39:10 +0200
tags: golang, kubernetes, docker, performance
comments: true
---

After having set up a consumer app as a service and setting it up under kubernetes cluster, I have set a minimum of 2 replicas with the possibility of horizontal scaling up to 8. The consumers, or worker service on start up makes a connection to postgres and rabbitmq and have some queueing jobs to be run. The api which is a separate service does the publishing of the jobs to the rabbitmq and the consumer services picks them up, based on the job type and payload builds the appropriate job and runs it.

I have set the consumer services as a separate service because I want it to scale horizontally under higher load so each pod will have its own connection to rabbitmq and the queues are represented by channels and goroutines. On higher load kubernetes would add another pod or more pods and each of them will create a new connection to rabbitmq with their queues/channels, so for example if i have a connection with 4 queues which will subscribe to exchange rabbitmq for example each by 5 channels/goroutines which will total of 20 goroutines running, effectively I have 20 consumers. After k8s will add me another pod this will duplicate. After the traffic subsides the pods should scale down and the number of goroutines will also double down.

The other part is the publishing of messages. For each of the api services connecting to the rabbitmq I am keeping a persistent rabbitmq connection with 4 channels for publishing. When requests come in on the api endpoints that publish to rabbitmq will reuse that connection. On high traffic the apis will be scaled up and new pods added and new connections for publishing created.

Both of the services, api and consumers connect to the postgres with setting max/min open connections, which makes a difference in the publishing as I would first get things from the database and then publish by ID to the queue.

To check the active open connection to postgres:

```
SELECT count(*) FROM pg_stat_activity;
```

<br/>

The way to monitor the k8s cluster adding and removing pods:

```
minikube addons enable metrics-server
kubectl top pods
```

<br/>

```
with setting connection pool on postgres:
ab -n 3000 -c 100 http://127.0.0.1:53942/api/test/1

Concurrency Level:      100
Time taken for tests:   6.065 seconds
Complete requests:      3000
Failed requests:        0
Total transferred:      2370000 bytes
HTML transferred:       1998000 bytes
Requests per second:    494.62 [#/sec] (mean)
Time per request:       202.177 [ms] (mean)
Time per request:       2.022 [ms] (mean, across all concurrent requests)
Transfer rate:          381.59 [Kbytes/sec] received

```

This means it will query and pulibhs 3K requests in 6 seconds.

<br/>

```
Before:
api-shop-deployment-dcbfc5877-b5jn7     13m          36Mi
api-shop-deployment-dcbfc5877-vrvk8     13m          35Mi
consumers-deployment-757d987d9d-dkd2j   15m          40Mi
consumers-deployment-757d987d9d-dq7tl   14m          39Mi
```

I am running the `ab` command now for publishing 3K requests to check the scaling of the pods:

<br/>

```
During, it got scaled:
api-shop-deployment-dcbfc5877-5dwvg     1m           52Mi
api-shop-deployment-dcbfc5877-b5jn7     1m           72Mi
api-shop-deployment-dcbfc5877-c4zv7     1m           73Mi
api-shop-deployment-dcbfc5877-vrvk8     2m           68Mi
api-shop-deployment-dcbfc5877-wphs9     2m           59Mi
consumers-deployment-757d987d9d-clm88   12m          40Mi
consumers-deployment-757d987d9d-dkd2j   1m           57Mi
consumers-deployment-757d987d9d-dq7tl   2m           55Mi
consumers-deployment-757d987d9d-dsv2h   2m           56Mi
consumers-deployment-757d987d9d-n6l56   2m           56Mi
consumers-deployment-757d987d9d-vdhfh   11m          39Mi
```

<br/>

After the traffic is processed by the consumers and api does not have anything more to publish, the pods go back to 2 for api and 2 for consumers. In the rabbitmq admin panel I can monitor the number of queues/goroutines going up and down based on the traffic need.
