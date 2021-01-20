# Android Activity Launch Mode
[http://inthecheesefactory.com/blog/understand-android-activity-launchmode/en](http://inthecheesefactory.com/blog/understand-android-activity-launchmode/en "源地址")

## standard
This is the default mode.</br>
The behavior of Activity set to this mode is a new Activity will always be created to work separately with each Intent sent. Imagine, if there are 10 Intents sent to compose an email, there should be 10 Activities launch to serve each Intent separately. As a result, there could be an unlimited number of this kind of Activity launched in a device.
#### <font color="brown">Behavior on Android pre-Lollipop</font>
This kind of Activity would be created and placed on top of stack in the same task as one that sent an Intent.
![](http://inthecheesefactory.com/uploads/source/launchMode/standardtopstandard.jpg)
An image below shows what will happen when we share an image to a standard Activity. It will be stacked in the same task as described although they are from the different application.
![](http://inthecheesefactory.com/uploads/source/launchMode/standardgallery2.jpg)
And this is what you will see in the Task Manager. (A little bit weird may be)
![](http://inthecheesefactory.com/uploads/source/launchMode/gallerystandard.jpg)</br>
If we switch the application to the another one and then switch back to Gallery, we will still see that standard launchMode place on top of Gallery's task. As a result, if we need to do anything with Gallery, we have to finish our job in that additional Activity first.
#### <font color="brown">Behavior on Android Lollipop</font>
If those Activities are from the same application, it will work just like on pre-Lollipop, stacked on top of the task.
![](http://inthecheesefactory.com/uploads/source/launchMode/standardstandardl.jpg)
But in case that an Intent is sent from a different application. New task will be created and the newly created Activity will be placed as a root Activity like below.
![](http://inthecheesefactory.com/uploads/source/launchMode/standardgalleryl.jpg)
And this is what you will see in Task Manager.
![](http://inthecheesefactory.com/uploads/source/launchMode/gallerystandardl1.jpg)</br>
This happens because Task Management system is modified in Lollipop to make it better and more make sense. In Lollipop, you can just switch back to Gallery since they are in the different Task. You can fire another Intent, a new Task will be created to serve an Intent as same as the previous one.</br>
![](http://inthecheesefactory.com/uploads/source/launchMode/gallerystandardl2.jpg)</br>
An example of this kind of Activity is a **Compose Email Activity** or a **Social Network's Status Posting Activity**. If you think about an Activity that can work separately to serve an separate Intent, think about standard one.
## singleTop
The next mode is singleTop. It acts almost the same as standard one which means that singleTop Activity instance could be created as many as we want. Only difference is if there already is an Activity instance with the same type at the top of stack in the caller Task, there would not be any new Activity created, instead an Intent will be sent to an existed Activity instance through <font color="blue">onNewIntent()</font> method.
![](http://inthecheesefactory.com/uploads/source/launchMode/singletop.jpg)
In singleTop mode, you have to handle an incoming Intent in both <font color="blue">onCreate()</font> and <font color="blue">onNewIntent()</font> to make it works for all the cases.

A sample use case of this mode is a Search function. Let's think about creating a search box which will lead you to a SearchActivity to see the search result. For better UX, normally we always put a search box in the search result page as well to enable user to do another search without pressing back.

Now imagine, if we always launch a new SearchActivity to serve new search result, 10 new Activities for 10 searching. It would be extremely weird when you press back since you have to press back for 10 times to pass through those search result Activities to get back to your root Activity.

Instead, if there is SearchActivity on top of stack, we better send an Intent to an existed Activity instance and let it update the search result. Now there will be only one SearchActivity placed on top of stack and you can simply press just back button for a single time to get back to previous Activity. Makes a lot more sense now.

Anyway singleTop works with the same task as caller only. If you expect an Intent to be sent to an existed Activity placed on top of any other Task, I have to disappoint you by saying that it doesn't work that way. In case Intent is sent from another application to an singleTop Activity, a new Activity would be launched in the same aspect as standard launchMode (*pre-Lollipop: placed on top of the caller Task, Lollipop: a new Task would be created*).