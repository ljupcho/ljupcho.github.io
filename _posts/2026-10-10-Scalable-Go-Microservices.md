---
layout: post
title: "Scalabele Go Microservices"
date: 2026-10-10 19:39:10 +0200
tags: golang, microservices, kubernetes, consumer, api, grpc
comments: true
---
I will be examining two approaches of syncing data that might come from third party or when we just want to move data or do some aggregation. The setup would be a trigger action would dispatch a job to the queue, a consumer will pick it up and start the sync. The sync would include fetching user IDs and send batches of user ids for which we would do querying in database and make a aggregated view. Each of the batches have to iterate for each user id and do a hash generation after fetching the user from database. We have 3M users to sync.

- Configuration about stravasync-api gRPC endpoints
- Configuration about stravaysnc-api DB connection pool
- Configuration about stravasync-api http endpoint for triggering the sync job
- Configuration about stravasync-consumer
- Configuration about stravasync-db-consumer
- Configuration about helm charts and deployment
- Configuration about Istio for load balancing of gRPC calls
- Configuration about prometheus and grafana with loki to check logs

<b>Approach 1: Have one standalone service that will fetch the IDs and query the database itself.</b>

![dbconsumer]({{ site.url }}/public/images/db-consumer-nats.png)

We can set up multiple workers for this approach so that multiple batches would send requests to the database. In this case the service holds the db connection. The question here is what happens if the cpu of the pod increase and eventually k8s kills off the pod. Answer to that would be simple increase the limits of the pod. If this synchronization happens every hour or so, or once a day, then we would be paying for resource that we won't be needing all the time just in the time of the sync, basically we top up just to facilitate the spike of the sync.

Running db-consumer with 10 workers/go-routines, 10K fetched data and 500 batchSize per worker
The db-consumer is going over the 100% and k8s adds another pod that does nothing due to the high concurrency i have set:

`kubectl get all -n stravasync`
```
horizontalpodautoscaler.autoscaling/stravasync-db-consumer-dev   Deployment/stravasync-db-consumer-dev   117%/75%, 42%/75%              1         10        1          2m1s
```
and we are risking that the k8s will kill off this pod any time or slow it down, the avg will say 60% due to 2 db-consumers pods but only the one is working. So, the db-consumer got killed off after 8mins of running and managed to insert 1.8M records, like 60% of 3M that should be the total job.
This means we need to lower the concurrency in order not to make the cpu spike.

Running db-consumer with 3 workers/go-routines, 10K fetched data and 500 batchSize per worker
Even with 3 workers the cpu again jumped over:
```
horizontalpodautoscaler.autoscaling/stravasync-db-consumer-dev   Deployment/stravasync-db-consumer-dev   110%/75%, 42%/75%   1         10        2          4m31s 
```
and k8s added another pod. The new pod lowers the percentage but does not do anything. The worst part about this is that you have to make sure that you run the full sync on this one pod and that it won't crash. It managed to run fully even tho it added one more pod with:
```
{"time":"2026-09-26T15:19:26.750982138Z","level":"INFO","msg":"success process","time":"26m16.137008716s","total":"1000000"}
```
otherwise you will have to bump that one pod/service with lots of cpu and pay for it like that.

<b>Approach 2: Distribute the load over multiple api pods via gRPC calls</b>

![consumer]({{ site.url }}/public/images/scalable-api-grpc-nats.png)

-	Decoupling. The consumer (worker) pulls jobs from a queue (in this case NATs message broker) and offloads the actual DB-heavy work to dedicated API pods via gRPC. This keeps the consumer lightweight and focused on orchestration/batching. Here as well we would be sending batches of IDs, so each gRPC call to the api would be payload of IDs that when the request reaches the api will do the database work.
-	Load Balancing. We want to load balance between api, so that when the consumer which can have multiple workers/go-routines sending to the apis all those gRPC requests, we would have increse in number of api pods in the time of the sync. This way we facilitate for the cpu spike on the api side. The consumer will remain quite as it only sends gRPC calls. We pay resource only for the time of the sync.
-	Fault tolerance: We can have retries on batch and since we can have multiple api pods we are in low risk that the process will break down.

Running consumer with 6 workers/go-routines, 10K fetched data and 500 batchSize per worker
I started out with 4 api pods, the cpu got to 70% so it added one more api pod, now I have 5 pods with cpu around 60%.
Only once consumer is being activated to send gRPC calls to the apis. The consumer is sleeping in terms of cpu.
We have 6 workers to send requests the api pods got auto-scalled, fetch 10K per iteration and spread that out to 6 go-routines per 500 batch.
```
horizontalpodautoscaler.autoscaling/stravasync-api-dev        Deployment/stravasync-api-dev        62%/60%, 35%/60%   4         10        5          5m38s
```
finish the job in:
```
{"time":"2026-09-26T17:29:59.249193763Z","level":"INFO","msg":"success process","time":"15m45.630512049s","total":"1000000"}
```
after the cool down it goes back to original state of pods (4 pods).
db-server around 30-40%.

One aspect and probably most important is not the time of the sync, how quickly you get the the sync done, but how correct it is, if you've done it without errors and failed states and if the CPU was not throttling. In my example I've managed to speed it up and finish the sync 10min quicker because i was able to just add more pods in the time of the sync. You could potentially do the same with adding more cpu for the first approach for the db-consumer, but you will always have the fear of the pod crashing.

Most of the app have CPU problem that could slow things down tremendously. When you have lots of customer and data to consume and aggregated the most important part would be to have stable and resilient system so adding/onboarding new customer won't necessarily mean just put more ram and cpu on it.




