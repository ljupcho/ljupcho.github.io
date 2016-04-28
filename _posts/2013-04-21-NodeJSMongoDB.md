---
layout: post
title:  "Multiple queries per page load with NodeJS and MongoDB"
date:   2013-08-21 21:18:10 +0200
categories: nodejs
tags: nodejs mongodb
comments: true
---	

So, I started using NodeJS with express framework, so I could first figure out how it does things and then make some performance tests to see if it is worth the hustle. I've combined it with mongodb and my plan was first to insert data in database and see the behaving as I increase the volume of it. I wanted to call multiple queries with one page call and the way nodejs handles them is like fire them sequentially, continue executing the script and after the db finishes with the queries it will print out the results. It means this:


{% highlight js %}
exports.index = function(req, res){
	var all_results=[];
	db.users.find({'where':'below 3000'}).sort({'number':1}).toArray(function(err,data){
		accumulateData(data);
	});
	db.users.find({'where':'above 6000'}).sort({'number':1}).toArray(function(err,data1){
		accumulateData(data1);
		render(all_results);
		//console.log(all_results);
	});

	console.log(all_results.length);
	var render =  function(data) {
		res.render('index', { title: 'Title',users: data});
	}
   
	var  accumulateData = function(arr) {
		for (var i=0;i<arr.length;i++) {
			all_results.push(arr[i]);
		}
	}
}
{% endhighlight %}


Here, I am trying with two queries, the first one should give me 3000 rows and the second one 4000 rows, by the total I've inserted. The thing here is that the console.log (the uncommented one) will print out total of 0 of the array length. It is like that because the script doesn't wait for the results from the database, furthermore it doesn't sync the results per query. If I use the commented console.log it will give the the total number of rows. To finalize my point I use the following example, for the first query I would use this:

{% highlight js %}
db.users.find({'where':'above 100000'}).sort({'number':1}).toArray(function(err,data){
	accumulateData(data);
});
{% endhighlight %}

First, I have inserted 100000 new records, because I wanted to know what nodejs would do if the first query has more db work or more rows to fetch. In this case the console.log, the one the print out the total length would give me 4000, only the number of the second query! This means that it doesn't wait for the results of the first query, even though it is fired first, but because it required more processing time, they are not in the total. 
So, I would move the render function in this query and this time it will give the results from both queries, I had to put the print out function in the most consuming query to get all the rows from all the queries. And if I have a lot queries to process per page load, how would I find the “heaviest” query and call the render function there? To make sure that all will be in total and I wouldn't guess which query makes the most cpu usage I do this:

{% highlight js %}
db.users.find({'where':'above 100000'}).sort({'number':1}).toArray(function(err,data){
	accumulateData(data);
	db.users.find({'where':'above 6000'}).sort({'number':1}).toArray(function(err,data1){
		accumulateData(data1);
		render(all_results);
		console.log(all_results.length);
	});
});
{% endhighlight %}

This way I am forcing the second query to run after I get the data from the first and merge the results together, which is basically running one after another. What's the point, don't quite get the benefit of non-blocking IO in this case.
