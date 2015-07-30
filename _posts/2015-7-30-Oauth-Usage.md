---
layout: post
title: OAuth2 Usage
---

Oauth2的使用手册

```
最近需要使用Oauth2.0认证登陆，所以简单的学习了一下相关知识。
这里记录一下学习过程中获取的部分资料。
```

### OAuth相关文档供参考 
  
首先了解一下OAuth 2.0的基本知识  
[理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)
   
深入了解OAuth各个不同组件，这是一个台湾人写的。内容很丰富，建议可以深入学习  
[OAuth笔记](https://blog.yorkxin.org/posts/2013/09/30/oauth2-2-cilent-registration/)

一篇关于ruby & rails使用OAuth2认证API的文章也是上面这个人写的,ruby-china的一篇文章   
[OAuth 2.0 教程: Grape API 整合 Doorkeeper](https://ruby-china.org/topics/14656)
   
OAuth 2.0英文文档，相当详细  
[The OAuth 2.0 Authorization Framework](http://tools.ietf.org/html/rfc6749#section-1.3)



### 目前NET263使用的OAuth

#### *OAuth常见的认证流程

* Authorization Code Grant Type Flow —— 认证码验证流程
  
* Implicit Grant Type Flow —— 简单认证流程
  
* Resource Owner Password Credentials Grant Type Flow —— 密码认证流程
  
* Client Credentials Grant Type Flow —— 客户端认证流程

