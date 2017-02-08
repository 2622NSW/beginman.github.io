---
layout: post
title: "python RESTful浅知"
category: "Python"
tags: [python]
---

# 一.RESTful概念

>REST (REpresentational State Transfer, 表现层状态转换) 已经变成了 web services 和 web APIs 的标配。它有下面几个核心：

1. 资源：一个URI映射一个资源，如图片、音频、视频、文档、文件等。
2. 表现层：资源是一种实体，实体该如何表示？资源的表现形式就是表现层，如文档资源常用JSON/XML等表示。URI可以确定一个资源，但是如何确定它的表现形式呢？通过HTTP的ACCEPT和Content-Type字段来确定。
3. 状态转换：因为HTTP协议是无状态的,这里所谓的状态是由HTTP方法和服务器配合完成。

![](http://beginman.qiniudn.com/2017-02-08-14864405658941.jpg)

REST 设计不需要特定的数据格式。在请求中数据可以以 JSON 形式, 或者有时候作为 url 中查询参数项。

**API 规范：**

```bash
http://[hostname]/name/api/version/ 
# 如
http://[hostname]/todo/api/v1.0/
```

![](http://beginman.qiniudn.com/2017-02-08-14864407886892.jpg)

实例：[gist](https://gist.github.com/BeginMan/a31c9f51dc3547774dcde01d87a81323)

- v1:flask实现简单的rest
- v2:Flask-RESTful 设计 RESTful API
- v3:Flask 设计简单的基于RESTful 的认证

# 二.身份认证

然而 REST 的规则之一就是 “无状态”， 因此我们必须要求客户端在每一次请求中提供认证的信息，目前我总结的有如下方案：

1. Http Basic Authentication
2. Access Token
3. Api Key + Security Key + Sign
4. JWT
5. OAuth

## 2.1 Http Basic Authentication

最简单也是最不安全的一种，就是通过`username:password` 再用Base64编码后的字符串随HTTP首部发出，再由接收者解析校验。

可运行上面的例子，进行如下带认证的访问：

```bash
$ curl -u 用户名:密码 -i http://localhost:5000/todo/api/v1.0/tasks
```

如果缺少用户名及密码，则会出现`401 UNAUTHORIZED`状态，

**这种方案没啥安全性，一解码立马破解掉了.如果不加HTTPS,则相当于裸奔。**

```python
>>>"admin:12345".encode('base64')
'YWRtaW46MTIzNDU=\n'

>>>'YWRtaW46MTIzNDU=\n'.decode('base64')
'admin:12345'
```

## 2.2 Access Token

原理就是客户端登录后返回给它一个Token, 这个Token包含认证信息，同时要设置过期时间，然后每次访问都携带这个Token.服务端进行认证。

这里涉及三个重要因素：

1. Token如何设计才能不易破解和伪造，要包含哪些信息？
2. 过期时间，过长存在盗用风险，过短则重新生成并获取Token次数增多。
3. HTTPS加密

基于此，这套方案相当于穿着裤衩裸奔。相关代码在[auth-token-api.py](https://gist.github.com/BeginMan/a31c9f51dc3547774dcde01d87a81323#file-auth-token-api-py)

## 2.3 Api Key + Security Key + Sign

这个用的比较多。原理如下：

客户端构造请求：

![](http://beginman.qiniudn.com/2017-02-08-14865443118992.jpg)

![](media/14865443435855.jpg)

**Server校验**

![](http://beginman.qiniudn.com/2017-02-08-14865443607473.jpg)

实例代码在：[gist:Api Key + Security Key + Sign](https://gist.github.com/BeginMan/231f479f30244fa35b7718e41b249490)

## 2.4 JWT (JSON Web Token)

[JWT](https://jwt.io/) 由头部、载荷与签名这三部分组成，如下图：

![](http://beginman.qiniudn.com/2017-02-08-14865455976696.jpg)

```python
>>> import jwt
>>> encoded = jwt.encode({'some': 'payload'}, 'secret', algorithm='HS256')
'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzb21lIjoicGF5bG9hZCJ9.4twFt5NiznN84AWoo1d7KO1T_yoc0Z6XOpOVswacPZg'

>>> jwt.decode(encoded, 'secret', algorithms=['HS256'])
{'some': 'payload'}

# ref:https://github.com/jpadilla/pyjwt/
```

# 三.参考

- [《App 后台开发运维和架构实践》](https://book.douban.com/subject/26792337/)
- [使用 Flask 设计 RESTful APIs](http://www.pythondoc.com/flask-restful/index.html)

