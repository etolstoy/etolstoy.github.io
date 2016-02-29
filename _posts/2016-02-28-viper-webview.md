---
layout: post
title: Using UIWebView with VIPER
description: How to separate UIWebView responsibilities using VIPER architecture
---

I was working on [Rambler.Mail](https://itunes.apple.com/app/id981598964) for the whole year. It's a client for one of the most popular Russian e-mail providers. The application itself is rather ordinary, it has the very same features you'd expect from an e-mail application - and nothing more. However, I spent a lot of time building a solid and maintainable architecture for both business logic and presentation layers.

One of the most interesting features was a rendering engine for displaying e-mail content. I won't dive into implementation details - all you need to know at the moment is that we used `UIWebView` and [WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge) - a great library for bridging native code and javascript. The result system was monlithic, difficult to understand and reason about - undoubtedly not the best piece of software I've ever built. Before I had a chance to refactor it I was switched to another project. To my great surprise I've encountered there a very similar task - I had to build a rendering engine for displaying blog posts. This time I was prepared well - I already knew how **NOT** to build such systems. So, instead of spaghetti code, I've used another approach - the cell containing `UIWebView` was built using the **VIPER** architecture.

The fundamental idea of my implementation was to separate `UIWebView` responsibilities in two groups: `<WebEngine>`, which is a dependency of the `Interactor`, and `<WebPresentation>`, which belongs to the `View`.

Their responsibilities are:

**<WebEngine>**

- Loads HTML code,
- Executes JS scripts,
- Renders content with a set of predefined actions,
- Notifies the `Interactor` of drawing, rendering and loading processes completion.

**<WebPresentation>**

- Notifies the `View` of different events: links, images and different embeds taps,
- Provides an interface of obtaining real content size and other view properties.

![WebView VIPER module scheme](/public/img/posts/viper-webview.png)

The data flow in this module is really simple:

1. The `Assembly` setups two delegates for  the `WebViewImplementation` object - one is the `View` and another is the `Interactor`.
2. The `View` (which is `UITableViewCell` in my case) is an entry point of the module. It's only input is raw html data.
3. The `View` passes `UIWebView` as a parameter in setup method of `<WebPresentation>`.
4. The `View` passes HTML data to the `Interactor` through the `Presenter`.
5. The Interactor setups the `<WebEngine>` environment by executing a number of JS scripts.
6. The Interactor modifies this HTML and passes it to the `<WebEngine>` using `-loadHtml:` method.
7. The `<WebEngine>` notifies the `Interactor` on loading completion.
8. The `Interactor` initiates content rendering by passing a `RenderStrategy` to the `<WebEngine>`. This strategy incapsulates all of the rendering steps for the current document.
9. `<WebEngine>` notifies the `Interactor` on rendering completion.
10. The `Interactor` notifies the `Presenter`, which obtains the content size from the `View` and passes it to the module output.

These are very simple steps - each method consists of no more than 5 lines of code.

Let's investigate the implementation of each component.

**`PostContentView`**

```objc
@interface PostContentCell : UITableViewCell <PostContentViewInput, WebPresentationDelegate>

@property (weak, nonatomic) IBOutlet UIWebView *contentWebView;
@property (strong, nonatomic) id<PostContentViewOutput> output;
@property (strong, nonatomic) id<WebPresentation> webPresentation;

@end

@implementation PostContentCell

- (BOOL)shouldUpdateCellWithObject:(PostContentCellObject *)object {
    [self.output didTriggerModuleSetupEventWithHtmlContent:object.htmlContent];
    
    return YES;
}

#pragma mark - PostContentViewInput

- (void)setupInitialState {
    [self.webPresentation setupWithWebView:self.contentWebView];
}

#pragma mark - WebPresentationDelegate

- (void)webPresentation:(id<WebPresentation>)webPresentation
             didTapLink:(NSURL *)link {
    [self.output didTriggerLinkTapEventWithURL:link];
}

- (void)webPresentation:(id<WebPresentation>)webPresentation
    didTapImageWithLink:(NSURL *)link {
    [self.output didTriggerImageTapWithURL:link];
}

@end
```

**`PostContentPresenter`**

```objc
@implementation PostContentPresenter

#pragma mark - PostContentViewOutput

- (void)didTriggerModuleSetupEventWithHtmlContent:(NSString *)htmlContent {
    self.rawHtmlContent = htmlContent;
    
    [self.interactor renderContent:self.rawHtmlContent];
}

- (void)didTriggerLinkTapEventWithURL:(NSURL *)url {
    [self.router showBrowserModuleWithURL:url];
}

- (void)didTriggerImageTapWithURL:(NSURL *)url {
    [self.router showFullscreenImageModuleWithImageURL:url];
}

#pragma mark - PostContentInteractorOutput

- (void)didCompleteRenderingContent {
    CGFloat contentHeight = [self.view obtainCurrentContentHeight];
    [self.moduleOutput didUpdatePostContentWithContentHeight:contentHeight];
}

@end

```

**`PostContentInteractor`**

```objc
@interface PostContentInteractor : NSObject <PostContentInteractorInput, WebEngineDelegate>

@property (nonatomic, weak) id<PostContentInteractorOutput> output;

@property (nonatomic, strong) id<WebEngine> webEngine;
@property (nonatomic, strong) ContentHTMLComposer *contentComposer;
@property (nonatomic, strong) JSScriptProvider *scriptProvider;
@property (nonatomic, strong) ContentRenderStrategyFactory *renderStrategyFactory;

@end

@implementation PostContentInteractor

#pragma mark - PostContentInteractorInput

- (void)renderContent:(NSString *)content {
    NSArray *scripts = [self.scriptProvider obtainScriptsForPostEnvironmentSetup];
    for (NSString *script in scripts) {
        [self.webEngine executeScript:script];
    }
    
    NSString *processedContent = [self.contentComposer composeValidHtmlForRenderingFromRawHtml:content];
    
    [self.webEngine loadHtml:processedContent];
}

#pragma mark - WebEngineDelegate

- (void)didCompleteLoadingContentWithWebEngine:(id<WebEngine>)webEngine {
    ContentRenderStrategy *renderStrategy = [self.renderStrategyFactory postContentRenderStrategy];
    [self.webEngine renderContentWithStrategy:renderStrategy];
}

- (void)didCompleteRenderingContentWithWebEngine:(id<WebEngine>)webEngine {
    [self.output didCompleteRenderingContent];
}

@end
```

**`WebEngineImplementation`**

```objc
@interface WebEngineImplementation : NSObject <WebPresentation, WebEngine>

@property (weak, nonatomic) id<WebEngineDelegate> webEngineDelegate;
@property (weak, nonatomic) id<WebPresentationDelegate> webPresentationDelegate;
@property (strong, nonatomic) WebBridgeFactory *bridgeFactory;

@end

@implementation WebEngineImplementation

#pragma mark - WebPresentation

- (void)setupWithWebView:(UIWebView *)webView {
    self.webView = webView;
    
    self.bridge = [self.bridgeFactory obtainBridgeForWebView:webView
                                             webViewDelegate:self];
    
    [self setupJavascriptHandlers];
}

#pragma mark - Handlers Setup

- (void)setupJavascriptHandlers {
    @weakify(self);
    [self.bridge registerHandler:kImageTapHandler
                         handler:^(id data, WVJBResponseCallback responseCallback) {
                             @strongify(self);
                             NSString *imageURLString = data[kDataLinkKey];
                             NSURL *imageURL = [NSURL URLWithString:imageURLString];
                             [self.webPresentationDelegate webPresentation:self
                                                       didTapImageWithLink:imageURL];
                         }];
}

#pragma mark - WebEngine

- (void)loadHtml:(NSString *)html {
    [self.webView loadHTMLString:html
                         baseURL:nil];
}

- (void)executeScript:(NSString *)script {
    [self.webView stringByEvaluatingJavaScriptFromString:script];
}

- (void)renderContentWithStrategy:(ContentRenderStrategy *)renderStrategy {
    dispatch_group_t group = dispatch_group_create();
    JSResponseBlock responseBlock = ^(NSDictionary *responseData) {
        NSString *logDescription = responseData[JSLogDescriptionKey];
        DDLogVerbose(@"%@", logDescription);
        dispatch_group_leave(group);
    };
    
    for (NSString *handlerKey in renderStrategy.scriptNames) {
        dispatch_group_enter(group);
        [self.bridge callHandler:handlerKey
                            data:renderStrategy.scriptPayloads[handlerKey]
                responseCallback:responseBlock];
    }
    
    [self.webEngineDelegate didCompleteRenderingContentWithWebEngine:self];
}

#pragma mark - UIWebViewDelegate

- (void)webViewDidFinishLoad:(UIWebView *)webView {
    [self.webEngineDelegate didCompleteLoadingContentWithWebEngine:self];
}

- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType {
    NSString *urlString = request.URL.absoluteString;
    
    if (navigationType == UIWebViewNavigationTypeLinkClicked) {
        [self.webPresentationDelegate webPresentation:self
                                           didTapLink:request.URL];
        return NO;
    }
    
    return YES;
}

```

Besides the clear separation of concerns, we've also hidden the fact of using a third party library for bridging - it's just an implementation detail of our `<WebEngine>`. Unit tests for this setup are easy and reliable - all that we have to test is the data flow.

Sometimes VIPER may seem like an overcomplicated abstraction. However, that's definitely not the case. Separating the module logic into different layers helped me to break the indivisible `UIWebView` between them.

The rendering process itself is a really interesting topic for another blog post. I'm going to cover content preprocessing mechanism, `WebViewJavascriptBridge` usage, callbacks organization, rendering steps for different types of content and so on.