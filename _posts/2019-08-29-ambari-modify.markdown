---
layout: post
title:  "ambari模拟自动登录"
date:   2019-08-29 13:10:46 +0800
categories: huangxiaomin update
---


为了从自己的系统中跳转到Ambari的时候不需要进行登录操作，研究了一下实现了，这里记录一下。

------

## 模拟自动登录达到免密效果（账号：admin，密码：admin）

### 1. 修改`index.html`页面。
```
vim /usr/lib/ambari-server/web/index.html
```
在开始执行脚本处添加以下代码：
```
$(document).ready(function() {
    require('initialize');
    // make favicon work in firefox
    $('link[type*=icon]').detach().appendTo('head');
    $('#loading').remove();
    //为了解决登录问题添加的代码  huangxiaomin
        var int = self.setInterval(function(){
            if($("#i18n-4").length > 0){
                $("#i18n-4").click();
                window.clearInterval(int);
            }
        },50)     //每隔50毫秒检测是否出现登录按钮，出现的话就执行登录操作，激活click()方法。
      });
```


### 2. 隐藏登录的样式。
```
vim /usr/lib/ambari-server/web/stylesheets/app.css
```
添加login类的样式
```
.login {
    position : absolute;
    top : -9999px;
    left : -9999px;
}
```

### 3. 隐藏登录的样式。
```
vim /usr/lib/ambari-server/web/javascripts/app.js
```
在这个文件中写定登录的账号和密码，这样执行click()方法时，就可以用写定的账号和密码登录了。
```
name: 'loginController',

loginName: 'admin',                   //在此处将账号和密码写定
password: 'admin',

errorMessage: '',

isSubmitDisabled: false,

```
```
var controller = this.get('loginController');
var loginName = "admin";                  //以下三行是要修改的地方
controller.set('loginName', loginName);
controller.set('password',"admin");
var hash = misc.utf8ToB64(loginName + ":" + controller.get('password'));
var usr = '';

if (App.get('testMode')) {
    if (loginName === "admin" && controller.get('password') === 'admin') {
        usr = 'admin';
    } else if (loginName === 'user' && controller.get('password') === 'user') {
        usr = 'user';
    }
}
```

这种方法实际上还是要登录的，只不过是通过定好的账号和密码自动登录，登录的自检查时间为50毫秒，检查到了就直接登录，肉眼基本看不出登录的过程，所以相当于模拟了免登录。
