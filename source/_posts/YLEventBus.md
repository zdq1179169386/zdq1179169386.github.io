---
title: Swift 实现一个简单的事件订阅/发布框架
date: 2021-03-09 21:59:24
tags: [swift]

---

利用 `NSHashTable`，实现一个简单的事件订阅/发布框架 

<!--more-->

### 一，YLEventBus 实现

先贴源码

```
class YLEventBus {
    
    /// 事件map
    fileprivate static var eventsMap = [String: NSHashTable<AnyObject>]()
    
    /// 加锁
    fileprivate static let notificationQueue = DispatchQueue(label: "com.swift.notification.center.dispatch.queue", attributes: .concurrent)
    
    /// 注册
    public static func subscribe<T>(_ proto: T.Type, observer: T) {
        let key = keyName("\(proto)")
        safeAdd(key: key, object: observer as AnyObject)
    }
    
    /// 解除注册,可以不用主动调用，不会对 observer 强引用
    public static func dispose<T>(_ proto: T.Type, observer: T) {
        let key = keyName("\(proto)")
        safeRemove(key: key, object: observer as AnyObject)
    }
    
    /// 解除注册
    public static func dispose<T>(_ proto: T.Type) {
        let key = keyName("\(proto)")
        safeRemove(key: key)
    }
    
    /// 通知
    public static func publish<T>(_ proto: T.Type, block: (T) -> Void ) {
        let key = keyName("\(proto)")
        guard let map = safeGetObjectSet(key: key) else {
            return
        }
        
        map.allObjects.compactMap({ $0 as? T}).forEach { obj in
            block(obj)
        }
    }
}

///线程安全
private extension YLEventBus{
    static func safeAdd(key: String, object: AnyObject) {
        notificationQueue.async(flags: .barrier) {
            if let set = eventsMap[key] {
                set.add(object)
                eventsMap[key] = set
            }else{
                let set = NSHashTable<AnyObject>.weakObjects()
                set.add(object as AnyObject)
                eventsMap[key] = set
            }
        }
    }
    
    static func safeRemove(key: String, object: AnyObject) {
        notificationQueue.async(flags: .barrier) {
            if let set = eventsMap[key] {
                set.remove(object)
                if set.count <= 0 {
                    eventsMap.removeValue(forKey: key)
                } else {
                    eventsMap[key] = set
                }
            }
        }
    }
    
    static func safeRemove(key: String) {
        notificationQueue.async(flags: .barrier) {
            eventsMap.removeValue(forKey: key)
        }
    }
    
    static func safeGetObjectSet(key: String) -> NSHashTable<AnyObject>? {
        var objectSet: NSHashTable<AnyObject>?
        notificationQueue.sync {
            objectSet = eventsMap[key]
        }
        return objectSet
    }
    
    static func keyName(_ key: String) -> String{
        return "yl_" + key
    }
}
```


1，YLEventBus 持有一个 `[String: NSHashTable<AnyObject>]` 的静态变量 `eventsMap` ，字典的 key  是我们定义的 `protocol `类型字符串 + 前缀，作为 key , value 是  `NSHashTable` 类型的 table, 主要是 `NSHashTable ` 可以对其持有元素设置弱引用。

2，订阅调用 `public static func subscribe<T>(_ proto: T.Type, observer: T)` 方法,  会往字典里存订阅者 observer，如果对应 key 的 value 为空，会先创建一个新的 table ，将 observer 添加进去，然后存起来

3，发布调用 `public static func publish<T>(_ proto: T.Type, block: (T) -> Void )` 方法，先根据 key 取出 table, 然后 for 循环，调用 block，将 observer 作为参数传进去， 这样外面就能拿到 observer ，并调用相应的协议方法


### 三， 使用 `YLEventBus`
先定义一个协议 

```
protocol TestDelegate{
    func send(_ text: String)
}
```

然后在 `ViewController` 中订阅并实现这个协议

```
class ViewController: UIViewController, TestDelegate {

    override func viewDidLoad() {
        super.viewDidLoad()
        navigationController?.view.backgroundColor = .white
        title = "ViewController"
        
        YLEventBus.subscribe(TestDelegate.self, observer: self)
    }


    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        navigationController?.pushViewController(SecondViewController(), animated: true)
    }
    
    func send(_ text: String) {
        print("ViewController: \(text)")
    }
}
```

在 `SecondViewController` 中也订阅这个协议

```
class SecondViewController: UIViewController, TestDelegate {
    func send(_ text: String) {
        print("SecondViewController: \(text)")
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        title = "SecondViewController"
        YLEventBus.subscribe(TestDelegate.self, observer: self)
    }
    
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        navigationController?.pushViewController(ThirdViewController(), animated: true)
    }
    
    deinit{
        print("SecondViewController deinit")
        YLEventBus.dispose(TestDelegate.self, observer: self)
        YLEventBus.dispose(TestDelegate.self)
    }
}
```

最后在 `ThirdViewController ` 点击 touch 方法， 发布


```
class ThirdViewController: UIViewController {
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        title = "ThirdViewController"
    }
    
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        YLEventBus.publish(TestDelegate.self) { obj in
            obj.send("111")
        }
    }
    
    deinit{
        print("ThirdViewController deinit")
    }
}
```

可以看到  ViewController 和 SecondViewController 都收到协议方法的调用

```
ViewController: 111
SecondViewController: 111
ThirdViewController deinit
SecondViewController deinit
```

并且在返回  ViewController 的时候， SecondViewController 和 ThirdViewController 都被释放掉了，说明 YLEventBus 不会强引用订阅者

注意： 在 SecondViewController 的 deinit 方法中调用了 `YLEventBus.dispose(TestDelegate.self, observer: self)` (其实可以不用调用) ，也不会被 `over-released`