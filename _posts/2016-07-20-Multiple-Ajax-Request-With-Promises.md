---
layout: post
title:  "Multiple Ajax/Requests with JavaScript Promises"
date:   2016-07-20 18:12:10 +0200
tags: promises javascript
comments: true
--- 

We have multiple ajax requests that need to be executed one by one. Each of the ajax requests is run after the success of the previous ajax. This usually happens when we want to get something from API, display some information and do another action and so on in multi nested levels. 

Problem one:

We can write the code with ajax calls in the success function of the previous ajax call and that would work just fine and as expected. We also want to put pop-ups between the calls that the user can choose an action, like a confirm dialog that would trigger the next ajax call or cancel the whole chain. This code might look very long and messy, that's how I started, it sure works but I didn't like it a bit.

Problem two:

Since I am using jQuery for ajax requests, jQuery itself offers promises solution, but in case of an reject(error from the ajax) instead of jumping to the catch part it goes to the next then in the chain. Probably you've come across blog posts regarding this, I've also tried it and I could manage the passed resolved data in the next then chain (where it shouldn't go in in case of reject) with another if, but that's not really the way to go. 

Solution:

Instead, what I wanted to do is have a promise for each ajax call, but without using the all() or race() that would fire the requests in parallel or get the fasted one. It's more of a solution that would allow me to configure listeners and dispatch them at the end. I continue using jQuery for ajax calls, but put each of the ajax into a promise with passed in object for configurations.

I create an object from PromiseHandler, which will be explained after this configuration. 

{% highlight js %}

var promiseHanlder = new PromiseHandler();

{% endhighlight %}


I register the first AJAX with an object that has the name of the module whose controller to be called and the action to be run for this request. jQuery ajax expects data to be passed and in this case I pass in a function with data_after_resolve as input argument. That would be the object that I get from resolving the previous ajax call, that data I would normally json_encode from server side. If this is the first ajax than no passed object is needed. In case of a resolve I would display the successMsg, in case of reject the errorMsg. I put the listeners in an array and fire them one by one and check if the current listener has an 'has_next' ajax call, if so then display a pop-up with two buttons, one for triggering the next ajax call and one cancel for dropping the whole thing.

{% highlight js %}
// First AJAX
promiseHanlder.addAjaxListener({
    'module_name' : 'Controller1',
    'action' : 'Action1',
    'data' : function(){
            return {
                id: record
            }
    },
    'successMsg' : function(data){
            return 'Success: '+ data.msg
    },        
    'errorMsg' :  function(data){
            return 'Error: '+ data.msg;
    },
    'has_next' : true, // this is important to trigger the next request
    'button_label_next_request' : 'Label for Action2',
});

{% endhighlight %}

The input for the second AJAX is the output(data_after_resolve) from the first AJAX. In my case this is the final AJAX and I put onFinalSuccessFunction that returns a function that should reload the parent window (this can be anything) when click the button on the popup after the second and final AJAX.

{% highlight js %}
// Second AJAX
promiseHanlder.addAjaxListener({
    'module_name' : 'Controller2',
    'action' : 'Action2',
    'data' : function (data_after_resolve) {
            return {
                id: record,
                amount: data_after_resolve.amount,
                type: data_after_resolve.type
            }
    },
    'successMsg' : function(data){
            return 'Success: '+ data.msg
    },
    'errorMsg' :  function(data){
            return 'Error: '+ data.msg;
    },             
    'onFinalSuccessFunction' : function(){
            return function(){
                location.reload();
            }
    }
});
{% endhighlight %}

The idea is that I can have as many AJAX requests as I want, at the end I should just dispatch them and have this whole ajax-popup-ajax-popup-ajax.... in motion. This means that the dispatch function is called recursively executing the listeners with output data from the previous as input data in the next request.

{% highlight js %}
promiseHanlder.dispatch();
{% endhighlight %}


So, like this I could just copy/paste the above configuration skeleton, put as many ajax requests as I need, put the module/controller name, action name, input data with messages. I could do that, but I couldn't copy/paste the whole nested ajax calls otherwise.


How it's made:

This is my Handler that can be extended if needed. For example for popups I am using another widget I've made that is also based on jQuery and if I need something else for popups I can extend from this object and override regularPopUpWithMsg and popUpWithFunc methods or any other method if needed.

{% highlight js %}

function PromiseHandler() 
{
    this.listeners = [];
    this.order = 0;
}

{% endhighlight %}

addAjaxListener method accepts the passed objects and creates listeners with that configuration object and a function that returns a promise. The promise is being resolved or rejected based on data.status field. That is the field I am using as a success or failure from AJAX from server side which is part of my json_encode data.

{% highlight js %}

PromiseHandler.prototype.addAjaxListener = function(obj)
{
    this.listeners.push(
        {
            'obj' : obj,
            'func' : function(data_after_resolve){
                return new Promise(function(resolve, reject) {

                    $.ajax({
                        cache: false,
                        async: true,
                        type: 'POST',
                        url: 'index.php?module='+obj.module_name+'&action='+obj.action,
                        dataType: 'json',
                        data: obj.data(data_after_resolve),
                        success: function(data) {
                            if (data.status) {
                                resolve(data);
                            } else {
                                reject(data);
                            }
                        }
                    });

                }) 
            }
        }
    );
}

{% endhighlight %}

The popup methods:

{% highlight js %}
PromiseHandler.prototype.regularPopUpWithMsg = function(data)
{
    var popup = new PopUp();
    popup.error_msg = data.msg;
    popup.show();
}

PromiseHandler.prototype.popUpWithFunc = function(data, func)
{
    var popup = new PopUp();
    popup.error_msg = data.msg;
    popup.show('info', func); // don't call the function just pass it
}

{% endhighlight %}

The dispatch method is called recursively for each of the listeners that has a promise. I see the configuration of the listener, if it has the has_next this method calls itself again with previously showing a popup for the user to choose to fire the next request or cancel, but if it doesn't than a regular popup with just ok button or ok button with attached function that reloads the parent window. In case of success ajax, resolve data is passed and in the popup I show successMsg and set the data_after_resolve that will be input for the next ajax/promise.
If the first AJAX fails, meaning it is rejected it goes to catch block, it doesn't go into the then block and displays a popup with the errorMsg in which function the rejected data from the promise is being passed.

{% highlight js %}

PromiseHandler.prototype.dispatch = function()
{
    var listener = this.listeners[this.order].func;
    var listener_obj = this.listeners[this.order].obj;
    // get data from previous ajax
    var data_after_resolve = {};
    if (this.order > 0) {
        data_after_resolve = typeof(this.listeners[(parseInt(this.order) - 1)].data_after_resolve) != "undefined" ?
                                this.listeners[(parseInt(this.order) - 1)].data_after_resolve : {};
    } 

    listener(data_after_resolve)
        .then(function(data){
            
            var html = listener_obj.successMsg(data);
            
            if (listener_obj.has_next) {
                this.listeners[this.order].data_after_resolve = Object.create(data);
                   
                var popup = new PopUp();
                popup.error_msg = html;
                popup.popup_id = 'popup_dlg';
                popup.showMultiple(
                [
                    {
                        text: listener_obj.button_label_next_request,
                        click: function() {
                            $('#popup_dlg').dialog("close");
                            // get next listener
                            this.order +=1;
                            // call recursively 
                            this.dispatch();
                        }.bind(this)
                    },
                    {
                        text: 'Cancel',
                        click: function() {
                            $('#popup_dlg').dialog("close");
                        }
                    }
                ]);

            } else {
                 
                data.msg = html;
                if (typeof(listener_obj.onFinalSuccessFunction) == 'function') {
                    this.popUpWithFunc(data, listener_obj.onFinalSuccessFunction());
                } else {
                    this.regularPopUpWithMsg(data);    
                }                  

            }               

        }.bind(this))
        .catch(function(data){
            data.msg = listener_obj.errorMsg(data);
            this.regularPopUpWithMsg(data);
        }.bind(this))
}

{% endhighlight %}


