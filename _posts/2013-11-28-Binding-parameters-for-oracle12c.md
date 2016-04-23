---
layout: post
title:  "Binding parameters for oracle12c"
date:   2013-11-27 21:18:10 +0200
categories: oracle
tags: oracle
comments: true
---	


Tests are performed on CentOS VM with 4GB RAM. I am running 10 queries from application w/o binding parameters in the sql statements in order to verify if i would get less time for bind rather than without. It turns out i am not getting much performance gain using the sql example and data set below. 


![binding12c]({{ site.url }}/public/images/binding12c.png)


Testing with two data sets, first with 20000 records, and second (the bigger bars) with 170000 records. 

{% highlight sql %}
SELECT a.id from (SELECT la_ljupcho.id , la_ljupcho.name , (NVL(jt0.first_name,'') || ' ' || NVL(jt0.last_name,'')) assigned_user_name , jt0.created_by assigned_user_name_owner , 'Users' assigned_user_name_mod, la_ljupcho.assigned_user_id 
FROM la_ljupcho 
LEFT JOIN users jt0 ON la_ljupcho.assigned_user_id=jt0.id AND jt0.deleted=0 AND jt0.deleted=0 
where ((la_ljupcho.name like ''||:name||'%')) AND la_ljupcho.deleted=0 
UNION ALL
SELECT la_ljupcho.id , la_ljupcho.name , (NVL(jt0.first_name,'') || ' ' || NVL(jt0.last_name,'')) assigned_user_name , jt0.created_by assigned_user_name_owner , 'Users' assigned_user_name_mod, la_ljupcho.assigned_user_id 
FROM la_ljupcho 
LEFT JOIN users jt0 ON la_ljupcho.assigned_user_id=jt0.id AND jt0.deleted=0 AND jt0.deleted=0 
where ((la_ljupcho.name like ''||:name||'%')) AND la_ljupcho.deleted=0) a 
group by a.id order by a.id
{% endhighlight %}

Using bind variables doesn't necessarily improve performance. The sql might use the execution plan and save time deciding on it, but still has to fetch different data set depending on the new value. Also, it means that in same cases it might be worse if that execution plan is not the ideal one, because not using bind parameters enables the optimizer to always opt for the best execution plan. So, i would use binding only for those sqls i am sure the execution plan is the best one.
