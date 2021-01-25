---
title: Shiro使用之自定义realm
date: 2020-08-03 15:09:52
tags:
- Shiro
categories: 
- blog
---

## 什么是Realm

Realm直译为王国或领地，再Shiro中，realm负责链接认证授权服务与其所使用的数据源。

如果说Shiro是小区保安，负责筛查所有进出小区的车辆，realm就是保安手上的住户名单，负责记录每个用户的信息，包含`Principals`（用户识别信息，通常是用户名或邮箱手机号），`Credentials`（用户证明信息，通常就是密码），和`Authorization`（用户身份）。

<!--more-->
## 如何自定义Realm

Shiro提供了很多Realm实现，其中常用的有`IniRealm`，`PropertiesRealm`，`JdbcRealm`。通常在项目中用户信息是存储于数据库中，对应可使用`JdbcRealm`。但`JdbcRealm`的使用中需要我们定义身份与用户信息的查询语句，这些内容通常是我们在实现DAO和Service层时就已经做过的。加上有事在获取用户信息的过程中会有一些特殊的业务逻辑，我们通常会通过继承`AuthorizingRealm`来实现自定义Realm。

`AuthorizingRealm`是一个抽象类，提供了`doGetAuthorizationInfo`抽象方法，同时通过继承`AuthenticatingRealm`提供`doGetAuthenticationInfo`。我们在继承`AuthorizingRealm`时需要实现这两个抽象方法。

`doGetAuthenticationInfo`方法用于获取系统存储的用户认证信息，它接收一个用户的Token，返回该用户的认证信息`AuthenticationInfo`。

`doGetAuthorizationInfo`方法用于获取系统所存储的用户身份信息，即用户角色，它接收用户的Principals信息，返回用户的身份信息`AuthorizationInfo`。

## 具体实现

```java
public class MyRealm extends AuthorizingRealm {

    // 注入用户Service
    @Resource
    AdminUserService aus;

    //实现doGetAuthorizationInfo方法，通过用户principals获取用户身份并返回
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
		//由于用户可以有多个Principals这里需要获取到用户的主要Principal
        String principal = (String) principalCollection.getPrimaryPrincipal();
        //通过用户Service获取用户domain对象，本例中用户principal为email
        AdminUser au = aus.selectByEmail(principal);
        //如用户不存在则直接返回null
        if(au==null){
            return null;
        }
        //用户可以有多个身份，存放于一个Set中
        Set<String> authSet = new HashSet<>();
        authSet.add(au.getAuth());
        //通过Set创建用户身份信息
        return new SimpleAuthorizationInfo(authSet);
    }

    //实现AuthenticationInfo方法，通过用户principal信息获取用户认证信息并返回
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        //从登录Token中取得用户的Principal
        String principal = (String) authenticationToken.getPrincipal();
        //通过Principal从用户Service中获取用户domain对象
        AdminUser au = aus.selectByEmail(principal);
        //如用户不存在则直接返回null
        if(au==null){
            return null;
        }
        //创建用户认证信息并返回，这里使用SimpleAuthenticationInfo实现，SimpleAuthenticationInfo提供了多种构造方式。
        //这里使用的是hash+salt加密的认证信息构造方法 SimpleAuthenticationInfo(Object principal, Object hashedCredentials, ByteSource credentialsSalt, String realmName)
        //若不使用hash加密，则可直接使用 SimpleAuthenticationInfo(Object principal, Object credentials, String realmName)
        return new SimpleAuthenticationInfo(au.getEmail(),au.getPassword(), ByteSource.Util.bytes(au.getSalt()),this.getName());
    }
}

```

