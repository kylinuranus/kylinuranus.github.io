---
layout: post
title:  "NSURLProtocol"
date:   2019-02-02
tag: iOS
permalink: /page/NSURLProtocol.html
type: legacy

---


### 定义
 
  NSURLProtocol能够让你去重新定义苹果的URL加载系统 (URL Loading System)的行为,是处于iOS APP端网络请求和网络层的中间层, URL Loading System里有许多类用于处理URL请求，比如NSURL，NSURLRequest，NSURLConnection和NSURLSession等
  
### 一些场景

- 重定向网络请求
- 无论网络请求，使用本地缓存
- 自定义网络请求的返回结果
- 一些全局的网络请求设置


### 基本使用

```
#import "MYURLProtocol.h"

@interface MYURLProtocol()<NSURLSessionDelegate>

@property (nonatomic, strong) NSOperationQueue *queue;
@property (atomic,strong,readwrite) NSURLSessionDataTask *task;
@property (nonatomic,strong) NSURLSession *session;

@end

@implementation MYURLProtocol

// 是否过滤该URL
+ (BOOL)canInitWithRequest:(NSURLRequest *)request {
    
    if ([NSURLProtocol propertyForKey:@"protocolKey" inRequest:request]) {
        return NO;
    }
    
    NSString *url = [request.URL absoluteString];
    if ([url isEqualToString:@"https://www.baidu.com/"]) {
        return YES;
    }
    return NO;
}

// 重定向，添加头部等
+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)reques
{
    NSString *webURL = @"https://www.zhangkylin.com/";
    NSURLRequest * urlReuqest = [[NSURLRequest alloc]initWithURL:[NSURL URLWithString:webURL]];
    return urlReuqest;
}

- (id)initWithRequest:(NSURLRequest *)request cachedResponse:(NSCachedURLResponse *)cachedResponse client:(id<NSURLProtocolClient>)client
{
    return  [super initWithRequest:request cachedResponse:cachedResponse client:client];
}

//开始重新加载，我这边使用NSUrlSession开始重新加载
- (void)startLoading
{
    NSMutableURLRequest *mutableReqeust = [[self request] mutableCopy];
    // 标示改request已经处理过了，防止无限循环
    [NSURLProtocol setProperty:@YES forKey:@"protocolKey" inRequest:mutableReqeust];
    
    NSURLSessionConfiguration *configure = [NSURLSessionConfiguration defaultSessionConfiguration];
    self.session  = [NSURLSession sessionWithConfiguration:configure delegate:self delegateQueue:self.queue];
    self.task = [self.session dataTaskWithRequest:mutableReqeust];
    [self.task resume];
    
}

- (void)stopLoading
{
    [self.session invalidateAndCancel];
    self.session = nil;
}


- (NSOperationQueue *)queue
{
    if (!_queue) {
        _queue = [[NSOperationQueue alloc] init];
    }
    return _queue;
}

- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error
{
    if (error != nil) {
        [self.client URLProtocol:self didFailWithError:error];
    }else
    {
        [self.client URLProtocolDidFinishLoading:self];
    }
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask
didReceiveResponse:(NSURLResponse *)response
 completionHandler:(void (^)(NSURLSessionResponseDisposition disposition))completionHandler
{
    [self.client URLProtocol:self didReceiveResponse:response cacheStoragePolicy:NSURLCacheStorageNotAllowed];
    
    completionHandler(NSURLSessionResponseAllow);
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask didReceiveData:(NSData *)data
{
    [self.client URLProtocol:self didLoadData:data];
}

- (void)URLSession:(NSURLSession *)session dataTask:(NSURLSessionDataTask *)dataTask willCacheResponse:(NSCachedURLResponse *)proposedResponse completionHandler:(void (^)(NSCachedURLResponse * _Nullable))completionHandler
{
    completionHandler(proposedResponse);
}

//TODO: 重定向
- (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task willPerformHTTPRedirection:(NSHTTPURLResponse *)response newRequest:(NSURLRequest *)newRequest completionHandler:(void (^)(NSURLRequest *))completionHandler
{
    NSMutableURLRequest *redirectRequest;
    redirectRequest = [newRequest mutableCopy];
    [[self class] removePropertyForKey:@"protocolKey" inRequest:redirectRequest];
    [[self client] URLProtocol:self wasRedirectedToRequest:redirectRequest redirectResponse:response];
    
    [self.task cancel];
    [[self client] URLProtocol:self didFailWithError:[NSError errorWithDomain:NSCocoaErrorDomain code:NSUserCancelledError userInfo:nil]];
}

@end 
 
```


### 注意点

- 让WKWebView支持NSURLProtocol [参考链接](https://blog.moecoder.com/2016/10/26/support-nsurlprotocol-in-wkwebview/)

```

从 registerSchemeForCustomProtocol: 这个方法名来猜测，它的作用的应该是注册一个自定义的 scheme，
这样对于 WebKit 进程的所有网络请求，都会先检查是否有匹配的 scheme，有的话再走主进程的 NSURLProtocol 这一套流程

Class cls = NSClassFromString(@"WKBrowsingContextController");
SEL sel = NSSelectorFromString(@"registerSchemeForCustomProtocol:");
if ([(id)cls respondsToSelector:sel]) {
    // 把 http 和 https 请求交给 NSURLProtocol 处理
    [(id)cls performSelector:sel withObject:@"http"];
    [(id)cls performSelector:sel withObject:@"https"];
}
    
// 这下 AwesomeURLProtocol 就可以用啦
[NSURLProtocol registerClass:[MYURLProtocol class]];


```


- 用NSURLSession拦截NSURLProtocol获取httpbody是为nil的,所以解决方案是使用NSURLConnection或者把body放到Header中

- 若一个项目中存在多个NSURLProtocol，那么NSURLProtocol的拦截顺序跟注册的方式和顺序有关。

对于使用registerClass方法注册的情况：多个NSURLProtocol拦截顺序为注册顺序的反序，即后注册的的NSURLProtocol先拦截。

对于通过配置NSURLSessionConfiguration对象的protocolClasses属性来注册的情况：protocolClasses这个数组里只有第一个NSURLProtocol会起作用。

- startLoading则进入死锁，直至超时，原因是执行实例方法所在的线程并没有启动runloop，而NSURLConnection这些网络请求需要依赖于runloop的，因此这些请求根本发不出去，所以必须使用异步请求，NSURLConnection/NSURLSession的异步请求的线程保证启动了runloop。


### 拓展

  NSURLProtocol是拦截请求去做缓存的，此时网络请求结果还没有回来。
  
  而NSURLCache则是响应数据的持久化层。NSURLConnection、NSURLSession和UIWebView默认都会使用NSURLCache，所有经过他们请求的数据都将被NSURLCache处理。NSURLCache不仅提供了内存和磁盘缓存方式，还有完善的缓存策略可配置。
  
  先了解下关键字段
 
- If-Modified-Since:与响应头Last-Modified相对应，其值为最后一次响应头中的Last-Modified。

- If-None-Match:与响应头Etag相对应，其值为最后一次响应头中的Etag。

- 响应头信息 Response cache headers

- Last-Modified:资源最近修改时间

- Etag:（Entity tag缩写）是请求资源的标识符，主要用于动态生成、没有Last-Modified值的资源。

- Cache-Control:缓存控制，只有包含此设置可能使用默认缓存策略。可能包含如下选项：

- max-age：缓存时间（单位：秒）。 

- public：可以被任何区缓存，包括中间经过的代理服务器也可以缓存。通常不会被使用，因为 

- max-age本身已经表示此响应可以缓存。 private:只能被当前客户端缓存，中间代理无法进行缓存。 

- no-cache:必须与服务器端确认响应是否发生了变化，如果没有变化则可以使用缓存，否则使用新请求的响应。 

- no-store:禁止使用缓存

Vary:决定请求是否可以使用缓存，通常作为缓存key值是否唯的确定因素，同一个资源不同的Vary设置会被作为两个缓存资源（注意：NSURLCache会忽略Vary请求缓存）。

默认缓存策略下当客户端发起一个请求时首先会检查本地是否包含缓存，如果有缓存则检查缓存是否过期（通过Cache-Control:max-age或者Expires判断），如果没有过期则直接使用缓存数据。如果缓存过期了，则发起一个请求给服务器端，此时服务器端对比资源Last-Modified或者Etags(二者都存在的情况下下如果有一个不同则认为缓存已过期)，如果不同则返回新数据，否则返回304 Not Modified继续使用缓存数据（客户端可以继续使用"max-age"秒缓存数据）。这个过程中客户端发送不发送请求主要看max-age是否过期，而过期后是否继续使用缓存则需要重新发起请求，服务器端根据情况通知客户端是否可以继续使用缓存（返回结果可能是200或者304）

缺点：只有在缓存过期的情况下才会去直接请求

1. 跟服务器配合缓存，通过cache-control去判断是否去服务器拉去新的信息（上面那种），
2. 完全本地缓存策略，如果设置了缓存策略，则下一次就不会进行网络请求，只要清空缓存或超过缓存限制

   


参考链接

- [https://shannonchenchn.github.io/2018/01/25/URL-Loading-System-%E6%A6%82%E8%A7%88/](https://shannonchenchn.github.io/2018/01/25/URL-Loading-System-%E6%A6%82%E8%A7%88/)
- [https://juejin.im/entry/59e02c6e6fb9a0451b038e3d](https://juejin.im/entry/59e02c6e6fb9a0451b038e3d)
- [webview预加载](https://github.com/ShannonChenCHN/iOSDevLevelingUp/
issues/55)
- [NSURLProtocol拦截wkwebview](https://blog.moecoder.com/2016/10/26/support-nsurlprotocol-in-wkwebview/)
    
 