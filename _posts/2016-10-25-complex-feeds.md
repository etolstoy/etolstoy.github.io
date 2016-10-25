---
layout: post
title: Feed, why so complex?
description: How to implement a complex feed with multiple types of data.
---

A feed is probably the most common data representation in mobile applications. Just have a look at any app on your phone - Facebook, Instagram, Tweetbot, and even your favourite e-mail or RSS client. Feeds are everywhere. Despite it's a pretty common task we still got a lot of questions about its implementation. The problem gets even harder when there are different content types in one feed.

I've faced this problem not very long ago in LiveJournal application. Our product manager suddenly decided to add new content types in post feeds - marketing notifications, like *don't forget to turn push-notifications on*, and beloved advertisements.

<!--more-->

> By the way, I've already made [a blog post](http://etolstoy.com/2016/03/20/promises-and-ads/) were I covered advertisement integration using promise pattern in more details.

The requirements for these elements were not straightforward at all.

Advertisement:

- Show the first element with a specific advertisement identifier after the first 3 posts.
- After the first advertisement show ads with a different identifier after every 5 posts.

Notifications (we'll look only at one type, "Do not forget to turn push notifications on"):

- The item should be shown only in a specific kind of feed.
- The item should be second in the feed, but not the last one.
- The item should be shown on the first application launch.
- If a user scrolls the item without interacting it should be shown after 5 days or after 20 launches.
- If a user dismisses the item it should be shown after updating the application to a new version.
- If a user interacts with the item but doesn't register, it should be shown on the next launch.
- ...and a couple of more rules.

The existance of such business requirements literally guarantees that they are not last and in the future we will be asked to integrate other types of feed content.

So, we had to design a module that would work independently of its content. Another nice feature to have was a kind of plugin system, that would allow to add new data sources and cells without touching the whole module.

Until this moment the logic of the feed VIPER module was pretty straightforward:

![Pure Feed VIPER Module](/public/img/posts/complex-feed-1.png)

The first decision was switching to a declarative style of programming. I've created a simple `ContentModel` class. Its purpose was to describe a feed and unambiguously identify the content type for each indexPath.

```
@interface ContentModel : NSObject <NSCopying>

- (void)registerSlotForItemType:(ContentListItemType)type
                       position:(NSUInteger)position;

- (NSUInteger)obtainNumberOfItems;

- (ContentItemType)obtainTypeForPosition:(NSUInteger)position;

- (NSUInteger)obtainRelativeIndexForPosition:(NSUInteger)position
                                        type:(ContentItemType)type;

@end
```

This model is retained by the presenter of the module and updates on every feed change event triggered either by user action (removing a notification cell) or by CoreData notification (inserting new posts). 

There is another way. You can give up this intermediate state and form a new `ContentModel` on every event - but sometimes it can affect the perfomance of your app. Consider these options when implementing your own feed.

The most interesting thing happens in the Interactor. When we want to obtain new ContentModel we call the method:

```
- (ContentModel *)obtainActualContentListContentModel {
    ContentModel *contentModel = [ContentModel new];
    for (id<DataProvider> provider in self.dataProviders) {
        contentModel = [provider enrichContentModel:contentModel];
    }
    return contentModel;
}
```

The Interactor stores an array of `<DataProvider>` objects responsible for providing elements only of a specific type.

![Data Providers](/public/img/posts/complex-feed-2.png)

Let's look at the implementation of a DataProvider responsible for working with posts. The first method is used to understand the kind of supported item:

```
- (BOOL)isResponsibleForItemType:(ContentItemType)itemType {
    return itemType == ContentPostItem;
}

@end
```

The second method is used for enriching the `ContentModel`. It's implementation can rely on the information from `ContentModel` which was added by previous providers. E.g. the `AdvertisementProvider` may count the number of posts and insert its elements on the right places:

```
- (ContentModel *)enrichContentModel:(ContentModel *)contentModel {
    NSUInteger numberOfPosts = self.controller.fetchedObjects.count;
    
    for (NSUInteger i = 0; i < numberOfPosts; i++) {
        [contentModel registerSlotForItemType:ContentListPostItem
                                     position:i];
    }
    
    return contentModel;
}
```

![Enriching Content](/public/img/posts/complex-feed-3.png)

And the last method is used to obtain a specified item from the cache. Interactor passes its index relative to the current `DataProvider` to simplify calculations.

```
- (id)obtainElementForRelativePosition:(NSUInteger)position {
    NSIndexPath *indexPath = [NSIndexPath indexPathForRow:position
                                                inSection:0];
    PostModelObject *post = [self.controller objectAtIndexPath:indexPath];
    return post;
}
```

That's all for `DataProvider`s. The other part of the system is a `CellObjectFactory`, which is responsible for creating a correct `CellObject` (just a dumb view model) for a given model object. The implementation of the factory is straightforward:

```
@protocol ContentCellObjectFactory <NSObject>

- (id<ContentListCellObject>)cellObjectWithViewModel:(id)viewModel;

- (BOOL)canProcessViewModel:(id)viewModel;

@end
```

So, when a `UITableViewDataSource` needs a cell, it queries the array of cell object factories until one of them takes responsibility for a given type of model object.

Finally, let's see the whole data flow:

![Enriching Content](/public/img/posts/complex-feed-4.png)

Let's see what we've got:

- Adding a new type of content is really simple. We just have to add two new objects:
  - `DataProvider`, which is responsible for enriching ContentModel and obtaining a model object
  - `CellObjectFactory`, which is responsible for creating a cell from a model object.
- We can decide whether to show a specific type of content based on a very complex set of rules.
- Each of content types can be easily disabled by using simple [feature toggles](http://devalloy.github.io/feature-toggle).
- The whole feed VIPER module is pretty simple, easy to use, extend maintain.

Besides this article, you can have a look at [one of our internal meetups](https://www.youtube.com/watch?v=t297guLQ7SE) where my colleague gave a brief talk on the subject. And if you've got questions on how to implement feed pagination - [my own talk](https://www.youtube.com/watch?v=yVL-01AwVOc) will definitely help you with that.