---
layout: post
title: Divide et Impera
description: Cleaning up the massive AppDelegate class.
---

This is just another story inspired by problems encountered in Rambler.Mail application. Our AppDelegate was really huge. And that wasn't great at all.

Spotlight, Handoff, state restoration, push notifications - all of this technologies require specific logic. Apple provided us with the only visible solution - `AppDelegate` class. And we put these methods in it. Besides system callbacks, this class seems like a nice container for other things as well - CoreData setup, analytics integration,  checking different permissions and showing onboarding screens. At the end `AppDelegate` becomes a real mess, which is hard to read and understand.

<!--more-->

One possible solution is applying the [Single Responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle) and moving each piece of logic in a separate class. So, say hello to: `StartUpConfigurator`, `PushNotificationCenter`, `StateRestorationManager`, `InitialRouter`, `CoreDataStackCoordinator` and their friends. The interface of the `AppDelegate` after this refactoring will look like:

```objc
@interface AppDelegate : UIResponder <UIApplicationDelegate>

@property (nonatomic, strong) id<StartUpConfigurator> startUpConfigurator;
@property (nonatomic, strong) id<PushNotificationCenter> pushNotificationCenter;
@property (nonatomic, strong) id<StateRestorationManager> stateRestorationManager;
@property (nonatomic, strong) id<InitialRouter> initialRouter;
...
@property (nonatomic, strong) id<CoreDataStackCoordinator> coreDataStackCoordinator;

@end
```

That's also not great. By adding multiple dependencies **we increased the overall complexity of the container**. It still implements a huge number of methods with the same implementation - choose a dependency and forward the method call to it. At first, his code is not [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) at all. Second, it's pretty easy to forgot to add forwarding somewhere, or to confuse one dependency with another. If it's not obvious, imagine unit tests for this class. We should verify every method call - and each of these tests would be the same, they just verify the concrete method forwarding.

> At this point I should stop and clarify, that I'm dealing with Objective-C case right now. For Swift this approach may be the only acceptable one.

Objective-C us a powerful technique just for this matter - it's message forwarding, especially handy with `NSProxy`. Instead of implementing all of the `AppDelegate` methods, we give the proxy class an responsibility to handle it automatically. It has a collection of mini-`AppDelegates`, responsible for dealing with a strictly defined piece of logic, and forwards the method call to the one capable of handling it.

My colleague created a nice component to abstract away from the inner complexity of the method - [RamblerAppDelegateProxy](https://github.com/rambler-ios/RamblerAppDelegateProxy). The usage is simple:

- Create a `RemoteNotificationAppDelegate` class which conforms to `UIApplicationDelegate` protocol.

```objc
@interface RemoteNotificationAppDelegate : NSObject <UIApplicationDelegate>

@end
...
@implementation RemoteNotificationAppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    NSDictionary *notification = [launchOptions objectForKey:UIApplicationLaunchOptionsRemoteNotificationKey];
    if (notification) {
        NSLog(@"Launching from push %@", notification);
    }
    return YES;
}

- (void)application:(UIApplication *)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
    NSLog(@"Did register for remote notifications");
}

- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo {
   NSLog(@"Did receive remote notification");}
}

@end
```

- Change the `main.m` file implementation.

```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        [[RamblerAppDelegateProxy injector] addAppDelegates:@[[RemoteNotificationAppDelegate new]]];

        return UIApplicationMain(argc, argv, nil, NSStringFromClass([RamblerAppDelegateProxy class]));
    }
}
```

- Use your `RemoteNotificationAppDelegate` in the same way as the standart `AppDelegate` class - just remember to not mess its responsibilities.

This approach helps to keep one of the ugliest parts of the application clean, and the future developers happy.