---
layout: post
title:  "Controllers don't hold data"
date:   2017-05-20 19:18:10 +0200
tags: php rest
comments: true
---	

Couple of things about how I use controllers. I have seem many cases, many of them I've been part of where we extend the functionality of a controller beyond being that a REST endpoint with the usual CRUD operations and having more than that, which usually includes AJAX calls, some business logic, inclusion of third party libraries etc. So, how do we reuse them, well we don't, we copy/paste the same if-else conditions and put that in another place/class/controller. And what do we do when we change something about it later down the road, well we find/replace that and we're done, with that we might mess something up but the testers will find it and we will fix it. This has been done for too long and for too many times.

As the code-base increases it becomes more and more unforgiving. Good code is suppose to lower estimations times not stay the same or increase them. Nor do I feel I need to go through all the code to do my work or understand what all it does. If the logic is mainly in one class it usually happens that I need to go through all of it so many if-else to grasp. That would be generally speaking.

If I need to write the method with name like restoreBackup inside of ApiController or some complex name it is an indicator that I need to create a new controller that would be called BackupController and inside of that a method restore. Method names inside of controllers should be short and having multiple controllers instead of squeezing them in one. I would restrict the controllers to only crud operations and create a new one functionality based controller. That would also allow me better manage role permission per controller, better grouping routes and lot more readable code. Most modern frameworks have this, when you can create standalone controller instead of creating a module or bundle folder based structure to include a controller, that would not work and I would better find a way inside that framework to create a single controller.

Basically controllers don't hold data, they shouldn't have the business logic, but that if-else logic should go in another class, let's call it service or provider class that will always implement an interface or extend an abstract class that extends an interface, which I would type hint interface inside the controller or some other service class. The interface itself will allow me to bootstrap the logic and allow me to get inside a component very fast like a memory trick since the interface tells me what method that object has that is being called from external object, bootstrapping me quickly into what I need to know.

