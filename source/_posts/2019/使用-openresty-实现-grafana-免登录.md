---
layout: post
title: 使用 OpenResty 实现 Grafana 免登录
date: '2019-06-20 18:44'
tags:
  - grafana
  - openresty
  - lua
categories:
  - 监控
---
# 背景
公司内部都有管理后台，一般都会把第三方软件集成到后台，收敛登录入口。 本文主要介绍一种集成 Grafana 的方法。
<!-- more -->

# 问题
假设，只有一个 `token` ，通过这个 `token` 可以获取到用户信息，在这种情况下，怎么可以免登录进入 `Grafana`？

例：访问：http://grafana.com?token=xxxxxxx 后，不应该跳转到 http://grafana.com/login ，而应该直接进入 http://grafana.com/home

# 基本思路
通过查看[文档](https://grafana.com/docs/auth/overview/)，可以发现 `Grafana` 支持多种权限认证；但没有一个能完美解决我们问题的方案；唯一比较接近的是 `Auth Proxy`，说是可以在 `Grafana` 外部进行权限验证，经过通读文档和试验，发现其本质是：通过 web server 校验后，在每个请求上增加一个 header (默认 `X-WEBAUTH-USER`)来实现免登录；文档中的示例是通过 `htpasswd` 来实现的权限验证，那自然想到可以通过 `OpenResty` 进行权限校验。

假设 token 对应的用户信息存在 Redis 中，基本流程如下：
![grafana_login](/images/2019/06/grafana-login.png)

# 实现细节
## Auth Proxy
本质是增加一个`header`（默认 `X-WEBAUTH-USER`）；

`Grafana` 开启`Auth Proxy`；配置：
``` ini
[auth.proxy]
enabled = true
; defalut header name
header_name = X-WEBAUTH-USER
; username or email
header_property = username
; 自动注册
auto_sign_up = false
; 白名单
whitelist = 127.0.0.1
```
## OpenResty
`OpenResty`的配置为：
``` nginx
server {
  listen      80;
  server_name grafana.com;

  location / {
    content_by_lua_file "/usr/local/etc/openresty/conf.d/grafana.lua";
    # 本质是为了加这个 header
    # proxy_set_header X-WEBAUTH-USER "longfeihe";
  }

  location @grafana {
    proxy_pass http://127.0.0.1:3000;
  }

  # 若使用 url 接口鉴权，则开启此 location
  #location ^~ /proxy/ {
  #  internal; # 内部请求
  #  proxy_pass http://someone-proxy.com/; # 必须有 /
  #  proxy_set_header Accept "*/*";
  #}
}
```

`grafana.lua`
```lua
-- config redis start
local redisIp = '127.0.0.1'
local redisPort = '6379'
-- config redis end

local cjson = require "cjson"
local redis = require "resty.redis"
local aes = require "resty.aes"

local function getToken()
    local args = ngx.req.get_uri_args()
    local token = args['token']
    return token
end

function conRedis(ip, port)
    local red = redis.new()
    local ok, err = red:connect(ip, port)
    red:set_timeout(1000) -- 1s
    if not ok then
        ngx.log(ngx.ERR, 'connect redis fail')
        ngx.exit(ngx.HTTP_SERVICE_UNAVAILABLE)
    end
    ngx.log(ngx.INFO, 'connect redis success')
    return red
end

local function getUsernameByRedis(token)
    local red = conRedis(redisIp, redisPort)
    token = ngx.md5(token .. '_sso')
    ngx.log(ngx.INFO, 'fetch redis key:' .. token)
    local value = red:get(token)
    closeRedis(red)
    if value ~= ngx.null then
        obj = cjson.decode(value)
        local username = obj['username']
        return username
    end
    return
end

-- 使用 url 鉴权
function getUsernameByUrl(token)
    local uri = "/proxy/get-user-info" -- 这里接口地址按实际情况进行修改
    --ngx.req.set_header("X-TOKEN", token)
    res = ngx.location.capture(uri, {
        method = ngx.HTTP_POST
    })
    if res.body then
        local data = cjson.decode(res.body)
        if data["code"] ~= 0 then
            ngx.log(ngx.ERR, "request model for get user info was failed; msg:" .. data["message"])
            ngx.exit(ngx.HTTP_BAD_GATEWAY)
        end
        local email = data["data"]["email"]
        return email
    else
        ngx.log(ngx.ERR, "request model for get user info was failed; code:" .. res.status)
        ngx.exit(ngx.HTTP_BAD_GATEWAY)
    end
end

function closeRedis(redis)
    local ok, err = redis:close()
    if not ok then
        ngx.log(ngx.ERR, 'close redis connect fail')
        return
    end
    ngx.log(ngx.INFO, 'close redis connect success')
end

local function setHeader(username)
    ngx.log(ngx.INFO, 'X-WEBAUTH-USER:' .. username)
    ngx.req.set_header("X-WEBAUTH-USER", username)
end

local function setCookie(username)
    ngx.header['Set-Cookie'] = { 'token=' .. encrypt(username) }
end

local function getUsernameByCookie()
    local token = ngx.var.cookie_token
    return decrypt(token)
end

local function logout()
    if ngx.re.match(ngx.var.request_uri, "logout") then
        ngx.header['Set-Cookie'] = { 'token=' }
    end
end

function encrypt(encryptString)
    local aes_128_cbc_md5 = aes:new("AKeyForAES")
    return ngx.encode_base64(aes_128_cbc_md5:encrypt(encryptString))
end

function decrypt(decryptString)
    local aes_128_cbc_md5 = aes:new("AKeyForAES")
    local decrypt = ngx.decode_base64(decryptString)
    return aes_128_cbc_md5:decrypt(decrypt)
end

-- main
logout()

local token = getToken()
if token then
    ngx.log(ngx.INFO, 'get token:' .. token)
    local username = getUsernameByRedis(token)
    if username then
        setHeader(username)
        setCookie(username)
    end
    ngx.exec("@grafana")
else
    local username = getUsernameByCookie()
    if username then
        setHeader(username)
    end
    ngx.exec("@grafana")
end
```

# 参考资料
1. [Auth Proxy Authentication](https://grafana.com/docs/auth/auth-proxy/)
2. [OpenResty](http://openresty.org/en/)
