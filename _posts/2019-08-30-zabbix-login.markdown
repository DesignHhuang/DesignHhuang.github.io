---
layout: post
title:  "模拟请求实现zabbix免登录"
date:   2019-08-30 13:10:46 +0800
categories: huangxiaomin update
---

为了从自己的系统中跳转到ZABBIX的时候不需要进行登录操作，研究了一下实现了，这里记录一下。

------

## 模拟页面请求登录达到免密效果

### 1. 修改`index.php`页面。
```
vim /usr/share/zabbix/index.php
```
在php中添加以下代码：
```
$origin = isset($_SERVER['HTTP_ORIGIN']) ? $_SERVER['HTTP_ORIGIN'] : '';
$allowOrigin = array(
        "http://地址1",
        "http://地址2",
        "http://地址3",
        "http://地址4"
);
if (in_array($origin, $allowOrigin)) {
        header("Access-Control-Allow-Origin:".$origin);
}
```


### 2. 在本系统登录的地方写请求。

```
let bodyFormData = new FormData();
bodyFormData.set('name', 'admin');
bodyFormData.set('password', 'zabbix');
bodyFormData.set('enter', 'login');
axios.post("zabbix的登录地址", bodyFormData, {
    withCredentials: true,
    crossDomain: true,
    headers: { "Content-Type": "multipart/form-data" }
}).then(function (response) {
    console.log("ZABBIX LOGIN SUCCESS")
}).catch(function (error) {
    console.log("ZABBIX LOGIN FAIL")
});
```

有时候zabbix.php也需要改一下 Access-Control-Allow-Origin 视情况而定。