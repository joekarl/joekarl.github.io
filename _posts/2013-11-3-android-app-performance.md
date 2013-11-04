---
layout: post
title: Android performance problems
---
One of the biggest pet peeves I have when it comes to software, is when software is written without regard to performance (or at least without regard to the tools available to handle performance problems). This has long plagued Android, with people shrugging it off saying \"you shouldn\'t design for performance, the platform is immature and will make things better\". That may have been the case a couple of years ago, but that is hardly the case now. Yet people still accept sluggish Android performance because \"that\'s just the way it is\".

##Common performance bottlenecks
Most Android performance problems boil down to a couple of specific areas. Here are three that are particularly important to be mindful of.

###Ignoring memory usage
Too often developers treat the memory on their target platform as an unlimited resource. Android does not implement paging or swap storage, so if you are storing something in your app, it is in RAM. The big things to avoid here are: 

* large application global objects (i.e. http caches)
* not using the correct image resource sizes for current screen density (i.e. you only have high res images for your app, despite smaller screen sizes having to resize them to use them)
* not reclaiming memory when not being in use (i.e. keeping images in memory when not being used)

If you see your app using a lot of memory or you\'re experiencing out of memory errors, these things are generally a good place to start.

###Executing IO tasks or any CPU intensive task on the main thread
Android, as with iOS (or any platform really), makes it easy to perform IO operations wherever the developer wants to. Just don\'t do it. IO should be performed on a background thread. Whether it\'s network requests, file access, or db access, use an async task or the loader interface. If you are doing IO access on the main thread, it\'s relatively easy to spot. Your app will be sluggish while the IO code is performed. Another way to find these problems is to enable \"Strict Mode\" in the developer options. This will cause the screen to flash when long operations on the main thread are being performed. 

One thing to keep in mind is that network access on the main thread is (by default) not allowed. You will see a NetworkOnMainThreadException. If you have disable this, that\'s probably bad...

This doesn\'t just go strictly for IO tasks, you should also background any task that does a lot of calculation or is CPU heavy. Async tasks are your friend.

###Using heavily nested View hierarchies
Heavily nested view hierarchies aren\'t always a bad thing (sometimes you need the complexity/flexibility), but if you\'re not careful this nesting can lead to a lot of overdraw. 

When Android does it\'s drawing, it starts at highest level container, draws that then draws all of that container\'s children. Current versions of Android aren\'t smart enough to know that pixels are going to be drawn on top of, so what ends up happening is you draw a background layer and the pixels of any child views are drawn again on top of the background. When people talk about overdraw, that is what they\'re talking about. The idea that individual pixels will be drawn more than once in a single drawing cycle. 

An overdraw of 2 is generally seen as ok, 3 is fine occasionally, 4+ should be avoided. If you\'re lucky enough to be running Android 4.2 or higher, you can turn on the \"Show GPU Overdraw\" setting in the developer options. Using that you can get a visual reference of overdraw that your app is performing. 

Long story short, be clever with your view hierarchy. 

* make fragment views invisible if not completely visible
* compound drawables are expensive, leverage nine patch images with content holes for complex drawables
* use transparent backgrounds for child views that don\'t need a background
* avoid setting backgrounds on list views if you\'re setting backgrounds on the list view items (i.e. alternating row colors)
* turn off backgrounds behind dynamically loaded images once they\'re loaded

**Extra** Romain Guy has a very good post detailing different things people can do to identify and correct graphical performance problems in Android apps (http://www.curious-creature.org/2012/12/01/android-performance-case-study/). Check it out, will be well worth your time.

##Takeaways
If your list view doesn\'t scroll smoothly, look to your view hierarchy. If your app loads data or views slowly, look to your memory usage. If you\'re seeing sluggish performance while loading data or your app becomes un-responsive, look to your IO or CPU intensive tasks.

Always be mindful of performance in your Android app. Your app, your users, and the Android ecosystem will be better for your efforts.