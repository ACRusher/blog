---
layout: post
title: apache-http-client(一) 
categories: [blog ]
tags: [软件开发  ]
description: apache-http-client(一) 
---


# apache-http-client(一)


## quick start

### 前言

​   虽然java net 包提供了基本的Http 接口，但是不能满足大多数web应用程序对扩展性和丰富功能性的要求。鉴于此，apache httpclient 提供高效的、功能丰富的http协议实现软件包。http传输协议基于 httpcore软件包，网络IO基于经典的Block IO。

​   httpclient并不是一个完整的web浏览器，它的目的是接受http消息，但不会处理消息内容。

引入httpclient maven 依赖

```xml
<dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>4.5.2</version>
</dependency>
```

发送http get 和 http post 请求代码

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpGet = new HttpGet("http://www.taobao.com/");
CloseableHttpResponse response1 = httpclient.execute(httpGet);
try {
  System.out.println(response1.getStatusLine());
  HttpEntity entity1 = response1.getEntity();
  String response=EntityUtils.toString(entity1,"utf-8");
  System.out.println(response);
  //必须完全的consume，否则connection manager可能无法复用连接
  EntityUtils.consume(entity1);
} finally {
  //必须close response ， 否则无法释放持有的connection
  response1.close();
}

HttpPost httpPost = new HttpPost("https://www.taobao.com/");
List<NameValuePair> nvps = new ArrayList<NameValuePair>();
nvps.add(new BasicNameValuePair("username", "vip"));
nvps.add(new BasicNameValuePair("password", "secret"));
httpPost.setEntity(new UrlEncodedFormEntity(nvps));
CloseableHttpResponse response2 = httpclient.execute(httpPost);

try {
  System.out.println(response2.getStatusLine());
  HttpEntity entity2 = response2.getEntity();
  String response=EntityUtils.toString(entity2,"utf-8");
  System.out.println(response);
  EntityUtils.consume(entity2);
} finally {
  response2.close();
}
```

## http client tutorial


### 基本原理

请求执行的基本代码结构如下

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    <...>
} finally {
    response.close();
}
```



Http协议定义了 `get,post,put,delete,head,trace,options` 方法，对应到实现类分别为: `HttpGet,HttpPost,HttpPut,HttpDelete,HttpHead,HttpTrace,HttpOptions`

所有的方法定义构造函数，都需要一个请求uri，httpclient提供了uri的工具类`URIBuilder`,使用方法：

```java
URI uri = new URIBuilder()
        .setScheme("http")
        .setHost("www.google.com")
        .setPath("/search")
        .setParameter("q", "httpclient")
        .setParameter("btnG", "Google Search")
        .setParameter("aq", "f")
        .setParameter("oq", "")
        .build();
HttpGet httpget = new HttpGet(uri);
System.out.println(httpget.getURI());
```

有时我们需要解析http返回的header数据，有三种使用方式：

> 直接取header

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, 
    HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie", 
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie", 
    "c2=b; path=\"/\", c3=c; domain=\"localhost\"");
Header h1 = response.getFirstHeader("Set-Cookie");
System.out.println(h1);
Header h2 = response.getLastHeader("Set-Cookie");
System.out.println(h2);
Header[] hs = response.getHeaders("Set-Cookie");
System.out.println(hs.length);
```

> 使用header迭代器

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, 
    HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie", 
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie", 
    "c2=b; path=\"/\", c3=c; domain=\"localhost\"");

HeaderIterator it = response.headerIterator("Set-Cookie");

while (it.hasNext()) {
    System.out.println(it.next());
}
```

> 使用header element 迭代器

```java
HttpResponse response = new BasicHttpResponse(HttpVersion.HTTP_1_1, 
    HttpStatus.SC_OK, "OK");
response.addHeader("Set-Cookie", 
    "c1=a; path=/; domain=localhost");
response.addHeader("Set-Cookie", 
    "c2=b; path=\"/\", c3=c; domain=\"localhost\"");

HeaderElementIterator it = new BasicHeaderElementIterator(
    response.headerIterator("Set-Cookie"));

while (it.hasNext()) {
    HeaderElement elem = it.nextElement(); 
    System.out.println(elem.getName() + " = " + elem.getValue());
    NameValuePair[] params = elem.getParameters();
    for (int i = 0; i < params.length; i++) {
        System.out.println(" " + params[i]);
    }
}
```

`HttpEntity` 是 http协议传输的内容主体，在client端http协议定义了post 和 put 方法可以传输`HttpEntity` ， 在server端的大部分response都是要包含 `HttpEntity`的，当然也有例外： 当server回复client的`HttpHead`请求时，如果是 `204(no content)/304(not modified)/205(reset content)` 则不需要传回`HttpEntity`.

`HttpClient` 将`HttpEntity`区分为三种类型：

- streamed  数据流类型，不可重复，只能从http response中检索一次
- self-contained 自持有类型，可重复，保存于客户端内存中
- wrapping 包裹类型，取决于被包含的Entity属性

上面三种类型的区分对应 http连接管理器 `ConnectManager` 是十分重要的，因为连接管理器需要在合适的时机回收和重复利用connection，无法回收会导致连接被抛弃和重建。

从server response中读取`HttpEntity` 可以使用两种方法：

- `HttpEntity#getContent()`  获取 `java.io.InputStream`
- `HttpEntity#writeTo(OutputStream)`  写入到输出流

下面是`StringEntity`的使用demo:

```java
StringEntity myEntity = new StringEntity("important message", 
   ContentType.create("text/plain", "UTF-8"));

System.out.println(myEntity.getContentType()); 
System.out.println(myEntity.getContentLength());
System.out.println(EntityUtils.toString(myEntity));
System.out.println(EntityUtils.toByteArray(myEntity).length);
```

> 重要！请一定记得正确关闭资源，典型的代码结构如下

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        InputStream instream = entity.getContent();
        try {
            // do something useful
        } finally {
            //正确关闭流，会完整读取response后再关闭
            //这个等同于 EntityUtils.consume(entity)
            //如果有特殊业务需求，不需要完整消费server的回复，可以注释掉
            //需要知道的是，一旦注释掉，connection manager 将无法复用这个链接
            instream.close();
        }
    }
} finally {
    // 无论底层的 socket是否消费完成，强制关闭链接，并释放系统资源
    response.close();
}
```

> 关于EntityUtils的说明

除非server端是被充分信任的，并且response不会过长，否则不建议使用 `EntityUtils`，在response过长时，使用 `EntityUtils` 会导致内存耗尽。

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/");
CloseableHttpResponse response = httpclient.execute(httpget);
try {
    HttpEntity entity = response.getEntity();
    if (entity != null) {
        long len = entity.getContentLength();
        if (len != -1 && len < 2048) {
            System.out.println(EntityUtils.toString(entity));
        } else {
            // Stream content out
        }
    }
} finally {
    response.close();
}
```



如果需要多次读取server端的response，可以使用`BufferedHttpEntity`

```java
CloseableHttpResponse response = <...>
HttpEntity entity = response.getEntity();
if (entity != null) {
    entity = new BufferedHttpEntity(entity);
}
```

> 如何在client端生产 HttpEntity

`HttpClient` 提供了多种 `HttpEntity` 来应对不同的使用场景，包括: `StringEntity/ByteArrayEntity/UrlEncodedFormEntity/FileEntity/InputStreamEntity` 等.

`FileEntiy`的demo代码: 

```java
File file = new File("somefile.txt");
FileEntity entity = new FileEntity(file, 
    ContentType.create("text/plain", "UTF-8"));        

HttpPost httppost = new HttpPost("http://localhost/action.do");
httppost.setEntity(entity);
```

`FormEntity` 的demo代码:

```java
List<NameValuePair> formparams = new ArrayList<NameValuePair>();
formparams.add(new BasicNameValuePair("param1", "value1"));
formparams.add(new BasicNameValuePair("param2", "value2"));
UrlEncodedFormEntity entity = new UrlEncodedFormEntity(formparams, Consts.UTF_8);
HttpPost httppost = new HttpPost("http://localhost/handler.do");
httppost.setEntity(entity);
```

> 工具类ResponseHandler的使用

由于`ResponseHandler` 会自动释放连接，无论处理reponse数据时是否发生异常，都会正确将连接返回给 connection manager ，官方推荐使用此类解析response。

```
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpGet httpget = new HttpGet("http://localhost/json");

ResponseHandler<MyJsonObject> rh = new ResponseHandler<MyJsonObject>() {

    @Override
    public JsonObject handleResponse(
            final HttpResponse response) throws IOException {
        StatusLine statusLine = response.getStatusLine();
        HttpEntity entity = response.getEntity();
        if (statusLine.getStatusCode() >= 300) {
            throw new HttpResponseException(
                    statusLine.getStatusCode(),
                    statusLine.getReasonPhrase());
        }
        if (entity == null) {
            throw new ClientProtocolException("Response contains no content");
        }
        Gson gson = new GsonBuilder().create();
        ContentType contentType = ContentType.getOrDefault(entity);
        Charset charset = contentType.getCharset();
        Reader reader = new InputStreamReader(entity.getContent(), charset);
        return gson.fromJson(reader, MyJsonObject.class);
    }
};
MyJsonObject myjson = client.execute(httpget, rh);
```

> HttpClient 底层执行细节基于注入的具体插件

`httpClient` 等价为一个facade设计模式，底层的执行细节可以根据需要自行定制，比如keep alive 属性设置:

```java
ConnectionKeepAliveStrategy keepAliveStrat = new DefaultConnectionKeepAliveStrategy() {

    @Override
    public long getKeepAliveDuration(
            HttpResponse response,
            HttpContext context) {
        long keepAlive = super.getKeepAliveDuration(response, context);
        if (keepAlive == -1) {
            // Keep connections alive 5 seconds if a keep-alive value
            // has not be explicitly set by the server
            keepAlive = 5000;
        }
        return keepAlive;
    }

};
CloseableHttpClient httpclient = HttpClients.custom()
        .setKeepAliveStrategy(keepAliveStrat)
        .build();
```

`httpClient`是线程安全的，可以被多线程同时使用。当不再使用时，一定记得释放占用的系统资源`CloseableHttpClient#close()`.

> http协议是无状态的，但是多个http请求可能是逻辑相关的，因此http client提供了`HttpContext` 来代表一系列http请求的上下文。

`HttpContext` 包含以下属性：

- HttpConnection 与远程服务器的连接实例
- HttpHost 远程主机实例
- HttpRoute 完整的连接路由
- HttpRequest 实际的http请求实例
- HttpResponse 实际的server返回实例
- RequestConfig 请求配置参数
- java.util.List<URI> 重定向的完整路径
- java.lang.Boolean 请求是否已完全发送到server端

```java
HttpContext context = <...>
HttpClientContext clientContext = HttpClientContext.adapt(context);
HttpHost target = clientContext.getTargetHost();
HttpRequest request = clientContext.getRequest();
HttpResponse response = clientContext.getResponse();
RequestConfig config = clientContext.getRequestConfig();
```

下面是一个复用http context的代码例子:

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
RequestConfig requestConfig = RequestConfig.custom()
        .setSocketTimeout(1000)
        .setConnectTimeout(1000)
        .build();

HttpGet httpget1 = new HttpGet("http://localhost/1");
httpget1.setConfig(requestConfig);
CloseableHttpResponse response1 = httpclient.execute(httpget1, context);
try {
    HttpEntity entity1 = response1.getEntity();
} finally {
    response1.close();
}
HttpGet httpget2 = new HttpGet("http://localhost/2");
//此处将会复用第一个请求的 request config 
CloseableHttpResponse response2 = httpclient.execute(httpget2, context);
try {
    HttpEntity entity2 = response2.getEntity();
} finally {
    response2.close();
}
```

> http interceptors 用来在发送或接受response时做统一的处理逻辑，典型应用是用来压缩content，或统一添加header。interceptor可以基于http context在多个连续请求中传播信息。

http interceptor 使用代码示例:

```java
CloseableHttpClient httpclient = HttpClients.custom()
        .addInterceptorLast(new HttpRequestInterceptor() {

            public void process(
                    final HttpRequest request,
                    final HttpContext context) throws HttpException, IOException {
                AtomicInteger count = (AtomicInteger) context.getAttribute("count");
                request.addHeader("Count", Integer.toString(count.getAndIncrement()));
            }

        })
        .build();

AtomicInteger count = new AtomicInteger(1);
HttpClientContext localContext = HttpClientContext.create();
localContext.setAttribute("count", count);

HttpGet httpget = new HttpGet("http://localhost/");
for (int i = 0; i < 10; i++) {
    CloseableHttpResponse response = httpclient.execute(httpget, localContext);
    try {
        HttpEntity entity = response.getEntity();
    } finally {
        response.close();
    }
}
```

http client 一般有三种异常

- IOException   一般是 socket io 超时，对于幂等方法会重试
- HttpException  数据流，不符合http规范，fatal error，不会重试。
- Transmit Exception 向server发送数据时发生异常。会重试。

一般使用 ``StandardHttpRequestRetryHandler` ` 即可满足需求，默认重试三次，下面是自定义handler代码：

```java
HttpRequestRetryHandler myRetryHandler = new HttpRequestRetryHandler() {

    public boolean retryRequest(
            IOException exception,
            int executionCount,
            HttpContext context) {
        if (executionCount >= 5) {
            // Do not retry if over max retry count
            return false;
        }
        if (exception instanceof InterruptedIOException) {
            // Timeout
            return false;
        }
        if (exception instanceof UnknownHostException) {
            // Unknown host
            return false;
        }
        if (exception instanceof ConnectTimeoutException) {
            // Connection refused
            return false;
        }
        if (exception instanceof SSLException) {
            // SSL handshake exception
            return false;
        }
        HttpClientContext clientContext = HttpClientContext.adapt(context);
        HttpRequest request = clientContext.getRequest();
        boolean idempotent = !(request instanceof HttpEntityEnclosingRequest);
        if (idempotent) {
            // Retry if the request is considered idempotent
            return true;
        }
        return false;
    }

};
CloseableHttpClient httpclient = HttpClients.custom()
        .setRetryHandler(myRetryHandler)
        .build();
```

`httpRequest` 可以通过调用 abort 方法放弃请求，http client 保证：即使 执行线程正阻塞在IO上，也会响应abort而抛出 Interrupted异常。

http client会自动处理所有重定向。也可以自己定制重定向策略:

```java
LaxRedirectStrategy redirectStrategy = new LaxRedirectStrategy();
CloseableHttpClient httpclient = HttpClients.custom()
        .setRedirectStrategy(redirectStrategy)
        .build();
```

下面是获取最终url的一个例子：

```java
CloseableHttpClient httpclient = HttpClients.createDefault();
HttpClientContext context = HttpClientContext.create();
HttpGet httpget = new HttpGet("http://localhost:8080/");
CloseableHttpResponse response = httpclient.execute(httpget, context);
try {
    HttpHost target = context.getTargetHost();
    List<URI> redirectLocations = context.getRedirectLocations();
    URI location = URIUtils.resolve(httpget.getURI(), target, redirectLocations);
    System.out.println("Final HTTP location: " + location.toASCIIString());
    // Expected to be an absolute URI
} finally {
    response.close();
}
```



### 连接管理器

*未完待续。*






