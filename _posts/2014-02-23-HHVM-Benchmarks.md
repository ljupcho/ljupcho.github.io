---
layout: post
title:  "Install HHVM & benchmarks"
date:   2014-02-23 21:18:10 +0200
categories: hhvm
tags: hhvm
comments: true
---	

HHVM, facebook hiphop virtual machine gets a lot of popularity recently due to good performance results. There are some benchmarks made by people showing how good hhvm is as a result of non-blocking I/O and better caching. It is also result of hiphop bytecode, the caching mahanism that is implement that translates eventually everything to c++ but in this case hhvm goes one step further instead of just keeping the php scripts in memory as apc does.

Here, I will try to do the same, but on different machines and in different scenarios. My idea was to see if I can make hhvm work with SugarCRM, since I beleave it can help us a great deal in cutting down the page load, the response.

There are couple of scenarios to try out:

- using the usual mod_php that we normally go by default in lamp stack
- cgi mode or php-fpm as a separate process outside from apache
- using cache with apc or it's replacement opcache
- using hhvm with mod_proxy_fcgi apache module

Of course there are more scenarios if you combine from above. One of the differences between these caching types and hhvm is that apc/opcache will store the files in memory whereas hhvm has sqlite database on its own. hhvm won't get best results in first run, it has so called warm-up period.
Good articles about hhvm: 

[http://www.hhvm.com/blog/1817/fastercgi-with-hhvm](http://www.hhvm.com/blog/1817/fastercgi-with-hhvm)
[https://github.com/facebook/hhvm/wiki/FastCGI](https://github.com/facebook/hhvm/wiki/FastCGI)
[http://wiki.apache.org/httpd/PHP-FPM](http://wiki.apache.org/httpd/PHP-FPM)

In order to install hhvm, first you need to make sure you got the mod_proxy_fcgi mode in your apache. To have that you need apache 2.4 version, in some distributions when you do an update of the system it gets to 2.2. Apache is also suppose to forward the request to hhvm as a separate server and hhvm will take care of the php code, as apache should just be just a proxy and you set it in the apache config file. As stated in the article above you should have something like this:

<code>
	ProxyPass / fcgi://127.0.0.1:9000/srv/http/
</code>

, and run from document root:

<code>
	sudo hhvm --mode server -vServer.Type=fastcgi -vServer.Port=9000
</code>


First, I tried instlling hhvm on ArchLinux. The apache version is 2.2 so I needed to install apache24 from AUR. The hhvm package I used is from AUR, got it from [here](https://github.com/mtorromeo/archlinux-packages/tree/master/hhvm) , since previous version got some errors, at-least this one will install. Installation took quite a while and it really heat up the machine, core temp reached 100 degrees! After installing apache24 and hhvm on arch linux, I wasn't able to run both of them in bundle, I could run a php script from hhvm but not from http. My question on stackoverflow still unanswered, [here](http://stackoverflow.com/questions/21740532/configure-hhvm-and-apache-for-archlinux). 

So, Arch linux is out, moving on to CentOS 6.5. I had problems with centos because of resolving dependencies. There are couple of tutorials how to install hhvm under centOS, what I couldn't work out was installing rpmdevtools, it says “Requires: pkgconfig >= 1:0.24”, and after upgrading the system the best I got for pkgconfig was .23 version. At this point I couldn't waiste more time as switch to Ubuntu, as hhvm must work there as explained in their tutorial and it should be stable. And it was, good old ubuntu. 

Ubuntu 13.10 has apache 2.4 by installing lamp by default. Installing hhvm is also one command and runs with no problems. At this point I was able to run some tests. I did it with mod_php with and without opcache and hhvm. 
I could verify if the system runs with hhvm on or off with checking 

<code>
	if (defined('HHVM_VERSION')) { echo "hip hop is on"; }
</code>

note: I need to uninstall hhvm to run the test without hhvm, stopping its service still runs the hhvm! 
Here's the results for fibinacii:
Document path of http request is something like /index.php?q=30, where I change “q”, running the test is done as usual with:

<code>
	ab -n 100 -c 10 http://192.168.1.107/index.php?q=30
</code>

![phpvshhvm]({{ site.url }}/public/images/phpvshhvm.png)


after q=30 the difference is really starting to show and on q=35 is impressive. mod_php processes q=35 in 12.5sec whereas hhvm in 0.5 sec. 
Since the plan was to do benchmarks for SugarCRM, after I run it under hhvm I got lots of errors, javascript mainly and I haven't fix those yet or haven't looked into those, maybe it's just some configuration for hhvm, but I will definitely try to work that out. Meanwhile I did for mod_php with and without opcache. Further I might update these graphs with other technologies.
The test here are performed using the following command, changing the c parameter or number of multiple request.

<code>
	ab -n 100 -c 10 http://192.168.1.107/sugarcrm/index.php?module=Accounts&action=DetailView&record=1506f08c-e553-b587-b084-5308cf4a58f2
</code>

![sugarcrmopcache]({{ site.url }}/public/images/sugarcrmopcache.png)

As of php 5.5. APC no longer is supported or apc doesn't not support php 5.5. so we switch to opCache , which as apc previously is giving better results than using without.