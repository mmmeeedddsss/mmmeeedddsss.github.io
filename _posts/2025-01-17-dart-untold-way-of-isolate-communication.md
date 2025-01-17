---
title: Not mentioned well but - You can communicate between dart isolates using IsolateNameServer
description: Somehow the official dart documentation overlooks the IsolateNameServer for communication between isolates even though it can be necessary to use in some cases.
date: 2025-01-17 17:10 +0100
categories: [CS, Dart]
tags: [dart, flutter, isolates, flutter_local_notifications, background_isolate, isolate_communication]     # TAG names should always be lowercase
---

### I develop a flutter application and I use [flutter_local_notifications](https://pub.dev/packages/flutter_local_notifications)

For submitting my notification schedules to the respective OS that the app is running. Since now, the library
seems very impressive due to its wide range of support and extensive documentation.

One use-case I need to provide is to allow the users of my app to be able to do some quick actions with the notifications actions.

![IOS Notification Action Example](/assets/img/ios_notif_ex.png) |  ![Android Notification Action Example](/assets/img/android_notif_ex.png)
:-------------------------:|:-------------------------:
Notification Actions on IOS  |  Notification Actions on Android

The not-so-weird need was that I would like the app to not open, and also make some changes on the locally persisted
data when a specific action is selected. It should be a common need; it is like marking a mail as read without opening the mail app.

This strategy is great from the UX perspective, you just provide a fast way to do an operation on the app with a single click on notification, without even opening the app.

### The problem arises
When you need to communicate between the main isolate and the background isolate for such a background action.
You might want to ask why would I need to, but the answer is simply because I just cannot be sure if the main isolate is running or not during the action.
Since the action does some state changes, I would like these changes to be reflected on the main isolate and so to the UI, IF there is a UI exists at that moment.

### Why is it a problem that causes me to write a post?

So naturally, I went for a google search on how to communicate between isolates in dart. I found the official dart documentation on [isolates](https://dart.dev/language/isolates).
I would like to share an excerpt from the documentation:
1.  > A newly spawned isolate only has the information it receives through the Isolate.spawn call. If you need the main isolate to continue to communicate with a spawned isolate past its initial creation, you must set up a communication channel where the spawned isolate can send messages to the main isolate. Isolates can only communicate via message passing. They can’t “see” inside each others’ memory, which is where the name “isolate” comes from.

So my understanding is that I have to set up a communication channel between isolates during the creation of them.

Let's turn and look for an excerpt from [flutter_local_notifications](https://pub.dev/packages/flutter_local_notifications) now,

:::note
In some cases when you don't have easy access to the point the isolates are created
and cannot pass the communication channel during `Isolate.spawn` call,
you can instead use the [IsolateNameServer](https://api.flutter.dev/flutter/dart-ui/IsolateNameServer-class.html).
It is a global mapping of names to ports where isolates can dynamically register themselves.
:::

{:start="2"}
2. > This plugin contains handlers for iOS & Android to handle these background isolate cases and will allow you to specify a Dart entry point (a function). When the user selects a action, the plugin will start a separate Flutter Engine which will then invoke the onDidReceiveBackgroundNotificationResponse callback.

Important part here is the part
> ".. the plugin will start a separate Flutter Engine which will then invoke ..".

So the plugin is the one starting the background isolate. In the parameters that library provides us to use as a callback interface,
`onDidReceiveBackgroundNotificationResponse`, there is no capability to provide a "communication channel".

### So is it the end? (No)

According to the documentations, yes it is the end. There is nothing I can do since (1) the only way to communicate between isolates to pass a communication channel during their initialization,
and (2) the library does not provide a way to pass a communication channel to the background isolate.

Having the sadness of this fact and the possible hacky solutions like polling the shared db, or subscribing to a file state(which also turned out to be not possible in Android(?)),
With the no good solution in my mind, I created an [issue](https://github.com/MaikuB/flutter_local_notifications/issues/2517) to the library's Github page, and

### I decided to solve another bug

Which arrived recently with the existence of multiple isolates using the same db.

I use [drift](https://pub.dev/packages/drift), which I also liked so much. Their documentation is also great and has a [section](https://drift.simonbinder.eu/isolates/) for
possible utilization of isolates for long running db tasks, and setting up your db for multi isolate access. The solution for
multi isolate access turned out to be pretty trivial, but it is of course not the case for the bug or not the topic of this blogpost.

Then an incredible thing happened. I saw the following, a bit longer excerpt on the documentation of drift:
>All setups mentioned here assume that there will be one main isolate responsible for spawning a DriftIsolate that it (and other isolates) can then connect to.\
>In Flutter apps, this model may not always fit your use case. For instance, your app may use background tasks or receive FCM notifications while closed. These tasks will run in a background FlutterEngine managed by native platform code, so there's no clear communication scheme between isolates. Still, you may want to share a live drift database between your UI engine and potential background engines, even without them directly knowing about each other.\
>An IsolateNameServer from dart:ui can be used to transparently share a drift isolate between such workers. You can store the connectPort of a DriftIsolate under a specific name to look it up later. Other clients can use DriftIsolate.fromConnectPort to obtain a DriftIsolate from the name server, if one has been registered.

### That is how I met with [IsolateNameServer](https://api.flutter.dev/flutter/dart-ui/IsolateNameServer-class.html)

So was it the end, No.

`IsolateNameServer` allows you to dynamically register your isolates to a global map/dictionary of isolates by their identifiers.

Then, the other isolates can get the communication channel of the registered isolate by its identifier and use it to send the information of

_"I changed some stuff, please adjust yourself, or you can be outdated"._

That is exactly what my background isolate tells to my main isolate for it to know that it should adapt to the changes and refresh the UI.
If the main isolate is not running, fine, background isolate cannot send this message and nothing happens.

### The moral of the story is

I am disappointed by the documentation of dart, specifically the one page on isolates: [isolates](https://dart.dev/language/isolates).

In addition to not even referring that ugly kind of global dictionary solution, it even does not mention the existence of `IsolateNameServer`.
And in addition to not even mentioning `IsolateNameServer`, it says "A newly spawned isolate only has the information it receives through the Isolate.spawn call."(and the continuation on above).

I found it both important to mention the existence of `IsolateNameServer` and also to mention that the communication channel can be passed after the initialization of the isolates.

The good thing is, I could write a new blogpost over this thing, and planning to open an issue to dart and see if I will become a contributor of it.


---
\* Your app is actually opened in any scenario, what I mean is opening the app to the front, not just running in the background.
For that reason, the library asks you to define a separate callback and an entrypoint for your app to handle the action.
