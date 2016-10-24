

[TOC]



## Android 端网络框架缓存详解

这里说的网络框架缓存，指的是，网络框架基于 Http 缓存建立的一条缓存机制，从表象上看，程序员不需要手动去处理任何缓存，但是实际上程序员基于Http 协议约定了缓存的实现机制，这里将向大家详细的介绍这一套机制，当然在实现细节上，不同的网络框架，可能有细微的差异。

在这里也简单讨论一下，http 缓存实现的必要性。在客户端的架构上，一般将图片框架和普通网络请求框架进行了分离，图片框架，肯定会做缓存处理的，至于网络请求框架，需不需要做网络缓存处理？这点因业务而异同，我们主要从收益和代价去考虑。

##### 从收益上考虑







### Http 缓存







### AsyncHttpClient 解析



**使用类介绍（4.0-4.3）**



* SchemeRegistry 类 4.0-4.3 使用，通过CurrentHashMap保存了注册的Scheme（网络协议），默认注册 http和https。
* HttpParams 接口，定义了每一个client Component 需要遵循的网络行为（可以理解为管理http请求体头部），这里设置了各种类型的set和get方法，实际上把params 的name作为键值，把parmas 的 value 内容存储起来，这里可以设置各种 params 例如设置超时时间。
* BasicHttpParams 类，这个类是 HttpParams 的默认实现，这里使用 ConcurrentHashMap 保存各种 params 并且实现了 setParameter()方法，通过这个方法为 client Component 设置不同的行为。
* ConnManagerPNames 接口，管理了 每个Connection的超时时间，连接总数，每条 router 的连接数目。
* ConnManagerParams 类，ConnManagerPNames 接口的实现类。
* CoreConnectionPNames 接口设置了一些 Connection 的参数
* HttpConnectionParams 类，CoreConnectionPNames  的实现，实现了一些设置，例如 Socket Buff Size 等设置。
* CoreProtocolPNames 接口，设置了协议的一些参数。
* HttpProtocolParams 类，CoreProtocolPNames 的实现类，提供一些参数的 set和get方法，例如User-Agent。
* ClientConnectionManager 接口， Connection 的管理类
* ​





一些细节：

AsyncHttpClient 会把 cookies 保存在 SharePerference 做持久化保存。



```java
String content = buildRequestContent(map); //构建 json 字符串
String token = getKugouToken(content);// MD5 加密方法如下


    @NonNull
    private static String getKugouToken(String content) {
        long timeSeconds = System.currentTimeMillis() / 1000; //系统时间
        // java 内部的 MD5  加密
        String md5 = MD5Utils.getMd5(content + API_KUGOU_SECRET + timeSeconds);
        String hexTimeSeconds = Long.toHexString(timeSeconds);
        return md5 + hexTimeSeconds;
    }
```



简单看了一下繁星这边使用的 asynchttpclient 是1.4.6版本的，目前 asynchttpclient 最新的是 1.4.9，查看了一下网络层的封装，这边封装的也不严重，所以如果考虑替换网络框架的话，从技术成本考虑，代价不是很大。例如以下是一段网络请求代码：

```java
HttpUtil.post(context, FxConstant.getKugouBiReportUrl(), se, CONTENT_TYPE, new AsyncHttpResponseHandler() {
    @Override
    public void onSuccess(int statusCode, Header[] headers, byte[] responseBody) {
        if (null != responseBody && responseBody.length > 0) {
            FxLog.d(TAG, "onSuccess eventId=%s,responseBody=%s", map.get("action_id"), new String(responseBody));
        }
    }

    @Override
    public void onFailure(int statusCode, Header[] headers, byte[] responseBody, Throwable error) {
        if (null != responseBody && responseBody.length > 0) {
            FxLog.d(TAG, "onFailure:eventId=%s,error=%s,responseBody=%s", map.get("action_id"), error, new String(responseBody));
        }
    }
});
```







