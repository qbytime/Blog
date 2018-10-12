---
layout:     post
title:      "从Retrofit看Java动态代理"
subtitle:   ""
date:       2018-01-16
author:     "Qiby"
header-img: "img/post-bg-unix-linux.jpg"
catalog: true
tags:
    - Android
    - Proxy
---


# 从Retrofit看Java动态代理



[](https://mp.weixin.qq.com/s?__biz=MzAxOTc0NzExNg==&mid=2665513926&idx=1&sn=1c43c5557ba18fed34f3d68bfed6b8bd&chksm=80d67b85b7a1f2930ede2803d6b08925474090f4127eefbb267e647dff11793d380e09f222a8#rd)

在写Retrofit之前，先回顾一下动态代理模式。
刚开始看动态代理的时候不是很懂，他是怎么就代理了一个类？有什么好处？InvocationHandler到底在干什么？

Java代理模式是提供了对目标对象另外的访问方式；即通过代理对象访问目标对象.这样做的好处是：可以在目标对象实现的基础上,增强额外的功能操作,即扩展目标对象的功能。
如果是静态代理，你就得创建出一个代理类，在代理类中添加一个目标类的一个实例，并重写出该目标类所有的方法，并进行方法扩展。如下
```
public class TargetProxy {

    private static final String TAG = "TargetProxy";

    private Target mTarget;

    public TargetProxy() {
        mTarget = new Target();
    }

    public void add() {
        //面向切面编程，扩展功能
        *******
        mTarget.add();
        *******
    }
}

```
其中`TargetProxy`这个代理类中的`add`方法代理了`Target`类的`add`方法，扩展增强了`Target`类的功能。但是有一个疑问如果`Target`类中方法很多，并且业务变大切相同，代理每个方法都要重写一次扩展功能的代码么？ 显然Java不会这么干的。

如此就出现了动态代理，动态代理就是要在

动态代理究竟要怎么做？先回想一个静态代理的要素
1. 代理类中覆盖目标类的调用契约
2. 代理类中目标类的实例
3. 对于目标方法的扩展

这3个要素对于动态代理同样适用。
首先代理类要和目标类有相同的方法，需要接口来定义方法，保证代理和目标类方法一致
```
public interface ISayHello {
    void sayHello();
    void goodBye();
}
//目标类
public class JavaSayHello implements ISayHello {
    private static final String TAG = "JavaSayHello";
    @Override
    public void goodBye() {
        Log.d(TAG, "goodBye: java");
    }
    @Override
    public void sayHello() {
        Log.d(TAG, "sayHello: java");
    }
}
```

有了目标类，需要借助`InvocationHandler`这个类来实现目标类方法扩展
```
public class LoggerHandler implements InvocationHandler {

    private static final String TAG = "LoggerHandler";

    private Object target;

    public LoggerHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        Log.d(TAG, "invoke: pre");
        Object o = method.invoke(target, args);
        Log.d(TAG, "invoke: post");
        return o;
    }
}

```
此方法实为扩展方法的模板，方法`method.invoke`调用目标类的方法

最后生成代理类
```
    ISayHello hello = new JavaSayHello();

    LoggerHandler handler = new LoggerHandler(hello);

    ISayHello proxy = (ISayHello) Proxy.newProxyInstance(
            Thread.currentThread().getContextClassLoader(),
            hello.getClass().getInterfaces(),
            handler);

	proxy.sayHello();
```
`proxy.sayHello();`即为代理调用。可以看见`LoggerHandler`这个扩展方法模板传入了目标类的实例作为参数，`Proxy.newProxyInstance`的方法传入了目标类接口的调用契约和扩展方法模板，最后生成了代理类。

动态代理看完了，那么现在来看看Retrofit到底是怎么实现的。

```
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(baseUrl)
                .client(mOkHttpClient)
                .addCallAdapterFactory(Java8CallAdapterFactory.create())
                .addConverterFactory(GsonConverterFactory.create())
                .build();
        Api proxy = retrofit.create(Api.class);
```
`retrofit.create()`通过门面Retrofit创建出一个Proxy，每个请求在调用代理方法的时候都要被拦截。

```
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
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```
其中扩展的内容为
获取请求方法，解析请求方法的注解，生成`ServiceMethod`,创建`HttpCall`，最后`call`被装入`CallAdapter`这个适配器。

```
//Java8CallAdapterFactory的内部类
private static final class BodyCallAdapter<R> implements CallAdapter<R, CompletableFuture<R>> {
    private final Type responseType;

    BodyCallAdapter(Type responseType) {
      this.responseType = responseType;
    }

    @Override public Type responseType() {
      return responseType;
    }

    @Override public CompletableFuture<R> adapt(final Call<R> call) {
      final CompletableFuture<R> future = new CompletableFuture<R>() {
        @Override public boolean cancel(boolean mayInterruptIfRunning) {
          if (mayInterruptIfRunning) {
            call.cancel();
          }
          return super.cancel(mayInterruptIfRunning);
        }
      };

      call.enqueue(new Callback<R>() {
        @Override public void onResponse(Call<R> call, Response<R> response) {
          if (response.isSuccessful()) {
            future.complete(response.body());
          } else {
            future.completeExceptionally(new HttpException(response));
          }
        }

        @Override public void onFailure(Call<R> call, Throwable t) {
          future.completeExceptionally(t);
        }
      });

      return future;
    }
```
请求最终还是交给`OkHttp`进行。

