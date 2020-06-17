---
layout:     post
title:      WKWebView
subtitle:   WKWebView总结
date:       2020-6-3
author:     eyuxin
header-img: img/post-bg-e7.jpg
catalog: true
tags:
    - iOS
    - WKWebView
---



>   这篇文章是看了Mattt大神的文章做的总结, 因为暂时还没有中文版翻译, 原文地址: [WKWebView](https://nshipster.com/wkwebview/) 
>
>   PS: 翻译的不好请见谅, 英文好的直接阅读原文就可以了.



UIWebView臃肿并且不好用，并且疯狂的泄漏内存。它落后于Mobile Safari，无法利用其更快的JavaScript和呈现引擎。

但是，随着WKWebView和其余WebKit框架的引入，这种情况改变了。

WKWebView是iOS 8和macOS Yosemite中引入的现代WebKit API的核心。它取代了UIKit中的UIWebView和AppKit中的WebView，在两个平台之间提供了一致的API。

WKWebView拥有响应式60fps滚动，内置手势，简化的应用程序与网页之间的通信以及与Safari相同的JavaScript引擎(Nitro JavaScript引擎)，是WWDC 2014上最重要的公告之一。

在WebKit框架中，曾经使用UIWebView和UIWebViewDelegate的单一类和协议被分解为14个类和3个协议。虽然看起来比较多, 但实际上这种新的架构要干净得多，并有很多新功能。

# 从UIWebView 迁移到WKWebView

自iOS 8以来，WKWebView一直是首选的API。但是，如果您的应用仍未进行切换，请注意，iOS 12和macOS Mojave中已正式弃用UIWebView和WebView，因此应尽快更新到WKWebView。

为了帮助实现迁移，下面是UIWebView和WKWebView API的比较：

|               UIWebView                |                      WKWebView                      |
| :------------------------------------: | :-------------------------------------------------: |
| `var scrollView: UIScrollView { get }` |       `var scrollView: UIScrollView { get }`        |
|                                        | `var configuration: WKWebViewConfiguration { get }` |
|   `var delegate: UIWebViewDelegate?`   |           `var UIDelegate: WKUIDelegate?`           |
|                                        |   `var navigationDelegate: WKNavigationDelegate?`   |
|                                        |  `var backForwardList: WKBackForwardList { get }`   |

### Loading

|                          UIWebView                           |                          WKWebView                           |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
|           `func loadRequest(request: URLRequest)`            |     `func load(_ request: URLRequest) -> WKNavigation?`      |
|     `func loadHTMLString(string: String, baseURL: URL?)`     | `func loadHTMLString(_: String, baseURL: URL?) -> WKNavigation?` |
| `func loadData(_ data: Data, mimeType: String, characterEncodingName: String, baseURL: URL) -> WKNavigation?` |                                                              |
|                                                              |           `var estimatedProgress: Double { get }`            |
|                                                              |           `var hasOnlySecureContent: Bool { get }`           |
|                       `func reload()`                        |               `func reload() -> WKNavigation?`               |
|                                                              |        `func reloadFromOrigin(Any?) -> WKNavigation?`        |
|                     `func stopLoading()`                     |                     `func stopLoading()`                     |
|              `var request: URLRequest? { get }`              |                                                              |
|                                                              |                   `var URL: URL? { get }`                    |
|                                                              |                 `var title: String? { get }`                 |

### History

|            UIWebView             |                          WKWebView                           |
| :------------------------------: | :----------------------------------------------------------: |
|                                  | `func goToBackForwardListItem(item: WKBackForwardListItem) -> WKNavigation?` |
|         `func goBack()`          |               `func goBack() -> WKNavigation?`               |
|        `func goForward()`        |             `func goForward() -> WKNavigation?`              |
|  `var canGoBack: Bool { get }`   |                `var canGoBack: Bool { get }`                 |
| `var canGoForward: Bool { get }` |               `var canGoForward: Bool { get }`               |
|   `var loading: Bool { get }`    |                 `var loading: Bool { get }`                  |

### Javascript Evaluation

|                          UIWebView                           |                          WKWebView                           |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| `func stringByEvaluatingJavaScriptFromString(script: String) -> String` |                                                              |
|                                                              | `func evaluateJavaScript(_ javaScriptString: String, completionHandler: ((AnyObject?, NSError?) -> Void)?)` |

### Miscellaneous

|                   UIWebView                   |                    WKWebView                    |
| :-------------------------------------------: | :---------------------------------------------: |
| `var keyboardDisplayRequiresUserAction: Bool` |                                                 |
|          `var scalesPageToFit: Bool`          |                                                 |
|                                               | `var allowsBackForwardNavigationGestures: Bool` |

``

### Pagination

“ WKWebView”目前缺乏用于分页内容的等效API。

-   `var paginationMode: UIWebPaginationMode`
-   `var paginationBreakingMode: UIWebPaginationBreakingMode`
-   `var pageLength: CGFloat`
-   `var gapBetweenPages: CGFloat`
-   `var pageCount: Int { get }`

### Refactored into `WKWebViewConfiguration`

UIWebView上的以下属性已拆分到一个单独的configuration中：

-   `var allowsInlineMediaPlayback: Bool`
-   `var allowsAirPlayForMediaPlayback: Bool`
-   `var mediaTypesRequiringUserActionForPlayback: WKAudiovisualMediaTypes`
-   `var suppressesIncrementalRendering: Bool`

## JavaScript ↔︎ Swift 交互

UIWebView的主要改进之一是如何在应用程序及其Web内容之间来回传递交互和数据。

#### 通过User Scripts注入行为

WKUserScript允许在文档加载开始或结束时注入JavaScript行为。 这项强大的功能允许跨页面请求以安全一致的方式处理Web内容。

举一个简单的例子，下面介绍了如何注入用户脚本来更改网页的背景颜色：

```swift
let source = """
    document.body.style.background = "#777";
"""

let userScript = WKUserScript(source: source,
                              injectionTime: .atDocumentEnd,
                              forMainFrameOnly: true)

let userContentController = WKUserContentController()
userContentController.addUserScript(userScript)

let configuration = WKWebViewConfiguration()
configuration.userContentController = userContentController
self.webView = WKWebView(frame: self.view.bounds,
                         configuration: configuration)
```

创建WKUserScript对象时，将提供要执行的JavaScript代码，指定是否应在加载文档的开始或结束时注入它，以及是否应将行为用于所有frames还是仅用于main frame。 然后将用户脚本添加到WKUserContentController，该对象在传递给WKWebView的初始化程序的WKWebViewConfiguration对象上进行设置。

此示例可以轻松扩展为执行更重要的修改，例如:  [changing all occurrences of the phrase “the cloud” to “my butt”](https://github.com/panicsteve/cloud-to-butt).

### Message Handlers

通过引入消息处理程序，从Web到应用程序的通信也得到了显着改善。

就像console.log如何将信息输出到Safari Web Inspector一样，可以通过调用以下命令将网页中的信息传递回应用程序：

```swift
window.webkit.messageHandlers.<#name#>.postMessage()
```

>   这个API的真正优点是JavaScript对象会自动序列化为原生的Objective-C或Swift对象。

处理程序的名称在add（_：name）中配置，该寄存器注册符合WKScriptMessageHandler协议的处理程序：

```swift
class NotificationScriptMessageHandler: NSObject, WKScriptMessageHandler {
    func userContentController(_ userContentController: WKUserContentController,
                               didReceive message: WKScriptMessage)
    {
        print(message.body)
    }
}

let userContentController = WKUserContentController()
let handler = NotificationScriptMessageHandler()
userContentController.add(handler, name: "notification")
```

现在，当通知进入应用程序（例如，通知页面上新对象的创建）时，可以通过以下方式传递信息：

```swift
window.webkit.messageHandlers.notification.postMessage({ body: "..." });
```

>   添加用户脚本为使用消息处理程序将状态传达回应用程序的网页事件创建hooks。

可以使用相同的方法从页面中抓取信息，以在应用程序内显示或分析。

例如，如果要构建专门用于NSHipster.com的浏览器，则可以使用一个按钮在弹出窗口中列出相关文章：

```javascript
// document.location.href == "https://nshipster.com/wkwebview"
const showRelatedArticles = () => {
  let related = [];
  const elements = document.querySelectorAll("#related a");
  for (const a of elements) {
    related.push({ href: a.href, title: a.title });
  }

  window.webkit.messageHandlers.related.postMessage({ articles: related });
};
```

```swift
let js = "showRelatedArticles();"
self.webView?.evaluateJavaScript(js) { (_, error) in
    print(error)
}

// Get results in a previously-registered message handler
```

## 内容过滤规则

尽管根据您的用例，您也许可以跳过与JavaScript进行双向通讯的麻烦。

从iOS 11和macOS High Sierra开始，您可以为WKWebView指定内容过滤规则，就像[Safari Content Blocker app extension](https://developer.apple.com/library/archive/documentation/Extensions/Conceptual/ContentBlockingRules/CreatingRules/CreatingRules.html).

例如，如果您想[Make Medium Readable Again](https://makemediumreadable.com/) ，则可以在JSON中定义以下规则：

```json
let json = """
[
    {
        "trigger": {
            "if-domain": "*.medium.com"
        },
        "action": {
            "type": "css-display-none",
            "selector": ".overlay"
        }
    }
]
"""
```

将这些规则传递给compileContentRuleList（forIdentifier：encodedContentRuleList：completionHandler :)，并在完成处理程序中使用结果规则列表配置Web视图：

```swift
WKContentRuleListStore.default()
    .compileContentRuleList(forIdentifier: "ContentBlockingRules",
                            encodedContentRuleList: json)
{ (contentRuleList, error) in
    guard let contentRuleList = contentRuleList,
        error == nil else {
        return
    }

    let configuration = WKWebViewConfiguration()
    configuration.userContentController.add(contentRuleList)

    self.webView = WKWebView(frame: self.view.bounds,
                        configuration: configuration)
}
```

通过声明性声明规则，WebKit可以将这些操作编译为字节码，这些字节码可以比注入JavaScript来执行相同操作的效率更高。

除了隐藏页面元素之外，您还可以使用内容阻止规则来阻止页面资源的加载（如图像或脚本），从请求中剥离cookie到服务器，并强制页面通过HTTPS安全地加载。

## 截图功能

从iOS 11和macOS High Sierra开始，WebKit框架提供了内置的API，用于获取网页的屏幕截图。

要在所有内容加载完毕后为网络视图的可见视口拍照，请实现`webView（_：didFinish：）`委托方法以调用`takeSnapshot（with：completionHandler：）`方法，如下所示：

```swift
func webView(_ webView: WKWebView,
            didFinish navigation: WKNavigation!)
{
    var snapshotConfiguration = WKSnapshotConfiguration()
    snapshotConfiguration.snapshotWidth = 1440

    webView.takeSnapshot(with: snapshotConfiguration) { (image, error) in
        guard let image = image,
            error == nil else {
            return
        }

        …
    }
}
```

以前，获取网页的屏幕快照意味着弄乱视图层和图形上下文。 因此，一个干净的单一方法选项是该API的一个受欢迎的补充。

实际上，您每天使用的许多应用程序都依赖WebKit来呈现特别棘手的内容。 您可能没有注意到的事实应该表明网页浏览量与应用开发最佳做法一致。

