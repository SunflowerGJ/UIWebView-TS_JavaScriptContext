I've worked on more “hybrid" iOS applications than I care to admit.  One of the major pain points of these apps is always communication across the web/native boundary - between JavaScript running in a `UIWebView` to ObjectiveC in the app shell. 

We all know that the only official way to call into a `UIWebView` from ObjectiveC is via `stringByEvaluatingJavaScriptFromString`.  And the typical way to call out from JavaScript is some manner of setting window.location to trigger a `shouldStartLoadWithRequest:` callback on the `UIWebView` delegate.   Another oft-considered technique is to implement a custom `NSURLProtocol` and intercept requests made via `XMLHttpRequest`.  

Apple gives us a public JavaScriptCore framework (part of WebKit) in iOS7, and JavaScriptCore provides simple mechanisms to proxy objects and methods between ObjectiveC and the JavaScript “context”.  Unfortunately, while it is common knowledge that `UIWebView` is built upon WebKit and in turn is using JavaScriptCore, Apple didn’t expose a mechanism for us to access this infrastructure.

It is possible to manhandle `UIWebView` and retrieve its internal JSContext object by using a KVO keypath to access undocumented properties deep within `UIWebView`.  [impathic describes this technique on his blog][1].  A major drawback of this approach, of course, is that it relies on the internal structure of the `UIWebView`.

I present an alternative approach to retrieving a `UIWebView`’s `JSContext`.  Of course mine is also non-Apple-sanctioned, and could break also.  I probably won’t try shipping this in an application to the App Store.  But it is less likely to break, and I think too that it does not have any specific dependency on the internals of `UIWebView` other than `UIWebView` itself using WebKit and JavaScriptCore.  (There is one small caveat to this, explained later.)


The basic mechanism in play is this:  WebKit communicates “frame loading events” to its clients (such as `UIWebView`) using “`WebFrameLoadDelegate`” callbacks performed in similar fashion to how `UIWebView` communicates page loading events via its own `UIWebViewDelegate`.   One of the `WebFrameLoadDelegate` methods is `webView:didCreateJavaScriptContext:forFrame:`   Like all good event sources, the WebKit code checks to see if the delegate implements the callback method, and if so makes the call.  Here’s what it looks like in the WebKit source (WebFrameLoaderClient.mm):

    if (implementations->didCreateJavaScriptContextForFrameFunc) {
        CallFrameLoadDelegate(implementations->didCreateJavaScriptContextForFrameFunc, webView, @selector(webView:didCreateJavaScriptContext:forFrame:),
            script.javaScriptContext(), m_webFrame.get());
    }

It turns out that in iOS, inside `UIWebView`, whatever object is implementing WebKit `WebFrameLoadDelegate` methods doesn’t actually implement `webView:didCreateJavaScriptContext:forFrame:`, and hence WebKit never performs this call.  If the method existed on the delegate object then it would be called automatically.

Well, there are a handful of ways in ObjectiveC to dynamically add a method to an existing class or object.  The easiest way is via a category.  My approach hinges on extending `NSObject` via a category, to implement `webView:didCreateJavaScriptContext:forFrame:`. 

Indeed, adding this method prompts WebKit to start calling it, since any object (including some sink object inside `UIWebView`) that inherits from `NSObject` now has an implementation of `webView:didCreateJavaScriptContext:forFrame:`.  If the sink inside `UIWebView` were to implement this method in the future then this approach would likely silently fail since my implementation on `NSObject` would never be called.

When our method is called by WebKit it passes us a WebKit `WebView` (not a `UIWebView`!), a JavaScriptCore `JSContext` object, and WebKit `WebFrame`.  Since we don’t have a public WebKit framework to provide headers for us, the WebView and WebFrame are effectively opaque to us.  But the `JSContext` is what we’re after, and it’s fully available to us via the JavaScriptCore framework.  (In practice, I do end up calling one method on the `WebFrame`, as an optimization.) 

The question becomes how to equate a given JSContext back to a particular `UIWebView`.  The first thing I tried was to use the `WebView` object we’re handed and walk up the view hierarchy to find the owning `UIWebView`.  But it turns out this object is some kind of proxy for a `UIView` and is not actually a `UIView`.  And because it is opaque to us I really didn’t want to use it.

My solution is to iterate all of the `UIWebViews` created in the app (see the code to see how I do this) and use `stringByEvaluatingJavaScriptFromString:` to place a token “cookie” inside its JavaScriptContext.  Then, I check for the existence of this token in the `JSContext` I’ve been handed - if it exists then this is the `UIWebView` I’ve Been Looking For!

Once we have the `JSContext` we can do all sorts of nifty things.  My test app shows how we can map ObjectiveC blocks and objects directly into the global namespace and access/invoke them from JavaScript.

**One Last Thing**  

(Sorry, couldn’t resist.)  With this bridge in place, it’s now possible to invoke ObjectiveC code in the app via the desktop Safari WebInspector console!  Try it out with the test app by attaching the WebInspector and typing `sayHello();` into the console input field.  Now that’s cool!

Nick Hodapp (a.k.a TomSwift), November 2013


  [1]: http://blog.impathic.com/post/64171814244/true-javascript-uiwebview-integration-in-ios7
  译文：
  我曾经做过很多的混合iOS应用，但是我不屑于承认。这些应用的一个主要痛点是通过web/native边界(运行在UIWebView中的JavaScript与运行在App中的ObjectiveC之间)进行交互。

我们都知道官方只给出一种方法从ObjectiveC调到网页里，是通过stringByEvaluatingJavaScriptFromString方法。还有一种调用JavaScript的典型办法是人为设置window.location去触发UIWebView的代理方法shouldStartLoadWithRequest:。另一种常常使用到的技术是实现自定义的NSURLProtocol并拦截通过XMLHttpRequest发出的请求。

在iOS7中苹果给出了一个公开的框架JavaScriptCore(WebKit的一部分)，这个框架提供了简单机制在ObjectiveC和JavaScript的环境中互相调用对象和方法。众所周知，UIWebView建立在WebKit之上最终也是使用了JavaScriptCore，不幸的是苹果没有暴露一些途径给我们去访问这套框架。

可以使用KVC简单粗暴的获取这个深植于UIWebView内部官方文档却未定义的属性JSContext。这篇博客介绍了这个技术。当然，这个方法的主要缺点是他依赖UIWebView的内部构造。

我介绍一个可供选择的方法去获取UIWebView的JSContext。当然我的方法也不是官方的，可能被拒。我应该不会尝试提交一个这样的应用到AppStore。但它看来至少不那么容易被拒，我认为它并没有特别地依赖于UIWebView的内部结构不同于UIWebView自己用WebKit和JavaScriptCore。(这有个小警告，一会解释)

基本原理是这样的：WebKit用WebFrameLoadDelegate回调与客户端进行通讯就好像UIWebView传达页面加载事件通过他自己的UIWebViewDelegate。WebFrameLoadDelegate其中一个方法是webView:didCreateJavaScriptContext:forFrame:就像所有事件源，WebKit的代码去检测他的代理是否实现了回调方法，如果实现了就调用此方法。下面是WebKit的部分源码(WebFrameLoaderClient.mm)

if (implementations->didCreateJavaScriptContextForFrameFunc) {
    CallFrameLoadDelegate(implementations->didCreateJavaScriptContextForFrameFunc, webView, @selector(webView:didCreateJavaScriptContext:forFrame:), script.javaScriptContext(), m_webFrame.get());
}
证实在iOS，UIWebView内，不论任何对象实现WebKit的WebFrameLoadDelegate方法，并不是真的实现webView:didCreateJavaScriptContext:forFrame:所以WebKit从不会调用此方法。如果此方法存在于代理对象中，它将会被自动调用。

既然如此，在OC中有很多的办法给现有的类和对象动态的增添一个方法。最简单的办法就是通过扩展。我给已有的类NSObject添加一个扩展去实现webView:didCreateJavaScriptContext:forFrame:方法。

的确，添加这个方法让WebKit开始调用它，因为任何对象(包括UIWebView中的一些sink object)都继承自NSObject，现在都实现了webView:didCreateJavaScriptContext:forFrame:这个方法。如果未来UIWebView内部的sink object实现了这个代理方法，那么这个途径就是失效因为我们自己实现的分类永远不会被调用。

当我们的方法被WebKit调用的时候会传给我们一个WebKit中的WebView(不是UIWebView)，一个JavaScriptCore的JSContext对象和WebKit的WebFrame。因为没有一个公开的WebKit框架的头文件提供给我们，所以WebView和WebFrame对我们来说非常透明。但是JSContext正是我们寻找的，通过JavaScriptCore框架对我们来说完全是适用的。(在实际中，我最终在WebFrame中调用方法，作为一个最佳状态)

问题现在就变成怎样根据JSContext反找到对应的UIWebView。首先我尝试使用WebView对象我们控制和沿着继承的view去找到他拥有的UIWebView.但是后来证明这个对象是一些UIView的代理，并不是一个真正的UIView。并且因为他对我们来说是透明的，我也没有打算使用它。

我的解决方案是迭代所有在app中所创建的UIWebViews(参考代码，我是怎么样做的)并且使用stringByEvaluatingJavaScriptFromString:去储存一个token"cookie"在JavaScriptContext中，然后我在JSContext中查找已经存在的这个token，如果他存在这个UIWebView就是我所要找的。

一旦我们有了JSContext我们就可以做一些很有趣的事情。我的测试App展示了我们怎样映射ObjectiveC的blocks和对象到全局命名空间并且通过JavaScript访问和调用它们。
