---
layout: post
title:  "Scaling developers’ work"
date:   2016-04-04 21:18:10 +0200
categories: svn
tags: svn scale
comments: true
---

There are different ways developers, managers, people involved in the project divide the work between themselves in order to be more efficient, usually assigning tasks to what the person is most comfortable with. 

What happens if I run this command, presumably you have your project under svn and you want to see the number of commits per user. 

<code>
svn log -v --xml | grep '<author.*/author>' | sort $* | uniq -c | sort -rn 
</code>

If this outputs doesn’t give you equal distribution between developers I think we haven’t divided the work properly. Sure that some developers might commit more often than others but at the end of the day the code needs to be written.
So, here's where I stand regarding this little dispute of mine. Why not all the developers work on all the tasks and be that a joined effort instead of having one developer doing the same thing over and over on different projects. Having in mind that problem first needs to be solved and only than the code to be written, we can have multiple people working on same functionality parsed into smaller little functionalities itself. The benefits: 

- two people can talk and contribute and be more creative from discussions
- they will be more familiar with the various functionalities instead of only one which they'll be responsible for
- code style to be close enough, the whole code shouldn't have much discrepancies 
- faster development
- not relying only on one person for specific thing, developers also have lifes and need vacations

But, what does this mean practically? It means that the functionality shouldn't be one file or all your functions inside a controller or calling actions with Ajax, instead making up a design with common patterns that would break down into more classes and packages, components if you will. That will surly help deployment and code re-usability as well.
