# Rxjava2源码解析一
## 目录
### 1、Observable与Observer
### 2、Rxjava线程调度原理
### 3、observeOn与subscribeOn控制线程的执行顺序解析

## 一、Observable与Observer
上游：Observable
下游：Observer

### demo代码
```java
   Observable observable = Observable.create(new ObservableOnSubscribe(){
       @Override
       public void subscribe(ObservableEmitter emitter) throw Exception{
            emitter.onNext(1);
            emitter.onNext(1);
            emitter.onNext(1);
            emitter.onComplete();
       }
   });
   Observer<Integer> observer = new Observer<Integer>(){
      @Override
      public void onSubscribe(Disposable d){

      }

      @Override
      public void onNext(Integer i){

      }

      @Override
      public void onError(Throwable e){

      }

      @Override
      public void onComplete(){

      }
   }

   observable.subscribe(observer);
```
### 涉及到的类
#### 1、Observable
` Observable  extends ObservableSource `
#### 2、ObservableEmitter
` ObservableEmitter extends Emitter`
#### 3、ObservableOnSubscribe
#### 4、ObservableCreate
```java
   public static <T> Observable<T> Observable.create(ObservableOnSubscribe<T> source){
       ...
       return Rxjavaplugins.onAssembly(new ObservableCreate<T>(source));
   }
   实际返回类型为 ObservableCreate
```
#### 5、Disposable与 AtomicReference<Disposable>
##### 相当于一个flag,与AtomicReference合用，获取一个常量，与枚举值 DISPOSED 比较


### 整个流程伪代码
```java
   ObservableOnSubscribe observableOnSubscribe = new ObservableOnSubscribe();
   ObservableCreate observable = Observable.create(observableOnSubscribe);
   Observer<T> observer = new Observer<T>(){
      void  onSubscribe(Disposable d);
      void  onNext(T t);
      void  onError(Throwable e);
      void  onComplete();
   }
   observable.subscribe(observer){
      ......
      Observable.subscribleActual(observer){
          CreateEmitter parent = new CreateEmitter(observer){
             public void  onNext(Object o){
                if (!isDisposed()) {
                  observer.onNext(o);
                }
             }

             public void onError(Throwable e){
               if (!isDisposed()) {
                  try {
                      observer.onError(t);
                  } finally {
                      dispose();
                  }
               }
             }

             public void onComplete(){
               if(!isDisposed()){
                 try {
                     observer.onComplete();
                 } finally {
                     dispose();
                 }
               }
             }
          };
          observer.onSubscribe(parent);
          observableOnSubscribe.subscribe(parent);
      };
   };
```

#### subscriber(observer)重载方法
```java
   observable.subscribe();
   observable.subscribe(Consumer onNext);
   伪代码流程
   public void subscribe(){
     subscribe(Functions.emptyConsumer(), Functions.ON_ERROR_MISSING, Functions.EMPTY_ACTION, Functions.emptyConsumer()){
         LambdaObserver<T> ls = new LambdaObserver<T>(onNext, onError, onComplete, onSubscribe){
            @Override
            public void onNext(T t) {
                if (!isDisposed()) {
                    try {
                        onNext.accept(t);
                    } catch (Throwable e) {
                        Exceptions.throwIfFatal(e);
                        get().dispose();
                        onError(e);
                    }
                }
            }
         };
     };
   }
```
## 二、Rxjava线程调度原理
### 2.1 demo代码
```java
Observable<Integer> observable = Observable.create(new ObservableOnSubscribe<Integer>() {
         @Override
         public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
             Log.e("subscribe",Thread.currentThread().getName());
             emitter.onNext(1);
         }
     });
     observable
             .subscribeOn(Schedulers.io())
             .subscribeOn(Schedulers.newThread())
             .observeOn(Schedulers.newThread())
             .observeOn(Schedulers.io())
             .subscribe(new Consumer<Integer>() {
                 @Override
                 public void accept(Integer integer) throws Exception {
                     Log.e("onNext",Thread.currentThread().getName());
                     Log.e("onNext",integer+"");
                 }
             }, new Consumer<Throwable>() {
                 @Override
                 public void accept(Throwable throwable) throws Exception {
                     Log.e("onError",Thread.currentThread().getName());
                     throwable.printStackTrace();
                 }
             });
 }
```
### 2.2 涉及相关类
#### 2.2.1、ObservableObserveOn
##### `extends AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T>`
##### 从继承关系可以看出这是一个 被观察者子类
#### 2.2.2、`ObservableObserveOn$ObserveOnObserver implements Observer<T>, Runnable`
##### 既是一个观察者，也是一个Runnable，
#### 2.2.3、ObservableSubscribeOn
##### `extends AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T>`
##### 从继承关系可以看出这是一个 被观察者子类
#### 2.2.4、`ObservableSubscribeOn$SubscribeOnObserver implements Observer`
##### 一个观察者子类
#### 2.2.5、Schedulers 任务调度器，以及对应的子类，这里有NewThreadScheduler,IoScheduler
#### 2.2.6、Worker 各子类，NewThreadWorker,EventLoopWorker,内部都存在对应的线程、以及线程池类

### 2.3 调度原理伪代码描述执行过程
####  模拟`.subscribeOn(Schedulers.newThread()).observeOn(Schedulers.newThread())`过程
```java
    ObservableOnSubscribe observableOnSubscribe = new ObservableOnSubscribe();
    ObservableCreate oc = Observable.create(observableOnSubscribe){
          this.source = observableOnSubscribe;
    };
    ObservableSubscribeOn oso = oc.subscribeOn(Schedulers.io()){
       this.source = oc;
       this.scheduler = ioScheduler;
    };
    ObservableObserveOn OOO= oso1.observeOn(Schedulers.newThread()){
       this.source = oso;
       this.scheduler = newThreadScheduler;
    };
    LambdaObserver<T> lo = new LambdaObserver<T>(onNext, onError, onComplete, onSubscribe);
    OOO.subscribe(lo){
       OOO.subscribeActual(lo){
         Schedulers.Worker w1 = scheduler.createWorker();
         ObserveOnObserver OOnO = new ObserveOnObserver(lo,w,...){
             this.actual = lo;
             this.worker = w1;
         };
         oso.subscribe(OOnO){
             oso.subscribeActual(OOnO){
                SubscribeOnObserver soo = new SubscribeOnObserver(OOnO){
                    this.actual = OOnO;
                };
                SubscribeTask st = new SubscribeTask(soo){
                    public void run(){
                       oc.subscribe(soo){
                          oc.subscribeActual(soo){
                            CreateEmitter<T> ce = new CreateEmitter<T>(soo);
                            observableOnSubscribe.subscribe(ce){
                                ce.onNext(1){
                                   soo.onNext(1){
                                      actual(OOnO).onNext(1){
                                         //observer线程调度
                                         schedule(){
                                            OOnO.run(){
                                               drainNormal(){
                                                  lo.onNext(1);
                                               };
                                            }
                                         };
                                      }
                                   };
                                }
                            };
                          }
                       };
                    }
                };
                //Observable线程调度
                scheduler.scheduleDirect(st);
             }
         }
       }
    };
```
#### 2.4 伪代码文字解析
* 1、首先弄清楚上面有几个被观察者，几个观察者，被观察者有ObservableCreate、ObservableObserveOn、ObservableSubscribeOn，观察者有
ObserveOnObserver，SubscribeOnObserver,LambdaObserver
* 2、被观察者相互之间都是包含于被包含的关系，比如 `ObservableSubscribeOn.source = ObservableCreate` `ObservableObserveOn.source=
ObservableSubscribeOn`,每次调用subscribe方法时候都是递归调用source的subscribe直到 最内层 ObservableCreate.subscribeActual()内部
CreateEmitter 发射器 subscribe方法即 emmitter.onNext(1);
* 3、观察者中也存在着包含与被包含关系，比如`ObserveOnObserver.actual = LambdaObserver` `SubscribeOnObserver.actual =
ObserveOnObserver`,从`CreateEmitter.onNext()`开始依次递归调用`SubscribeOnObserver,ObserveOnObserver,LambdaObserver`
* 4、被观察者的线程调度在`ObservableSubscribeOn`中在ScribeTask线程中执行了 `ObservableCreate.subscribe()`从而改变的执行线程
* 5、ObserveOnObserver本身就是一个Runnable接口，所以本身就是可以控制线程的执行

### 三、observeOn与subscribeOn控制线程的执行顺序解析
```java
//写法一
observableCreate.observeOn(Schedulers.newThread()).subscribeOn(Schedulers.io());
//伪代码执行顺序为
ObservableSubscribeOn.subscribeActual(observer){
  SubscribeOnObserver<T> soo = new SubscribeOnObserver<T>(observer);
  st = new SubscribeTask(soo){
    run(){
      ObservableObserveOn.subscribeActual(soo){
        OOnO = new ObserveOnObserver<T>(observer, w, delayError, bufferSize);
        observableCreate.subscribeActual(OOnO){
            emitter.onNext(1){
              OOnO.onNext(1){
                //切换了线程
                worker.schedule(this){
                   soo.onNext(1){
                     observer.onNext(1);
                   }
                };
              }
            }
        }
      }
    }
  }
}

```
```java
//写法二
observableCreate.subscribeOn(Schedulers.io()).observeOn(Schedulers.newThread());
//伪代码执行顺序
ObservableObserveOn.subscribeActual(observer){
  OOnO = new ObserveOnObserver<T>(observer, w, delayError, bufferSize);
  ObservableSubscribeOn.subscribeActual(OOnO){
    SubscribeOnObserver<T> soo = new SubscribeOnObserver<T>(OOnO);
    st = new SubscribeTask(soo){
      //切换了线程
      run(){
        emitter.onNext(1){
          soo.onNext(1){
            OOnO.onNext(1){
              worker.schedule(this){
                observer.onNext(1);
              }
            }
          }
        }
      }
    }
  }
}
```
* 从上面执行的结果可以看出，subscribeOn、observeOn的顺序 并不影响观察者与被观察者线程
的切换，主要原因是因为，ObservableSubscribeOn是在subscribeActual中先切换线程再向上迭代，而ObservableObserveOn总是先向上迭代，只有在emitter.onNext发射完之后迭代到ObserveOnObserver时才会在自身这里切换线程
#####  可以得出结论，多个subscribeOn，与observeOn多个切换线程时，切换的顺序为
* subscribeOn(N) ——> subscribeOn(N-1)——>....——>subscribeOn(1)——>observeOn(1)——>
....observeOn(N);
