---
title: Retrofit源码浅析
date: 2016-08-17 00:49:43
tags: Android
---

 这篇主要会走读一下`Retrofit`的源码，解析一下里面遇到的一些设计模式，网络请求的过程等。
 
### 开始
从创建`Retrofit`开始，看一下常见的创建`Retrofit`的实例的方式
<!--more-->

```
HttpLoggingInterceptor interceptor = new HttpLoggingInterceptor();
        interceptor.setLevel(HttpLoggingInterceptor.Level.BODY);
        OkHttpClient client = new OkHttpClient.Builder()
                .addInterceptor(interceptor)
                .retryOnConnectionFailure(true)
                .connectTimeout(15, TimeUnit.SECONDS)
                .addNetworkInterceptor(createHeaderInterceptor())
                .build();
                
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(BuildConfig.BASE_API_URL)
                .client(client)
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .addConverterFactory(BolomeGsonConverterFactory.create())
                .build();
          Net net = retrofit.create(Net.class)
```
从上面的代码可以看出，`Retrofit`并没有去真正的执行网络请求，还是交给了`OkHttp`来进行实现，`Retrofit`可以看作的对`OkHttp`的一层非常棒的封装，`Retrofit`的关注点是在如何让你更快捷更灵活的去进行网络请求。如果你使用过`Retrofit`的话，你也会明白用`Retrofit`去实现一次网络请求是多方便的事情。

### `retrofit.create`
从`Net net = retrofit.create(Net.class)`这里开始入手，用过`Retrofit`的几乎都知道`Retrofit`是用了动态代理，动态代理说简单点就是动态的去生成接口的实现类，也可以在原始的结果返回前对参数或者结果进行修改，这个特性也使得一些`Hook`框架大量使用了动态代理，比如很著名的360的`DroidPlugin`。有点扯远了，还是去源码里面去看看：

```
  public <T> T create(final Class<T> service) {
    ...
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            ...
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```
这里省略了一下判断的逻辑，但是主要的就是最后的三行代码，从方法的签名来看，是传进来一个接口类，返回一个接口类型的实例，但是这个实例的具体实现类型是由`serviceMethod.callAdapter.adapt`方法动态决定了，而且这个接口里面的每一次方法调用，都会进入这个`invoke`方法，也就是由接口的实现类去完成功能。例如，`Retrofit`默认的请求方式是`Call<T>`,这里默认的实现类就是`ExecutorCallbackCall<T> implements Call<T>`，而在这个类里面又是用代理类的方式去交给`Okhttp`的`call`去执行真正的请求，这个就是后面再说了。

### `ServiceMethod`
这个类主要是储存了请求的信息和`Retrofit`的配置信息以及对请求注解的解析生成正常的地址和请求。

### `OkHttpCall`
第二行`OkHttpCall`可以看成是`OkHttp`的`call`在`Retrofit`层面的一次封装。从上面一步看到，把请求参数和配置信息交给了这个`OkHttpCall`，然后让`OkHttp`去执行请求操作。

### `callAdapter`
不知道该怎么去具体描述这个类的意思，可以理解成将请求转换成不同的请求形式，例如默认的`call`或者常见的`Rxjava`的` Observable<T>`的形式亦或者是`Agera`的`Supplier<Result<T>> `的形式等等。以`@drakeet`的[retrofit-agera-call-adapter][1]为例：

```
public final class AgeraCallAdapterFactory extends CallAdapter.Factory {

    public static AgeraCallAdapterFactory create() {
        return new AgeraCallAdapterFactory();
    }


    private AgeraCallAdapterFactory() {
    }


    @Override
    public CallAdapter<?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
        ...
        支持Supplier<Result<T>>形式
            return new BodyCallAdapter(innerTypeOfInnerType);
        }
        ...支持Supplier<Result<Response<T>>>形式
        return new ResponseCallAdapter(responseType);
    }
    private static class BodyCallAdapter implements CallAdapter<Supplier<?>> {

        private final Type responseType;


        BodyCallAdapter(Type responseType) {
            this.responseType = responseType;
        }


        @Override public Type responseType() {
            return responseType;
        }


        @Override
        public <T> Supplier<Result<T>> adapt(Call<T> call) {
            return new CallSupplier(call);
        }
    }
}

class CallSupplier<T> implements Supplier<Result<T>> {

    private final Call<T> originalCall;


    CallSupplier(@NonNull final Call<T> call) {
        this.originalCall = checkNotNull(call);
    }


    @NonNull @Override public Result<T> get() {
        Result<T> result;
        try {
            Response<T> response = originalCall.clone().execute();
            //将Response转换成Result<T>的形式 因为数据要从Result中取，而不像默认的call那样直接走回调
            if (response.isSuccessful()) {
                result = Result.success(response.body());
            } else {
                result = Result.failure(new HttpException(response));
            }
        } catch (IOException e) {
            e.printStackTrace();
            result = Result.failure(e);
        }
        return result;
    }
}

```
代码没太多难度，看看就明白了。也是非常棒的自定义`CallAdapter`教程。

### `enqueue`
异步请求的时候是这个样子的

```
 mNet.getLiveShows(20, 1).enqueue(new Callback<LiveBlock>() {
            @Override
            public void onResponse(Call<LiveBlock> call, Response<LiveBlock> response) {
                
            }

            @Override
            public void onFailure(Call<LiveBlock> call, Throwable t) {

            }
        });
```
而`getLiveShows`返回的`call`其实是上面说的`ExecutorCallbackCall`，这个的`enqueue`是这样的

```
@Override public void enqueue(final Callback<T> callback) {
      if (callback == null) throw new NullPointerException("callback == null");

      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override public void onFailure(Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }

```
调用了`delegate.enqueue`这个`delegate`就是上面说到的`OkHttpCall`看这个里面的

```
call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response<T> response;
        try {
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        callSuccess(response);
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callSuccess(Response<T> response) {
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
      
Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
   .....//使用responseConverter.convert(body)
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    ....
  }
      
```
注意的是这里几个`call`的区别和三个`enqueue`的区别和作用

###`Converter`
这个是用来将`RequestBody`和`responseBody`转换成相应的类型，具体去看看官方的`GsonConverterFactory`就可以了


 


  [1]: https://github.com/drakeet/retrofit-agera-call-adapter