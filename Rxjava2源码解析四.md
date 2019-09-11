# Rxjava2源码解析四
## 目录
### 1、背压的产生
### 2、Flowable 源码解析

## 一、背压？
### 1.1、背压如何产生的
* 在某些场景下，上游生产的速度远远大于下游处理的速度，导致内存不断的被消耗，因此可能出现的OOM内存溢出问题

### 1.2、背压的解决方式
#### 1.2.1、降低上游的生产速度
* 因为没法确定下游处理额速度，哪怕你每生产一个产品等待一定的时间，也没法完全保证不会出现内存消耗过大情况，只能说大大降低OOM的可能性

#### 1.2.2、减少或者过滤掉部分生产数量
* 直接过滤掉部分数据，实际问题中，大部分数据都是有用的，这个很可能导致最终的结果有问题

#### 1.3、Flowable的产生
* 在可观察者`Observable extends ObservableSource`，观察者`Observer`体系中没法很好的处理背压问题，所以出现了可观察者`Flowable extends Publisher` ，观察者`Subscriber`体系

## 二、Flowable
### 2.1、背压模式
#### 2.1.1、`BackpressureStrategy.ERROR`
##### 同步情况下：上游生产之后，下游不处理就会直接抛出异常`MissingBackpressureException`,异步情况：上游生产的会先放到一个128长度的queue容器中，一旦下游不处理，导致容器存储溢出也会抛出`MissingBackpressureException`
#### 2.1.2、`BackpressureStrategy.BUFFER`
##### 相当于容器大小不做限制了，因此可能会出现OOM情况
#### 2.1.3、`BackpressureStrategy.DROP`
##### 上游生产的存储到128的容器中，一旦容器满了之后，上游生产的数据直接丢弃
#### 2.1.4、`BackpressureStrategy.LATEST`
##### 跟Drop类似，但是最新的那条数据总是会保留下来
### 2.2、示例demo
```java
Flowable.create(new FlowableOnSubscribe<Integer>() {
            @Override
            public void subscribe(FlowableEmitter<Integer> emitter) throws Exception {
                Log.e("flowableEmitter", "First requested = " + emitter.requested());
                boolean flag;
                for (int i = 0; ; i++) {
                    flag = false;
                    while (emitter.requested() == 0) {
                        if (!flag) {
                            Log.e("flowableEmitter", "Oh no! I can't emit value!");
                            flag = true;
                        }
                    }
                    emitter.onNext(i);
                    Log.e("flowableEmitter", "emit " + i + " , requested = " + emitter.requested());
                }
            }
        }, BackpressureStrategy.ERROR)
                .subscribeOn(Schedulers.io())
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {

                    private Subscription subscription;

                    @Override
                    public void onSubscribe(Subscription s) {
                        subscription = s;
                        s.request(1);
                    }

                    @Override
                    public void onNext(Integer integer) {
                        Log.e("flowableEmitter", "integer = [" + integer + "]");
                        try {
                            Thread.sleep(1000);
                            subscription.request(1);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }

                    @Override
                    public void onError(Throwable t) {
                        t.printStackTrace();
                    }

                    @Override
                    public void onComplete() {
                        Log.e("flowableEmitter", "onComplete");
                    }
                });
```
### 2.3、涉及相关类
#### 2.3.1 命名规律：
* 前缀Flowable 可观察者
* 后缀Subscriber 观察者

#### 2.3.2、FlowableOnSubscribe
* 简写 fos

#### 2.3.3、FlowableCreate
* 继承关系`extends Flowable`,对比ObservableCreate
* 简写 fc
* 成员变量：
```java
source = fos
mode = BackpressureStrategy.ERROR
```
#### 2.3.4、FlowableSubscribeOn
*继承关系`extends AbstractFlowableWithUpstream<T, R> extends Flowable<R> implements HasUpstreamPublisher<T>`
*简写 fso
*成员变量
```java
source = fc
scheduler = ioScheduler
```
#### 2.3.5、FlowableObserveOn
*继承关系`AbstractFlowableWithUpstream<T, R> extends Flowable<R> implements HasUpstreamPublisher<T>`
*简写：foo
*成员变量
```java
source = fso;
prefetch = 96;
scheduler = newThreadScheduler;
```
#### 2.3.6、StrictSubscriber
*继承`extends AtomicInteger implements FlowableSubscriber<T>, Subscription`
*简写：ss
*成员变量
```java
actual = new Subscriber();
s = new AtomicReference<Subscription>();
```
#### 2.3.7、ObserveOnSubscriber
*继承`extends BaseObserveOnSubscriber<T> extends BasicIntQueueSubscription<T>
    implements FlowableSubscriber<T>, Runnable`
*简写：oos
*成员变量
```java
actual = ss;
prefetch = 128;
limit = 96;
Subscriber s 未初始化，onSubscribe中进行初始化
```
#### 2.3.8、SubscribeOnSubscriber
*继承`extends AtomicReference<Thread>
    implements FlowableSubscriber<T>, Subscription, Runnable`
*简写：sos
*成员变量
```java
actual = oos;
s = new AtomicReference<Subscription>();
```

### 2.4、伪代码流程
```java
mSubscriber = null;
foo.subscribeActual(ss){
  fso.subscribeActual(oos){
    oos.onSubscribe(sos){
      oos.s = sos;
      ss.onSubscribe(oos){
        subscriber.onSubscribe(ss){
          mSubscriber = ss;
        }
      }
    }
    fc.subscribeActual(sos){
      ErrorAsyncEmitter ee = ErrorAsyncEmitter(sos);
      sos.onSubscribe{
        SubscriptionHelper.setOnce(this.s, s);
      }
      fos.subscribe(ee){
        ee.onNext(1){
          sos.onNext(1){
            oos.onNext(1){
              oos.runAsync(){
                long e = produced;
                for(;;){
                  long r = requested.get();
                  while(e!=r){
                    T v = queue.poll();
                    ss.onNext(v){
                      subscriber.onNext(v){
                         mSubscriber.request(1){
                           oos.runAsync(){
                             ....
                           }
                         };
                      }
                    };
                    e++;
                    if (e == limit) {
                        r = requested.addAndGet(-e);
                        s.request(e);
                        e = 0L;
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```
* Flowable与Subscriber 订阅大体逻辑基本和Observable体系差不多，唯一的是要知道mSubscriber.request(1)是如何在线程之间交互修改request的值的，从而达到通过下游消费
速度控制上游生产速度；还有为什么在容器大小为128的情况下，下游只有消费达到96的时候才会重新开启生产，这部分逻辑是存在于`FlowableObserveOn.runAsync()`中
