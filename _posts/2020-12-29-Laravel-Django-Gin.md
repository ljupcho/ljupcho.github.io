---
layout: post
title:  "Laravel(php) vs Django(python) vs Gin(golang)"
date:   2019-12-19 16:08:10 +0200
tags: software architecture, php, python, golang, performance
comments: true
---	

## The environment setup

I am using dockers to set up each of the framework specific environment with same services. Tests are run on ubuntu desktop i5 6 cores 32GB RAM. Each of the environments is done the same way, with mysql database, web app that listens on port 9000, same nginx configuration as proxy to the web app. Each of the nginx containers is exposed to the host to specific port so I can run http request from the host machines.

*the php case:* <br/>
docker: https://github.com/ljupcho/docker-nginx-php<br/>
	php 8, php-fpm, opcache enabled, mysql v5.7.25, redis, consumers container for running queue jobs managed by supervisor
	./startup.sh creates the containers and prepares the environment like composer update, migrations, clears cache etc..
application: https://github.com/ljupcho/test-laravel<br/>
	Laravel v8.12<br/>

*the python case:* <br/>
docker: https://github.com/ljupcho/docker-nginx-python<br/>
	python 3.6, uwsgi, mysql v5.7.25, redis, consumers container for running celery/queue tasks managed by supervisor
	./startup.sh creates the containers and prepares the environment like install requirements, migrations etc..
application: https://github.com/ljupcho/test-django<br/>
	Django v2.2.3<br/>

*the golang case:* <br/>
docker: https://github.com/ljupcho/docker-nginx-golang <br/>
	golang 1.14.13, mysql v5.7.25 <br/>	
application: https://github.com/ljupcho/docker-nginx-golang/tree/master/morningo <br/>
	Gin-gonic v1.6.3

Each of the framework support ORM that handles queries and migrations as I wouldn’t really run queries on my own so good ORM was a requirement. It was important that i have all the database structure for all 3 database servers, same type of columns with same type of indexes and foreign keys. In case of laravel and django I am using redis to store the queue jobs whereas for golang I am using goroutines with channels to accomplish concurrency. 

## Case 1 (Imports)
I want to see how the frameworks would perform in a task like bulk imports, a task I have had a few many times on real projects like import csv files. I would skip the part of the actual upload of the file and test the time it takes for each framework to import 20K rows of data.

Insert is done by chunks of 500 for all cases. Changing the chunk did not have much difference but 500 seems to be optimal. Number of processes is 6, as i manage that with supervisor and verify it with `ps aux` on each of the consumers containers.

The golang case is the most interesting. We have concept of goroutines that are threads that run in parallel, same thing as the workers we have in laravel/django queues, but in this case they can communicate via channels and in golang case they are not exactly processes, but there's a better memory management. The example is [here](https://github.com/ljupcho/docker-nginx-golang/blob/master/morningo/controllers/MainController.go#L194). The idea behind this is that we are creating a channel that we push data to and continuously read from in our concurant threads/goroutines. After all imported we can continue with the rest of the logic which in the laravel or django case is not possible without making a workaround as there we have abstraction layer with processes and normally we will need to additional check (either db/redis related) to figure out that all the processes have finished or use `async` packages. In golang case I have the flexibility to write concurrent code as I find fit not just running queries or other logic one by one.

### Insert into single table only
The difference between inserting into single table or multiple tables that are in some kind of a relationship is that in the first case i don't need the objects back in order to make the relationship, but the frameworks still provide methods that i would insert into a table and it would return me only the id instead of whole object which implies one additional query and inserting 20K would actually double the number of queries and the time it takes for the whole operation. That's why it is crucial to make sure that only queries that are needed are run.

| Laravel | Django | Gin
| :--- | :--- | :---
| 20s | 22s | 21s

The basic idea behind this is that you want to push most of the heavy logic in your queues and process as background job as your code is in RAM and makes no difference if you're using a compiled or an interpreted language. In such case the results show that the database is the bottleneck here and in these 3 cases the database is same so makes little to no difference. The management of the processes is also adjustable to your needs and resources at your disposal, meaning if you want to run the workers on the same servers you're serving users on or totally separate worker environment. Load balancing over multiple servers with restricted number of processes in most cases is fine.

### Insert into multiple tables
Next, I have a nested case, more realistic scenario when a user can belong to a group and the user can have many posts. I seed the database first with 300 groups and pick one to set the FK in users for a group and for each user i create 2 posts. So, in total here it happens to have 60K inserts and 40 selects for finding the group, as for job i am running 1 select and 1500 inserts (chunks size by 500 - 1 for the user and 2 for posts for that user).

| Laravel | Django | Gin
| :--- | :--- | :---
| 65s | 67s | 30s

Well golang is outperforming both of them even though queries are the same.


## Case 2 (API endpoint)

The point here is to run concurrent requests increased over time on an endpoint that returns a json. The database structure with indexes and FKs is the same for all three and the response looks like this for all three as well:

![response-json]({{ site.url }}/public/images/response-json.png)

First i am sending 1000 request with 100 concurrent, than 3000 request with 100 concurrent.
```
laravel: `ab -n 1000 -c 100 http://localhost:7081/api/getUsers/`
django: `ab -n 1000 -c 100 http://localhost:6061/todolist/getUsers/`
gin: `ab -n 1000 -c 100 http://localhost:10091/api/getUsers`
```

On a single request the queries that are run are:
```
select * from `users` where `users`.`deleted_at` is null order by `users`.`id` desc limit 50;
select * from `groups` where `groups`.`id` in ({group_ids}) and `groups`.`deleted_at` is null;
select * from `posts` where `posts`.`user_id` in ({post_ids}) and `posts`.`deleted_at` is null;
```
<br/>
I am using already imported users with groups and posts from the previous test case. I am pre-loading the group and posts relationships with the user what makes them `whereIn` queries with passed ids. With using a join for the group i am down to 2 queries per request instead of 3 but not much is changing in terms of the response times. But, it does matter if I would loop through the users and run a query for the group and since I am fetching 50 users that many queries for the group multiple by the concurrent request would make a big difference even doubling the times.

Here i would stick to the 3 queries per request with pre-loading the group and the posts in all three frameworks.

|num of req/con req.  | Laravel | Django | Gin
|:---:		      | :--- | :--- | :---
|1000/100	      | 2.9s | 7.7s | 1.4s
|3000/100	      | 7.8s | 16.6s | 4.1s

Again golang is a clear winner, even when i refresh the page in the browser for this response I get times in the range 10-20ms, for laravel in the range 30-40ms, and django in the range 40-50ms.
Moreover that was expected and coincides with other results on net, but i have to admit was pleasantly surprised by laravel performance slightly better than i had expected and django is just not performing well even though i have the .pyc files, debug false etc..
For the php and python to compete with golang in such case when your API gets hit by concurrent users and you get cpu spikes, most probably the first thing you can do is optimized your queries, make sure you have all the in-build caching mechanism enabled, the right nginx configuration. The faster approach would be to put more servers under load balancer, but ultimately in such cases in order to improve your performance you have to cache things. That means if you're serving static sites you can cache and server html either nginx cache or varnish cache will do, but in other cases you would want to have the most frequent data cached in redis (in most cases). Most of the counter you have that comes from count, group by should be already compiled using a cron job most probably. 
Because these results show that if I have one server for golang i would need 2 for php and 4 for python to serve the same number of users, you'd have a load balancer to put more servers to make sure you don't get timeouts. The point is that if you have come to having 4 servers most probably having them means very little cost to your business anyway.









