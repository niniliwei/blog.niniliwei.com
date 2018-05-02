---
title: Retrofit 源码解析第二部分：请求对象创建解析
tags:
- Android
- Retrofit
- 源码解析
---

在  [`Retrofit` 源码解析第一部分](https://blog.niniliwei.com/2018/05/02/retrofit-souce-code-analysis-part1/)的最后，请求发送的逻辑分为了三步，在这篇文章中，我们分析第一步，即用于发送请求的 `okhttp3.Call` 对象是如何创建的。

`OkHttpCall#createRawCall()` 方法的源码如下：

```java
private okhttp3.Call createRawCall() throws IOException {
  okhttp3.Call call = serviceMethod.toCall(args);
  if (call == null) {
    throw new NullPointerException("Call.Factory returned null.");
  }
  return call;
}
```

可以看出 `okhttp3.Call` 对象其实是通过 `serviceMethod.toCall(args)` 方法创建的。

<!--more-->

`ServiceMethod.toCall(Object...)` 源码如下：

```java
/** Builds an HTTP request from method arguments. */
okhttp3.Call toCall(@Nullable Object... args) throws IOException {
  RequestBuilder requestBuilder = new RequestBuilder(httpMethod, baseUrl, relativeUrl, headers,
      contentType, hasBody, isFormEncoded, isMultipart);

  @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
  ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

  int argumentCount = args != null ? args.length : 0;
  if (argumentCount != handlers.length) {
    throw new IllegalArgumentException("Argument count (" + argumentCount
        + ") doesn't match expected count (" + handlers.length + ")");
  }

  for (int p = 0; p < argumentCount; p++) {
    handlers[p].apply(requestBuilder, args[p]);
  }

  return callFactory.newCall(requestBuilder.build());
}
```

从注释可以看出，该方法的作用是根据接口方法的参数构造一个 HTTP 请求。