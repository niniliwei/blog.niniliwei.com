---
title: Retrofit 源码解析第一部分：请求流程解析
tags:
  - Android
  - Retrofit
  - 源码解析
date: 2018-05-02 17:30:00
---


## 研究对象：一个小例子

我将 `Retrofit` 源码 `samples` 模块中的 `SimpleService` 这个小例子稍作修改，取名为 `SimplerService` ，以更基本的形式展示一下 `Retrofit` 的典型使用方式，然后以 `SimplerService` 为研究对象对 `Retrofit` 源码做一下探究。

<!--more-->

### 整体代码简单介绍

`SimplerService` 整体代码如下 。

```java
public final class SimplerService {
  public static final String API_URL = "https://api.github.com";

  public interface GitHub {
    @GET("/repos/{owner}/{repo}/contributors")
    Call<ResponseBody> contributors(
            @Path("owner") String owner,
            @Path("repo") String repo);
  }

  public static void main(String... args) throws IOException {
    // 创建一个很简单的 Retrofit 对象
    Retrofit retrofit = new Retrofit.Builder()
        .baseUrl(API_URL)
        .build();

    // 创建 GitHub 接口的一个实例
    GitHub github = retrofit.create(GitHub.class);

    // 创建一个查询 Retrofit 项目贡献者信息的请求对象
    Call<ResponseBody> call = github.contributors("square", "retrofit");

    // 发送请求得到响应对象
    Response<ResponseBody> response = call.execute();

    // 从响应对象中获取 ResponseBody 对象
    ResponseBody body = response.body();

    // 通过 ResponseBody 对象的相关方法获取具体的数据
    String string = body.string();
    System.out.println(string);
  }
}
```
上面的代码使用 `Retrofit` 请求了 [https://api.github.com/repos/square/retrofit/contributors](https://api.github.com/repos/square/retrofit/contributors) 这个接口，并将返回结果进行打印。

### 代码逻辑拆分

`SimplerService` 的整体代码可以拆分为下面几步：

#### 1. 定义用于发送 HTTP 请求的接口 `GitHub`

```java
public interface GitHub {
  @GET("/repos/{owner}/{repo}/contributors")
  Call<ResponseBody> contributors(
      @Path("owner") String owner,
      @Path("repo") String repo);
}
```

通过该接口实例对象的方法调用将 HTTP 请求发送出去。

#### <a name="step2"/>2. 创建 Retrofit 对象

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl(API_URL)
    .build();
```

这个 `Retrofit` 对象很简单，只配置了 base url 。

#### <a name="step3"/>3. 创建 GitHub 接口的实例

```java
GitHub github = retrofit.create(GitHub.class);
```

通过调用 `Retrofit#create(Class)` 方法实现。

#### <a name="step4"/>4. 调用 GitHub 接口方法获取请求(`retrofit2.Call`)对象

```java
Call<ResponseBody> call = github.contributors("square", "retrofit");
```

#### <a name="step5"/>5. 发送请求并获取响应对象

```java
Response<ResponseBody> response = call.execute();
```

调用 `retrofit2.Call#execute()` 方法得到响应(`retrofit2.Response`)对象。

#### 6. 从响应对象中获取响应体(`okhttp3.ResponseBody`)对象

```java
ResponseBody body = response.body();
```

调用 `retrofit2.Response#body()` 方法。

#### 7. 通过 `okhttp3.ResponseBody` 对象的相关方法获取具体数据

```java
// 获取字节流
InputStream inputStream = body.byteStream();
// 获取字符流
Reader reader = body.charStream();
// 获取 Okio 的 BufferedSource 对象
BufferedSource source = body.source();
// 获取字节数组
byte[] bytes = body.bytes();
// 获取字符串
String string = body.string();
```

前三个是流的形式，后两个会把数据全部加载进内存。**注意，每次请求只能调用上面的一个方法。**

## 请求发送流程分析

在 `SimplerService` 中，[第 5 步](#step5)的时候发送请求并得到响应对象。

```java
Response<ResponseBody> response = call.execute();
```

可以看出请求发送的逻辑应该发生在 `call.execute()` 方法中，而 `call` 是 `retrofit2.Call` 接口的实例，要想知道 `execute()` 方法中发生了什么，必须知道 `call` 对象对应具体的类是什么。而 `call` 对象又是在[第 4 步](#step4)中通过 `github` 对象的方法调用得到的，`github` 对象是 `GitHub` 接口的实例，它是在[第 3 步](#step3)通过 `retrofit` 对象创建的。

按照上面的分析，一步一步看一下请求是怎么发送出去的。

### <a name="github_instance">1. `GitHub` 实例是什么

`Retrofit#create(Class)` 方法源码如下：

```java
  public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.adapt(okHttpCall);
          }
        });
  }
```

可以看到 `Retrofit.create(Class)` 方法返回了一个**动态代理**对象，在这里不对动态代理做深入探讨，我们只要知道**被代理对象（即 `GitHub` 接口）**的所有方法调用最终会走到代理对象 `InvocationHandler` 的 `invoke` 方法中就够了。

所以，在这一步我们得到结论：**`GitHub` 实例其实是一个动态代理对象**。

### 2. `retrofit2.Call` 实例是什么？

我们已经知道 [`call` 对象是 `github.contributors("square", "retrofit")` 的返回值](#step4)，而 `github` 是一个动态代理对象，它的方法调用会走到 `InvocationHandler#invoke()` 方法中，所以 **`call` 其实是 `invoke` 方法返回的**。

来看 `InvocationHandler#invoke()` 的源码：

```java
new InvocationHandler() {
    private final Platform platform = Platform.get();

    @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
        throws Throwable {
        // 省略不重要的代码
        ...
        
        ServiceMethod<Object, Object> serviceMethod =
            (ServiceMethod<Object, Object>) loadServiceMethod(method);
        OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
        return serviceMethod.adapt(okHttpCall);
    }
}
```

可以看到 `InvocationHandler#invoke()` 方法的返回值是 `serviceMethod.adapt(okHttpCall)` ，所以，**`call` 其实是 `serviceMethod.adapt(okHttpCall)` 返回的**。

这里涉及到两个新的类，一个是 `ServiceMethod` ，一个是 `OkHttpCall`。

#### `OkHttpCall` 是什么？

先看一下 `OkHttpCall` 类的定义：

```java
final class OkHttpCall<T> implements Call<T> {
  ...
}
```

结论很简单，**它是 `retrofit2.Call` 接口的实现类。**

那 [`call` 对象](#step4)是 `OkHttpCall` 的实例吗？在这里我们还不知道，因为在 `InvocationHandler#invoke()` 中不是将 `OkHttpCall` 对象直接返回的。

#### `ServiceMethod` 是什么？

再看看 `ServiceMethod` 这个类，其注释如下：

```
/** Adapts an invocation of an interface method into an HTTP call. */
```
意思是**“将接口方法的执行转换为一个 HTTP 请求（对象）”**，`SimplerService` 中的 `GitHub` 就是这里所说的**接口**。

<a name="retrofit_loadservicemethod" />这里的 `serviceMethod` 对象是通过 `Retrofit#loadServiceMethod(Method)` 获取的，其源码如下：

```java
ServiceMethod<?, ?> loadServiceMethod(Method method) {
  ServiceMethod<?, ?> result = serviceMethodCache.get(method);
  if (result != null) return result;

  synchronized (serviceMethodCache) {
    result = serviceMethodCache.get(method);
    if (result == null) {
      result = new ServiceMethod.Builder<>(this, method).build();
      serviceMethodCache.put(method, result);
    }
  }
  return result;
}
```

可以看出 `serviceMethod` 是通过 `new ServiceMethod.Builder<>(this, method).build()` 创建的。

#### “转换”过程解析

这个**“转换”**过程就发生在 `ServiceMethod#adapt(OkHttpCall)` 方法中。

##### <a name="servicemethod_adapt" />`serviceMethod.adapt(call)`

```java
T adapt(Call<R> call) {
  return callAdapter.adapt(call);
}
```

看起来很简单，只有一行代码，调用了 `callAdapter.adapt(call)` 方法。所以，“转换”过程其实发生在 `CallAdapter#adapt(Call)` 中。

`callAdapter` 是 `retrofit2.CallAdapter` 的实例对象，它的注释如下：

```java
/**
 * Adapts a {@link Call} with response type {@code R} into the type of {@code T}. Instances are
 * created by {@linkplain Factory a factory} which is
 * {@linkplain Retrofit.Builder#addCallAdapterFactory(Factory) installed} into the {@link Retrofit}
 * instance.
 */
```

这个类的作用是将 `retrofit2.Call<R>` 转换成 `T` ，它的实例是由它的工厂类（`CallAdapter.Factory`）创建的（<a name="calladapter_creation">*通过 `CallAdapter.Factory#get(Type, Annotaion[], Retrofit)` 方法*</a>），而其工厂类对象是通过 `Retrofit.Builder#addCallAdapterFactory(Factory)` “安装”到 `Retrofit` 对象中的。

但在 `SimplerService` 中[创建 `Retrofit` 对象](#step2)的时候并没有调用 `Retrofit.Builder#addCallAdapterFactory(Factory)` 方法，那这里的 `callAdapter` 是什么？

##### <a name="servicemethod_calladapter" />`serviceMethod` 的 `callAdapter` 是什么？

`callAdapter` 是 `ServcieMethod` 对象的成员变量，它是在 `ServiceMethod` 的构造方法中被赋值的 。

```java
ServiceMethod(Builder<R, T> builder) {
  ...
  this.callAdapter = builder.callAdapter;
  ...
}
```

而且 `ServiceMethod` 的构造方法是在 `ServiceMethod.Builder#build()` 中调用的。

```java
public ServiceMethod build() {
  callAdapter = createCallAdapter();
  ...
  return new ServiceMethod<>(this);
}
```

由此可以看出 `ServiceMethod` 对象的 `callAdapter` 变量是由 `ServiceMethod.Builder` 对象的 `callAdapter` 赋值的，而 `ServiceMethod.Builder` 对象的 `callAdapter` 是由其 `createCallAdapter()` 方法创建的，该方法源码如下：

```java
private CallAdapter<T, R> createCallAdapter() {
  Type returnType = method.getGenericReturnType();
  ...
  Annotation[] annotations = method.getAnnotations();
  try {
    //noinspection unchecked
    return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
  } catch (RuntimeException e) { // Wide exception range because factories are user code.
    throw methodError(e, "Unable to create call adapter for %s", returnType);
  }
}
```
可以看到 `ServiceMethod#Builder` 的 `callAdapter` 通过 `retrofit.callAdapter(returnType, annotations)` 获得， `retrofit` 是 `Retrofit` 实例对象，是[通过 `ServiceMethod#Builder` 的构造参数传入](#retrofit_loadservicemethod)的。

所以，**`serviceMethod` 的 `callAdapter` 是通过 `retrofit.callAdapter(returnType, annotations) 方法得到的。`**

##### `retrofit.callAdapter(returnType, annotations)` 解析

其源码如下：

```java
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
  return nextCallAdapter(null, returnType, annotations);
}
```

```java
public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
    Annotation[] annotations) {
  checkNotNull(returnType, "returnType == null");
  checkNotNull(annotations, "annotations == null");

  int start = callAdapterFactories.indexOf(skipPast) + 1;
  for (int i = start, count = callAdapterFactories.size(); i < count; i++) {
    CallAdapter<?, ?> adapter = callAdapterFactories.get(i).get(returnType, annotations, this);
    if (adapter != null) {
      return adapter;
    }
  }

  ...
}
```
`callAdapterFactories` 是 `Retrofit` 对象的成员变量，它是一个 `CallAdapter.Factory` 列表，上面的代码对这个列表进行遍历并[通过每个 `CallAdapter.Factory` 对象创建对应的 `CallAdapter` 对象](#calladapter_creation)，将第一个不为 `null` 的 `CallAdapter` 对象返回。

##### `retrofit` 的 `callAdapterFactories` 包含什么？

`callAdapterFactories` 是在 `Retrofit` 的构造方法中被赋值的。

```java
Retrofit(okhttp3.Call.Factory callFactory, HttpUrl baseUrl,
    List<Converter.Factory> converterFactories, List<CallAdapter.Factory> callAdapterFactories,
    @Nullable Executor callbackExecutor, boolean validateEagerly) {
  ...
  this.callAdapterFactories = callAdapterFactories; // Copy+unmodifiable at call site.
  ...
}
```

而 `Retrofit` 的构造方法是在 `Retrofit.Builder#build()` 方法中被调用的。

```java
public Retrofit build() {
  ...

  // Make a defensive copy of the adapters and add the default Call adapter.
  List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
  callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

  ...

  return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
      unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
}
```

在上面代码中，基于 **`Retrofit.Builder` 对象的 `callAdapterFactories`** 新建了一个 `ArrayList` 对象，然后将**通过 `platform.defaultCallAdapterFactory(callbackExecutor)` 得到的 `CallAdapter.Factory` 对象**也加入其中，这个新建的 `ArrayList` 对象被传给了 `Retrofit` 构造方法。

###### `Retrofit.Builder` 对象的 `callAdapterFactories` 包含什么？

`Retrofit.Builder` 对象的 `callAdapterFactories` 变量创建时是一个空列表。

```java
private final List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>();
```

在调用 `Retrofit.Builder#addCallAdapterFactory(CallAdapter.Factory)` 方法时将作为参数传入的 `CallAdapter.Factory` 对象放入了 `callAdapterFactories` 中。

```java
public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
  callAdapterFactories.add(checkNotNull(factory, "factory == null"));
  return this;
}
```

已经知道，`SimplerService` 中 [`Retrofit` 对象的创建](#step2)并没有调用 `Retrofit.Builder#addCallAdapterFactory(CallAdapter.Factory)` 方法，因此对于 `SimplerService` 来说 `Retrofit.Builder` 对象的 `callAdapterFactories` 是空的。

###### `platform.defaultCallAdapterFactory(callbackExecutor)` 可以得到什么

`platform` 是 `retrofit2.Platform` 类的实例，在 `Retrofit.Builder` 的构造方法被赋值。

```java
Builder(Platform platform) {
  this.platform = platform;
}

public Builder() {
  this(Platform.get());
}
```
`SimplerService` 中 [`Retrofit` 对象的创建](#step2)调用的是 `Retrofit.Builder` 的无参构造方法，所以，`platform` 是通过 `Platform.get()` 得到的。

```java
private static final Platform PLATFORM = findPlatform();

static Platform get() {
  return PLATFORM;
}

private static Platform findPlatform() {
  try {
    Class.forName("android.os.Build");
    if (Build.VERSION.SDK_INT != 0) {
      return new Android();
    }
  } catch (ClassNotFoundException ignored) {
  }
  try {
    Class.forName("java.util.Optional");
    return new Java8();
  } catch (ClassNotFoundException ignored) {
  }
  return new Platform();
}
```

对于 Android 开发来说 `platform` 是 `Android` 类的实例。

```java
static class Android extends Platform {
  @Override public Executor defaultCallbackExecutor() {
    return new MainThreadExecutor();
  }

  @Override CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
    if (callbackExecutor == null) throw new AssertionError();
    return new ExecutorCallAdapterFactory(callbackExecutor);
  }

  static class MainThreadExecutor implements Executor {
    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override public void execute(Runnable r) {
      handler.post(r);
    }
  }
}
```

看它的 `defaultCallAdapterFactory` 方法可以得知，**`platform.defaultCallAdapterFactory(callbackExecutor)` 得到的是 `ExecutorCallAdapterFactory` 实例对象**。

###### 结论

通过以上分析，可以知道，**`retrofit` 的 `callAdapterFactories` 只包含一个 `ExecutorCallAdapterFactory` 对象**。

##### 通过 `ExecutorCallAdapterFactory` 创建的 `CallAdapter` 对象是否为 `null` ？

<a name="executorcalladapterfactory" />`ExecutorCallAdapterFactory` 源码如下：

```java
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
  final Executor callbackExecutor;

  ExecutorCallAdapterFactory(Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }

  ...
}
```

看其 `get()` 方法的实现，返回值是否为 `null` 在于下面的判断条件：

```java
if (getRawType(returnType) != Call.class) {
  return null;
}
```

如果 `returnType` 的 rawType 不是 `Call.class` 的话就返回 `null` ，否则的话就返回一个 `CallAdapter` 的匿名内部类。

什么是 rawType ？举例说明，对于 `List<String>` 来说它的  rawType 是 `List` 。什么是 `returnType` ？`returnType` 是用于请求网络的接口方法的返回值类型。对于 `SimplerService` 来说，`GitHub#contributors()` 方法的返回值类型是 `Call<ResponseBody>` ，它的 rawType 是 `Call.class` ，符合条件。

所以，**对 `SimplerService` 来说， `ExecutorCallAdapterFactory` 创建的 `CallAdapter` 对象不为 `null`，并且是一个 `CallAdapter` 的匿名内部类**。

##### `serviceMethod` 的 `callAdapter` 最终是什么？

由以上的分析可以得出结论，**`serviceAdapter` 的 `callAdapter` 最终是通过 `ExecutorCallAdapterFactory` 工厂类得到的一个 `CallAdapter` 的匿名内部类**。

##### 所以，`retrofit2.Call` 实例是什么？

我们已经知道 `serviceMethod` 的 `callAdapter` 是通过 `ExecutorCallAdapterFactory` 创建的一个 `CallAdapter` 匿名内部类，那[“转换”过程](#servicemethod_adapt)其实是发生在这个匿名内部类的 `adapt()` 方法中，看其[源码](#executorcalladapterfactory)可以知道，**`retrofit2.Call` 实例是一个 `ExecutorCallbackCall` 对象**。

### 3. `call.execute()` 方法中发生了什么

我们已经知道了 `call` 是 `ExecutorCallbackCall` 的实例，直接去看它的 `execute()` 方法。

```java
@Override public Response<T> execute() throws IOException {
  return delegate.execute();
}
```

调用了 `delegate.execute()` 方法，这个 `delegate` 是它的构造方法传进来的一个 `retrofit2.Call` 对象。

```java
ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
  this.callbackExecutor = callbackExecutor;
  this.delegate = delegate;
}
```

那这个 `delegate` 到底是什么？先说结论，**`delegate` 就是在 `Retrofit#create(Class)` 方法返回的动态代理对象的 `InvocationHandler` 的 `invoke()` 方法中创建的 `OkHttpCall` 对象**。分析如下：

* `OkHttpCall` 对象创建之后传给了 `serviceMethod.adapt(okHttpCall)` 方法；
*  `serviceMethod.adapt(okHttpCall)` 方法中又将其传给了 `callAdapter.adapt(call)` 方法；
* `callAdapter.adapt(call)` 方法中将其作为  `ExecutorCallbackCall` 构造函数的第二个进行了传递；
* `ExecutorCallbackCall` 构造函数的第二个参数正是赋值给了 `ExecutorCallbackCall` 对象的 `delegate` 变量。

接下来看 `OkHttpCall` 的 `execute()` 方法，其源码如下：

```java
@Override public Response<T> execute() throws IOException {
  okhttp3.Call call;

  synchronized (this) {
    ...

    call = rawCall;
    if (call == null) {
      try {
        call = rawCall = createRawCall();
      } catch (IOException | RuntimeException | Error e) {
        throwIfFatal(e); //  Do not assign a fatal error to creationFailure.
        creationFailure = e;
        throw e;
      }
    }
  }

  ...

  return parseResponse(call.execute());
}
```

这里的逻辑可以分为三步：

1. 通过 `createRawCall()` 方法创建一个 `okhttp3.Call` 对象；
2. 调用 `okhttp3.Call#execute()` 得到一个 `okhttp3.Response` 对象；
3. 将第2步得到的 `okhttp3.Response` 对象传入 `parseResponse(okhttp3.Response)` 方法中得到一个 `retrofit2.Response` 对象并返回。

所以说，**`Retrofit` 其实最终是通过 `OkHttp` 进行网络请求的**。

