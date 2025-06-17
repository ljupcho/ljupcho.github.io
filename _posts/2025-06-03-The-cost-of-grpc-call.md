---
layout: post
title: "The cost of a grpc call"
date: 2025-06-03 11:39:10 +0200
tags: golang, kubernetes, microservices, grpc, performance
comments: true
---

In this blog post I am looking more into inner-server communication between golang microservices. Obviously there is a cost to having multiple services that need to exchange data rather than having one bigger service.
But, not going too much into the micro-services discussion here, I would prefer having smaller services for lots of reasons. Structuring them is an art on its own and deciding what should be service and what kind. I would say there are three main types of services: api, worker and consumer. Having a MQ to load balance between the same service that gets autos-laced and fan-out on the same topic for different services. That's all there is to it.

First, I am having an api that holds the data and running requests directly to it normally would give us the best performance results.
Second, I have another api that will send grpc calls to the first api, which will have a grcp server running. A 2.1 test case would be native grcp server versus http multiplexing.
Third, I want to have yet another type of calls, through NATs that are quite flexible and have a request-reply pattern.

The results are as expected. Running benchmarks on mac laptop.

```
// Sending requests from api02 to api01 with http2 multiplexing.
➜  goapp02-api git:(main) ✗ wrk http://localhost:9102/api/v1/products
Running 10s test @ http://localhost:9102/api/v1/products
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.73ms    2.89ms  55.29ms   89.26%
    Req/Sec     2.23k     1.14k    3.54k    52.50%
  44421 requests in 10.01s, 15.50MB read
Requests/sec:   4436.43
Transfer/sec:      1.55MB

// Sending requests from api02 to api01 with grpc native server.
➜  goapp02-api git:(main) ✗ wrk http://localhost:9102/api/v2/products
Running 10s test @ http://localhost:9102/api/v2/products
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.34ms  676.81us  36.12ms   97.60%
    Req/Sec     3.80k   193.26     4.14k    89.60%
  76397 requests in 10.10s, 26.67MB read
Requests/sec:   7562.07
Transfer/sec:      2.64MB

// Sending requests from api02 to api01 with nats server.
➜  goapp02-api git:(main) ✗ wrk http://localhost:9102/api/v3/products
Running 10s test @ http://localhost:9102/api/v3/products
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     6.14ms    0.98ms  29.30ms   96.78%
    Req/Sec   820.94     49.18     0.87k    94.50%
  16351 requests in 10.01s, 5.71MB read
Requests/sec:   1633.64
Transfer/sec:    583.90KB

// Sending requests directly to api01.
➜  goapp02-api git:(main) ✗ wrk http://localhost:9101/api/v1/products
Running 10s test @ http://localhost:9101/api/v1/products
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     0.99ms  308.60us   9.49ms   86.75%
    Req/Sec     5.06k   266.24     5.42k    88.61%
  101744 requests in 10.10s, 35.51MB read
Requests/sec:  10073.47
Transfer/sec:      3.52MB
```

| Method              | Avg Latency | Max Latency | Req/Sec   | Total Requests | Transfer/Sec |
|---------------------|-------------|-------------|-----------|----------------|--------------|
| HTTP/2 Multiplexing | 2.73ms      | 55.29ms     | 4436.43   | 44,421         | 1.55MB       |
| gRPC Native Server  | 1.34ms      | 36.12ms     | 7562.07   | 76,397         | 2.64MB       |
| NATS Server         | 6.14ms      | 29.30ms     | 1633.64   | 16,351         | 583.90KB     |
| Direct to api01     | 0.99ms      | 9.49ms      | 10073.47  | 101,744        | 3.52MB       |


Analysis: <br/>
- Direct to api01 is the fastest, with the lowest average latency (0.99ms) and highest throughput (10,073.47 req/sec, 3.52MB/sec). It also has the lowest max latency (9.49ms), making it the most efficient.

- gRPC Native Server performs strongly, with a low average latency (1.34ms) and high throughput (7,562.07 req/sec, 2.64MB/sec). It’s a good balance of speed and scalability.

- HTTP/2 Multiplexing is slower than gRPC, with higher latency (2.73ms) and lower throughput (4,436.43 req/sec, 1.55MB/sec). Its max latency (55.29ms) is the highest, indicating potential bottlenecks under load.

- NATS Server is the slowest, with the highest average latency (6.14ms) and lowest throughput (1,633.64 req/sec, 583.90KB/sec). However, its max latency (29.30ms) is lower than HTTP/2, suggesting more consistent performance under lighter loads.

