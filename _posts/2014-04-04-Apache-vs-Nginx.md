---
layout: post
title:  "Apache vs nginx web server comparison"
date:   2014-04-04 21:18:10 +0200
categories: nginx apache
tags: nginx apache
comments: true
---	

Using php-fpm seems to be a good idea so I wanted to check how thing stand regarding web server performance. Apache is standard and I've being using it for lots of years, but read that nignx should be faster and more stable. Version of apache is 2.4.7 and nignx's is 1.4.7. 

if you want to use apache 2.4 with the standard, let's say old mod_php, you'd need to switch to mod_mpm_prefork (as well install the modules/libphp5.so and include the php5_module.conf file), whereas using it with php-fpm is allowed with the new mod_mpm_event which stands for multi processing module. Installing the php-fpm and making it work with the web server should give “FPM/FastCGI” under Server API in php info. 

Nginx is event driven and apache is process driven, the first doesn't create new process for every request, apache does. Nginx also should be consuming less memory and should be faster serving static pages. 

Environment is set using Arch Linux as application server and CentOS with database server with oracle12c. Arch is the host and CentOS is the VM. 

So, I've tested both of them using the php-fpm module and SugarCRM and this is what I got:

<code>	
	ab -n 100 -c 10 http://127.0.0.1/mkt/index.php?module=Accounts&action=DetailView&record=7c8b058e-edb7-afa2-e61e-533d6dc29960
</code>


![Apache vs Nginx with php-fpm]({{ site.url }}/public/images/apachevsnginxwithphp-fpm.png)

Lowers means better or less time for processing per request. The results show that using apache is better then using ngnix with php-fpm for SugarCRM. 
Also, here's how it is with apache only using different mode, once with standard mod_php and with php-fpm. 


![mod_php vs php-fpm]({{ site.url }}/public/images/mod_phpvsphp-fpm.png)


It seems like php-fpm is a lot faster and mod_php is catching up only for bigger values of 'c' and only if all request tested are sent in parallel.
