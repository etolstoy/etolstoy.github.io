---
layout: post
title: The Story of One Unit Test
description: What makes a clean test? Three things. Readability, readability, and readability.
---

> What makes a clean test? Three things. Readability, readability, and readability. Readability is perhaps even more important in unit tests than it is in production code. What makes tests readable? The same thing that makes all code readable: clarity, simplicity, and density of expression. In a test you want to say a lot with as few expressions as possible.

*Robert Martin, Clean Code*

It's very easy to turn your unit tests into an unreadable and unmaintainable mess. The main reason of such attitude is a wrong understanding of test cases role. It's not *"write once, never read"*. It's not just in verifying your methods behaviour. Unit tests are all about documentation. It's *"write once, refactor often, consult daily"*. The easier a test is, the more it tells to a developer.

<!--more-->

I won't repeat all of Uncle Bob's thoughts here. This post is meant to show how this rule of thumb can affect a real-world problem.

One of the key objects in LiveJournal application is `OperationScheduler`. It distributes `NSOperations` between multiple operation queues, handle their priority and does other related things. One of possible application usage scenarios is authorization token expiration. If we catch this error, we initiate token refresh by executing a corresponding operation. All pending data operations should be paused until the session is restored.

The easiest approach is making use of the `-addDependency` method. Pending data operations are dependent on token refresh operation. Our `OperationScheduler` implements this logic in few lines of code:

```objc
for (NSOperation *generalOperation in self.generalQueue.operations) {
    [generalOperation addDependency:operation];
}

[self.authQueue addOperation:operation];
```

But testing this behaviour is not that easy. To be sure everything works as expected, I've implemented the following test scenario:

- The initial data operation is passed to a scheduler,
- 5 other data operations are passed to the scheduler,
- This initial operation creates authorization operation on execute.

If everything is fine, the sequence of operations is: 
`initial -> authorization -> general (x5)`

[The first revision](https://gist.github.com/etolstoy/dd9552a247df0b327abc898ddb8be5ad#file-operationschedulertests-1-m) was not so great. Let's explore it block by block.

The test should be asynchronyous. XCTestExpectation is the best option around.

```objc
XCTestExpectation *expectation = [self expectationWithDescription:@"Last operation fired"];
```

The exact mechanism of creating an expectation is just an implementation detail, so we can safely hide it in a category and change this line to:

```objc
XCTestExpectation *expectation = [self expectationForCurrentTest];
```

Each operation type has its identifier:

```objc
NSString *const kAuthOperationName = @"AuthOperation";
NSString *const kInitialOperationName = @"InitialOperation";
NSString *const kGeneralOperationName = @"GeneralOperation";
```

It's not crucial to declare this constants right in the unit test body, so they can be safely moved to a separate file.

The `NSBlockOperation` syntax is really wordy:

```objc
NSBlockOperation *authOperation = [NSBlockOperation blockOperationWithBlock:^{
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        @synchronized(operationNames) {
            [operationNames addObject:kAuthOperationName];
        }
        [NSThread sleepForTimeInterval:0.05];
    });
}];
```

The implementation of authorization and general operations is very straightforward - it just adds their identifiers to an array representing the call sequence. Once again, it's just an implementation detail, so we can go forward and extract these operations somewhere else. However, the `initialOperation` is important for understanding what's going on, so it shouldn't be ignored. This results in an `Environment` class:

```objc
@interface TestBlockingByAuthOperationEnvironment : NSObject

- (void)setupEnvironmentWithTestCase:(XCTestCase *)testCase
              generalOperationsCount:(NSUInteger)generalOperationsCount
               initialOperationBlock:(void (^)(void))initialOperationBlock;

@property (strong, nonatomic) NSBlockOperation *initialOperation;
@property (strong, nonatomic) NSBlockOperation *authOperation;
@property (strong, nonatomic) NSArray *generalOperations;
@property (strong, nonatomic) NSArray *firedOperationNames;

@end
```

This class encapsulates 60 lines of unimportant details. The operation setup can be refactored using cleaner syntax:

```objc
TestBlockingByAuthOperationEnvironment *environment = [TestBlockingByAuthOperationEnvironment new];
[environment setupEnvironmentWithTestCase:self
                   generalOperationsCount:kGeneralOperationsCount
                    initialOperationBlock:^{
    [self.scheduler addAuthOperation:environment.authOperation];

    for (NSOperation *operation in environment.generalOperations) {
        [self.scheduler addGeneralOperation:operation];
    }
}];
```

This is the last change - `// then` section is already simple and easy to understand.

[The initial unit test](https://gist.github.com/etolstoy/dd9552a247df0b327abc898ddb8be5ad#file-operationschedulertests-1-m) had 56 lines of code. [After refactoring](https://gist.github.com/etolstoy/dd9552a247df0b327abc898ddb8be5ad#file-operationschedulertests-2-m) it's shortened to 26 lines. What's more important, the readability of the second version is certainly much greater. No more magic numbers, constants, ugly nested blocks and `@synchronized` directives - the next one who reads this file will focus on the test itself.