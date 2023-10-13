---
layout: post
title: "Rust vs Golang "
date: 2023-08-20 18:27:10 +0200
tags: rust, golang, performance
comments: true
---

Yet another one rust vs go comparison, but I wanted to check for myself and have a bit of the rust dev experience. The idea is to create a simple endpoint and test it out with benchmark tools. Of-course rust would be better just wanted to have my first rust dev experience and let me tell you it took me a few weeks to get it running. The biggest problems developers have today is the ability to unlearn things that they considered to be solid and live by and learn new concepts. It seems to me it is hard to let go of things that took you a lot of time to learn and now that you have finally felt comfortable using the inheritance and design patterns with your classes and building concepts with them you would need to forget all of that. Simple things like DI, dependency injection that is more or less present in all languages and widely accepted might be troublesome to implement using Rust and even more trying to have the same code structure of handlers/controllers, services or repositories.

In both Rust and Golang you would have singleton objects that live as long as your server does and are shared among requests. In other interpreted languages usually you'd hit an endpoint and it has to build everything from scratch, all the database objects then your other business logic objects and so on. But the difference in Rust is the borrow checker you cannot just simply pass objects as you'd normally do, an object can be taken ownership of and be dropped at the end of that function and you cannot use it anymore in the outer scope. You can also pass it as reference and the function will borrow it, but it might not live long enough, meaning the original object you are borrowing from has been dropped and your reference points to nothing on the heap. Data is usually stored on the heap and the pointers or anything fixed like primitives on the stack as that one is faster than accessing it from the heap. So, what usually happens in situations when something is borrow but being dropped before, you can either clone it, that would actually make a copy or use a lifetime. With lifetimes you can pass-in the reference but again the main object can be dropped. Such a case is having a database object and I need to clone that object when passing into different router objects for different routers that I wanted to create. If I use lifetimes and borrow the db object and create this db object in the main.rs the `axum::Server` will block hence the db object is never dropped then why would it say that `borrowed value does not live long enough`? Anyhow I did run a single endpoint and have no need to clone it but I would've if I had added another route. Another confusion about this is that if you use lifetimes then you use the `<’a>,` but that object you might want to DI into another object so you pass-in a lifetime object into another lifetime, seems a bit dubious.

In Rust you can have multiple packages and have them referenced, but the libraries that are related need to be the same version. So, you would usually have migrations as a separate package and I personally prefer sea ORM that would put the entities as well as a separate package.

The error handling is a bit different in Rust than golang and I have to admin it is more likable instead of always having to check the `err != nil` in golang.

Using traits to have a polymorphism is yet another thing I am used to and Rust does support it. Rust would probably do different variations on compile time and build a big folder to run so the polymorphism won't take anything from the runtime. This is true also for Generics and while in Golang most of the things happen on runtime Rust does most of the things on compile time. Generics make more sense in Rust, whereas in Go it does offer that type safety but they look strange and you would think they improve performance, they do not. The strongest take on using Generics in Go is reusability.

For performance sake I run some tests with and without cloning the database, having the case with clones have the trait and all. It really shows no perf impact maybe in different test runs in comparison to having a single handler endpoint that will call a service method that does the query from db. But, in reality I would prefer to have a trait and a struct implementing those functions. In fairness I do have the impression between multiple test runs that handler function without cloning or traits might be slightly faster.

I have now compared the test runs between Golang vs Rust endpoints and they do nothing special really. Golang uses GORM and Rust uses sea ORM to query a single record from a database. Golang uses the gin framework, Rust uses the axum framework. The endpoints are exposed from the terminal, that's how I run the app, no proxies/nginx is used. Both apps are run in release mode.

I have run golang in release mode from terminal using:

```
go run main.go
export DB_HOST=127.0.0.1
export DB_USER=shop
export DB_PASSWORD=shop
export DB_NAME=shop
export DB_PORT=5483
export APP_DEBUG=false
export APP_PORT=9000
```

for Rust running did this:

```
login to db server and create testrun database.
insert into post (id, name, title, text, created_at) values (1, 'test', 'test', 'test', '2023-10-01');
sea-orm-cli migrate up
cargo build --release
cargo run --release
```

Configuring the DB connection pool is done for both Go and Rust apps with `max_open_connections` set to 25 and `max_idle_connections` set to 25. This is actually very important so the db connections are being reused among the requests when running tests with high concurrency, otherwise there are errors such as too many clients open.

```
Golang:
ab -n 1000 -c 100 http://127.0.0.1:9000/api/test/1

Concurrency Level:      100
Time taken for tests:   0.088 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      154000 bytes
HTML transferred:       31000 bytes
Requests per second:    11369.71 [#/sec] (mean)
Time per request:       8.795 [ms] (mean)
Time per request:       0.088 [ms] (mean, across all concurrent requests)
Transfer rate:          1709.90 [Kbytes/sec] received

wrk http://127.0.0.1:9000/api/test/1

Running 10s test @ http://127.0.0.1:9000/api/test/1
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.12ms    3.92ms  89.90ms   99.11%
    Req/Sec     6.05k   566.23     6.67k    94.06%
  121453 requests in 10.10s, 17.84MB read
Requests/sec:  12025.29
Transfer/sec:      1.77MB
```

```
Rust
ab -n 1000 -c 100 http://127.0.0.1:9900/api/posts/1

Concurrency Level:      100
Time taken for tests:   0.269 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      139000 bytes
HTML transferred:       31000 bytes
Requests per second:    3723.31 [#/sec] (mean)
Time per request:       26.858 [ms] (mean)
Time per request:       0.269 [ms] (mean, across all concurrent requests)
Transfer rate:          505.41 [Kbytes/sec] received

wrk http://127.0.0.1:9900/api/posts/1

Running 10s test @ http://127.0.0.1:9900/api/posts/1
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.26ms    5.51ms 117.30ms   97.91%
    Req/Sec     3.12k   446.91     3.45k    96.04%
  62798 requests in 10.10s, 8.32MB read
Requests/sec:   6217.17
Transfer/sec:    843.93KB

```

What we’re seeing here is that Golang is twice faster than Rust when running from the terminal. Go serves around 121K requests in 10s with 6K requests per second and 8ms for a request. Rust serves around 63K requests in 10s with 3K requests per second and 26ms for requests. How is this so? Wasn't Rust supposed to be a lot faster?

Let’s try a slightly different test. I have created a kubernetes cluster using minikube locally. So I am using the same setup for both Golang and Rust. I have built images for the apis, pushed them to the registry and k8s will pull them. For both setups I have the exact same metrics in terms of memory and cpu limits for horizontal scaling and have 2 pods for each to begin with. I have also tried to test them using nginx on top, but that gives very similar results as with k8s. With the k8s setup I am using ingress-nginx, but these results are based after I run `minikube service rust-api` to run the service on the api pods.

```
Golang:
wrk http://127.0.0.1:60074/api/test/1

Running 10s test @ http://127.0.0.1:60074/api/test/1
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     7.90ms   17.32ms 243.56ms   90.88%
    Req/Sec     1.65k   298.51     2.03k    90.10%
  33093 requests in 10.10s, 4.86MB read
Requests/sec:   3275.82
Transfer/sec:    492.65KB

ab -n 1000 -c 100 http://127.0.0.1:60074/api/test/1

Concurrency Level:      100
Time taken for tests:   0.670 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      154000 bytes
HTML transferred:       31000 bytes
Requests per second:    1492.24 [#/sec] (mean)
Time per request:       67.013 [ms] (mean)
Time per request:       0.670 [ms] (mean, across all concurrent requests)
Transfer rate:          224.42 [Kbytes/sec] received
```

```
Rust
wrk http://127.0.0.1:65432/api/posts/1

Running 10s test @ http://127.0.0.1:65432/api/posts/1
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     3.92ms   10.39ms 162.96ms   94.03%
    Req/Sec     3.20k   768.90     3.94k    87.44%
  63523 requests in 10.02s, 8.42MB read
Requests/sec:   6339.12
Transfer/sec:    860.49KB

ab -n 1000 -c 100 http://127.0.0.1:65432/api/posts/1

Time taken for tests:   0.549 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      139000 bytes
HTML transferred:       31000 bytes
Requests per second:    1821.79 [#/sec] (mean)
Time per request:       54.891 [ms] (mean)
Time per request:       0.549 [ms] (mean, across all concurrent requests)
Transfer rate:          247.29 [Kbytes/sec] received

```

This would be the setup we would normally use in production using a kubernetes cluster. In this setup Rust is twice better with serving 63K requests in 10s (we don’t see a difference here with the test when Rust was run from the terminal) with 3.2K requests per second and 54ms for a request. Golang serves 33K requests in 10s (huge drop from the test when run from terminal) with 1.6K requests per second and 67ms for a request. I am kind of perplexed with these results, the k8s setup is probably the one I can trust more, but still don’t know the Golang case when run from the terminal as I was not able to find any errors.

There's this point that you would use Rust in case you have high memory cases or cpu for that matter, but isn't everything now highly consuming. Golang hasn't yet still picked up to the point I was hoping it would, rather than talking about rust ever becoming popular, but if people continue adopting it for these edge cases I am sure they will start using it for any other tasks as they get used to the concepts.
