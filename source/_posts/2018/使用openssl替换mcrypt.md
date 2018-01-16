---
layout: "post"
title: "使用Openssl替换mcrypt加解密"
date: "2018-01-16 14:15"
tags:
  - php
  - openssl
  - mcrypt
categories:
  - PHP
---

# 背景

PHP官方文档显示：
> Warning 本扩展从 PHP 7.1.0 开始废弃；自 PHP 7.2.0 起，会移到 PECL。

地址：https://secure.php.net/manual/zh/intro.mcrypt.php

<!-- more -->

# 加解密方法替换公式

```
openssl_encrypt() = base64_encode(mcrypt_encrypt())
openssl_decrypt() = mcrypt_decrypt(base64_decode($encryptString))
```

# 钉钉的开放平台消息体加解密
因官方给的是`mcrypt`方案，所以参考官方给的示例，修改为php7版本。
Github 地址：[dingtalk-crypto](https://github.com/hlf513/dingtalk-crypto)

## 安装
```
composer require hlf_513/dingtalk-crypto
```
## 使用
``` php
$crypto = new Crypto(
    $token,
    $encodingAesKey,
    $suiteKey
);

$string = 'success';
$timestamp = '1515664989185';
$nonce = '53IP1CdM';

// 加密
$ret = $crypto->encryptMsg($string, $timestamp, $nonce);
$ret = json_decode($ret, 1);
// 解密
$ret = $crypto->decryptMsg($ret['msg_signature'], $timestamp, $nonce, $ret['encrypt']);
// output: $ret === 'success'
```


# 参考资料

* [php7.1微信公众平台消息安全模式的加密及解密](http://blog.csdn.net/sapperlab/article/details/56672443)
