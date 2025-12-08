---
title: Managing Tomcat Logs
categories:
- tech
- java
tags:
- tomcat
---

我喜欢提出问题，逐步解决的方式来构建文章。

- tomcat 的日志是用的什么库，配置文件在哪。
- 配置文件如何理解。
- 日志按照日期记录成文件，保留最近n天日志。

# tomcat 核心日志

主要使用 JULI，配置在 $CATALINA_BASE/conf/logging.properties

一个日志事件从产生到输出的流程是：

- 事件产生： Tomcat 某个组件（如 org.apache.catalina.startup.HostConfig）产生一条日志消息。

- 级别检查 (Logger): 消息首先与该组件对应的 Logger (org.apache.catalina.startup.HostConfig.level) 进行比较。如果消息级别低于 Logger 级别，则丢弃。

- 日志路由 (Handlers): 消息被发送到 Logger 配置的 Handlers (.handlers 属性)。

- 级别检查 (Handler): 消息与 Handler 自身的级别 ([Handler 别名].level) 进行比较。如果消息级别低于 Handler 级别，则丢弃。

- 输出： 消息通过 Handler 定义的格式 (.formatter) 写入到指定的位置 (.directory + .prefix)。

# tomcat 核心配置文件理解

```
# 第一部分 根 Logger 和全局 Handlers 定义
## 全局激活的 Handler 列表。 列出了所有在 Tomcat 启动时需要实例化的 Handler 别名。
## Tomcat 的 JULI 实现为了简化配置和保证唯一性，将 Handler 的别名和它的类名组合成一个唯一的 Key 来进行配置
## [Handler 别名].[Handler 类的 FQCN].[属性名]
handlers = 1catalina.org.apache.juli.AsyncFileHandler, 2localhost.org.apache.juli.AsyncFileHandler, 3manager.org.apache.juli.AsyncFileHandler, 4host-manager.org.apache.juli.AsyncFileHandler, java.util.logging.ConsoleHandler

##  定义根 Logger（没有指定名字的顶级 Logger）默认使用的 Handlers。
.handlers = 1catalina.org.apache.juli.AsyncFileHandler, java.util.logging.ConsoleHandler
## 定义根 Logger 的默认日志级别。如果其他 Logger 没有明确指定级别，会继承这个级别。
.level = INFO

# 第二部分 Handler 属性定义 (配置输出目标)

## 设置 1catalina 这个 Handler 自身能处理的最低日志级别。
1catalina.org.apache.juli.AsyncFileHandler.level = FINE
## 指定日志文件的输出目录（通常是 logs 文件夹）。
1catalina.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
## 指定日志文件的文件名前缀，最终文件名类似于 catalina.2025-12-02.log。
1catalina.org.apache.juli.AsyncFileHandler.prefix = catalina.
## 指定日志文件的字符编码。
1catalina.org.apache.juli.AsyncFileHandler.encoding = UTF-8
## 指定日志消息的输出格式。Tomcat 默认使用 OneLineFormatter
1catalina.org.apache.juli.AsyncFileHandler.formatter = org.apache.juli.OneLineFormatter

2localhost.org.apache.juli.AsyncFileHandler.level = FINE
2localhost.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
2localhost.org.apache.juli.AsyncFileHandler.prefix = localhost.
2localhost.org.apache.juli.AsyncFileHandler.encoding = UTF-8

3manager.org.apache.juli.AsyncFileHandler.level = FINE
3manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
3manager.org.apache.juli.AsyncFileHandler.prefix = manager.
3manager.org.apache.juli.AsyncFileHandler.encoding = UTF-8

4host-manager.org.apache.juli.AsyncFileHandler.level = FINE
4host-manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
4host-manager.org.apache.juli.AsyncFileHandler.prefix = host-manager.
4host-manager.org.apache.juli.AsyncFileHandler.encoding = UTF-8

## Console Handler 的级别（用于控制台输出，对应 catalina.out）
java.util.logging.ConsoleHandler.level = FINE
java.util.logging.ConsoleHandler.formatter = org.apache.juli.OneLineFormatter
java.util.logging.ConsoleHandler.encoding = UTF-8

# 3. Logger 级别和 Handler 关联 (控制什么被记录)
## Logger 级别 和 Handler 级别 都是独立存在的过滤屏障，只有通过了两者，日志消息才会被最终记录
## Logger 级别 (INFO) 决定了 Handler 永远收不到 DEBUG/FINE 级别的消息。
## Handler 级别 (WARNING) 决定了它会丢弃 Logger 传来的 INFO 级别的消息
## org.apache.catalina 包下所有 Logger 的默认日志级别
org.apache.catalina.level = INFO

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].handlers = 2localhost.org.apache.juli.AsyncFileHandler

## Tomcat 容器采用树状结构，由以下主要组件组成（从上到下）：## Server -> Service -> Engine -> Host-> Context
#当 Tomcat 在日志配置中引用一个特定组件时，它使用中括号 [] 来引用该组件的 name 属性，从而精确地定位到日志记录器 (Logger)。 
# 在 Tomcat 中，Context Path 是用来区分部署在 Host 上的各个 Web 应用程序的标识符
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].handlers = 3manager.org.apache.juli.AsyncFileHandler

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/host-manager].level = INFO
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/host-manager].handlers = 4host-manager.org.apache.juli.AsyncFileHandler
```

|级别名称 (Level)|优先级 (High to Low)|描述 (Description)|对应的数值 (Numeric Value)|
|----|----|----|----|
|OFF|最高 (All Off)|关闭所有日志记录。|Integer.MAX_VALUE|
|SEVERE|次高|表示非常严重的错误事件，可能会导致应用程序中止。|1000|
|WARNING||表示潜在的问题，但不会导致应用程序中断。|900|
|INFO||提供一般性的运行时信息，如启动/停止消息。|800|
|CONFIG||提供静态配置信息的详细转储。|700|
|FINE||提供跟踪应用程序流程的详细信息。|500|
|FINER||提供更精细的跟踪，比 FINE 更详细。|400|
|FINEST||提供最详细的跟踪信息，用于深度调试。|300|
|ALL|最低 (All On)|启用所有消息的日志记录。|Integer.MIN_VALUE|

# 按天归档，只保留7天日志
JULI 的 FileHandler 默认就是按天滚动的
- .rotatable 启用日志文件轮换功能。（默认通常为 true）
- .fileDateFormat 定义日期格式，确保文件名中包含日期，例如 catalina.2025-12-03.log
- .prefix 文件名前缀
- .suffix 文件名后缀  
  
保留固定天数 (Retention/Cleaning)，是通过配置 日志清理器 (Rotator/Cleaner) 来实现

- .maxDays 设置日志文件保留的最大天数。系统会在轮换时检查旧文件并删除超过这个天数的文件
