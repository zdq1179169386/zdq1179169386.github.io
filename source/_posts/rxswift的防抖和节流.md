---
title: RxSwift 的函数防抖和函数节流
date: 2020-06-18 18:17:51
tags: [RxSwift]

---


当我们在开发的时候，经常会碰到这种情况，比如用户多次点击了按钮，从而进行了多次网络请求，或者在搜索的时候，用户每输入一个字符，就进行了一次网络请求，而实际上我们需要的是，用户输入了一个完整的词之后，我们才去请求。函数防抖和节流就是为了优化这些情况的。其实函数防抖和函数节流是前端提出来的概念，具体请看这篇文章
[函数节流与函数防抖](https://juejin.im/entry/58c0379e44d9040068dc952f) 。

<!--more-->

### 函数防抖(debounce)
简单的理解，就是用户手抖了，多点击几下按钮，从而进行了多次网络请求，或者其他的数据操作。如何防止这种情况，就是函数防抖。

正常情况下多次点击按钮，就会出现下面的情况
![](/images/btnclick.gif)

如果按钮里需要进行网络请求，也会进行多次，这显然是不好的，这是我们就可以用`RxSwift` 的 `debounce` 操作符来修改

```
button.rx
    .tap
    .asDriver()
    .debounce(1)
    .drive(onNext: { (_) in
        print("tap")
    }).disposed(by: disposeBag)
```
效果图：

![](/images/debounce.gif)

利用 `debounce ` 操作符，就能够有效的解决这种情况。

### 函数节流(throttle) 
通常在搜索页面，都会出现频繁触发搜索请求的现象，

![](/images/search.gif)

上次搜索结果还没回来，又开始下次请求了， 这样很浪费资源，我们可以利用`RxSwift` 提供的 `throttle` 操作符，来修改

```
searchBar.rx.text
    .orEmpty
    .throttle(RxTimeInterval.seconds(1), scheduler: MainScheduler.instance)
    .distinctUntilChanged()
    .filter{ !$0.isEmpty}
    .subscribe(onNext: { (value) in
        print("request")
    }).disposed(by: disposeBag)
```
效果图: 

![](/images/throttle.gif)

在用户输入的时候，不是一直不停的去请求，而是隔个1秒，再去请求，可以有效的节省资源

### 总结
* 函数节流: 指定时间间隔内只会执行一次任务；

* 函数防抖: 任务频繁触发的情况下，只有任务触发的间隔超过指定间隔的时候，任务才会执行。

使用函数防抖和函数节流都是为了优化用户体验和节省计算机资源。