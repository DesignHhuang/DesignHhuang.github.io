---
layout: post
title:  "系统跳转到superset时设置免密登录的一种方式"
date:   2019-08-29 13:10:46 +0800
categories: huangxiaomin update
---

为了从自己的系统中跳转到SUPERSET的时候不需要进行登录操作，研究了一下实现了，这里记录一下。

先说个题外话吧，就是设置`superset`的默认语言为中文，只要修改配置文件`config.py`中的`BABEL_DEFAULT_LOCALE`就可以了。

------

## 模拟请求登录达到免密效果

### 1. 修改`config.py`中的配置。
```
# Flask-WTF flag for CSRF
WTF_CSRF_ENABLED = False

# Add endpoints that need to be exempt from CSRF protection
WTF_CSRF_EXEMPT_LIST = ["superset.views.core.log","你的地址1","你的地址2","你的地址3"]

```

```
# CORS Options
ENABLE_CORS = True
CORS_OPTIONS = {"supports_credentials":True}

```

### 2. 前端代码实现。
```
//在登录自己的系统时同时登录SUPERSET，模拟登录一次，之后访问就不需要在登录了
    let superseturl = 'http://superset的地址/login/';
    
    axios.post(superseturl, { username: "admin", password: "admin" }, {
        withCredentials: true, headers: { "Content-Type": "application/json" }
    }).then(function (response) {
        console.log("SUPERSET LOGIN SUCCESS")   //登录成功
    }).catch(function (error) {
        console.log("SUPERSET LOGIN FAIL")
    });

```

ok,这样就可以啦，有问题可以发文件我一起研究。（ihuanglimin@qq.com）