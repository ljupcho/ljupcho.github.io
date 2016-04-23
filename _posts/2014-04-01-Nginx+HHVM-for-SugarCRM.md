---
layout: post
title:  "Nginx + HHVM for SugarCRM"
date:   2014-04-01 21:18:10 +0200
categories: nginx 
tags: nginx sugarcrm
comments: true
---	

After installing nginx web server I managed to make it work with hhvm. Further more SugarCRM works just fine in this configuration except for some errors on some pages. I wasn't able to run it under apache, my app server is Arch Linux since apache 2.4.7 gives HipHop notice of file not found. It was able under ubuntu as in one of my previous posts but the app wasn't working properly. So now with nginx I got my luck to try some results for SugarCRM. 

My environment settings are, using arch both as app server and db server, because I am using mysql in this case, couldn't make hhvm work for oracle yet. But this setup gives me the chance to get some results. When running the hiphop virutal machine and you go to php info you will only get HipHop instead of all the info. Also, I am using php-fpm as better than mod_php in comparison to hhvm. 

I am changing number of request with concurrency 60 (time per request is across all concurrent requests) and this is what I got:

<code>
ab -n 200 -c 60 http://127.0.0.1/sugarcrm/index.php?action=ajaxui#ajaxUILoc=index.php%3Fmodule%3DTasks%26action%3Dindex%26parentTab%3DActivities
</code>

![Nginx vs phpfpm]({{ site.url }}/public/images/hhvmvsphp-fpmnginx.png)

We can see how big of a gap it is between hhvm and php-fpm. HHVM knocks down the php. These incomparable differences in favor of hhvm show why it is or will be a huge success. HHVM needs on average 2 ms pre request and php-fpm needs on average 25ms pre second and mod_php on average around 300ms (previous post). I try testing it with changing the concurrency and getting the same results. 

Still, in this setup I must say that SugarCRM gives some errors, for example navigating to detail view of an account it will give errors:

<code>
	HipHop Fatal error: $this is null in /usr/share/nginx/html/sugarcrm/include/SubPanel/SubPanelTilesTabs.php on line 62	
</code>

<code>
	HipHop Notice: Use of undefined constant JSON_LOOSE_TYPE - assumed 'JSON_LOOSE_TYPE' in /usr/share/nginx/html/sugarcrm/include/utils.php on line 3693
</code>