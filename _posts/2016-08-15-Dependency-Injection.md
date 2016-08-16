---
layout: post
title:  "On Dependency Injection"
date:   2016-08-16 20:12:10 +0200
tags: DI
comments: true
--- 

Having the code decoupled seems like the way to go for re-usability outside the scope of a framework. That's the main reason I prefer Symfony, I think it has the best approach since you can easily use some of its components someplace else in some other framework or as is.

The concept of service provider and use everything as a service is done thanks to injecting one object into another with type hinting, even more type hint the interface that that object's class should implement and not the exact class. 

If I have a class that would accept let's say 3 object DI into the constructor, so every-time I create an object of that class I have to create those 3 objects as well. This is a good solution because if I were to extend and override some methods of those objects, I can and that would work fine. This is also good for mocking, because I find myself usually null-ing the constructor of a mock, especially if it is a class that has DB connection in its constructor. 

So, if I extend some of those classes I won't have to extend the main class those objects are injected into, which wouldn't be the case otherwise. If the objects were created inside the main class than if some of than gets extended I will have to extend the main class and override the method where object of the first class is being created to accept the newly extended class.

What bothers me is that I will have to inject all of these 3 objects all the time when I create the main object. That's just me being lazy and I would look for a workaround that and examine how offer I create this object and what is the probability of any of these 3 objects getting extended. If they are not to be extended I would allow those objects to be created inside the class.