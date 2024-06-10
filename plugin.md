# 插件编写指南

## 编写插件

Kong网关插件由 Lua 模块组成，Kong网关提供了一套用于插件开发的**Plugin Development Kit**(PDK) ，通过PDK可以与请求/响应对象或流进行交互，实现任意逻辑。PDK 是一组 Lua 函数。

## 插件的文件结构

Kong插件需要以特定的文件结构组织。

一个最简单的插件必须包含下面两个文件：

handler.lua: 插件的核心。它是一个要实现的接口，其中每个函数将在请求/连接生命周期中的所需时刻运行。
schema.lua: 定义插件的运行规则，定义插件所需要的配置结构（以便用户在使用时配置）

``` javascript
[my-first-plugin]$ tree
.
├── handler.lua  # 插件的核心，逻辑实现。
└── schema.lua   # 定义配置结构和运行规则。
```
如果需要与Kong进行更深入的交互，实现更加复杂的插件。比如需要与数据库交互，需要公开管理API的endpoint。 这时就需要更加复杂的文件结构。

``` javascript
complete-plugin
├── api.lua        # 定义管理 API 中可用的端点列表
├── daos.lua       # 定义 DAO（数据库访问对象）列表
├── handler.lua    # 要实现的接口。
├── migrations     # 数据库迁移（例如创建表）
│   ├── init.lua
│   └── 000_base_complete_plugin.lua
└── schema.lua     # 保存插件配置的架构，
```
本文主要介绍简单插件开发和调试流程，高级插件的开发后面有空会另开一文介绍。


## 生命周期和切入点

本文只介绍HTTP请求的生命周期和切入点。

Kong中的**HTTP Module** 定义了如下接口用于为HTTP请求编写插件



| 函数名            | 语义                                                         | 适用协议                | 描述                                                         |
| :---------------- | :----------------------------------------------------------- | :---------------------- | :----------------------------------------------------------- |
| init_worker       | https://github.com/openresty/lua-nginx-module#init_worker_by_lua_block | *                       | Nginx worker进程启动时执行                                   |
| certificate       | https://github.com/openresty/lua-nginx-module#ssl_certificate_by_lua_block | https, grpcs, wss       | SSL握手时执行                                                |
| rewrite           | https://github.com/openresty/lua-nginx-module#rewrite_by_lua_block | *                       | 用于在Kong收到客户端请求时的重写阶段执行。在此阶段Server和Consumer还没被识别，因此只能用于全局插件。 |
| access            | https://github.com/openresty/lua-nginx-module#access_by_lua_block | http(s), grpc(s), ws(s) | 在收到请求后，转发给upstream前执行.                          |
| ws_handshake      | https://github.com/openresty/lua-nginx-module#access_by_lua_block | ws(s)                   | 在每个WebSocket请求完成 WebSocket 握手前执行.                |
| response          | https://github.com/openresty/lua-nginx-module#access_by_lua_block | http(s), grpc(s)        | 从upstream收到相应后，转发给客户端前执行。                   |
| header_filter     | https://github.com/openresty/lua-nginx-module#header_filter_by_lua_block | http(s), grpc(s)        | 从upstream收到所有应答header时执行。                         |
| ws_client_frame   | https://github.com/openresty/lua-nginx-module#content_by_lua_block | ws(s)                   | 从客户端收到WebSocket Message时执行。                        |
| ws_upstream_frame | https://github.com/openresty/lua-nginx-module#content_by_lua_block | ws(s)                   | 从upstream收到WebSocket Message时执行。                      |
| body_filter       | https://github.com/openresty/lua-nginx-module#body_filter_by_lua_block | http(s), grpc(s)        | 每收到upstream应答的chunk时执行.  由于响应被流式传输回客户端，因此它可能会超出缓冲区大小并被逐块流式传输。 这个函数在一个请求中可能会被多次执行。 |
| log               | https://github.com/openresty/lua-nginx-module#log_by_lua_block | http(s), grpc(s)        | 最后一个response返回给客户端后执行。                         |
| ws_close          | https://github.com/openresty/lua-nginx-module#log_by_lua_block | ws(s)                   | WebSocket关闭时执行。                                        |
|                   |                                                              |                         |                                                              |

## 简单插件的编码示例
这里开发一个不带配置的建议插件，并在HTTP请求常见切入点做一些处理逻辑。

目录结构：

```javascript
my-plugins/
└── my-first-plugin
    ├── handler.lua
    ├── schema.lua
```

**schema.lua：**

```javascript
//schema.lua
local typedefs = require "kong.db.schema.typedefs"
  

local schema = {
    name = "my-first-plugin",
    fields = {
        {
            consumer = typedefs.no_consumer,
        },
        {
            -- 插件只在nginx http模块中生效
            protocols = typedefs.protocols_http,
        },
        {
            config = {
                type = "record",
                fields = {
               },
            },
        },
    },
}

return schema
```

**handler.lua:**

```javascript
//handler.lua
local MyFirstHandler = {
    -- 插件的优先级，决定了插件的执行顺序；数字越大，优先级越高，越早执行
    PRIORITY = 1101,
    -- 插件的版本号
    VERSION = "0.1.0-1",
}

-- 在Nginx worker启动时执行
function MyFirstHandler:init_worker()
        kong.log("data:init_worker") -- 用来确认是否加载成功的日志
end

-- 收到请求,还没进入server处理时执行, 
-- 此处判断路径如果不是/sayHello和/sayBye直接返回字符串"only support /sayHello and /sayBye"
function MyFirstHandler:rewrite()
        kong.log("MyFirstHandler:rewrite")
        local rawPath = kong.request.get_raw_path() -- 使用PDK获取请求URL
        kong.log("rewrite rawpath: " .. rawPath)
        if rawPath ~= "/sayHello" and rawPath ~= "/sayBye" then
                kong.log("not support rawPath: " .. rawPath)
                return kong.response.exit(404, "only support /sayHello and /sayBye")
        end
        kong.log("rewrite finish")
end

function MyFirstHandler:access()
        kong.log("access")
        kong.service.request.set_header("req-key", "plugin-header-value")
end

-- 注意，即使rewrite中使用了kong.response.exit， 这里也会执行
function MyFirstHandler:header_filter()
        kong.log("header_filter")
        local header = kong.service.response.get_header("rsp-key")
        if header ~= nil then
          kong.log(header)
          kong.response.set_header("rsp-key", header .. " modify by plugin")
        end
end

function MyFirstHandler:body_filter()
        kong.log("body_filter")
end

return MyFirstHandler
```



## 测试验证

#### **测试准备：upstream服务Demo**

这里使用python的flask框架搭建了一个web服务，接受两个路由 `sayHello` 和 `sayBye`s

```javascript
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json

from flask import Flask, request

app = Flask(__name__)

@app.route("/sayHello")
def sayHello():
    req_header = request.headers.get("req-key")
    print(req_header)

    name = request.args.get("name")
    return "Hello " + name, 200, [('rsp-key', 'rsp-head-by-server')]

@app.route("/sayBye")
def sayBye():
    req_header = request.headers.get("req-key")
    print(req_header)
    name = request.args.get("name")
    return "Bye " + name, 200, [('rsp-key', 'rsp-head-by-server')]

app.run(host="0.0.0.0", port=5000, debug=True)
```

并未这个服务配置 upstream、server、router

确保在未配置插件是，可以正常通过 `http://{kong网关IP}:8000/sayHello` 访问

## 安装插件

新建/修改 `/etc/kong/kong.conf`

```javascript
[windealli@VM-52-29-tencentos kong]$ cat etc/kong/kong.conf
plugins = bundled,my-first-plugin  # 指定了要加载的插件
lua_package_path = /kong/plugins/?.lua;; # 指定了自定义插件的目录
[windealli@VM-52-29-tencentos kong]$
```

修改 `lua_kong/kong/constants.lua`

> 如果不修改constants.lua，会出现konga的info页面可以看到插件my-first-plugin，但是在添加插件页面却找不到。

```javascript
# 在plugins中添加我们的插件
local plugins = {
  "my-first-plugin",
 ...
}
```

![image-20240610171206590](https://raw.githubusercontent.com/yuezu1026/typora_pic/main/images/202406101712691.png)







## 使用插件

直接将插件配置到全局

## 验证



1） 验证rewrite

1. 验证完整链路
2. 查看服务端日志打印了插件添加的header