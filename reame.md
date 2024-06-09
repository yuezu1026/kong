# 云原生API网关-Kong部署与konga基本使用

## 1.简介
官网：https://docs.konghq.com/gateway/3.2.x/

## 扩展知识
### 2.1 运行方式

    
Kong Gateway 是一个运行在 Nginx 中的 Lua 应用程序。Kong Gateway 与OpenResty一起分发，OpenResty 是一组扩展lua-nginx-module 的模块。


这为模块化架构奠定了基础，插件可以在运行时启用和执行。Kong Gateway 的核心是实现数据库抽象、路由和插件管理。插件可以存在于单独的代码库中，并可以注入到请求生命周期的任何地方，只需几行代码。


Kong 提供了许多插件供您在网关部署中使用。您还可以创建自己的自定义插件。有关详细信息，请参阅 插件开发指南、PDK 参考以及使用其他语言（ JavaScript、Go和Python ）创建插件的指南。


### 2.2Konga

Konga不是官方应用程序，并且和Kong没有隶属关系。
官网：https://github.com/pantsel/konga

#### 2.2.1 介绍
```
Konga是一款基于Kong Admin API的GUI图形化管理界面。
```
#### 2.2.2 特征

```
（1）管理所有 Kong Admin API 对象。
（2）从远程源（数据库、文件、API 等）导入消费者。
（3）管理多个 Kong 节点。
（4）使用快照备份、恢复和迁移 Kong 节点。
（5）使用健康检查监控节点和 API 状态。
（6）电子邮件和 Slack 通知。
（7）多个用户。
（8）简单的数据库集成（MySQL、postgresSQL、MongoDB）。
```
### 2.3 Kong插件

    
Kong Gateway 是一个 Lua 应用程序，旨在加载和执行 Lua 或 Go 模块，我们通常称之为插件。Kong 提供了一组与 Kong Gateway 捆绑在一起的标准 Lua 插件。您有权访问的插件集取决于您的安装：开源、企业或在 Kubernetes 上运行的这些 Kong Gateway 选项之一。
自定义插件也可以由 Kong 社区开发，并由插件创建者支持和维护。如果它们发布在 Kong Plugin Hub 上，则称为社区或第三方插件。


==地方==
####  下划线（有道支持）
++下划线++


