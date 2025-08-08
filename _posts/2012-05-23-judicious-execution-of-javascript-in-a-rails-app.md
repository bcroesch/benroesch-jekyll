---
layout: post
title: "Judicious execution of Javascript in a Rails app"
author: "Ben Roesch"
---
When you start a new rails app, you get a new javascript file for each controller that you scaffold. Assuming that you\'re using the asset pipeline, these will all get concatenated into a single file called application.js. This results in all of your javascript being executed by the browser each time the file is pulled down. In many cases, I would find myself tossing controller-specific JS into jQuery\'s document-ready event. Many times this isn\'t really an issue, but there were numerous times that it would cause me problems. One example would be if I wanted a jQuery plugin applied to certain elements on one page, but only after a certain action on other pages. On top of that, I\'m a slave to the left brain and it bothered me that code for TripsController was executing on a page in the PlansController.  
  
While building [Minimundo Travel][1], I came up with a solution that I felt was pretty solid. There are probably other similar solutions out there, but here\'s mine.  
  
I started by putting all controller-specific JS into a CoffeeScript class. For example, the JS file for my PlansController starts with:  
>     class MMT.Plans

Then, if I have JS that I want executed for every action in that controller, I give the class an init method.  
>     class MMT.Plans  init: ()->    /* do some controller-wide initialization here */

Next, for each action that has relevant JS, I give the class an init method for that particular action.  
>     class MMT.Plans  init: ()->    /* do some controller-wide initialization here */  initIndex: ()->    /* do some index-action-specific initialization here */

In order to inform the JS of the current controller and action, I added data attributes to the body tag in my application layout.  
> `<body data-controller=\"<%=params[:controller].camelize%>\"
> data-action=\"<%=params[:action].camelize%>\">`

Lastly, I created another file called base.js.coffee that ties everything together and fires on document-ready. All of my classes are namespaced under MMT, so I create a new instance of the controller\'s class, if it exists. Then, I call the relevant init functions to ultimately fire the JS corresponding to the given action.  
>     $ ()->  controller = $("body").data("controller")  action = $("body").data("action")  if MMT[controller]?    instance = new MMT[controller]()    instance["init"]() if typeof(instance["init"]) == "function"    instance["init#{action}"]() if typeof(instance["init#{action}"]) == "function"

  
  
If you enjoyed this post, I\'d love it if you followed me on twitter: <a>@bcroesch</a>

[1]: http://www.minimundotravel.com
