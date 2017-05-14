---
title: RxJava和Retrofit的那些事
date: 2016-08-01 22:24:32
tags: Android
---

##关于Retrofit
这是一个最近很火的网络请求库，官网在这[官网](http://square.github.io/retrofit/),当然这也是一个开源的项目[GitHub地址](https://github.com/square/retrofit)。可能你对这个不是很熟悉，但是说起他的兄弟，okhttp估计就比较熟悉了，还有大名鼎鼎的依赖注入dagger和图片加载库picasso还有检查内存泄露的LeakCanary都是出自Square公司。当然这只是在JAVA和Android端，在js和Python上也有很棒的项目，有兴趣就自己看了，扯远了。。<!--more-->

为什么要用Retrofit呢，因为他除了提供给我们常见的请求Callback形式，还封装了Observable的请求形式，这就很方便的支持了Rxjava,而Rxjava的强大就是体现在异步任务上。下面来简单的介绍一下使用。

对于RxJava的语法就不说了，有一篇写了关于这方面的，下面就直接开始了。

以github的开放API作为数据来源，我们要去获取某个项目的contributors，以square的retrofit为例，可以调用这个`https://api.github.com/repos/square/retrofit/contributors`返回的json是这样的array

```
{
login: "JakeWharton",
id: 66577,
avatar_url: "https://avatars.githubusercontent.com/u/66577?v=3",
gravatar_id: "",
url: "https://api.github.com/users/JakeWharton",
html_url: "https://github.com/JakeWharton",
followers_url: "https://api.github.com/users/JakeWharton/followers",
following_url: "https://api.github.com/users/JakeWharton/following{/other_user}",
gists_url: "https://api.github.com/users/JakeWharton/gists{/gist_id}",
starred_url: "https://api.github.com/users/JakeWharton/starred{/owner}{/repo}",
subscriptions_url: "https://api.github.com/users/JakeWharton/subscriptions",
organizations_url: "https://api.github.com/users/JakeWharton/orgs",
repos_url: "https://api.github.com/users/JakeWharton/repos",
events_url: "https://api.github.com/users/JakeWharton/events{/privacy}",
received_events_url: "https://api.github.com/users/JakeWharton/received_events",
type: "User",
site_admin: false,
contributions: 619
},
```
这里我们只使用到`login`和`contributions`

 1.定义一个接口，这个看代码就明白了

```
public interface GithubApi {
 @GET("/repos/{owner}/{repo}/contributors")
    Observable<List<Contributor>> contributors(@Path("owner") String owner,
                                               @Path("repo") String repo);
     }
```
通过注解来提供参数，这里注解有很多，还有`@Query`，`@QueryMap `，`@Body`等等，也可以通过`@Headers`设置请求头。参数基于base url,其实就是拼接成的。这个返回的是一个`Observable`，这样就很方便的去发布一个事件了。

 2.设置`RestAdapter`和`service`,先用`Endpoint(API)`并调用buid()方法来创建一个RestAdapter对象。

```
RestAdapter adapter = new RestAdapter.Builder().setEndpoint(
              "https://api.github.com/").setLogLevel(RestAdapter.LogLevel.FULL).build();
```
然后使用我们的adapter来创建一个服务适配器(service for adapter)。`adapter.create(GithubApi.class);`
完整的代码

```
  private GithubApi _createGithubApi() {
        RestAdapter adapter = new RestAdapter.Builder().setEndpoint(
              "https://api.github.com/").setLogLevel(RestAdapter.LogLevel.FULL).build();

        return adapter.create(GithubApi.class);
    }
```

 3.这就到了Rxjava的地方了，上一步获取到了GithubApi对象，就可以去调用函数了：

```
_subscriptions.add(
                _api.contributors(_username.getText().toString(), _repo.getText().toString())//事件源
                        .subscribeOn(Schedulers.io())
                        .observeOn(AndroidSchedulers.mainThread())
                        .subscribe(new Observer<List<Contributor>>() {//订阅者
                            @Override
                            public void onCompleted() {
                                
                            }

                            @Override
                            public void onError(Throwable e) {
                               
                            }

                            @Override
                            public void onNext(List<Contributor> contributors) {
                                for (Contributor c : contributors) {
                                    _adapter.add(format("%s has made %d contributions to %s",
                                            c.login,
                                            c.contributions,
                                            _repo.getText().toString()));
                                }
                            }
                        }));
```
当然这是一个比较简单的处理过程，并没有用到一下通配符，但是基本的流程就是这样的。

到现在发现我们并没有去解析json数据就自动生成了bean类，这是因为Retrofit已经集成了Gson，并且默认使用Gson来进行解析，当然你也可以自己去定义。在创建`RestAdapter`的时候使用`setConverter(new GsonConverter(gson))`。默认是使用okhttp进行加载的，当然你也可以去配置一些关于`OkHttpClient`的数据，然后通过`setClient(new OkClient(client))`进行设置进去。

现在`Retrofit`的2.0版本已经出了，主要就是 取消了同步和异步，可以取消正在进行中的业务，取消了Gson的依赖等等，具体可以看这里[Retrofit 2.0：有史以来最大的改进](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0915/3460.html)
这篇参考了GitHub上的项目[RxJava-Android-Samples](https://github.com/70kg/RxJava-Android-Samples)，这里只是拣取了关于Retrofit的部分，有兴趣可以看看其他的。

参考：
http://www.tuicool.com/articles/26jUZjv
http://www.cnblogs.com/angeldevil/p/3757335.html
