# RxJava

> RxJava是ReactiveX的一种Java实现。ReactiveX是Reactive Extensions的缩写，一般简写为Rx。开发者可以用Observables表示异步数据流，用LINQ操作符查询异步数据流，用Schedulers参数化异步数据流的并发处理，Rx可以这样定义：Rx = Observables + LINQ + Schedulers。

* 我为什么要选择RxJava ？

```
一般我们在开发的时候，只要说到异步操作，就会想到Android的AsyncTask和Handler。但是随着请求的数量越来越多，
代码逻辑将会变得越来越复杂，而RxJava却仍旧能保持清晰的逻辑。RxJava的原理就是创建一个Observable对象来干活，
然后使用各种操作符建立起来的链式操作，就如同流水线一样，把你想要处理的数据一步一步加工成你想要的成品，
然后发射给Subscriber处理。
```

* RxJava与观察者模式

```
RxJava的异步操作是通过扩展的观察者模式来实现的。RxJava有4个角色：Observable、Observer、Subscriber和Suject。
Observable和Observer通过subscribe方法实现订阅关系，Observable就可以在需要的时候通知Observer。
```

* RxJava的实现

在使用RxJava前请先在AndroidStudio配置gradle:(此处以RxJava为例,不使用RxJava2)

```
dependencies {
    ...

    compile 'io.reactivex:rxjava:1.2.0'
    compile 'io.reactivex:rxandroid:1.2.1'
}
```

看到这里可能会问：我使用的是RxJava，为什么还要引入RxAndroid ？

RxAndroid是RxJava在Android平台的扩展。包含了一些能够简化Android开发的工具，比如特殊的调度器。


* RxJava的使用

RxJava的使用可以分为3个步骤：创建Observable(被观察者)、创建Observer(观察者)、Subscribe(订阅)。

**创建Observable(被观察者)**

Observable它决定什么时候触发事件以及触发怎样的事件。RxJava使用 *create* 方法来创建一个Observable，并未它定义时间触发规则。例如：


```
        // 创建Observable(被观察者)
        Observable<String> observable = Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("张三");
                subscriber.onNext("李四");
                subscriber.onCompleted();
            }
        });
```

通过调用Subscriber的方法，不断的将事件添加到任务队列中。

**创建Observer(观察者)**

Observer 它决定事件触发的时候将有怎么样的行为。


```
        // 创建Observer(观察者)
        Subscriber<String> subscriber = new Subscriber<String>() {
            @Override
            public void onStart() {
                Log.e(TAG, "onStart");
            }

            @Override
            public void onCompleted() {
                Log.e(TAG, "onCompleted");
            }

            @Override
            public void onError(Throwable e) {
                Log.e(TAG, "onError");
            }

            @Override
            public void onNext(String s) {
                Log.e(TAG, "onNext" + s);
            }
        };
```

其中，onCompleted、onError和onNext是必须要实现的方法。其具体含义如下：
* onCompleted:事件队列完结。RxJava不仅把每个事件单独处理，其还会把它们看作一个队列。当不会有新的 onNext 发出时，需要触发 onComplete() 方法作为完成标志。

* onError:事件队列异常。在事件处理过程中出现异常时，onError() 方法会被触发，同时队列自动终止，不允许再有事件发出。

* onNext():普通的事件。将要处理的事件添加到事件队列中。

* onStart():它会在事件还未发送之前被调用，可以用于做一些准备工作。例如数据的清零或重置。这是一个可选方法，默认情况下它的实现为空。

当然，如果要实现简单的功能，也可以用到Observer来创建观察者。Observer是一个接口，而上面用到的Subscriber是在Observer的基础上进行的扩展。在后文的Subscribe订阅过程中Observer也会被转换为Subscriber来使用。如果用Observer创建观察者对象，如下：

```
Observer<String> observer = new Observer<String>() {
            @Override
            public void onCompleted() {
                Log.e(TAG, "onCompleted");
            }

            @Override
            public void onError(Throwable e) {
                Log.e(TAG, "onError");
            }

            @Override
            public void onNext(String s) {
                Log.e(TAG, "onNext" + s);
            }
        };
```

**Subscribe(订阅)**


```
observable.subscribe(subscriber);
```

运行代码可查看Log:

```
E/RxJavaActivity: onStart
E/RxJavaActivity: onNext张三
E/RxJavaActivity: onNext李四
E/RxJavaActivity: onCompleted
```


## RxJava操作符

> RxJava操作符类型可以分为:创建操作符、变换操作符、过滤操作符、组合操作符、错误处理操作符、辅助操作符、条件和布尔操作符、算术和聚合操作符及连续操作符等，而这些操作符类型下又有很多操作符，每个操作符可能还有很多变体。此处，简单的介绍一些创建操作符、变换操作符、过滤操作符、辅助操作符、错误处理操作符。



**创建操作符**

* **Interval**

创建一个按固定时间间隔发射整数序列的Observable，相当于定时器，如下：

```
Observable.interval(3, TimeUnit.SECONDS)
                .subscribe(new Action1<Long>() {
                    @Override
                    public void call(Long aLong) {
                        Log.e(TAG, "interval:" + aLong.intValue());
                    }
                });
```

上面的代码每个3s就会调用call方法并打印Log。

* **Range**

创建发射指定范围的整数序列的Observable，可以拿来代替for循环，发射一个范围内的有序整数序列。第一个参数是起始值，并不小于0;第二个参数为整数序列的个数。


```
Observable.range(0, 5)
                .subscribe(new Action1<Integer>() {
                    @Override
                    public void call(Integer integer) {
                        Log.e(TAG, "range:" + integer.intValue());
                    }
                });
```

运行代码可查看Log:


```
E/RxJavaActivity: range:0
E/RxJavaActivity: range:1
E/RxJavaActivity: range:2
E/RxJavaActivity: range:3
E/RxJavaActivity: range:4
```

* **Repeat**

创建一个N次重复发射特定数据的Observable，如下：

```
Observable.range(0, 3)
                .repeat(2)
                .subscribe(new Action1<Integer>() {
                    @Override
                    public void call(Integer integer) {
                        Log.e(TAG, "repeat:" + integer.intValue());
                    }
                });
```

运行代码可查看Log：


```
E/RxJavaActivity: repeat:0
E/RxJavaActivity: repeat:1
E/RxJavaActivity: repeat:2
E/RxJavaActivity: repeat:0
E/RxJavaActivity: repeat:1
E/RxJavaActivity: repeat:2
```

**变换操作符**

* **Map**

map操作符通过指定一个Func对象，将Observable转换为一个新的Observable对象并发射，观察者将收到最新的Observable处理。例如：


```
final String host = "https://github.com/";
        Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {

                subscriber.onNext("JiaYang627");
            }
        }).map(new Func1<String, String>() {
            @Override
            public String call(String s) {
                return host + s;
            }
        }).subscribe(new Action1<String>() {
            @Override
            public void call(String s) {
                Log.e(TAG, "map:" + s);
            }
        });
```

运行代码查看Log：

```
E/RxJavaActivity: map:https://github.com/JiaYang627
```

**过滤操作符**

* **Filter**

Filter操作符是对源Observable产生的结果自定义规则进行过滤，只有满足条件的结果才会交给订阅者。例如：


```
 Observable.create(new Observable.OnSubscribe<Integer>() {
            @Override
            public void call(Subscriber<? super Integer> subscriber) {
                subscriber.onNext(1);
                subscriber.onNext(2);
                subscriber.onNext(3);
                subscriber.onNext(4);

            }
        }).filter(new Func1<Integer, Boolean>() {
            @Override
            public Boolean call(Integer integer) {
                return integer > 2;
            }
        }).subscribe(new Action1<Integer>() {
            @Override
            public void call(Integer integer) {
                Log.e(TAG, "filter:" + integer);
            }
        });
```

运行代码查看Log：

```
E/RxJavaActivity: filter:3
E/RxJavaActivity: filter:4
```

**辅助操作符**

* **Do**

Do系列操作符是为原始Observable的生命周期事件注册一个回调，当Observable的某个事件发生时就会调用这些回调。RxJava中有很多Do系列操作符，如下：

* doOnEach：为Observable注册这样一个回调：当Observable每当发射一项数据时就会调用它一次，包括onNext、onError和onCompleted。
* doOnNext：只有执行onNext的时候会被调用。
* doOnSubscribe：当观察者订阅Observable时就会被调用。
* doOnUnsubscribe：当观察者取消订阅Observable时就会被调用;Observable通过onError或者onCompleted结束时，会取消订阅所有的Subscriber。
* doOnCompleted：当Observable正常终止调用onCompleted时会被调用。
* doOnError：当Observable异常终止调用onError时会被调用。
* doOnTerminate：当Observable终止(无论是正常终止还是异常终止)之前会被调用。
* finallyDo：当Observable终止(无论是正常终止还是异常终止)之后会被调用。

此处以doOnNext为例，如下：

```
 Observable.create(new Observable.OnSubscribe<Integer>() {
            @Override
            public void call(Subscriber<? super Integer> subscriber) {
                subscriber.onNext(1);
                subscriber.onNext(2);
            }
        }).doOnNext(new Action1<Integer>() {
            @Override
            public void call(Integer integer) {
                Log.e(TAG, "doOnNext:" + integer);
            }
        }).subscribe(new Subscriber<Integer>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(Integer integer) {
                Log.e(TAG, "subscribe_onNext:" + integer);
            }
        });
```

运行代码查看Log:

```
E/RxJavaActivity: doOnNext:1
E/RxJavaActivity: subscribe_onNext:1
E/RxJavaActivity: doOnNext:2
E/RxJavaActivity: subscribe_onNext:2
```

* **SubscribeOn、ObserverOn**

SubscribeOn操作符用于指定Obserable自身在哪个线程上运行。如果Observable需要执行耗时操作，一般可以让其在新开的一个子线程上运行。ObserverOn用来指定Observer所运行的线程，也就是发射出的数据在哪个线程上使用。一般情况下会指定砸死主线程中运行，这样就可以修改UI。如下：

```
 Observable.create(new Observable.OnSubscribe<Integer>() {
            @Override
            public void call(Subscriber<? super Integer> subscriber) {
                Log.e(TAG, "Observable:" + Thread.currentThread().getName());
                subscriber.onNext(1);
            }
        }).subscribeOn(Schedulers.io())
          .observeOn(AndroidSchedulers.mainThread())
          .subscribe(new Action1<Integer>() {
              @Override
              public void call(Integer integer) {
                  Log.e(TAG, "Observer:" + Thread.currentThread().getName());
              }
          });
```

运行代码查看Log：

```
E/RxJavaActivity: Observable:RxIoScheduler-2
E/RxJavaActivity: Observer:main
```
其中，AndroidSchedulers是RxAndroid库中提供的Scheduler。

**错误处理操作符**

* **Retry**

Retry操作符不会将原始Observable的onError通知传递给观察者，它会订阅这个Observable再给它一次机会无错误地完成其数据序列。Retry总是传递onNext通知给观察者，由于重新订阅，这可能会造成数据项重复。RxJava中的实现为Retry和RetryWhen。此处以Retry为例说明。

它指定最多重新订阅的次数，如果次数超了，它不会尝试再次订阅，而会把最新的一个onError通知传递给自己的观察者。如下：

```
Observable.create(new Observable.OnSubscribe<Integer>() {
            @Override
            public void call(Subscriber<? super Integer> subscriber) {
                for (int i =0 ; i < 5 ; i ++) {
                    if (i == 1) {
                        subscriber.onError(new Throwable("Throwable"));
                    } else {
                        subscriber.onNext(i);
                    }
                }
            }
        }).retry(2)
          .subscribe(new Subscriber<Integer>() {
              @Override
              public void onCompleted() {
                  Log.e(TAG, "onCompleted");
              }

              @Override
              public void onError(Throwable e) {
                  Log.e(TAG, "onError:" + e.getMessage());
              }

              @Override
              public void onNext(Integer integer) {
                  Log.e(TAG, "onNext:" + integer);
              }
          });
```

如上所示，Observable中走了一个for循环，当i为1时走onNext，当i为1时让走onError，然后，此时会重新订阅，再次走for循环，此为第一次重新订阅。在第二次重新订阅的时候，走到onError后不会再次重新订阅，运行代码查看Log：

```
E/RxJavaActivity: onNext:0
E/RxJavaActivity: onNext:0
E/RxJavaActivity: onNext:0
E/RxJavaActivity: onError:Throwable
```

## RxJava结合Retrofit访问网络

此处使用自己写的MVP框架Demo为例，使用的是RxJava2、Retrofit2。

* 首先配置build.gradle:


```
dependencies {
    ...
    compile 'io.reactivex.rxjava2:rxandroid:2.0.1'
    compile 'io.reactivex.rxjava2:rxjava:2.1.0'
    
    compile 'com.squareup.retrofit2:retrofit:2.3.0'
    compile 'com.squareup.retrofit2:adapter-rxjava2:2.3.0'
    compile 'org.ligboy.retrofit2:converter-fastjson-android:2.1.0'
    compile 'com.squareup.retrofit2:converter-scalars:2.1.0'
    
    compile 'com.alibaba:fastjson:1.2.33'

}
```
以上为配置依赖，Demo中使用FastJson数据解析。

**配置Retorfit：**

```
private Retrofit creatRetrofit(Retrofit.Builder builder, String baseUrl, OkHttpClient client){
        builder.client(client)
                .baseUrl(baseUrl)
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .addConverterFactory(ScalarsConverterFactory.create())
                .addConverterFactory(FastJsonConverterFactory.create());
        return builder.build();
    }
```

* 注意：由于MVP框架使用的是Dagger2 + RxJava2 + Retrofit2 + OkHttp3 + FastJson。此处只说配置Retrofit。

上面配置Retrofit的时候，添加 addCallAdapterFactory方法，这样此前Retrofit定义的接口返回的就不是Call了，而是Observalbe，这样就能用RxJava提供的方法来对请求网络的回调进行处理。添加的addConverterFactory方法支持对返回结构的支持类型，此处支持的为FastJson 和String类型。

最后，给出MVP框架的地址，此项目框架采用 OkHttp3 + RxJava2 + Retrofit2 + Dagger2 + FastJson 为主体的 mvp框架。里面有详细的Retrofit配置，已经RxJava请求联网时线程转换

[MVPFramework](https://github.com/JiaYang627/MVPFramework)