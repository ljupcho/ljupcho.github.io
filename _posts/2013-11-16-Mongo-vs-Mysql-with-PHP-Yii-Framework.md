---
layout: post
title:  "MongoDB vs Mysql with PHP Yii Framework"
date:   2013-11-27 21:18:10 +0200
categories: php
tags: mongodb mysql php yii
comments: true
---	

Graphics show the results I got from measuring response time using mysql and mongo with php yii framework. First I took Yii so I could get an environment up and running quickly and concentrate on the results. I inserted 110000 records in both databases and did simple selects to print data out. I print out 6 sqls with measuring time for each of them with incrementing the total. 

The script I use is [here](https://github.com/ljupcho/php/tree/master/mognoClient)

1. Using the MongoYii extension from http://sammaye.github.io/MongoYii/ , this require setting in the main.php of the config folder in yii. 
2. I use MongoClient from php 
3. I use yii to set mysql driver in config folder 
The results show that I increase the number of Records, also I show the size of it, and the time of the response. 


![mongovsmysql0]({{ site.url }}/public/images/mongovsmysql0.png)

This picture shows that using mongoYii extension didn't give the results I was expecting and it won't justify the mongo hype. It shows that it will process the nearly 60,000 records for 8.5 seconds which is by far a lot more than acceptable. 
If I turn off the extension and use native MongoClient that comes with php, I get this:


![mongovsmysql1]({{ site.url }}/public/images/mongovsmysql1.png)

Here we see that mongo shows better results than mysql and it is even lower than a second for the 60K records. That's more like it.
