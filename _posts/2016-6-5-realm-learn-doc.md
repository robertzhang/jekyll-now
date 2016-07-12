---
layout: post
title: Realm 杂谈
---

Realm 在五月终于发布了1.0正式版。不知道是不是见证了一个历史性的时刻，毕竟 Realm 是号称用来取代 SQLite 而诞生的。话不多说，下面就来揭开她的面纱，说一说我是怎么和她愉快的玩耍。

故事的背景要说到，最近开始的一个新项目。项目选定使用 Swift 开发，所以需要在前期的时候制定相关的技术方案。数据存储是老生常谈的模块，IOS 开发无非 SQLIte 和 Core Data 可选。Realm 在之前就有听说过，但一直没有静下心来研究。这次经过一番研究后，觉得甚是妙哉。再考虑到她对 Android 平台也有不俗的表现，以后业务逻辑方便迁移。故最终选定 Realm。

先附上 Realm Swift 中文文档地址：https://realm.io/cn/docs/swift/latest/ 内容相当详细。当然这都是基础用法，更高级的用法还是参考官方例子吧。以下是我封装的 RealmHelper 类：

~~~
class RealmHelper {

    /// realm 数据库的名称
	private let username = "MY-DB"
	
    static let sharedInstance = try! Realm()
    
//--MARK: 初始化 Realm
    /// 初始化进过加密的 Realm， 加密过的 Realm 只会带来很少的额外资源占用（通常最多只会比平常慢10%）
    static func initEncryptionRealm() {
        // 说明： 以下内容是可以合并操作的，但为了能最大限度的展示各个操作内容，故分开设置 Realm
        
        // 产生随机密钥
        let key = NSMutableData(length: 64)!
        SecRandomCopyBytes(kSecRandomDefault, key.length,
                           UnsafeMutablePointer<UInt8>(key.mutableBytes))
        
        // 获取加密 Realm 文件的配置文件
        var config = Realm.Configuration(encryptionKey: key)
        
        // 使用默认的目录，但是使用用户名来替换默认的文件名
        config.fileURL = config.fileURL!.URLByDeletingLastPathComponent?
            .URLByAppendingPathComponent("\(username).realm")
        
        // 获取我们的 Realm 文件的父级目录
        let folderPath = config.fileURL!.URLByDeletingLastPathComponent!.path!
        
        /**
         *  设置可以在后台应用刷新中使用 Realm
         *  注意：以下的操作其实是关闭了 Realm 文件的 NSFileProtection 属性加密功能，将文件保护属性降级为一个不太严格的、允许即使在设备锁定时都可以访问文件的属性
         */
        // 解除这个目录的保护
        try! NSFileManager.defaultManager().setAttributes([NSFileProtectionKey: NSFileProtectionNone],
                                                          ofItemAtPath: folderPath)
        
        // 将这个配置应用到默认的 Realm 数据库当中
        Realm.Configuration.defaultConfiguration = config
        
    }
    
    /// 初始化默认的 Realm
    static func initRealm() {
        var config = Realm.Configuration()
        
        // 使用默认的目录，但是使用用户名来替换默认的文件名
        config.fileURL = config.fileURL!.URLByDeletingLastPathComponent?
            .URLByAppendingPathComponent("\(username).realm")
        
        // 获取我们的 Realm 文件的父级目录
        let folderPath = config.fileURL!.URLByDeletingLastPathComponent!.path!
        
        // 解除这个目录的保护
        try! NSFileManager.defaultManager().setAttributes([NSFileProtectionKey: NSFileProtectionNone],
                                                          ofItemAtPath: folderPath)
        
        // 将这个配置应用到默认的 Realm 数据库当中
        Realm.Configuration.defaultConfiguration = config
    }

//--- MARK: 操作 Realm
    /// 做写入操作
    static func doWriteHandler(clouse: ()->()) { // 这里用到了 Trailing 闭包
        try! sharedInstance.write {
            clouse()
        }
    }
    
    /// 添加一条数据
    static func addCanUpdate<T: Object>(object: T) {
        try! sharedInstance.write {
            sharedInstance.add(object, update: true)
        }
    }
    static func add<T: Object>(object: T) {
        try! sharedInstance.write {
            sharedInstance.add(object)
        }
    }
    /// 后台单独进程写入一组数据
    static func addListDataAsync<T: Object>(objects: [T]) {
        let queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0)
        // Import many items in a background thread
        dispatch_async(queue) {
            // 为什么添加下面的关键字，参见 Realm 文件删除的的注释
            autoreleasepool {
                // 在这个线程中获取 Realm 和表实例
                let realm = try! Realm()
                // 批量写入操作
                realm.beginWrite()
                // add 方法支持 update ，item 的对象必须有主键
                for item in objects {
                    realm.add(item, update: true)
                }
                // 提交写入事务以确保数据在其他线程可用
                try! realm.commitWrite()
            }
        }
    }
    
    static func addListData<T: Object>(objects: [T]) {
        autoreleasepool {
            // 在这个线程中获取 Realm 和表实例
            let realm = try! Realm()
            // 批量写入操作
            realm.beginWrite()
            // add 方法支持 update ，item 的对象必须有主键
            for item in objects {
                realm.add(item, update: true)
            }
            // 提交写入事务以确保数据在其他线程可用
            try! realm.commitWrite()
        }
    }
    
    /// 删除某个数据
    static func delete<T: Object>(object: T) {
        try! sharedInstance.write {
            sharedInstance.delete(object)
        }
    }
    
    /// 批量删除数据
    static func delete<T: Object>(objects: [T]) {
        try! sharedInstance.write {
            sharedInstance.delete(objects)
        }
    }
    /// 批量删除数据
    static func delete<T: Object>(objects: List<T>) {
        try! sharedInstance.write {
            sharedInstance.delete(objects)
        }
    }
    /// 批量删除数据
    static func delete<T: Object>(objects: Results<T>) {
        try! sharedInstance.write {
            sharedInstance.delete(objects)
        }
    }
    
    /// 批量删除数据
    static func delete<T: Object>(objects: LinkingObjects<T>) {
        try! sharedInstance.write {
            sharedInstance.delete(objects)
        }
    }
    
    
    /// 删除所有数据。注意，Realm 文件的大小不会被改变，因为它会保留空间以供日后快速存储数据
    static func deleteAll() {
        try! sharedInstance.write {
            sharedInstance.deleteAll()
        }
    }
    
    /// 根据条件查询数据
    static func selectByNSPredicate<T: Object>(_: T.Type , predicate: NSPredicate) -> Results<T>{
        return sharedInstance.objects(T).filter(predicate)
    }
    
//--- MARK: 删除 Realm
    /*
     参考官方文档，所有 fileURL 指向想要删除的 Realm 文件的 Realm 实例，都必须要在删除操作执行前被释放掉。
     故在操作 Realm实例的时候需要加上 autoleasepool 。如下:
        autoreleasepool {
            //所有 Realm 的使用操作
        }
     */
    /// Realm 文件删除操作
    static func deleteRealmFile() {
        let realmURL = Realm.Configuration.defaultConfiguration.fileURL!
        let realmURLs = [
            realmURL,
            realmURL.URLByAppendingPathExtension("lock"),
            realmURL.URLByAppendingPathExtension("log_a"),
            realmURL.URLByAppendingPathExtension("log_b"),
            realmURL.URLByAppendingPathExtension("note")
        ]
        let manager = NSFileManager.defaultManager()
        for URL in realmURLs {
            do {
                try manager.removeItemAtURL(URL)
            } catch {
                // 处理错误
            }
        }
        
    }
}
~~~
注释写的很清楚，主要包括初始化 Realm 和其相关的操作。 所以就不赘述封装类，而是讲解一些 Realm 使用中我觉得有意思的地方。

## 有趣的 Realm
首先来一个例子：

~~~
class Dog: Object {
	let friends = List<Dog>
	let owners = LinkingObjects(fromType: Person.self, property: "dogs")
}
class Person: Object {
	let dogs = List<Dog>()
	override static func primaryKey() -> String? {
    return "id"
  }
}
~~~

* 反向关系 -- 关键字 LinkingObjects。在 M-N 的关系中，我们虽然可以在两个数据模型中都使用 List<T> 来双向绑定。但由于手动同步关系会很容易出错，并且还会让内容变得复杂、冗余。反向关系就解决了这个问题。
* 内容更新 -- add VS create。虽然他们都有更新的作用，既如果主键 id 为1的 Person 对象已经存在于数据库当中了，那么对象就会简单地进行更新。而如果不在数据库中存在的话，那么这个操作将会创建一个新的 Person 对象并添加到数据库当中。那该怎么选择他们呢？create 通过传递想要更新值的集合，从而更新带有主键的某个对象的部分值；add 更新的是对象，在更新的时候，向上的关系会保留，因为向上的关系是通过主键id联系在一起的。向下的关系，既对象内部List属性，如果不重新设置，之前的关系会丢失。这里需要十分小心。
* 内容删除。会删除向上的关系。但不会将向下关系的对象一起被清除。所以，在删除的时候，需要考虑其向下的关系是否还有存在的意义，如果没有，应一并删除其向下关系的对象。例如删除一个 Dog 对象，会清除该对象向上的 owners 关系，但不会清除 friends 对象。如果其 friends 对象在该 Dog 对象被删除后，就没有存在的必要。则应该一并删除 friends 连接的 Dog 对象。
* 通知的理解。官方文档的例子很简单也很清楚。在此我只说说不同的见解。通过 addNotificationBlock 可以轻松的监察某个 Results<T> 实例的变化。这样就可以监察某一个特定的查询结果（Realm自动更新的原则）。并对这个查询结果做出相应的处理。我们完全可以通过这种方式将每一个 view 都绑定上 Results，这样就看起十分像 Android DataBinding。但这是不是一个好的设计方案呢？我想 Realm 不一定会这样推荐使用，即便她的查询效率很高。Realm 本事是支持自动更新的，当其他地方更新数据后，之前的对象也会被动态更新。通知只是告知 UI 在什么时候响应更新，所有的修改 Realm 都为我们做完了。
* 通过 Person 查看 dogs 和 Dog 的 friends 非常简单，就像点出来一个 public 方法和属性一样简单。当然通过 Dog 的 owners 也可以向上追溯 Person。建议在建好数据模型后，打印出整个关系树来查看。会对理解 Realm 的模型关系有很大帮助。

## 总结

Realm 是一个轻量级的关系型数据库。她和 SQLite 比起来还要轻。SQLite 需要通过 SQL 语句来操作数据库，虽然也可以通过一些第三方控件封装，使得上层不再关心 SQL 语句的编写。但对比起 Realm 天生已经将这繁琐的工作帮我们封装过，只给我提供较为直观的关系树（关系树是我根据打印出来的数据，自己理解的出来的一个词），让使用方法更符合上层逻辑的编写习惯。我相信如果用的得当，她会大大减轻我们对数据库开发的成本。
## 说明
本着分享加记录学习心得的严谨态度，最后还是要声明一下。以上内容皆为阅读官方文档和项目开发中对 Realm 的理解和感悟，不一定是正确的。如果有误导的地方，请联系我。我会在第一时间修正。当然，它也会帮助你从不同的角度理解 Realm，分享的乐趣就是让更多的人看到你的态度。

