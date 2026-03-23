---
title: "What is the best way to pass NSManagedObjectContext around in iOS applications?"
date: 2012-01-18 00:09:00
slug: "best-way-to-pass-nsmanagedobjectcontext-around-in-ios-applications"
tags:
  - ios
---

**TL;DR**
Passing an `NSManagedObjectContext` in each and every `NSViewController` that
needs CoreData can be done by different means, but some are frowned upon. In doubt, inject it.


While pair-reviewing some code with my colleague [Vincent Tourraine](http://www.vtourraine.net/blog/),
we realized we wrote this kind of code into each `viewDidLoad:` method of each `NSViewController` subclasses we created:

```objc
managedObjectContext = [[[UIApplication sharedApplication] delegate] managedObjectContext];
```

Doing so is more or less the same as using a global variable, as calling `sharedApplication` on `UIApplication` returns the singleton
application instance. I kind of felt this was a code smell so I investigated.

After a couple of minute of browsing Apple's documentation, here's what [we can read](http://developer.apple.com/library/ios/#DOCUMENTATION/DataManagement/Conceptual/CoreDataSnippets/Articles/stack.html#//apple_ref/doc/uid/TP40008283-SW2)

<blockquote>
A view controller typically **shouldn’t retrieve** the context from a global object
such as the application delegate. This tends to make the application
architecture rigid. [...]
[...]
When you create a view controller, you pass it a context. [...]
</blockquote>

If you use the good old `nib|xib`s (I'm acting as if I was confident with them) and
instantiate your controllers programmatically, this sounds dead easy, but what
if you use the brand new shiny `Storyboard`s?

The only solution I came up with, that will comply with Apple's recommendation, is to use `prepareForSegue`, like this:

```objc
#import "MyMainController.h"
#import "MyOtherController.h"
@implementation MyMainController
-(void)prepareForSegue:(UIStoryboardSegue *)segue sender:(id)sender {
    if([[segue identifier] isEqualToString:@"SegueToMyOtherController"]) {
        MyOtherController *controller = [segue destinationViewController];
        controller.managedObjectContext = managedObjectContext;
    }
}
@end
```

If your main controller doesn't have any `NSManagedObjectContext`, you might
want to inject it from within the method
`application:didFinishLaunchingWithOptions:` of your AppDelegate, as you can see if you create a new CoreData project from the templates.

So next time, no more abuse of the `sharedApplication` !
