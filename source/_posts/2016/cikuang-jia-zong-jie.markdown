---
layout: post
title: CI 框架总结
date: '2016-03-27 03:09:54 +0800'
tags:
  - CI
categories:
  - PHP
---

>本文主要是参考2.2.6的源码

# 设计思想

使用 &get_instance(); 可以引用所有已加载的类。
<!-- more -->
# 中文手册
http://codeigniter.org.cn/userguide2/index.html
# 框架运行图解

## 运行流程图
{% img center /images/2016/0327/CI.png %}

## 运行生命周期
{% img center /images/2016/0327/CI2.png %}


# 开发注意事项

## Controller 中

* `_remap` 方法（接管路由）
* `_output` 方法（接管输出）
* `_`前缀的方法名都会被路由屏蔽

# 如何扩展框架？

## 扩展/替换 core 类
>此类都是在系统使用的核心类，常用的是扩展控制器类

在 application/core 下新建文件

- 扩展

	{% codeblock lang:php %}
	class MY_controller extends CI_controller{}
	{% endcodeblock %}
- 替换

	{% codeblock lang:php %}
	class CI_controller{}
	{% endcodeblock %}

## 使用/新建/替换/扩展类库

1. 使用内置

	{% codeblock lang:php %}
	$this->load->library('name');
	{% endcodeblock %}

2. 建立新的类
	在 applicatioin/libraries 目录下
3. 扩展已有类
	在 applicatioin/libraries 目录下，使用定义好的子类前缀，并继承父类
	比如扩展 email：

	{% codeblock lang:php %}
	class MY_Email extends CI_Email{}
	{% endcodeblock %}
3. 替换已有类
	在 applicatioin/libraries 目录下，声明和默认的类名一样的类



## 使用/新建适配器

- 内置

``` php
$this->load->driver('some_parent');
$this->some_parent->some_method();
$this->some_parent->child_one->some_method();
```

- 自定义

目录结构：

```
/application/libraries/Driver_name
  Driver_name.php
  drivers
  	Driver_name_subclass_1.php
  	Driver_name_subclass_2.php
  	Driver_name_subclass_3.php
```



## 集成自己的独立应用

目录结构：

```
/application/third_party/foo_bar
config/
helpers/
language/
libraries/
models/
```

使用方法：

```php
$this->load->add_package_path(APPPATH.'third_party/foo_bar/');
$this->load->library('foo_bar');
...
$this->load->remove_package_path(APPPATH.'third_party/foo_bar/');
```

# 坑
1. 默认保存 SQL；cli 模式下会内存溢出;修复方式如下：

``` php
在配置文件中增加：
$db['default']['save_queries'] = false;
或者在代码里增加:
$this->load->database();
$this->db->save_queries = false;
```
