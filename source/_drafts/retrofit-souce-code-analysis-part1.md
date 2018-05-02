---
title: Retrofit 源码解析：Part 1
date: 2018-04-26 00:31:46
tags:
- Android
- Retrofit
- 源码解析
---

## 从一个小例子讲起

我从 `Retrofit` 源码的 `samples` 模块中拿出 `SimpleService` 这个小例子稍作修改，以更基本的形式展示一下 `Retrofit` 请求的典型使用方式，然后顺着这个小例子一步一步对其源码做一下探究。

先看一下整体的代码，因为是从 `SimpleService` 修改而来，所以我给其取名为 `SimplerService` 。

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
<!--more-->
上面的整体代码可以拆分为下面几步：

### 1. 定义访问接口的 interface

```java
public interface GitHub {
  @GET("/repos/{owner}/{repo}/contributors")
  Call<ResponseBody> contributors(
      @Path("owner") String owner,
      @Path("repo") String repo);
}
```

通过该接口实例对象的方法调用将 HTTP 请求发送出去。

### <a name="step2"/>2. 创建 Retrofit 对象

```java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl(API_URL)
    .build();
```

这个 `Retrofit` 对象很简单，只配置了 base url 。

### <a name="step3"/>3. 创建 GitHub 接口的实例

```java
GitHub github = retrofit.create(GitHub.class);
```

通过调用 `Retrofit#create()` 方法实现。

### <a name="step4"/>4. 调用 GitHub 接口方法获取请求(`retrofit2.Call`)对象

```java
Call<ResponseBody> call = github.contributors("square", "retrofit");
```

### <a name="step5"/>5. 发送请求并获取响应对象

```java
Response<ResponseBody> response = call.execute();
```

调用 `retrofit2.Call#execute()` 方法得到响应(`retrofit2.Response`)对象。

### 6. 从响应对象中获取响应体(`okhttp3.ResponseBody`)对象

```java
ResponseBody body = response.body();
```

调用 `retrofit2.Response#body()` 方法。

### 7. 通过`okhttp3.ResponseBody` 对象的相关方法获取具体数据

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

前三个是流的形式，后两个会把数据全部加载进内存。

## 请求到底是怎么发送出去的

在上面的例子中，[第 5 步](#step5)的时候发送请求并得到响应对象。

```java
Response<ResponseBody> response = call.execute();
```

可以看出整个的逻辑应该发生在 `call.execute()` 方法中，而 `call` 是 `retrofit2.Call` 接口的实例，要想知道 `execute()` 方法中发生了什么，必须知道 `call` 对象对应具体的类是什么。而 `call` 对象又是在[第 4 步](#step4)中通过 `github` 对象的方法调用得到的，`github` 对象是 `GitHub` 接口的实例，它是在[第 3 步](#step3)通过 `retrofit` 对象创建的。

我们按照下面的顺序一步一步看一下请求是怎么发送出去的。

### 1. `GitHub` 实例是什么

`Retrofit#create()` 方法源码如下：

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

可以看到 `create()` 方法返回了一个**动态代理**对象，在这里不对动态代理做深入探讨，我们只要知道被代理对象的所有方法调用最终会走到代理对象 `InvocationHandler` 的 `invoke` 方法中就够了。

所以，在这一步我们得到结论：`GitHub` 示例是一个动态代理对象。

### 2. `retrofit2.Call` 实例是什么？

我们已经知道 `call` 对象是 `github` 对象方法调用的返回值，而 `github` 是一个动态代理对象，它的方法调用

会走到 `InvocationHandler#invoke()` 方法中，所以 **`call` 其实是 `invoke` 方法返回的**。

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

可以看到这里有两个对象创建一次方法调用，两个对象分别是 `ServiceMethod` 和 `OkHttpCall` ，一次方法调用是 `serviceMethod.adapt(okHttpCall)` 并且该方法的返回值就是 `invoke()` 方法的返回值。所以，**`call` 其实是 `serviceMethod.adapt(okHttpCall)` 返回的**。

先看一下 `OkHttpCall` 类的定义：

```java
final class OkHttpCall<T> implements Call<T> {}
```

它是 `retrofit2.Call` 接口的实现类，那 `call` 对象是 `OkHttpCall` 的实例吗？在这里我们还不知道，因为在 `InvocationHandler#invoke()` 中不是将 `OkHttpCall` 对象直接返回的。

继续看 `ServiceMethod` 这个类，这个类是干嘛的我们还不知道，但从注释中可以看出它的大致作用，如下：

```
/** Adapts an invocation of an interface method into an HTTP call. */
```
翻译过来就是**“将接口方法的执行转换为一个 HTTP 请求（对象）”**，小例子中的 `GitHub` 就是这里所说的*接口*。

<a name="servicemethod_adapt" />这个“转换”过程就发生在 `ServiceMethod#adapt()` 方法中，源码如下：

```java
T adapt(Call<R> call) {
  return callAdapter.adapt(call);
}
```
看起来很简单，只有一行代码。虽然还不知道 `callAdapter` 是什么，但我们又可以得出一个结论：**小例子中的 `call` 对象是 `callAdapter.adapt(call)` 返回的**。

`callAdapter` 是 `retrofit2.CallAdapter` 的实例对象，它的注释如下：

```java
/**
 * Adapts a {@link Call} with response type {@code R} into the type of {@code T}. Instances are
 * created by {@linkplain Factory a factory} which is
 * {@linkplain Retrofit.Builder#addCallAdapterFactory(Factory) installed} into the {@link Retrofit}
 * instance.
 */
```

这个类的作用是将 `retrofit2.Call<R>` 转换成 `T` ,它的实例是由它的工厂类创建的，而其工厂类对象是通过 `Retrofit.Builder#addCallAdapterFactory(Factory)` “安装”到 `Retrofit` 对象中的。

但在我们的小例子中[创建 `Retrofit` 对象](#step2)的时候并没有调用 `addCallAdapterFactory(Factory)` “安装” `CallAdapter.Factory` 对象，那这里的 `callAdapter` 是哪里来的？分析一下。

`callAdapter` 是 `ServcieMethod` 对象的成员变量，通过搜索可以看到 `callAdapter` 是在 `ServiceMethod` 的构造方法中被赋值的，它的值是 `builder.callAdapter` 。

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

由此可以看出 `ServiceMethod.Builder` 对象的 `callAdapter` 是由其 `createCallAdapter()` 方法创建的，其源码如下：

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
到这里可以看出 `callAdapter` 最终来自 `retrofit.callAdapter(returnType, annotations)` ，其源码如下：

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
上面代码的逻辑简单说就是对一个 `CallAdapter.Factory` 列表对象 `callAdapterFactories` 进行遍历并调用每个工厂对象的 `get()` 方法得到 `CallAdapter` 对象，将第一个不为 `null` 的 `CallAdapter` 对象返回。返回的这个 `CallAdapter` 对象就是 [`ServiceMethod.adapt(Call)`](#servicemethod_adapt) 方法中用到的 `callAdapter` 对象。那接下来就要分析 `callAdapterFactories` 是如何来的以及通过其中的那个工厂对象得到的 `CallAdapter` 对象不为 `null` 。

通过在 `Retrofit` 源码中简单的搜索，可以知道 `callAdapterFactories` 是 `retrofit` 对象的成员变量，并且是在 `Retrofit` 的构造方法中被赋值的。

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

在 `Retrofit.Builder#build()` 方法中，基于 `this.callAdapterFactories ` 新建了一个 `ArrayList` ，**并且**将通过 `platform.defaultCallAdapterFactory(callbackExecutor)` 得到的一个 `CallAdapter.Factory` 对象也加入其中，然后将这个新建的 `ArrayList` 传给了 `Retrofit` 的构造方法。

`this.callAdapterFactories` 是 `Retrofit.Builder` 对象的成员变量，并且调用 `Retrofit.Builder#addCallAdapterFactory(CallAdapter.Factory)` 方法正是将作为参数的 `CallAdapter.Factory` 对象放入了 `this.callAdapterFactories` 中。

```java
public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
  callAdapterFactories.add(checkNotNull(factory, "factory == null"));
  return this;
}
```

已经知道，小例子中 `Retrofit` 对象的创建并没有调用 `Retrofit.Builder#addCallAdapterFactory(CallAdapter.Factory)` 方法，因此可以合理地推断，`ServiceMethod#adapt(Call)` 方法中的 `adapter` 对象应该就是通过 `platform.defaultCallAdapterFactory(callbackExecutor)` 得到的工厂类创建的 `CallAdapter` 对象，现在要验证的就是对于我们的小例子来说 `platform.defaultCallAdapterFactory(callbackExecutor)` 得到的工厂类创建的 `CallAdapter` 对象是否为 `null` 。

接下来，我们必须搞清楚 `platform` 对象是什么，再去看它的 `defaultCallAdapterFactory()` 方法。

`platform` 是 `retrofit2.Platform` 类的实例，赋值来自于 `Retrofit.Builder` 的构造方法。

```java
Builder(Platform platform) {
  this.platform = platform;
}

public Builder() {
  this(Platform.get());
}
```
在 `Platform` 的源码中可以看到：

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

由上可以知道对于 Android 开发来说 `platform` 是 `Android` 类的实例，`Android` 是 `Platform` 的子类。

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

看它的 `defaultCallAdapterFactory` 方法，返回了一个 `ExecutorCallAdapterFactory` 对象。

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

看 `get()` 方法的实现，首先是一个判断：

```java
if (getRawType(returnType) != Call.class) {
  return null;
}
```

如果 `returnType` 的 rawType 不是 `Call.class` 的话返回 `null` 。什么是 rawType ？不做过多的解释，举例说明，对于 `List<String>` 来说它的  rawType 就是 `List` 。那 `returnType` 是什么呢，它是我们定义的用于请求网络的接口方法的返回值类型，对于我们的小例子来说，就是 `GitHub#contributors()` 方法的返回值类型，是`Call<ResponseBody>` ，符合条件。（如果不理解的话，简单回溯一下 returnType 的传递流程）

继续看 `ExecutorCallAdapterFactory.get()` 方法的实现，后面就直接返回一个 `CallAdapter`  的匿名内部类了，所以 [`ServiceMethod.adapt()`](#servicemethod_adapt) 方法的返回值就是这个匿名内部类的 `adapt()` 方法的返回值，它是一个 `ExecutorCallbackCall` 类的实例。

最终，我们得到了这一部分想要的答案：**`github.contributors("square", "retrofit")` 返回的 `retrofit2.Call`  实例是一个 `ExecutorCallbackCall` 对象**。

### 3. `call.execute()` 方法中发生了什么

我们已经知道了 `call` 是 `ExecutorCallbackCall` 的实例，直接去看它的 `execute()` 方法。

```java
@Override public Response<T> execute() throws IOException {
  return delegate.execute();
}
```

直接调用了 `delegate.execute()` 方法，这个 `delegate` 是它的构造方法传进来的一个 `retrofit2.Call` 对象。

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

那接下来就要看 `OkHttpCall` 的 `execute()` 方法了。其源码如下：

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

上面的第2步属于 `OkHttp` 的范围了，这里不再说。简单看一下 `createRawCall` 方法和 `parseResponse(okhtt3.Response)` 的实现。

```java
private okhttp3.Call createRawCall() throws IOException {
  okhttp3.Call call = serviceMethod.toCall(args);
  if (call == null) {
    throw new NullPointerException("Call.Factory returned null.");
  }
  return call;
}
```

`okhttp3.Call` 对象其实是通过 `serviceMethod.toCall(args)` 方法得到的。

```
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
  ResponseBody rawBody = rawResponse.body();

  ...

  ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
  try {
    T body = serviceMethod.toResponse(catchingBody);
    return Response.success(body, rawResponse);
  } catch (RuntimeException e) {
    // If the underlying source threw an exception, propagate that rather than indicating it was
    // a runtime exception.
    catchingBody.throwIfCaught();
    throw e;
  }
}
```

这里我们看到通过 `serviceMethod.toResponse(catchingBody)` 得到的返回值传入 `retrofit2.Response.success()` 方法，得到一个 `retrofit2.Response` 对象并返回。

在这篇文章中我们不对 `serviceMethod.toCall(args)` 和 `serviceMethod.toResponse(catchingBody)` 做深入解析，放到下次进行。

至此，本次的解析到此结束。



