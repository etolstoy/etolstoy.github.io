---
layout: post
title: Promises and Ads
description: A promise is an extremely useful pattern. One of it's possible usages is advertising.
---

A promise is a really simple and easy to understand pattern. In general, it's just an object that represents a result of some action which is not finished yet. That's all, nothing more.

This simple concept reduces a complexity of asynchronous code a lot. Imagine that you don't have to write these awful nested completion blocks - every operation is synchronous. You just ask your service to return data - and it returns you something to work with.

<!--more-->

There are a couple of frameworks in iOS development that implement a promise pattern: [Bolts](https://github.com/BoltsFramework), [PromiseKit](http://promisekit.org/) and [others](https://github.com/search?l=Objective-C&q=promise&type=Repositories&utf8=%E2%9C%93). They are really great - but sometimes *(I'd rather say mostly)* they may be an overkill for your project. There is no need to make everything a promise - for example networking is done really easy with the use of chainable NSOperations. However, there still are some areas where the use of promises can simplify your code a lot. Just keep in mind the famous [KISS principle](https://en.wikipedia.org/wiki/KISS_principle) - you don't have to solve a problem in a general way, just focus on completing your current task.

A couple of days ago I took a new Jira ticket - integration of [MoPub advertisement banners](https://github.com/mopub/mopub-ios-sdk) in a posts feed. There were some non-trivial constraints:

- The advertisement should start loading only when the cell is about to show,
- All of the shown advertisements should be cached so that user will see them again after scrolling content backwards.
- There is an activity indicator while the banner is loading. If the SDK can't return it - the cell should collapse.
- The banner is continiously refreshing every 30 seconds while on the screen. This rotation should be paused just before the cell disappears.

We're using the [VIPER architecture](https://github.com/rambler-ios/The-Book-of-VIPER) for a presentation layer and [SOA](https://en.wikipedia.org/wiki/Service-oriented_architecture) for business logic. A straightforward solution would result in a simple `Service`, responsible for downloading advertisement views with `MoPub SDK`, and a `State`, which holds them after downloading.

```objc
@interface AdvertisementService

- (void)loadAdvertisementWithConfig:(AdvertisementConfig *)config 
                withCompletionBlock:()block;

@end
```

This approach may be enough for a simple application without many layers between the advertisement provider and receiver. In my case this would result in a number of useless methods in every component of the module. Otherwise the data flow is relatively simple, it has a lot of edge cases, described above.

The easiest solution was to make the `AdvertisementCell` an independent module and move there all the advertisement data flow logic. This cell still needs some data from its parent feed module - and in our case it's a promise. We don't need to wait for the advertisement to be downloaded anymore - the promise object can be obtained synchronously. Here is what the service looks like now:

```objc
@interface AdvertisementService

- (id<AdvertisementPromise>)obtainAdvertisementWithConfig:(AdvertisementConfig *)config;

@end
```

The data is available right at the moment when we need it, so we can safely configure the `UITableViewDataSource` and forget about handling edge-cases in the Feed module.

Let's investigate the `AdvertisementPromise` protocol:

```objc
typedef void(^BannerAdvertisementPromiseShowBlock)(UIView <AdvertisementView> *adView);
typedef void(^BannerAdvertisementPromiseErrorBlock)(NSError *error);

@protocol BannerAdvertisementPromise <NSObject>

- (id<BannerAdvertisementPromise>)show:(BannerAdvertisementPromiseShowBlock)block;

- (id<BannerAdvertisementPromise>)error:(BannerAdvertisementPromiseErrorBlock)block;

@end
```

It looks quite simple - the `AdvertisementPresenter` defines a behavior for two cases:

- The advertisement view is successfully loaded,
- The advertisement view has failed to download.

Here is how it looks like in our case:

```objc
- (void)didTriggerViewReadyEventWithPromise:(id<BannerAdvertisementPromise>)promise {
    self.promise = promise;
    [self.view showLoader];

    @weakify(self);
    [[self.promise show:^(UIView <AdvertisementView> *adView) {
        @strongify(self);
        [self.view hideLoader];
        [self.view showAdvertisementView:adView];
        [self.view startAdvertisementContiniousRefreshing];
    }] error:^(NSError *error) {
        @strongify(self);
        [self.interactor handleAdvertisementError:error];
        [self.view hideLoader];
        [self.moduleOutput didChangeHeightObjectFromList:self.view];
    }];
}
```

Both cases - when the advertisement is already cached and when we have to download it are handled with the same block of code. Super easy to implement, test and maintain. The last part of the puzzle is the implementation of the promise:

```objc
@implementation MoPubBannerAdvertisementPromise

#pragma mark - <AdvertisementPromise>

- (id<BannerAdvertisementPromise>)show:(BannerAdvertisementPromiseShowBlock)block {
    self.showBlock = block;

    if (!self.adView) {
        self.adView = [[MPAdView alloc] initWithAdUnitId:self.config.adUnitId
                                                    size:self.config.adSize];
        self.adView.delegate = self;
        // We don't know if the view would be displayed or not, so it's better to explicitly disable automatic content refreshing.
        [self.adView stopAutomaticallyRefreshingContents];
        [self.adView loadAd];
    } else if (self.showBlock && self.hasContentToDisplay) {
        self.showBlock(self.adView);
    }

    return self;
}

- (id<BannerAdvertisementPromise>)error:(BannerAdvertisementPromiseErrorBlock)block {
    self.errorBlock = block;
    return self;
}

#pragma mark - <MPAdViewDelegate>

- (UIViewController *)viewControllerForPresentingModalView {
    UIViewController *presentingViewController = [self obtainControllerFromResponderChain];
    return presentingViewController;
}

- (void)adViewDidLoadAd:(MPAdView *)view {
    if (self.showBlock) {
        self.showBlock(view);
    }
}

- (void)adViewDidFailToLoadAd:(MPAdView *)view {
    NSError *failError = [NSError errorWithDomain:kMoPubErrorDomain
                                             code:LiveJournalMoPubErrorLoadingFailed];
    if (self.errorBlock) {
        self.errorBlock(failError);
    }
}

#pragma mark - Private methods

- (UIViewController *)obtainControllerFromResponderChain {
    // We are sure that the first UIViewController from the responder chain can be used for displaying modal controllers with fullscreen advertisements.
    id responder = self.adView;
    while (![responder isKindOfClass:[UIViewController class]] && responder != nil) {
        responder = [responder nextResponder];
    }
    return responder;
}

@end
```

Besides aforementioned advantages this implementation provides encapsulation of specific advertisement framework details. If we'll ever want to add another one ads network - we'd just add another one promise, the table view cell will remain the same.

Working with advertisements is definitely not the only way of using promise pattern. Besides networking it can be really helpful for obtaining user input from different forms or with any other not synchronous action. If you don't want to add new huge dependencies to your project, don't worry. Due to a very simple nature of the pattern it's really easy to implement it yourself.