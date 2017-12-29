---
layout: post
title: Laravel自定义用户权限校验
date: '2017-12-30 00:05'
categories:
  - PHP
tags:
  - php
  - laravel
---

# 背景

在现实的架构中，帐号体系往往会单独维护，应用需要鉴权的时候会请求用户中心接口，而`laravel`使用`auth`中间件时默认采用的是`session`进行鉴权，不能满足我们的需求，所以需要自定义权限校验。

# `Auth` 中间件的工作原理

<!-- more -->
首先，在 `app/Http/Kernel.php` 中，发现 :
``` php
protected $routeMiddleware = [
       'auth' => \Illuminate\Auth\Middleware\Authenticate::class,
       'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
       ...
   ];
```
然后，跟踪到 `vendor/laravel/framework/src/Illuminate/Auth/AuthManager.php`

```PHP
public function guard($name = null)
{
   $name = $name ?: $this->getDefaultDriver();

   return isset($this->guards[$name])
               ? $this->guards[$name]
               : $this->guards[$name] = $this->resolve($name);
}

...
public function getDefaultDriver()
{
   // 注意这行
    return $this->app['config']['auth.defaults.guard'];
}
```
然后，查看 `config/auth.php` :
``` php
...

'defaults' => [
       'guard' => 'web', // 注意这行
       'passwords' => 'users',
   ],
...
'guards' => [
       'web' => [
           'driver' => 'session',
           'provider' => 'users',
       ],
       ...
   ],
...
```
由此可见，默认使用的`guard`是`web`，驱动是`session`；到此已经很明确了，我们需要做的只是新建一个`guard`。

具体如何创建`guard`？
见官方文档：https://laravel.com/docs/5.5/authentication#adding-custom-guards


# 举个例子

* 创建 `guard`

```php
namespace App;

use Illuminate\Auth\GuardHelpers;
use Illuminate\Contracts\Auth\Guard;
use Illuminate\Http\Request;

class KohanaGuard implements Guard
{
	use GuardHelpers;

	protected $request;

	public function __construct(Request $request)
	{
		$this->request = $request;
	}

	/**
	 * Get the currently authenticated user.
	 *
	 * @return \Illuminate\Contracts\Auth\Authenticatable|null
	 * @throws \Exception
	 */
	public function user()
	{
          // 校验规则
          // 成功返回 User Model 对象
          // 失败返回 null
	}

	/**
	 * Validate a user's credentials.
	 *
	 * @param  array $credentials
	 *
	 * @return bool
	 */
	public function validate(array $credentials = [])
	{
		// TODO: Implement validate() method.
	}

}

```

* 添加到 `app/Providers/AuthServiceProvider.php`:

```php
public function boot()
{
   $this->registerPolicies();

   //注册 kohana 的 guard
   \Auth::extend('kohana', function ($app, $name, array $config) {

     return new KohanaGuard($app['request']);
   });
}
```

* 修改 `config/auth.php`
```php
...
'defaults' => [
    'guard' => 'kohana', // 注意这行
    'passwords' => 'users',
],
...
'guards' => [
   ...
   // 以下为新增
   'kohana' => [
     'driver' => 'kohana',
     'provider' => 'users',
   ],
],
...
```
