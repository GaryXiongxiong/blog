---
title: Shiro通过注解实现REST风格API权限管理
date: 2020-08-04 15:24:39
tags: 
- Shiro
categories: 
- blog
---

最近在写个自己的springboot的前后端分离小项目，项目使用REST风格的API来进行前后端通信。在API使用权限管理上选择了Shiro。REST风格API的特点之一是同一个地址的不同请求方法会产生不同的效果，也需要不同的权限控制。例如对于`/blog/1`这个地址，使用`GET`方法为获取id为1的博客的内容信息，而使用`PATCH`方法为更新id为1的博客的信息。这时就需要对于`GET`和`PATCH`方法设定不同的权限要求。
<!--more-->
## 使用 Shiro Filter 时遇到的问题

在Shiro官方的[参考文档](https://shiro.apache.org/reference.html)中，对于Web应用建议使用Shiro提供的`HttpMethodPermissionFilter`（`rest`）对特定的请求方法进行权限控制。具体来说，我们可以在为角色添加权限时设定指定资源的指定方法。这些方法会被`HttpMethodPermissionFilter`对应到各个http请求方法上。例如我们要控制一角色对于blog资源的修改权限，我们可以为该角色添加以下permission：

```java
info.addStringPermission("blog:edit");
```

其中，blog为指定的资源名称，冒号后为指定的操作方法类型，在Shiro的定义中，edit被对应到了`PATCH`请求方法上。之后，我们就可以通过`HttpMethodPermissionFilter`过滤器来限制对于`/blog/**`地址的权限：

```java
shiroFilterFactoryBean.setFilterChainDefinitions("/blog/**=rest[blog]");
```

如此设置之后，拥有`blog:edit`权限的用户可以向该地址发送`PATCH`请求，拥有`blog:create`权限的的用户可以向该地址发送`POST`请求，拥有`blog:read`方法的用户可以向该地址发送`GET`请求。具体的请求方法与Shiro定义权限的对应如下：

| HTTP Method | Mapped Action | Example Permission | Runtime Check |
| ----------- | ------------- | ------------------ | ------------- |
| head        | read          | perm1              | perm1:read    |
| get         | read          | perm2              | perm2:read    |
| put         | update        | perm3              | perm3:update  |
| post        | create        | perm4              | perm4:create  |
| mkcol       | create        | perm5              | perm5:create  |
| options     | read          | perm6              | perm6:read    |
| trace       | read          | perm7              | perm7:read    |
| patch       | edit          | perm8              | perm8:edit    |

在我尝试使用这种方法配置filter时，发现了一个问题，有些资源我们希望用户可以匿名发送`GET`请求，而需要限制只有管理员才能发送`POST`等其他请用。如果我们使用该`HttpMethodPermissionFilter`，会对于所有请求方法进行鉴权，`GET`请求必须要拥有`read`权限的用户方可发送。由于匿名用户没有角色，无法对其进行授权，其`GET`请求就会被拦截。

为了解决这个问题，我们需要写一个自定义的Filter来放行`GET`方法，仅对其余方法进行权限检查。感觉比较麻烦。在网上查找其他项目的实现时发现Shiro可以基于注解对调用方法进行权限管理（在官方文档里居然没有说，或者是我没有找到），所以想出了一个比较便捷（懒）的实现REST API权限控制的方法

## 基于注解的实现

Shiro提供了以下几个注解来实现方法调用的权限控制：

- `@RequiresAuthentication`仅认证（登录）后的用户可以调用该方法
- `@RequiresGuest` 仅Guest（未认证）用户可以调用该方法，这个注解的作用与`@RequiresAuthentication`完全相反
- `@RequireUser`仅User可以可以调用，这个注解与`@RequiresAuthentication`的区别是，它会允许之前认证过，并使用`RememberMe`功能的用户调用该方法
- `@RequiresRoles`仅拥有指定角色的用户可以调用该方法
- `@RequiresPermissions`仅拥有指定权限的用户可以调用该方法

调用被注解的方法时没有指定的权限，Shiro会抛出一个`AuthorizationException`异常

使用这些注解，配合我们在上一节中对用户的授权，我们就可以在controller中设置每个mapping方法的使用权限啦：

```java
@RestController
@RequestMapping("/blog")
public class BlogController {

    @Resource
    BlogService bs;

    //这里不注解即表示所有人均可调用
    @GetMapping("")
    public RestResult<List<Blog>> selectAll(){
        return RestResult.success(bs.selectAll());
    }

    //同上
    @GetMapping("/{id}")
    public RestResult<Blog> selectById(@PathVariable int id){
        return RestResult.success(bs.selectByPrimaryKey(id));
    }

    //仅拥有blog:create权限的用户可以调用该方法
    @RequiresPermissions("blog:create")
    @PostMapping("")
    public RestResult<Blog> Insert(@RequestParam String title, @RequestParam String content, @RequestParam int author){
        Blog blog = new Blog();
        blog.setTitle(title);
        blog.setContent(content);
        blog.setAuthor(author);
        bs.insert(blog);
        return RestResult.success(bs.selectByPrimaryKey(blog.getId()));
    }

    @RequiresPermissions("blog:edit")
    @PatchMapping("/{id}")
    public RestResult<Blog> update(@PathVariable int id,@RequestParam(required = false) String title, @RequestParam(required = false) String content, @RequestParam(required = false) Integer author){
        Blog blog = new Blog();
        blog.setId(id);
        blog.setTitle(title);
        blog.setContent(content);
        blog.setAuthor(author);
        bs.updateByPrimaryKeySelective(blog);
        return RestResult.success(bs.selectByPrimaryKey(blog.getId()));
    }

    @RequiresPermissions("blog:delete")
    @DeleteMapping("/{id}")
    public RestResult<Blog> delete(@PathVariable int id){
        Blog blog = bs.selectByPrimaryKey(id);
        bs.deleteByPrimaryKey(id);
        return RestResult.success(blog);
    }

}
```

之后别忘了通过springboot对异常进行捕获并返回适当的结果：

```java
@ControllerAdvice
@ResponseBody
public class ControllerExceptionHandler {
    @ExceptionHandler(AuthorizationException.class)
    public RestResult<Object> handleAuthorizationException(AuthorizationException e){
        return RestResult.fail(403,'没有权限');
    }
}
```