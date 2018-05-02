---
title: 对 ExecutorCallAdapterFactory 中 returnType 的进一步说明
tags:
  - Android
  - Retrofit
  - 源码解析
date: 2018-05-02 21:08:47
---


在[Retrofit 源码解析第一部分：请求流程解析](https://blog.niniliwei.com/2018/05/02/retrofit-souce-code-analysis-part1/)中讲到 [`ExecutorCallAdapterFactory`](https://blog.niniliwei.com/2018/05/02/retrofit-souce-code-analysis-part1/#executorcalladapterfactory) 时，它的 `get()` 方法有一个类型为 `Type` 名为 `returnType` 的参数，通过判断 `returnType` 的 rawType 是否为 `Call.class` 来决定返回的 `CallAdapter` 对象是否为 `null` 。

这篇文章通过分析 `returnType` 的值传递过程，使大家更清楚的了解 `returnType` 是什么。

<!--more-->

本次分析依然是基于 `SimplerService` ，我通过一问一答的形式完成。

### 1. `ExecutorCallAdapterFactory#get(Type, Annotation[], Retrofit)` 在哪里被调用？

在 `Retrofit#callAdapter(Type, Annotation[])` 方法，也就是 `Retrofit#nextCallAdapter(CallAdapter.Factory, Type, Annotation[])` 中（因为 `callAdapter()` 直接调用了 `nextCallAdapter()`）。

在 `nextCallAdapter()` 中对 `callAdapterFactories` 进行遍历并调用了每个元素（`CallAdapter.Factory` 对象）的 `get()` 方法，`ExecutorAdapterFactory` 正是 `callAdapterFactories` 的一个元素。

### 2. `Retrofit#callAdapter(Type, Annotation[])` 在哪里被调用？

在 `ServiceMethod.Builder#createCallAdapter()` 方法中。

`retrofit.callAdapter(returnType, annotations)` 作为 `ServiceMethod.Builder#createCallAdapter()` 方法的返回值返回，而 `returnType` 通过 `method.getGenericReturnType()` 获得。

### 3. `method` 对象是什么？

是 `ServiceMethod.Builder` 对象的成员变量，是 `Method` 类的实例，在 `ServiceMethod.Builder` 的构造方法中被赋值。

### 4. `ServiceMethod.Builder` 的构造方法在哪里被调用？ 

在 `Retrofit#loadServiceMethod(Method)` 方法中，`method` 是该方法的参数。

### 5. `Retrofit#loadServiceMethod(Method)` 方法在哪里被调用？

在 `Retrofit#create(Class)` 方法的返回值，也就是动态代理对象的 `InvocationHandler#invoke(Object, Method, Object[])` 方法中。`method` 是 `InvocationHandler#invoke()` 方法的参数，表示被代理对象正在被执行的方法。

### 6. `returnType` 是什么？

我们已经知道 `returnType` 是 `method.getGenericReturnType()` ，而 `method` 表示被代理对象正在被执行的方法，在 `Retrofit#create(Class)` 方法中被代理对象是 `GitHub` 接口，所以在执行 `github.contributors()` 方法时，`returnType` 就是 `Call<ResponseBody>` 。

如果我们将 `GitHub#contributors()` 方法的返回值定义为 `io.reactivex.Observable<ResponseBody>` ：

```java
public interface GitHub {
  @GET("/repos/{owner}/{repo}/contributors")
  Observable<ResponseBody> contributors(
      @Path("owner") String owner,
      @Path("repo") String repo);
}
```

那 `returnType` 就是 `Observable<ResponseBody>` 。