##前言

在iOS开发过程中，经常用到一些H5交互的页面需要处理，iOS中H5开发的控件之一UIWebView的使用是必须熟练的掌握的。

##一、UIWebView 介绍
**UIWebView继承与UIView**，因此，其初始化方法和一般的View一样，通过alloc和init进行初始化。
**UIWebView  是用来加载加载网页数据的一个框**。UIWebView可以用来加载pdf、word、doc 等等文件

##二、UIWebView简单使用介绍

接下来介绍几种UIWebView网页加载展现的几种方法

####第一种方式
```
//使用 NSURLRequest 的方式加载网页
- (void)loadRequest:(NSURLRequest *)request;
```
这是加载网页最常用的一种方式，通过一个URL来进行加载，这个URL可以是**远程的**也可以是**本地的**。例如：

```
- (void)createUIWebViewTest {
// 1.创建webview
CGFloat width = self.view.frame.size.width;
CGFloat height = self.view.frame.size.height;
UIWebView *webView = [[UIWebView alloc] initWithFrame:CGRectMake(0, 0, width, height)];


// 2.1 创建一个远程URL
NSURL *remoteURL = [NSURL URLWithString:@"http://www.baidu.com"];

// 2.2 创建一个本地URL
NSString *urlStr = [[NSBundle mainBundle] pathForResource:@"first" ofType:@"html"];
NSURL *locationURL = [NSURL URLWithString:urlStr];   

// 3.创建Request
NSURLRequest *request =[NSURLRequest requestWithURL:remoteURL];
// 4.加载网页
[webView loadRequest:request];
// 5.最后将webView添加到界面
[self.view addSubview:webView];
self.webView = webView;
}

```

####第二种方式

```
/* 
功能：加载HTML字符串
string为要加载的本地HTML字符串
baseURL用来确定htmlString的基准地址，相当于HTML的<base>标签的作用，定义页面中所有链接的默认地址
*/
- (void)loadHTMLString:(NSString *)string baseURL:(nullable NSURL *)baseURL;
```
第一个参数：主要强调加载HMTL的 **HTML语句，并没强调必须是HTML完整的内容**，HTML文件需要一个完整的HTML格式的文件内容，而这个方法可以直接加载HMTL的语句。
```
// HTML是网页的设计语言  
// <>表示标记  
// 应用场景:截取网页中的某一部分显示  
// 例如:网页的完整内容中包含广告!加载完成页面之后,把广告部分的HTML删除,然后再加载  
// 被很多新闻类的应用程序使用  
[self.webView loadHTMLString:@"<p>Hello</p>" baseURL:nil];
```

第二个参数：baseURL是可以自定义的一个路径，用于寻找html文件中引用的图片等素材。

例如：
```
- (void)loadLocalHTMLFileToUIWebView{
// 获取本地html文件文件路径
NSString *localHTMLPageName = @"Test";
NSString *path = [[NSBundle mainBundle] pathForResource:localHTMLPageName ofType:@"html"];
// 从html文件中读取html字符串
NSString *htmlString = [NSString stringWithContentsOfFile:path 
encoding:NSUTF8StringEncoding 
error:NULL];
// 加载本地HTML字符串
[self.webView loadHTMLString:htmlString baseURL:[[NSBundle mainBundle] bundleURL]];
}
```
####第三种方式
```
- (void)loadData:(NSData *)data MIMEType:(NSString *)MIMEType textEncodingName:(NSString *)textEncodingName baseURL:(NSURL *)baseURL;
```
这个方法使用比较少，但是更加自由，其中data是文件数据，MIMEType是文件类型，textEncodingName是编码类型，baseURL是素材资源路径。例如：

```
#pragma 以二进制数据的形式加载文件  
- (void)loadDataFile  {  
// 最最常见的一种情况  
// 打开IE,访问网站,提示你安装Flash插件  
// 如果没有这个应用程序,是无法用UIWebView打开对应的文件的  

// 应用场景:加载从服务器上下载的文件,例如pdf,或者word,图片等等文件  
NSURL *fileURL = [[NSBundle mainBundle] URLForResource:@"iOS 7 Programming Cookbook.pdf" withExtension:nil];  

NSURLRequest *request = [NSURLRequest requestWithURL:fileURL];  
// 服务器的响应对象,服务器接收到请求返回给客户端的  
NSURLResponse *respnose = nil;  

NSData *data = [NSURLConnection sendSynchronousRequest:request returningResponse:&respnose error:NULL];  

NSLog(@"%@", respnose.MIMEType);  

// 在iOS开发中,如果不是特殊要求,所有的文本编码都是用UTF8  
// 先用UTF8解释接收到的二进制数据流  
[self.webView loadData:data MIMEType:respnose.MIMEType textEncodingName:@"UTF8" baseURL:nil];  
}

// 本地方法：
//从本地加载
NSString *thePath = [[NSBundle mainBundle] pathForResource:@"User_Guide" ofType:@"pdf"];

if (thePath) {
NSData *pdfData = [NSData dataWithContentsOfFile:thePath];
[self.webView loadData:pdfData MIMEType:@"application/pdf" textEncodingName:@"utf-8" baseURL:nil];
}
```
综上：UIWebView已经可以简单的使用到项目中，通过以上三种不同的方式

##三、UIWebView详细介绍
接下来会从UIWebView有以下三个方面的内容：
1.UIWebView的一些属性和方法。
2.UIWebView的代理方法
3.UIWebView和Javascript之间的交互

###1. UIWebView对应的方法和属性
```
#pragma mark - 判断属性
// 是否可以后退
@property (nonatomic, readonly, getter=canGoBack) BOOL canGoBack;
// 是否可以向前
@property (nonatomic, readonly, getter=canGoForward) BOOL canGoForward;
// 是否正在加载
@property (nonatomic, readonly, getter=isLoading) BOOL loading;

#pragma mark - 操作方法
// 刷新网页
- (void)reload;
// 停止加载网页
- (void)stopLoading;
// 后退
- (void)goBack;
// 前进
- (void)goForward;
```

####2.UIWebView代理

```
//是否允许加载网页，也可获取js要打开的url，通过截取此url可与js交互
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request 
navigationType:(UIWebViewNavigationType)navigationType;
//开始加载网页
- (void)webViewDidStartLoad:(UIWebView *)webView;
//网页加载完成
- (void)webViewDidFinishLoad:(UIWebView *)webView;
//网页加载错误
- (void)webView:(UIWebView *)webView didFailLoadWithError:(NSError *)error;
```
####3.UIWebView和JavaScript交互
UIWebView和JavaScript的交互主要涉及两个方面：JS执行OC代码，OC调用JS代码。
同时还有一些更加细节的内容，两者之间的调用参数问题。
关于UIWebView和JavaScript交互会专一做一篇专题的博客。

