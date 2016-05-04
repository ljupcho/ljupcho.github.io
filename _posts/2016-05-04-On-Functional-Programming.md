---
layout: post
title:  "On Functional Programming"
date:   2016-05-04 20:12:10 +0200
tags: closures
comments: true
--- 

Where are we with functional programming and should that improve code quality. It sure is interesting. This is how I understand it. Functional programming is based on immutability and statelessness. The first means the function doesn't change anything, it gets data and returns some other data (data in - data out) and statelessness which means it doesn't depend on anything previous as if it is declared in separate file.

JQuery has been doing this for 20 years now, so most of the how-to I've read come from JavaScript but the logic is the same or maybe not quite the same.

In OOP the point is we have objects that communicate between each other, usually we inject one object into other object's constructor (DI injection) and than use its methods in some way. So, we have objects commonly referred to like "nouns" and methods like "verbs". If we need a method from some objects we have to create an instance of it and call that method even though that's all we need, so we might want to declare and call that method static so we don't have to create an object. Creating an object just for using something from it is not very good practice and it faces the problems known as "composition over inheritance" or better referred as the famous quote "you asked for the banana but you got the whole jungle". It is a problem when we have huge chain of inheritance of many parent/child classes so the purpose of using something from some child class and create all in the background is not necessary and we don't need it and we shouldn't create it.

Moving these objects is some patterns is old news, moving the functions and not creating objects is also old news but somehow modern frameworks adopt it widely. They are known as closure, anonymous functions or functions that are injected into another functions as arguments. We have native functions like array map/filter/reduce that will do something for each of the elements of the passed data. So, instead of for/foreach loops we can use those. We move the "verbs" or the functions around and for that to be feasible we need to make them immutable and stateless. This doesn't mean we go back to having functions in files and include those files and call functions. 

If we have a objects that communicate in some way than for some function I might want to pass an array by reference and also return some other value and also calling that function recursively. Calling functions recursively is great way to make simple and re-usable code.

I might want to force using some other functions in one function, for example if I have an abstract class and declare an abstract private method, so children classes must have that function but with creating an object they can't assess that method from outside, because I want my main function to be called first so I can ensure that that object will have some logic done before calling a public method.

It is just a common sense that we should break our functions into smaller pieces, each of them doing one thing (you have one job) because we will always need small piece of some functions someplace else.

How do we inject one function into another, is that a function that we create it on the run or use it from a file or from another object or all of these we just know what the output from that function should be.








