---
layout: post
title: RESTful API 优化设计（Swift 版）
subtitle:   "API 优化设计"
date:       2018-09-30
author:     "Robert Zhang"
header-img: "img/post-bg-rwd.jpg"
tags:
    - IOS
---

> 关键词：iOS  
开发语言：Swift  
使用到的库：Alamofire，SwiftyJSON，ObjectMapper
# 背景
最近重构 API 相关代码，参考了 RocketChat.iOS 项目，获益匪浅故分享记录一下。RocketChat 对 API 的编写思路很有意思，这里只是对相关设计做了简单的整理。如果对此感兴趣，可以直接查看其开源项目。（文末附有地址）
# API 设计
API 设计属于网咯层开发十分重要的一部分。API 设计是否合理，直接影响数据层的逻辑。接下来我将从 RESTful API 来分析和讲解优化方案。

### 整体结构
先来看一下目录结构

![api_design.jpeg]({{ site.baseurl }}/img/api_design.jpeg)

主要的文件是 API、APIRequest、APIResource、APIParameters ，我们先从依赖关系最少的文件开始看起。
#### APIRequest
~~~ swift
import Alamofire

protocol APIRequest {
  associatedtype APIResourceType: APIResource 
  
  var path: String { get }
  var method: HTTPMethod { get }
  var parameters: Parameters? { get }
}
~~~
APIRequest 是一个协议，用来提供请求所需的漏油，HTTP请求类型，表单参数，以及返回数据的 APIResourceType。在后文 APIRequest 扩展中将详细说明各个属性的作用。在这里只需要知道，APIRequest 是我们管理请求所需参数的地方
#### APIResource
APIResource 在上面已经出现过，但没有讲她的具体作用。通过阅读代码发现 APIResource 仅存放了一个 JSON(SwiftyJSON 的基本类型) 类型的 raw。那我们就先认为 APIResource 就是 API 请求返回的 json 数据好了。
~~~ swift
import SwiftyJSON

class APIResource {
  let raw: JSON?

  required init(raw: JSON?) {
    self.raw = raw
  }
}

// 基本扩展
/**
API 请求返回的最基本数据结构：
{
  status: "error",
  message: "xxxx"    // 错误原因 
}
*/
extension APIResource {
  enum Status: String {
    case error
    case success
  }
  var status: Status? {
    return Status.init(rawValue: (raw?["status"].string)!)
  }
  
  var message: String? {
    return raw?["message"].string
  }
}
~~~
基本扩展新增了只读属性 status 和 message。在这里你可以根据自身的情况设计此模型
#### APIParameters
APIParameters 协议只有一个方法，为 Alamofire 提供 Parameters。我们也可以在子类中存放请求所需的表单参数，下文也会对其扩展做说明。
~~~ swift
import Alamofire

protocol APIParameters {
  func toParameters() -> Parameters?
}

// 基本扩展
extension APIParameters {
  func toParameters() -> Parameters? {
    let paramDic = ModelToDic(model: self as AnyObject)
    return paramDic as? Parameters
  }
  
  /**
   对象转换为字典
   - parameter model: 对象
   - returns: 转化出来的字典
   */
  private func ModelToDic(model:AnyObject) -> NSDictionary{
    let redic = NSMutableDictionary()
    
    let mirror: Mirror = Mirror(reflecting: model)
    
    for p in mirror.children{
      if(p.label! != ""){
        redic.setValue("\(p.value)", forKey: p.label!)
      }
    }
    
    return redic
  }
}
~~~
上面三个类都是为最后的重头戏 API 文件服务，话不多说一起来看看她吧
#### API
~~~swift
import Alamofire
import SwiftyJSON

typealias APIFetchCompletionBlock<R: APIRequest> = (R.APIResourceType) -> Void

final class API {
  
  let host: String // http 接口地址
  var authToken: String? // 根据需要添加
  var userId: String? // 根据需要添加
  
  init(host: String) {
    self.host = host
  }
  
  func fetch<R: APIRequest>(_ request: R, completion: (APIFetchCompletionBlock<R>)? = nil) {
    // 根据需要自行取舍
    let headers: HTTPHeaders = [
      "X-Auth-Token" : authToken!,
      "X-User-Id": userId!
    ]

    // 拼接 APIRequest 属性， 使用  Alamofire 请求 HTTP
    Alamofire.request(host + request.path, method: request.method,parameters: request.parameters, headers: headers).responseJSON { (response) in
      guard let jsonStr = response.result.value else {
        #if DEBUG
        print("[API.fetch][\(request.path)]: No data.")
        #endif
        
        return
      }
      
      #if DEBUG
      print("[API.fetch][request: \(response.request)] [params: \(request.parameters)]]")
      print("[respons: \(response.result.value)")
      #endif
      
      let json = JSON(jsonStr) // 将返回的 json 字符串转换成 JSON 对象
      //如果 Alamofire.request 没有在主线程中使用，下面需要切换到主线程调用block
      completion?(R.APIResourceType(raw: json)) // 使用APIRequest 中设置的 APIResourceType 来创建 APIResource 对象
    }
  }
}

extension API {
  // 初始化 API 对象
  static func current() -> API? {
    let api = API(host: "\(MeteorConfig.domain)/api/v1")
    
    api.authToken = MeteorWrapper.shared.getMeteorToken()
    api.userId = MeteorWrapper.shared.getUserId()
    
    return api
  }
}
~~~
总的来说，上面的代码难度不大。值得一说到就是 __R.APIResourceType(raw: json)__ 的使用。R 为 APIRequest 的实现，APIResourceType 为 R 对应的关联类型。听起来有点绕，我们重新组织一下语言。  
__R.APIResourceType(raw: json)__  ：创建 APIRequest 子类 R 关联类型 APIResourceType 的实例，该实例需要传入 json 字符串。  
下面分别给出 APIRequest、APIResource、APIParameter 的扩展，方便理解。
### APIRequest 扩展
我们实现一个处理会话的 ThreadRequest
~~~swift
import Alamofire
import RealmSwift

class ThreadRequest: APIRequest {
  // ThreadRequest 可以处理的事件
  enum Action {
    case addMembers(threadId: String, members: [String])
    case loadMissed(time: String, limit: Int?)
    case delete(threadId: String)
    case update(threadId: String, params: UpdateThreadInfoParameters)
  }
  
  /**  
    API 中的 R.APIResourceType(raw: json) 实际调用的下面这个类型。
    所以我们在实现 APIRequest 的时候需要对 APIResourceType 参数做相应的修改
  */
  typealias APIResourceType = ThreadResource
  
  var path: String
  
  var method: HTTPMethod
  
  var parameters: Parameters?
  
  init(action: Action) {
    switch action {      
    case .delete(let threadId):
      self.method = .delete
      self.path = "/threads/\(threadId)"
      
    case .update(let threadId, let params):
      self.method = .put
      self.path = "/threads/\(threadId)"
      self.parameters = params.toParameters()
      
    case .addMembers(let threadId, let members):
      self.method = .put
      self.path = "/threads/\(threadId)/members/\(members.joined(separator: ","))"

    case .loadMissed(let time, let limit):
      self.method = .get
      
      let paramLimit = limit != nil ? "&limit=\(limit!)" : ""
      
      self.path = "/threads/load_missed?time=\(time)\(paramLimit)"
    }
  }
  
  class ThreadResource: APIResource {
  // 根据你的需要解析 APIResource.raw 的 json 字符串，推荐尽量使用只读属性，统一结构
  }
}
~~~

### APIParameters 扩展
~~~swift
import Alamofire

class UpdateThreadInfoParameters: APIParameters {
  // 需要传入的参数
  var subject: String?
  var announcement: String?
  
  init(subject: String? = nil, announcement: String? = nil) {
    self.subject = subject
    self.announcement = announcement
  }
  
/** 
Alamofire 需要的 Parameters 是字典类型，所以这里需要将 APIParameters 子类的所有属性转换成 Parameters 。  
细心的你一定观察到，APIParameters 的基本扩展中已经实现了 toParameters() 方法。
这里复写是为本实现做了定制
*/
  func toParameters() -> Parameters? {
    if subject != nil, announcement != nil {
      return ["subject": subject!, "announcement": announcement!]
      
    } else if subject != nil, announcement == nil {
      return ["subject": subject!]
      
    } else if subject == nil, announcement != nil {
      return ["announcement": announcement!]
      
    } else {
      return nil
    }
  }
}
~~~
# 思考
以上 API 的优化设计就做完了。是不是觉得十分清爽，扩展新接口的成本几乎可以忽略。
回顾整个优化过程，APIRequest、APIResource、APIParameter 的抽离。使得API 接口组件化，我们可以根据需要定制 APIRequest，并将其传递给 API.current().fetch(::)。
# 参考
[RocketChat.iOS](https://github.com/RocketChat/Rocket.Chat.iOS)