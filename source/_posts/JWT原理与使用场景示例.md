---
title: JWT原理与使用场景示例
tags:
  - 安全
categories:
  - blog
date: 2020-09-22 15:09:15
---

JWT全称 JSON Web Token，是一种紧凑的，URL安全的请求表示方式(定义见[RFC7519](https://tools.ietf.org/html/rfc7519))。

<!--more-->

## JWT的组成原理

一组JWT包含三个部分：HEADER，PAYLOAD和SIGNATURE，之间由`.`分割，如：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

在讲这三个部分如何生成之前，有三个概念需要明确：编码、签名、加密：

#### 编码

编码是指将内容通过一定的算法将内容转换为更方便传输的形式。我们在开发中较为常见的是Base编码。编码的特征是可以进行解码，**也就是说经过编码的数据任何人都可以通过相同的算法进行解码并获取到明文**。所以所谓“Base64加密”等说法都是不准确的。

#### 签名

签名是通过哈希算法生成特征码以防止内容信息被篡改，在我们日常开发中很常见到，如MD5，SHA等等。我们知道哈希算法的特点是正向快，逆向困难。所以**签名通常是不可逆的**。通过核对签名是否一致就可基本判断出原始内容是否有被修改。

#### 加密

说完了前两种常被误称为“加密”的算法，再说说什么才是加密。严格来说，加密是通过特定算法，配合密钥使内容变为不可读的密文。之后密文可通过特定的方法配合密钥解密为明文。加密可分为对称加密（如AES）与非对称加密（RSA）。

#### JWT的原理

说完了3个概念，我们来看看JWT的生成原理。从JWT的定义中可知，JWT的三部分组成来源为：

```javascript
const sign = HMACSHA256(base64.encode(header) + '.' + base64.encode(payload), secret)
const jwt = base64.encode(header) + '.' + base64.encode(payload) + '.' + sign
```

其中JWT的HEADER与PAYLOAD为明文的BASE64**编码**。SIGNATURE为前两者拼接后配合密钥生成的**签名**。这个过程中不涉及加密，我们可以轻松的通过JWT解码得到HEADER与PAYLOAD的明文。故我们不应把JWT当作加密手段来传输敏感信息。而SIGNATURE为前两者信息的签名，由于签名是不可逆的，我们不可能通过签名得到HEADER与PAYLOAD，只能用SIGNATURE与密钥来校验HEADER与PAYLOAD是否被篡改。

#### JWT的用途

知道了JWT的用途，那他的用途也就很明确了。由于第三方不能再没有密钥的情况下篡改HEADER与PAYLOAD，那么JWT可以作为存储用户信息的TOKEN在用户鉴权时颁发给用户。之后用户可以在请求中带上该TOKEN给服务端来验明正身。

## JWT的使用场景

#### 发送给用户的重置密码与激活账户等链接

在通过邮箱重置密码时，应用通常会发给我们一个地址，如`http://jiangyixiong.top/reset_password?uid=xxxx`，在这里如果用户的身份id以明文放在链接中，用户就很可能可以通过修改地址中的uid参数来越权修改他人账户信息。在这里我们可以以uid等信息作为PAYLOAD，服务端以密钥生成JWT，并放在地址参数中。用户虽然依旧可以通过Base64解码获得PAYLOAD中的参数，但无法修改。因为一旦参数被修改，将无法通过服务端的签名校验。由于客户端没有生成JWT的密钥，也无法伪造新的签名。

#### 作为REST API的鉴权TOKEN

在需要权限控制的REST API中，可以要求调用方在调用时提供鉴权时颁发的JWT，该JWT的PAYLOAD中会包含调用方的角色信息，权限信息，该JWT的过期时间等。服务端接收到请求后可使用密钥校验Token的有效性以确定请求发起方的身份。

## References

> https://jishuin.proginn.com/p/763bfbd2d4b6
>
> https://blog.csdn.net/qq_28165595/article/details/80214994
>
> https://blog.csdn.net/achenyuan/article/details/80829401