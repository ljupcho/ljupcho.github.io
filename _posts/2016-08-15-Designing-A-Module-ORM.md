---
layout: post
title:  "On Designing a Module, ORM or not"
date:   2016-08-15 19:12:10 +0200
tags: module orm
comments: true
--- 

Most of the framework today are using the ORM approach when it comes to designing their modules/bundles. They would create a class which private or protected properties represent columns in database table. So ORMs would provide CRUD and auto-generate public functions for listing, creating, updating or deleting a record from the table that module lives on.

That's cool, especially that frameworks such as Symfony or Laravel will provide you with commands to run in terminal and do the whole work. Where's my problem with this? The problem with this is that whenever I watch or read a tutorial they would give a blog example generate all of the crud actions in the controller, do the database work with calling the Doctrine ORM and that's it. Well it is not always a blog that we develop right. I hate that we have a structure from the back-end known what it is on the front-end, like I know this form and this lists is a table.

When we design a front-end and say we need a button here and we need a form here that triggers our ORM brain to think that that would be a table or that would be many-to-one relationship table. That's what I don't prefer having to think the back-end and the front-end with the same logic, I would probably give the front-end to someone who has no idea what could happens in the back-end and vice-versa. I am not against ORM at all, but when we have complex logic on single page, data that needs to be put into multiple tables in different kind of relationships thinking about front-end should be raised with "how" this would be done. A single form might have elements from different tables but not over complicate things, ORM is there to give us simplicity. It's all about the user experience and if what we've come up with is not simple enough go through it again. I am pro ORM, even have classes that don't have to be shown on front-end but I need to do save/retrieve on them. I tend to forget about ORM, tables and such things when trying to think of forms just to make sure they are simple enough that no explanation is necessary.
