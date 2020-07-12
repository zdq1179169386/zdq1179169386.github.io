---
title: RxSwift 操作符
date: 2020-04-10 17:18:20
tags: [RxSwift]

---

操作符可以帮助大家创建新的序列，或者变化组合原有的序列，从而生成一个新的序列。

#### map

map 操作符将源 Observable 的每个元素应用你提供的转换方法，然后返回含有转换结果的 Observable

```
Observable.of(1,2,3)
        .map{$0 * 10}
        .subscribe(onNext: { (value) in
            print(value)
        }).disposed(by: disposeBag)
```

输出：

```
10
20
30
```

#### flatMap
flatMap 操作符将源 Observable 的每一个元素应用一个转换方法，将他们转换成 Observables。 然后将这些 Observables 的元素合并之后再发送出来。

```
//variable 要废弃掉了，要用 BehaviorRelay 替换
let subject1 = BehaviorSubject(value: "A")
let subject2 = BehaviorSubject(value: "1")
let variable = BehaviorRelay(value: subject1);
variable.asObservable()
    .flatMap{$0}
    .subscribe(onNext: { (value) in
        print(value)
    }).disposed(by: disposeBag)
    
subject1.onNext("B")
variable.accept(subject2)
subject2.onNext("2")
subject1.onNext("C")
```
输出：

```
A
B
1
2
C
```

#### map 和 flatMap 的区别

先看下swift 中的 map 和 flatMap 的区别

```
let array = [[1,2,3],[4,5]]
let array2 = array.map{$0}
print(array2)
    
let array3 = array.flatMap{$0}
print(array3)
```
输出：

```
[[1, 2, 3], [4, 5]]
[1, 2, 3, 4, 5]
```
明显就能看出
flatMap 把大数组拉平了，而 map 只会操作大数组中的小数组上，不会对小数组中元素有任何操作

RxSwift 中的 map 和 flatMap 

```
let array = [[1,2,3],[4,5]]

Observable.from(array)
.map{$0}
.subscribe(onNext: { (value) in
    print(value)
}).disposed(by: disposeBag)

//flatMap ： 不仅可以将大数组拉平，而且能将小数组中的每个元素，转换为字符    
Observable.from(array)
        .flatMap { (numbers) -> Observable<String> in
            return Observable.from(numbers).map{":"+String($0)}
    }.subscribe(onNext: { (value) in
        print(value)
        }).disposed(by: disposeBag)
```
输出：

```
[1, 2, 3]
[4, 5]

:1
:2
:4
:3
:5
```
区别: 

* map 变换后返回任意类型，flatMap : 返回 Observable<T> 类型

* map只能进行一对一的变换，而flatMap则可以进行一对多，多对多的变换

#### flatMapLatest

 flatMapLatest与flatMap 的唯一区别是：flatMapLatest只会接收最新的value 事件。

```
  let subject1 = BehaviorSubject(value: "A")
    let subject2 = BehaviorSubject(value: "1")
    
    let relay = BehaviorRelay(value: subject1)
    relay.asObservable().flatMapLatest{ $0 }
        .subscribe(onNext: { (value) in
            print(value)
        }).disposed(by: disposeBag)
    subject1.onNext("B")
    relay.accept(subject2)
    subject2.onNext("2")
    subject1.onNext("C")
```

#### concatMap

concatMap 与 flatMap 的唯一区别是：当前一个 Observable 元素发送完毕后，后一个Observable 才可以开始发出元素。或者说等待前一个 Observable 产生完成事件后，才对后一个 Observable 进行订阅。

```
let subject1 = BehaviorSubject(value: "A")
let subject2 = BehaviorSubject(value: "1")
    
let relay = BehaviorRelay(value: subject1)
    
relay.asObservable()
    .concatMap{$0}
    .subscribe(onNext: { (value) in
        print(value)
    }).disposed(by: disposeBag)
    
subject1.onNext("B")
relay.accept(subject2)
subject2.onNext("2")
subject1.onNext("C")
subject1.onCompleted()
```
输出:

```
A
B
C
2
```


#### filter

filter: 会过滤掉小于满足条件的元素

```
Observable.of(2,30,40,22,6,9).filter {
	$0 > 10
}.subscribe(onNext: { (value) in**
	print(value)
}).disposed(by: disposeBag)
```
输出:
 
```
30
40
22
```

#### distinctUntilChanged
distinctUntilChanged:  操作符将阻止 Observable 发出相同的元素。如果后一个元素和前一个元素是相同的，那么这个元素将不会被发出来

```
Observable.of(1,1,2,3,3,4)
    .distinctUntilChanged()
    .subscribe(onNext: { (value) in
        print(value)
    }).disposed(by: disposeBag)
```
 
 输出：
 
```
1
2
3
4
```

#### single
single: 限制只发送一次事件，或者满足条件的第一个事件。如果存在有多个事件或者没有事件都会发出一个 error 事件。如果只有一个事件，则不会发出 error事件。

```
Observable.of(1, 2, 3, 4)
    .single{ $0 == 2 }
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
```
输出：

```
2
```
#### elementAt
该方法只发送指定位置的元素

```
Observable.of(1, 2, 3, 4)
    .elementAt(2)
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
```
输出:

```
3
```

#### ignoreElements
ignoreElements 操作符将阻止 Observable 发出 next 事件，但是允许他发出 error 或 completed 事件。
如果你并不关心 Observable 的任何元素，你只想知道 Observable 在什么时候终止，那就可以使用 ignoreElements 操作符。

```
Observable.of(1,2,3)
    .ignoreElements()
    .subscribe(onCompleted: {
        print("onCompleted")
    }).disposed(by: disposeBag)
```
输出：

```
onCompleted
```
#### take
该方法实现仅发送 Observable 序列中的前 n 个事件，在满足数量之后会自动 .completed。

```
Observable.of(1,2,3,4)
    .take(2)
    .subscribe(onNext: { (value) in
        print(value)
    }).disposed(by: disposeBag)
```
输出：

```
1
2
```
#### takeLast
该方法实现仅发送 Observable序列中的后 n 个事件
 
```
Observable.of(1, 2, 3, 4)
    .takeLast(1)
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
```
输出：

```
4
```

#### takeWhile

该方法依次判断 Observable 序列的每一个值是否满足给定的条件。 当第一个不满足条件的值出现时，它便自动完成。

```
Observable.of(1,2,3,4)
        .takeWhile({$0 < 3})
        .subscribe(onNext: { (value) in
            print(value)
        }).disposed(by: disposeBag)
```
输出：

```
1
2
```
#### takeUntil

 除了订阅源 Observable 外，通过 takeUntil 方法我们还可以监视另外一个 Observable， 即 notifier。如果 notifier 发出值或 complete 通知，那么源 Observable 便自动完成，停止发送事件。
 
```
let source = PublishSubject<String>()
    let notifier = PublishSubject<String>()
    
    source.takeUntil(notifier)
        .subscribe(onNext: { (value) in
            print(value)
        }).disposed(by: disposeBag)
    
    source.onNext("a")
    source.onNext("b")
    //notifier.onCompleted()//好像不奏效
    notifier.onNext("---")
    source.onNext("1")
    source.onNext("2")
```


#### skip
 该方法用于跳过 Observable 序列发出的前 n 个事件。
 
```
Observable.of(1, 2, 3, 4)
    .skip(2)
    .subscribe(onNext: { print($0) })
    .disposed(by: disposeBag)
```

输出：

```
3
4
```

#### skipWhile

该方法用于跳过前面所有满足条件的事件。一旦遇到不满足条件的事件，之后就不会再跳过了。

```
Observable.of(1,2,3,4,5)
            .skipWhile({$0 < 4})
            .subscribe(onNext: { (value) in
                print(value)
            }).disposed(by: disposeBag)
```
输出：

```
4
5
```

#### skipUntil (跳过直到)

同上面的 takeUntil 一样，skipUntil 除了订阅源 Observable 外，通过 skipUntil方法我们还可以监视另外一个 Observable， 即 notifier 。
 与 takeUntil 相反的是。源 Observable 序列事件默认会一直跳过，直到 notifier 发出值或 complete 通知
 
 ```
  let source = PublishSubject<Int>()
        let notifier = PublishSubject<Int>()
        source.skipUntil(notifier)
            .subscribe(onNext: { (value) in
                print(value)
            }).disposed(by: disposeBag)
        
        source.onNext(1)
        source.onNext(2)
        source.onNext(3)
        notifier.onNext(0)
        
        source.onNext(4)
        source.onNext(5)
        
        notifier.onNext(100)
        
        source.onNext(6)
 ```
 
 输出：
 
 ```
4
5
6
 ```
 source 发送的 1，2 ，3 收不到，直到 notifier 发送了0， 后面发送的才能接收到
 

#### sample

sample 操作符将不定期的对源 Observable 进行取样操作。通过第二个 Observable 来控制取样时机。一旦第二个 Observable 发出一个元素，就从源 Observable 中取出最后产生的元素。

```
let source = PublishSubject<Int>()
let notifier = PublishSubject<String>()
source.sample(notifier)
    .subscribe(onNext: { (value) in
        print(value)
    }).disposed(by: disposeBag)
source.onNext(1)
    
notifier.onNext("A")
    
source.onNext(2)
    
notifier.onNext("B")
notifier.onNext("C")
    
source.onNext(3)
source.onNext(4)
    
//让源序列接收接收消息
notifier.onNext("D")
    
source.onNext(5)
    
//让源序列接收接收消息
notifier.onCompleted()
```
> 只有notifier 发送了onNext 或  onCompleted, source 才能接收到最后的元素

输出：

```
1
2
4
5
```
#### buffer

缓存元素，然后将缓存的元素集合，周期性的发出来

```
let subject = PublishSubject<String>()
subject.buffer(timeSpan: RxTimeInterval.seconds(1), count: 3, scheduler: MainScheduler.init())
    .subscribe(onNext: {
        print($0)
    }).disposed(by: disposeBag)
    
subject.onNext("1")
subject.onNext("2")
subject.onNext("3")
    
subject.onNext("A")
subject.onNext("B")
subject.onNext("C")
    
subject.onCompleted()
```
输出： 

```
["1", "2", "3"]
["A", "B", "C"]
```

#### window
window 操作符和 buffer 十分相似，buffer 周期性的将缓存的元素集合发送出来，而 window 周期性的将元素集合以 Observable 的形态发送出来。

buffer 要等到元素搜集完毕后，才会发出元素序列。而 window 可以实时发出元素序列。

```
let subject = PublishSubject<String>()
subject.window(timeSpan: RxTimeInterval.seconds(1), count: 3, scheduler: MainScheduler.instance)
    .subscribe(onNext: { [weak self] in
        print(":",$0)
        $0.asObservable().subscribe(onNext: {
            print($0)
        }).disposed(by: self!.disposeBag)
    }).disposed(by: disposeBag)
    
subject.onNext("A")
subject.onNext("B")
subject.onNext("C")
    
subject.onNext("1")
subject.onNext("2")
subject.onNext("3")
subject.onCompleted()
```
输出：
 
```
: RxSwift.AddRef<Swift.String>
A
B
C
: RxSwift.AddRef<Swift.String>
1
2
3
: RxSwift.AddRef<Swift.String>
```

#### scan

scan 就是先给一个初始化的数，然后不断的拿前一个结果和最新的值进行处理操作

```
Observable.of(1,2,3,4)
        .scan(1) { (ack, sum) in
            print(ack,sum)
            return  ack + sum
    }.subscribe(onNext: { (value) in
        print(value)
    }).disposed(by: disposeBag)
```

输出:

```
2
4
7
11
```

#### groupBy

groupBy 操作符将源 Observable 分解为多个子 Observable，然后将这些子 Observable 发送出来。

```
Observable<Int>.of(1,2,3,4,5,6,7,8,9)
        .groupBy { (value) -> String in
            return value % 2 == 0 ? "偶数" : "奇数"
    }.subscribe { (event) in
        switch event{
        case .next(let group):
            group.asObservable().subscribe { (event) in
                print(group.key,event)
            }.disposed(by: self.disposeBag)
        default :
            print("")
        }
    }.disposed(by: disposeBag)
```

输出：

```
奇数 1
偶数 2
奇数 3
偶数 4
奇数 5
偶数 6
奇数 7
偶数 8
奇数 9
```
#### amb

当传入多个 Observables 到 amb 操作符时，它将取第一个发出元素或产生事件的 Observable，然后只发出它的元素。并忽略掉其他的 Observables。

```
let subject1 = PublishSubject<Int>()
let subject2 = PublishSubject<Int>()
let subject3 = PublishSubject<Int>()
    
subject1
    .amb(subject2)
    .amb(subject3)
    .subscribe(onNext: { (value) in
        print(value)
    }).disposed(by: disposeBag)
    
subject2.onNext(1) //第一个
subject1.onNext(20)
subject2.onNext(2)
subject1.onNext(40)
subject3.onNext(0)
subject2.onNext(3)
subject1.onNext(60)
subject3.onNext(0)
subject3.onNext(0)
```
输出：

```
1
2
3
```
只会接受subject2 的,因为它是第一个发出元素的

#### startWith

该方法会在 Observable 序列开始之前插入一些事件元素。即发出事件消息之前，会先发出这些预先插入的事件消息。

```
Observable.of(1,2)
        .startWith(3)
        .startWith(4)
        .subscribe(onNext: { (value) in
            print(value)
        }).disposed(by: disposeBag)
```
输出：

```
4
3
1
2
```

#### merge

该方法可以将多个（两个或两个以上的）Observable 序列合并成一个 Observable序列。

```
let subject1 = PublishSubject<Int>()
let subject2 = PublishSubject<Int>()
Observable.of(subject1,subject2)
    .merge()
    .subscribe(onNext: { (value) in
        print(value)
    }).disposed(by: disposeBag)
subject1.onNext(1)
subject1.onNext(2)
subject2.onNext(3)
subject1.onNext(4)
subject2.onNext(5)
```

#### zip

该方法可以将多个（两个或两个以上的）Observable 序列压缩成一个 Observable 序列。
而且它会等到每个 Observable 事件一一对应地凑齐之后再合并。

```
let subject1 = PublishSubject<Int>()
let subject2 = PublishSubject<String>()
    
Observable.zip(subject1, subject2)
    .subscribe(onNext: { (value) in
        print(value)
    }).disposed(by: disposeBag)
    
subject1.onNext(1)
subject2.onNext("A")
subject1.onNext(2)
subject2.onNext("B")
subject2.onNext("C")
subject2.onNext("D")
subject1.onNext(3)
subject1.onNext(4)
subject1.onNext(5)
```
输出：

```
(1, "A")
(2, "B")
(3, "C")
(4, "D")
```
注意 subject1 最后发出的5 被丢掉了, 换成 combineLatest 就不会 。

#### combineLatest

该方法同样是将多个（两个或两个以上的）Observable 序列元素进行合并。
但与 zip 不同的是，每当任意一个 Observable 有新的事件发出时，它会将每个 Observable 序列的最新的一个事件元素进行合并。

```
let subject1 = PublishSubject<Int>()
let subject2 = PublishSubject<String>()
    
Observable.combineLatest(subject1, subject2)
    .subscribe(onNext: {
        print($0)
    }).disposed(by: disposeBag)
subject1.onNext(1)
subject2.onNext("A")
subject1.onNext(2)
subject2.onNext("B")
subject2.onNext("C")
subject2.onNext("D")
subject1.onNext(3)
subject1.onNext(4)
subject1.onNext(5)
```

输出：

```
(1, "A")
(2, "A")
(2, "B")
(2, "C")
(2, "D")
(3, "D")
(4, "D")
(5, "D")
```
#### zip 和  combineLast 的区别
仔细比较上面 zip 和 combineLast 的例子就知道，zip 必须两个 Subject 都发出数据，不对称则会丢掉其中一次。combineLast ，如果不对称，则会使用之前一次的数据，和最新的一次数据，进行合并

#### withLatestFrom

没清楚这个是干嘛的

#### switchLatest

### 算数聚合操作符
#### toArray

 该操作符先把一个序列转成一个数组，并作为一个单一的事件发送，然后结束。
 
```
Observable.of(1, 2, 3)
        .toArray()
        .subscribe(onNext: { print($0) })
        .disposed(by: disposeBag)
```
输出：

```
[1,2,3]
```

#### reduce

>这里的 reduce 不该翻译为减少，而是应该翻译为: 简化，归纳为，合并。这样更好理解。

reduce 接受一个初始值，和一个操作符号。
reduce 将给定的初始值，与序列里的每个值进行累计运算。得到一个最终结果，并将其作为单个值发送出去

```
Observable.of(1, 2, 3, 4, 5)
            .reduce(0, accumulator: +)
            .subscribe(onNext: { print($0) })
            .disposed(by: disposeBag)
```
输出：

```
15
```

#### concat

concat 会把多个 Observable 序列合并（串联）为一个 Observable 序列。
 并且只有当前面一个 Observable 序列发出了 completed 事件，才会开始发送下一个  Observable 序列事件。
 
```
let subject1 = BehaviorSubject(value: "A")
let subject2 = BehaviorSubject(value: "1")
 subject1.concat(subject2)
    .subscribe(onNext: { (value) in
        print(value)
    }).disposed(by: disposeBag)
    
subject2.onNext("2")
subject1.onNext("B")
subject1.onNext("C")
subject1.onCompleted()
subject2.onNext("3")
```
输出：

```
A
B
C
2
3
```

subject2 的输出始终在subject1  的后面
