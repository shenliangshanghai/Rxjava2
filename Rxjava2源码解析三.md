# Rxjava2源码解析三
## 目录
### 1、zip源码解析

## 一、zip
### 1.1、demo示例
```java
Observable<Integer> observable1 = Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                emitter.onNext(1);
                emitter.onNext(2);
                emitter.onComplete();
            }
        });
        Observable<String> observable2 = Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("A");
                emitter.onComplete();
            }
        });
        Observable.zip(observable1, observable2, new BiFunction<Integer, String, String>() {
            @Override
            public String apply(Integer integer, String s) throws Exception {
                return integer+s;
            }
        }).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                Log.e("zipResult",s);
            }
        });
```

### 1.2、涉及相关类
#### 1.2.1、ObservableZip
*这是一个被观察者，继承关系`ObservableZip<T, R> extends Observable<R>`
#### 1.2.2、ZipObserver
*这是一个观察者，继承关系`ZipObserver<T, R> implements Observer<T>`
#### 1.2.3、ZipCoordinator
*这是一个协调类

### 1.3、伪代码解析
```java
   //关键对象，及所含关键成员变量
   ObservableOnSubscribe oos1 = new ObservableOnSubscribe(){
     public void subscribe(ObservableEmitter emmiter){
       emmiter.onNext(1);
       emmiter.onNext(2);
     }
   }
   ObservableCreate oc1 = Observable.create(oos1);

   ObservableOnSubscribe oos2 = new ObservableOnSubscribe(){
     public void subscribe(ObservableEmitter emmiter){
       emmiter.onNext(A);
     }
   }
   ObservableCreate oc2 = Observable.create(oos2);
   ObservableSource[] ocArray = new ObservableSource[]{oc1,oc2};
   Array2Func af = new Array2Func<T1, T2, R>(f){
     public R apply(Object[] a) throws Exception {
            return f.apply((T1)a[0], (T2)a[1]);
     }
   }
   ObservableZip oz = new ObservableZip(ocArray,af,bufferSzie...);
   LambdaObserver lo = new LambdaObserver(new Consumer());
   oz.subscribeActual(lo){
      ZipCoordinator<T, R> zc = new ZipCoordinator<T, R>(lo, af, 2, delayError){
        this.actual = lo;
        this.zipper = af;
        this.observers = new ZipObserver[2];
        this.row = (T[])new Object[count];//
      };
      zc.subscribe(ocArray,2){
        for (int i = 0; i < 2; i++) {
          observers[i] = new ZipObserver<T, R>(this, 2){
            this.parent = parent;
            this.queue = new SpscLinkedArrayQueue<T>(bufferSize);
          };
        }
        for (int i = 0; i < 2; i++) {
          ocArray[i].subscribeActual(observers[i]){
            CreateEmmiter ce = new CreateEmmiter(observers[i]);
            ocArray[i].oos.subscribe(ObservableEmitter emmiter){
                emmiter.onNext(1){
                  observers[i].onNext(1){
                    queue.offer(t);//队列中加数据
                    zc.drain(){
                      for(;;){
                        for(;;){
                           for(ZipObserver z : observers){
                              if(row[i]==null){
                                T v = z.queue.poll();
                                if (v!=null) {
                                  os[i] = v;
                                } else {
                                  emptyCount++;
                                }
                              }
                              i++;
                            }
                            if (emptyCount != 0) {
                                break;
                            }
                            String s = af.apply(row){
                              return 1+"A";
                            }
                            lo.onNext(s);
                            Arrays.fill(os, null);
                          }
                        }
                      }
                    };
                  }
                }
            }
          };
       }
      };
   }

```
*主要就是根据ObservableCreate动态生成两个ZipObserver对象，每个ZipObserver中有都存在一个queue,emmiter发射的内容都会存到这个queue中，每次emmiter.onNext()之后都会循环遍历每个队列中对应的位置上的内容，用一个Object[]数组保存，分别从两个队列中都取到值存入数组时，才会执行到Array2Func进行变换，emptyCount就是判断是否都取到值的一个flag,因为第一个管道发射内容的时候，第二个管道还没有执行发射内容，所以肯定是第一个管道里的内容全部发射完毕，才会出现都取到值的情况
