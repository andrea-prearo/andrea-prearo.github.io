---
layout: post
title: "Boost Smooth Scrolling with iOS 10 Pre-Fetching API"
date: 2017-04-03
categories: [iOS, Mobile App Development, Swift]
---
In a [previous post](https://andrea-prearo.github.io/ios/mobile%20app%20development/swift/2017/01/25/Smooth-Scrolling-in-UITableView-and-UICollectionView.html), we explored some common strategies to achieve smooth scrolling in our iOS mobile apps. The main goal of applying those strategies was to avoid *choppy scrolling*, a common issue that negatively affects the user experience. To help developers with such a task, Apple made some very useful changes to `UICollectionView` in iOS10. But before reviewing this newly introduced functionality, let’s start by examining what prompted the need for them.

## What Causes Scrolling to Be Choppy? ##

Have you ever interacted with, or worked on an app that occasionally had choppy scrolling? If the answer is “yes”, then you know how disappointing it is when you try to scroll quickly and the app content appears to stutter. You may have asked yourself what was triggering such choppy scrolling behavior and the bad user experience that comes with it.

**The short answer is: The app was _dropping frames_. But what exactly does this mean?**

To ensure consistent smooth scrolling, an app needs to be able to steadily display 60 FPS (Frames Per Second). Or, in other words, the app needs to refresh its content 60 times per second. This means that each frame has approximately 16ms (1000ms/60frames ~ 16ms/frame) to be rendered. In the unfortunate case that displaying a frame takes longer than the allotted time, no data is shown for the next frame and it is said that the app *“dropped a frame”*. This unfortunate scenario is illustrated in the diagram below. The blue markings indicate drawing operations and their thickness represent the time required to complete the rendering. As we can see, in the second frame we have some rendering events that took more than the allotted time (~16ms) and, as a consequence, the third frame has been dropped.

![Dropped allowFragments](/assets/2017-04-03-Boost-Smooth-Scrolling-with-iOS-10-Pre-Fetching-API/Image1.png)

We can visualize the same scenario from the point of view of CPU time spent in refreshing operations. In the graph below, the spike corresponds to the occurrence of a dropped frame when the app took more than the expected ~16ms to refresh the current content.

![Display Refreshes](/assets/2017-04-03-Boost-Smooth-Scrolling-with-iOS-10-Pre-Fetching-API/Image2.png)

To achieve a good user experience, the refresh time must always be below the allowed maximum of ~16ms. Ideally, as we’d like to create a great user experience (and not just a good one), each refresh time should be:

* Consistently well below the allowed maximum time (~16ms).

* As low as possible, to save CPU time that can be used for other tasks.

The most common source of dropped frames is loading expensive data models for a cell from the main thread. Typical examples of this scenario are:

* Loading images from an URL.
* Accessing items from a database or CoreData.

With iOS10, Apple introduced some optimizations in the way cells are loaded and displayed. Let’s take a look at the improvements available with iOS10 and how they make it easier for developers to create smooth scrolling user experiences.

## Cell Lifecycle in iOS9 ##

The lifecycle of a `UICollectionViewCell` can be visualized as follows:

![iOS9 cell lifecycle](/assets/2017-04-03-Boost-Smooth-Scrolling-with-iOS-10-Pre-Fetching-API/Image3.png)

The main interactions between the collection view and its cells are:

* The collection view is requesting the content for a cell that is going to be displayed — *the cell is about to enter the visible field*: [`collectionView(_:cellForItemAt:)`](https://developer.apple.com/reference/uikit/uicollectionviewdatasource/1618029-collectionview).

* The collection view is requesting to display the cell — *the cell just entered the visible field*: [`collectionView(_:willDisplay:forItemAt:)`](https://developer.apple.com/reference/uikit/uicollectionviewdelegate/1618087-collectionview).

* The collection view is removing the cell — *the cell is outside the visible field*: [`collectionView(didEndDisplaying:forItemAt:)`](https://developer.apple.com/reference/uikit/uicollectionviewdelegate/1618006-collectionview).

## Cell Lifecycle in iOS10 ##

In iOS10, the lifecycle of a cell is mostly the same as in iOS9. However, there are a few significant differences.

The first difference is that the OS calls [`collectionView(_:cellForItemAt:)`] much earlier than it used to. This means two things:

* The heavy lifting of loading the cell can be done way before the cell needs to be displayed.

* The cell may not end up being displayed at all (as it may not ever be brought into the visible field).

The second difference is in what happens when a cell goes off the visible field. With iOS10, `collectionView(didEndDisplaying:forItemAt:)` is called as usual but the cell is not immediately recycled. The OS keeps it around for a little bit in case the user inverts the scrolling direction; if this happens, the cell is still available and can be displayed again (`collectionView(_:willDisplay:forItemAt:)` will be called) without having to reload its content.

![iOS10 cell lifecycle changes](/assets/2017-04-03-Boost-Smooth-Scrolling-with-iOS-10-Pre-Fetching-API/Image4.png)

Compare this with what happens on iOS9. You will notice that, for this specific use case, the heavy lifting of reloading the cell (`collectionView(_:cellForItemAt:)`) is not necessary anymore. This optimization allows us to render cells faster when the user is both rapidly scrolling and changing scroll direction.

A third important difference between iOS9 and iOS10 is in the way cells are loaded for collection views with multi-column layouts.

![iOS9](/assets/2017-04-03-Boost-Smooth-Scrolling-with-iOS-10-Pre-Fetching-API/Image5.png)

![iOS10](/assets/2017-04-03-Boost-Smooth-Scrolling-with-iOS-10-Pre-Fetching-API/Image6.png)

With iOS10, each cell about the enter the visible field is loaded separately (`collectionView(_:cellForItemAt:)`). As we saw while examining the cell lifecycle, this happens much earlier than when each cell actually needs to be displayed. This opens the door for optimization as the OS will be able to process different requests, interleaving them with cell loads.

When a row of cells is entering the visible field, the cells are displayed as a single batch (`collectionView(_:willDisplay:forItemAt:)` is called for the entire row at the same time) as this operation is not very expensive in terms of CPU cycles (at least compared to loading cell content).

This difference in loading multi-column layouts is at the core of the most important iOS10 `UICollectionView` optimization: *Pre-Fetching*. Let’s explore it in more detail.

## Pre-Fetching API ##

When presenting iOS10, Apple touted [**Pre-Fetching**](https://developer.apple.com/videos/play/wwdc2016/219/) as an *Adaptive Technology*. What this means is that *Pre-Fetching* will try take advantage of how users interact with the app to perform some optimizations targeted to improve scrolling performances. For instance, this new technology will look for idle times (when the user is scrolling slowly, or not scrolling at all) to load (*Pre-Fetch*) new cells. Depending on user scrolling patterns, there may be more (or less) opportunities for *Pre-Fetching* to perform such an optimization.

Before reviewing the available API, let’s take a look at some best practices to work with this technology. In order to fully take advantage of *Pre-Fetching*, the bulk of setting up cell content must be performed in `collectionView(_:cellForItemAt:)`. Operations performed in `collectionView(_:willDisplay:forItemAt:)` and `collectionView(didEndDisplaying:forItemAt:)` should be kept minimal and must be non-CPU intensive. Even better, it would be optimal if we could perform no operation at all for those lifecycle events! Also, keep in mind that, even if `collectionView(_:cellForItemAt:)` gets called for a cell, it is still possible that the cell will never be displayed.

Some very good news is that *Pre-Fetching* is enabled by default for apps compiled on iOS10. It is, however, possible to turn this feature off by setting the `isPrefetchingEnabled` property of `UICollectionView` to `false`. It is also important to note that *Pre-Fetching* works alongside the cell lifecycle. This means that the code we already wrote to implement a collection view doesn’t need to be changed - the only action required to take full advantage of *Pre-Fetching* is to implement the *Pre-Fetching API*.

# Pre-Fetching API and UICollectionView #

The *Pre-Fetching API* for `UICollectionView` is defined in the [`UICollectionViewDataSourcePrefetching`](https://developer.apple.com/reference/uikit/uicollectionviewdatasourceprefetching) protocol. The API defines the following two methods.

* [`collectionView(_:prefetchItemsAt:)`](https://developer.apple.com/reference/uikit/uicollectionviewdatasourceprefetching/1771767-collectionview) (required) — This method allows to initiate the asynchronous loading of the data required for the cells specified by the `[IndexPath]` parameter. The asynchronous loading can be performed either through the [Grand Central Dispatch](https://developer.apple.com/reference/dispatch) or by means of an [`OperationQueue`](https://developer.apple.com/reference/foundation/operationqueue). What is most important when implementing these methods is to write code that shifts the burden of data loading from the main queue to a background queue. The goal of this is to reduce the workload of the main queue to allow it to spend most of its time performing critical tasks as display refresh.

* [`collectionView(_:cancelPrefetchingForItemsAt:)`](https://developer.apple.com/reference/uikit/uicollectionviewdatasourceprefetching/1771769-collectionview) (optional) — This method communicates that the data for the cells specified by the `[IndexPath]` parameter is not needed anymore. Implementing this method allows us to cancel pending data loading as needed, which is a good way to save CPU time by canceling unnecessary work (typically because the user changed scrolling direction).

As we stated before, *Pre-Fetching* is an *Adaptive Technology*. Therefore, the above methods are triggered based on the way the user interacts with the app. One consequence of this is that the `collectionView(_:prefetchItemsAt:)` method may not be called for every cell in the collection view. This means that when loading a cell through `collectionView(_:cellForItemAt:)`, the app should be able to handle all the following scenarios:

* The data has been pre-fetched and is ready to be displayed.

* The data is currently being fetched and is not ready to be displayed.

* The data has not been requested yet.

# Pre-Fetching API and UITableView #

iOS10 also introduced *Pre-Fetching* for `UITableView`. All the main concepts we illustrated for `UICollectionView` apply in a similar way to `UITableView`. The *Pre-Fetching API* for `UITableView` is defined in the [`UITableViewPrefetchingDataSource`](https://developer.apple.com/reference/uikit/uitableviewdatasourceprefetching) protocol. The API defines the following two methods.

* [`tableView(_:prefetchRowsAt:)`(https://developer.apple.com/reference/uikit/uitableviewdatasourceprefetching/1771764-tableview) (required) — This method allows to initiate the asynchronous loading of the data required for the cells specified by the `[IndexPath]` parameter.

* [`tableView(_:cancelPrefetchingForRowsAt:)`](https://developer.apple.com/reference/uikit/uitableviewdatasourceprefetching/1771765-tableview) (optional) — This method communicates that the data for the cells specified by the ``[IndexPath]`` parameter is not needed anymore.

The suggestions presented for each `UICollectionView` *Pre-Fetching API* method apply in the same way to each analogue `UITableView` *Pre-Fetching API* method.

## Conclusion ##

I’ve updated my code sample for `UITableView` and `UICollectionView` to support *Pre-Fetching* on iOS10. You can find it [here](https://github.com/andrea-prearo/SwiftExamples/tree/master/SmoothScrolling/Client).

In this post, we reviewed the improvements available with iOS10 to boost smooth scrolling for both `UICollectionView` and `UITableView`. In particular, we saw that by implementing a specific *Pre-Fetching API* it is possible to take full advantage of all the new OS optimizations and provide the best possible user experience when interacting with our mobile apps.
