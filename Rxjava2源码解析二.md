# Rxjava2源码解析二
## 目录
### 1、map源码解析
### 2、FlatMap源码解析
### 3、concatMap源码解析
### 4、concatMap为何在变换中有序输出
### 5、常见场景中:模拟多个网络请求FlatMap整合流程以及线程调度过程源码解析

## 一、map
* 用于对Observable中泛型类型的转换
### 1.1、demo示例
```java
Observable.create(new ObservableOnSubscribe<Integer>() {
         @Override
         public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
             emitter.onNext(1);
         }
     }).map(new Function<Integer, String>() {
         @Override
         public String apply(Integer integer) throws Exception {
             return "emmitter integer" + integer;
         }
     }).subscribe(new Consumer<String>() {
         @Override
         public void accept(String s) throws Exception {
             Log.e("mapResult", s);
         }
     });
```
### 1.2、涉及相关类
#### 1.2.1、ObservableMap（前缀为Observable基本都是被观察者的子类）
*继承关系`extends AbstractObservableWithUpstream<T, U> extends Observable<U>`,可以看出这是一个被观察者的子类
#### 1.2.2、MapObserver(后缀为Observer基本都是观察者的子类)
*继承关系`extends BasicFuseableObserver<T, R> implements Observer<T>, QueueDisposable<R>`,可以看出这是一个观察者的子类
#### 1.2.3、Funtion
*`R apply(@NonNull T t) `这是一个接口，功能只有一个就是把 对象t转化为对象R

### 1.3、伪代码执行过程
```java
   ObservableOnSubscribe oos = new ObservableOnSubscribe(){
     public void subscribe(ObservableEmmiter emmitter);
   };
   ObservableCreate oc = Observable.create(oos);
   Function f = new Function<Integer,String>(){
     public String apply(Integer i);
   };
   ObservableMap om = oc.map(f){
     this.source = oc;
   };
   LambdaObserver lo = new LambdaObserver(new Consumer());
   om.subscribeActual(lo){
      MapObserver mo = new MapObserver(lo,f){
        this.actual = lo;
      };
      oc.subscribeActual(mo){
        CreateEmmiter ce = new CreateEmitter(mo);
        oos.subscribe(ce){
          ce.onNext(1){
            mo.onNext(1){
              String s = f.apply(1);
              lo.onNext(s);
            };
          };
        }
      }
   }
```
## 二、FlatMap
* 也是利用Function对象进行转换，不过是把一个非ObservableSource对象，转换为ObservableSource对象
### 2.1、demo示例
```java
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                emitter.onNext(1);
                emitter.onNext(2);
                emitter.onNext(3);
            }
        }).flatMap(new Function<Integer, ObservableSource<String>>() {
            @Override
            public ObservableSource<String> apply(Integer integer) throws Exception {
                final List<String> list = new ArrayList<>();
                for (int i = 0; i < 3; i++) {
                    list.add("I am value " + integer);
                }
                return Observable.fromIterable(list);
            }
        }).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                System.out.println(s);
            }
        });
```
### 2.2、涉及相关类
*基本每个操作符都对应一组被观察者，观察者对象，这里就是ObservableFlatMap,MergeObserver,这里多了一个InnerObserver
#### 2.2.1、ObservableFlatMap
*被观察者子类，继承关系`extends AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T>`
#### 2.2.2、MergeObserver
*观察者，继承关系`extends AtomicInteger implements Disposable, Observer<T>`
#### 2.2.3、InnerObserver
*观察者，继承关系`extends AtomicReference<Disposable> implements Observer<U>`一般来说不需要两个观察者的，这里因为Function转换出一个新的Observable所以需要重新订阅，所以有了这个对象

#### 2.2.4、ObservableFromIterable
*被观察者子类` extends Observable<T>`
#### 2.2.5、fromIterableDisposable
*

### 2.3、伪代码执行流程
```java
ObservableOnSubscribe oos = new ObservableOnSubscribe(){
  public void subscribe(ObservableEmmiter emmitter);
};
ObservableCreate oc = Observable.create(oos);
Function f = new Function<Integer,ObservableSource<String>>(){
  public ObservableSource<String> apply(Integer i);
}
ObservableFlatMap OFM = oc.flatMap(f){
  this.source = oc;
};
LambdaObserver lo = new LambdaObserver(new Consumer());
OFM.subscribeActual(lo){
  MergeObserver mo = new MergeObserver(lo,...){
    this.actual = lo;
  };
   oc.subscribeActual(mo){
      CreateEmitter ce = new CreateEmitter(mo);
      oos.subscribe(ce){
        ce.onNext(1){
          mo.onNext(1){
            ObservableSource p = f.apply(1){
              return Observable.fromIterable(list);
            };//此处就是ObservableFromIterable
            InnerObserver io = new InnerObserver(mo,...){
              this.parent = mo;
            };
            p(ObservableFromIterable).subscribeActual(io){
                Iterator it = list.iterator();
                FromIterableDisposable fid = new FromIterableDisposable(io,it){
                  this.actual = io;
                };
                fid.run(){
                  do{
                    T v = it.next();
                    io.onNext(v){
                      parent(mo).drainLoop(){
                        for(;;){
                          lo.onNext(v);
                        }
                      }
                    };
                  }while(hasNext)
              }
            }
          }
        };
      }
   }
}
```
### 三、concatMap
#### 3.1、demo示例
```java
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                emitter.onNext(1);
                emitter.onNext(2);
                emitter.onNext(3);
            }
        }).concatMap(new Function<Integer, ObservableSource<String>>() {
            @Override
            public ObservableSource<String> apply(Integer integer) throws Exception {
                final List<String> list = new ArrayList<>();
                for (int i = 0; i < 3; i++) {
                    list.add("I am value " + integer);
                }
                return Observable.fromIterable(list);
            }
        }).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                System.out.println(s);
            }
        });
```
#### 3.2、涉及相关类
##### 3.2.1、ObservableConcatMap
*继承关系`AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T>`
##### 3.2.2、SourceObserver
*继承关系`extends AtomicInteger implements Observer<T>, Disposable`
##### 3.2.3、SourceObserver$InnerObserver
*继承关系`extends AtomicReference<Disposable> implements Observer<U>`

#### 3.3、伪代码流程
```java
ObservableOnSubscribe oos = new ObservableOnSubscribe(){
  public void subscribe(ObservableEmmiter emmitter);
};
ObservableCreate oc = Observable.create(oos);
Function f = new Function<Integer,ObservableSource<String>>(){
  public ObservableSource<String> apply(Integer i);
}
ObservableConcatMap ocm = oc.concatMap(f){
  this.source = oc;
};
LambdaObserver lo = new LambdaObserver(new Consumer());
ocm.subscribeActual(lo){
  SerializedObserver SerializedObserver = new SerializedObserver(lo){
    this.actual = lo;
  }
  SourceObserver so = new SourceObserver(SerializedObserver,f,..){
    InnerObserver io = new InnerObserver<U>(SerializedObserver, this){
      this.actual = SerializedObserver;
      this.parent = SourceObserver;
    };
  };
  oc.subscribeActual(so){
    CreateEmitter ce = new CreateEmitter(so);
    oos.subscribe(ce){
      ce.onNext(1){
        so.onNext(1){
          SpscLinkedArrayQueue queue = new SpscLinkedArrayQueue<T>(bufferSize);
          queue.offer(1);
          for(;;){
            boolean active = false;
            if(!active){
              T  t  = queue.poll();
              ObservableFromIterable ofi = f.apply(t){
                retrun Observable.fromIterable(list);
              };
            }
            active=true;
            ofi.subscribeActual(io){
              Iterator it = list.iterator();
              FromIterableDisposable fid = new FromIterableDisposable(io,it){
                this.actual = io;
              };
              fid.run(){
                do{
                  T v = it.next();
                  io.onNext(v){
                     serializedObserver.onNext(v){
                       lo.onNext(v);
                     };
                  };
                  io.onComplete(){
                    active=false;
                  }
                }while(hasNext)
              }
            };
          }
        };
      };
    }
  };
};
```
### 四、concatMap为何有序一个
*SourceObserver中存在一个原子性的布尔值active，用来控制他们的有序，只有在InnerObserver执行完OnComplete之后，才能从队列中取值进行下一个数据的发射转换
