---
title: Spring Boot 读取配置字符串中转义字符的小坑
date: 2020-09-09 19:30:03
tags:
- Spring Boot
categories: 
- blog
---

今天遇到一个问题，从配置中读取的字符串内容中的`\n`不换行，类似如下：

```yml
wechat:
	message:
		error: 出现如下错误：\n 1.错误1 \n 2.错误2
```

```java
@Value("${wechat.message.error}")
String errorMessage;
```

得到的`errorMessage`的值为`"出现如下错误：\n 1.错误1 \n 2.错误2"`其中`\n`作为字符显示，并不能换行。一开始以为是View曾解析的问题，折腾半天后发现，在Spring Boot读取配置时，会将配置字符串中的`\n`作为字符读入，等价于`\\n`。

解决该问题也很简单，在配置文件中用引号包裹包含转义字符的字符串即可：

```yml
wechat:
	message:
		error: "出现如下错误：\n 1.错误1 \n 2.错误2"
```