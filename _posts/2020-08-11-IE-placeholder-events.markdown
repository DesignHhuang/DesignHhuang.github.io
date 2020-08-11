---
layout: post
title:  "antd(react)中的input在IE下会默认自检"
date:   2020-08-11 10:28:46 +0800
categories: huangxiaomin update
---

### 1. BUG的出现             

> * 环境是ANTD（react）中在Form中使用input
> * input中存在placeholder属性
> * input带有自检属性（自检为必输项）
> * 在IE中直接打开页面就会执行自检，出现必须项提示

### 2. BUG的原因
在IE中，如果input带有placeholder属性的话，在加载的时候会自动触发oninput事件，form就会检查input的输入项

### 3. 解决的办法
## 1. 因为是在form中的input，我们可以直接设置form中的input获取数据的时机，设置获取时机在onchange的时候，只有执行onchange的时候才会出发form的检查机制，因为onchange的执行顺序是在oninput之后的，这样placeholder自动触发oniput事件的时候，也不会使form组件自检。
## 2. 或者我们可以设置检查的时机，使用validateTrigger选项。

### 4. input的事件执行顺序
onkeydown  < onkeypress <	oninput <	onkeyup <	onchange < onblur 