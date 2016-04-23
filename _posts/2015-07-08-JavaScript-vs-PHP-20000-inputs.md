---
layout: post
title:  "JavaScript vs PHP on new 20,000 inputs"
date:   2015-07-08 21:18:10 +0200
categories: php javascript
tags: php javascript
comments: true
---

Basically I have couple of ways to build a form with a lot of inputs, maybe that won't be the best practical solution but let's just see where I am going with this. 
In one tr I have at least 20 inputs, it maybe be more and I want to multiply that by 1000. Solutions based on JavaScript are done using jQuery. These are the scenarios I have tried: 

1). JavaScript (cloning): Render/load the page first with basic inputs or it should have at-least one row(tr).
- Copy the last row, or maybe better have a template of that tr and make a clone.
- Find the last tr of that table and attach the newly clones tr. You may use append or next from jQuery functions.
- Find the last tr again, the cloned tr and switch the ids and name attributes of each input, select fields that you find in that tr. The switching means incrementing the counter or the tr which will be part of the unique id of each element and the nested array you're trying to build in the name attribute. Also null the onclick/onchange and other functions (calendars and such) on each input and bind its new functions with incremented id.

This might be a bit more work to it but works just find and it's fast comparing to making an ajax call to server side. 

2). PHP (file): Upload functionally allowing the user to upload an excel/csv file that will be parsed on server side. In the excel I would have all the rows I need to make inputs from and the logic is building the table and its rows in php. Nothing more to it. 

3). JavaScript (builder html): Instead of cloning and switching ids on client side I am trying to build the rows with building the html itself in JavaScript. In order to have better solution I would have the inputs configurations/ fields definitions json_econded into browser's memory. I could then access that array and loop through the fields' types to render appropriate input. As I go from one to another I am incrementing the id, acctually the same thing as in php. The difference here from the first approch is that I don't attach each of the row/tr after creating it, but build all the htlm and than find the table id (once) and attach the whole html. 

4). PHP (ajax): The standard way, making an ajax call to server side, use the field definitions array, loop and build html. Then return it to client side and attach the html. 
Each of the 2, 3, and 4 approach would require binding the functions on document ready. 

The results (Response time in miliseconds): 

<table>
  <thead>
    <tr>
      <th>Num. of rows</th>
      <th>JS (cloning)</th>
      <th>PHP (file)</th>
      <th>JS (builder)</th>
      <th>PHP (ajax)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>500</td>
      <td>83501</td>
      <td>30861</td>
      <td>1721</td>
      <td>5268</td>
    </tr>
    <tr>
      <td>1000</td>
      <td>270085</td>
      <td>56231</td>
      <td>3297</td>
      <td>9648</td>
    </tr>
  </tbody>
</table>
	

This shows that cloning the rows and attaching them is slowest way to do. As the DOM becomes larger it takes more and more time to find the last, find the inputs and switch the ids. 
The fastest way to do it is to build the html on JavaScript side and attach it all at once. Doing it the same way using ajax is 3 times slower due to making the server call even though it might be faster depending on the server configurations, this is just my case. 

Going with the JavaScript approach has also one major advantage besides the performance aspect. If making the save on this form with php you will have to increase the max_input_vars in php.ini and that parameter can't be set only for that script specified but globally. Instead, this would be another JS approach besides the building of the html, the save can be perform having all the inputs as array in json format and using json_decode on php to make the needed request structure.