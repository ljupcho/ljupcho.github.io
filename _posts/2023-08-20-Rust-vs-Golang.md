---
layout: post
title: "Rust vs Golang "
date: 2023-08-20 18:27:10 +0200
tags: rust, golang, performance
comments: true
---

Yet another one rust vs go comparison, but I wanted to check for myself and have a bit of the rust dev experience. The idea is to create a simple endpoint and test it out with benchmark tools. Of-course rust would be better just wanted to have my first rust dev experience and let me tell you it took me a few weeks to get it running. The biggest problems developers have today is the ability to unlearn things that they considered to be solid and live by and learn new concepts. It seems to me it is hard to let go of things that took you a lot of time to learn and now that you have finally felt comfortable using the inheritance and design patterns with your classes and building concepts with them you would need to forget all of that. Simple things like DI, dependency injection that is more or less present in all languages and widely accepted might be troublesome to implement using Rust and even more trying to have the same code structure of handlers/controllers, services or repositories.

In both Rust and golang you would have singleton objects that live as long as your server does and are shared among requests. In other interpreted languages usually you'd hit an endpoint and it has to build everything from scratch, all the database objects then your other business logic objects and so on. But the difference in rust is the borrow checker you cannot just simply pass objects as you'd normally do, an object can be taken ownership of and be dropped at the end of that function and you cannot use it anymore in the other scope. You can also pass it as reference and the function will borrow it, but it might not live long enough, meaning the original object you are borrowing from has been dropped and your reference points to nothing on the heap. Data is usually stored on the heap and the pointers or anything fixed like primitives on the stack as that one is faster than accessing it from the heap. So, what usually happens in situations like when something is borrow but being dropped before, you can either clone it, that would actually make a copy or use a lifetime. With lifetimes you can pass-in the reference but again the main object can be dropped. Such a case is having a database object and I need to clone that object when passing into different router objects for different routers that i wanted to create. If I use lifetimes and borrow the db object and create this db object in the main.rs the axum::Server will block hence the db object is never dropped then why would it say that `borrowed value does not live long enough`? Anyhow I did run a single endpoint and have no need to clone it but I would've if I had added another route. Another confusion about this is that if you use lifetimes then you use the “<’a>,” but that object you might want to DI into another object so you pass-in a lifetime object into another lifetime, seems a bit dubious.

In Rust you can have multiple packages and have them referenced, but the libraries that are related need to be the same version. So, you would usually have migrations as a separate package and I personally prefer sea ORM that would put the entities as well as a separate package.

The error handling is a bit different in Rust than golang and I have to admin it is more likable instead of always having to check the `err != nil` in golang.

Using traits to have a polymorphism is yet another thing I am used to and Rust does support it. Rust would probably do different variations on compile time and build a big folder to run so the polymorphism won't take anything from the runtime. This is true also for Generics and while in golang most of the things happen on runtime Rust does most of the things on compile time. Generics make more sense in rust, whereas in go it does offer that type safety but they look strange and you would think they improve performance, they do not.

For performance sake I run some tests with and without cloning the database, having the case with clones have the trait and all. It really shows no perf impact maybe in different test runs in comparison to having a single handler endpoint that will call a service method that does the query from db. But, in reality I would prefer to have a trait and a struct implementing those functions. In fairness I do have the impression between multiple test runs that handler function without cloning or traits might be slightly faster.

I have now compared the test runs between golang vs rust endpoint and they do nothing special really. Golang uses GORM and Rust uses Sea ORM to query a single record from a database. Golang uses the gin framework, Rust uses the axum framework. The endpoints are exposed from the terminal, that's how I run the app, no proxies/nginx is used. Both apps are run in release mode.

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

<br/>
for rust running did this:

```
login to db server and create testrun database.
insert into post (id, name, title, text, created_at) values (1, 'test', 'test', 'test', '2023-10-01');
sea-orm-cli migrate up
cargo build --release
cargo run --release
```

<br/>

```
Golang:
ab -n 1000 -c 50 http://127.0.0.1:9000/api/shops/1
Concurrency Level:      50
Time taken for tests:   0.666 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      245000 bytes
HTML transferred:       121000 bytes
Requests per second:    1500.96 [#/sec] (mean)
Time per request:       33.312 [ms] (mean)
Time per request:       0.666 [ms] (mean, across all concurrent requests)
Transfer rate:          359.12 [Kbytes/sec] received
```

<br/>

```
Rust:
ab -n 1000 -c 50 http://127.0.0.1:9900/api/postsnew/1
Concurrency Level:      50
Time taken for tests:   8.171 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      139000 bytes
HTML transferred:       31000 bytes
Requests per second:    122.38 [#/sec] (mean)
Time per request:       408.556 [ms] (mean)
Time per request:       8.171 [ms] (mean, across all concurrent requests)
Transfer rate:          16.61 [Kbytes/sec] received
```

<br/>

Golang time per request is 33ms, rust's is 9ms.
However there is a big twist in this, I have tweak the database connection pool settings for golang app:

```
sqlDB, err := DB.DB()
if err != nil {
	panic(err)
}

// SetMaxIdleConns sets the maximum number of connections in the idle connection pool.
sqlDB.SetMaxIdleConns(25)

// SetMaxOpenConns sets the maximum number of open connections to the database.
sqlDB.SetMaxOpenConns(25)

// SetConnMaxLifetime sets the maximum amount of time a connection may be reused.
sqlDB.SetConnMaxLifetime(5 * time.Minute)
```

and got these results:

```
ab -n 1000 -c 50 http://127.0.0.1:9000/api/test/1
Concurrency Level:      50
Time taken for tests:   0.151 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      145000 bytes
HTML transferred:       22000 bytes
Requests per second:    6624.75 [#/sec] (mean)
Time per request:       7.547 [ms] (mean)
Time per request:       0.151 [ms] (mean, across all concurrent requests)
Transfer rate:          938.08 [Kbytes/sec] received
```

<br/>
This yields same or better results than rust.

Now, let's move on to testing with `wrk` for both of the apps.

```
Golang:
wrk http://127.0.0.1:9000/api/shops/1
Running 10s test @ http://127.0.0.1:9000/api/shops/1
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     9.61ms   12.95ms  81.40ms   81.72%
    Req/Sec     1.71k   377.52     2.68k    68.50%
  33980 requests in 10.01s, 7.94MB read
Requests/sec:   3394.68
Transfer/sec:    812.20KB
```

<br/>

```
Rust:
wrk http://127.0.0.1:9900/api/postsnew/1
Running 10s test @ http://127.0.0.1:9900/api/postsnew/1
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.69ms    1.79ms  50.48ms   98.88%
    Req/Sec     3.19k   271.75     3.42k    94.55%
  64195 requests in 10.10s, 8.51MB read
Requests/sec:   6354.57
Transfer/sec:    862.58KB
```

<br/>

Golang serves 34K requests in 10s, Rust serves 64K in 10s. well it does make a big difference.

So there's this point that you would use Rust in case you have high memory cases or cpu for that matter, but isn't everything now highly consuming. Golang hasn't yet still picked up to the point I was hoping it would, rather than talking about rust ever becoming popular, but if people continue adopting it for these edge cases I am sure they will start using it for any other tasks as they get used to the concepts.

Edit:
However after changing the database connection pools for golang app, I get this:

```
Golang:
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

<br/>
So, this serves 121K requests in 10s, which is twice than my rust app. I don't know what's going on here, how do I get this results, is it something not configured right in my rust app?
The reason I have increased the pool in golang app and set the min and max connections parameters was that I was getting a lot of warnings for slow queiries, but after setting these up I am not longer getting them.
Now, I am looking at how to imporove the rust api to match the golang api, both api return the same result both query the postgres database by primery key and have set max connections the same. The research goes on...

Edit2:
Considing that the golang is fast running from terminal only, I have done metrics for using nginx or k8s on top.

```
Using nginx:
wrk http://localhost/api/test/1
Running 10s test @ http://localhost/api/test/1
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     5.02ms    4.53ms  91.66ms   95.24%
    Req/Sec     1.08k   146.91     1.28k    86.00%
  21572 requests in 10.01s, 3.97MB read
Requests/sec:   2155.56
Transfer/sec:    406.27KB

Concurrency Level:      100
Time taken for tests:   0.715 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      188000 bytes
HTML transferred:       31000 bytes
Requests per second:    1398.44 [#/sec] (mean)
Time per request:       71.508 [ms] (mean)
Time per request:       0.715 [ms] (mean, across all concurrent requests)
Transfer rate:          256.74 [Kbytes/sec] received

Using k8s:
wrk http://127.0.0.1:60074/api/test/1
Running 10s test @ http://127.0.0.1:60074/api/test/1
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     7.90ms   17.32ms 243.56ms   90.88%
    Req/Sec     1.65k   298.51     2.03k    90.10%
  33093 requests in 10.10s, 4.86MB read
Requests/sec:   3275.82
Transfer/sec:    492.65KB

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

Using nginx or k8 show similar results, but far behind from running the app from terminal. Might be some settings.
