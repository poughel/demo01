# DataSophon Worker 模块文件完整索引

## 一、索引概述

本文档提供 Worker 模块所有 62 个 Java 文件的完整索引，包括文件路径、行数、核心职责和相关文档链接。

### 索引统计

| 类别 | 文件数 | 总行数 | 占比 |
|------|--------|--------|------|
| **Actor 层** | 19 | 908 行 | 46.6% |
| **Handler 层** | 3 | 527 行 | 27.0% |
| **Strategy 层** | 31 | 1949 行 | 100% |
| **Utils 层** | 6 | 411 行 | 21.1% |
| **Log 层** | 2 | 147 行 | 7.5% |
| **启动类** | 1 | 199 行 | 10.2% |
| **总计** | 62 | ~4141 行 | 100% |

## 二、启动类 (1个文件)

### WorkerApplicationServer.java
- **路径**: `com/datasophon/worker/WorkerApplicationServer.java`
- **行数**: 199 行
- **职责**: Worker 节点的主启动类
- **核心功能**:
  - 初始化 Akka ActorSystem
  - 启动 Node Exporter 监控
  - 创建系统用户和组
  - 向 Master 注册 Worker 节点
  - 注册 JVM 关闭钩子
- **详细文档**: [02-WorkerApplicationServer启动类详解.md](./02-WorkerApplicationServer启动类详解.md)

## 三、Actor 层 (19个文件)

### 3.1 核心 Actor

#### WorkerActor.java
- **路径**: `com/datasophon/worker/actor/WorkerActor.java`
- **行数**: 108 行
- **职责**: Worker 的根 Actor，创建和监督所有子 Actor
- **管理的子 Actor**: 15个
- **详细文档**: [03-Actor层完整分析.md](./03-Actor层完整分析.md#21-workeractor---根-actor)

### 3.2 服务生命周期管理 Actor (6个)

#### InstallServiceActor.java
- **路径**: `com/datasophon/worker/actor/InstallServiceActor.java`
- **行数**: 87 行
- **职责**: 处理服务安装请求
- **特殊处理**: Kerberos yum 安装、软链接创建
- **详细文档**: [03-Actor层完整分析.md](./03-Actor层完整分析.md#22-installserviceactor---服务安装)

#### ConfigureServiceActor.java
- **路径**: `com/datasophon/worker/actor/ConfigureServiceActor.java`
- **行数**: 55 行
- **职责**: 生成服务配置文件
- **使用技术**: FreeMarker 模板引擎
- **详细文档**: [03-Actor层完整分析.md](./03-Actor层完整分析.md#23-configureserviceactor---配置生成)

#### StartServiceActor.java
- **路径**: `com/datasophon/worker/actor/StartServiceActor.java`
- **行数**: 62 行
- **职责**: 启动大数据服务
- **特性**: 支持自定义启动策略
- **详细文档**: [03-Actor层完整分析.md](./03-Actor层完整分析.md#24-startserviceactor---服务启动)

#### StopServiceActor.java
- **路径**: `com/datasophon/worker/actor/StopServiceActor.java`
- **行数**: 50 行
- **职责**: 停止运行中的服务
- **验证机制**: 轮询检查服务状态
- **详细文档**: [03-Actor层完整分析.md](./03-Actor层完整分析.md#25-stopserviceactor---服务停止)

#### RestartServiceActor.java
- **路径**: `com/datasophon/worker/actor/RestartServiceActor.java`
- **行数**: 40 行
- **职责**: 重启服务
- **详细文档**: [03-Actor层完整分析.md](./03-Actor层完整分析.md#26-restartserviceactor---服务重启)

#### CheckServiceStatusActor.java
- **路径**: `com/datasophon/worker/actor/CheckServiceStatusActor.java`
- **行数**: 48 行
- **职责**: 检查服务运行状态

### 3.3 通用操作 Actor (3个)

#### ExecuteCmdActor.java
- **路径**: `com/datasophon/worker/actor/ExecuteCmdActor.java`
- **行数**: 39 行
- **职责**: 执行任意 Shell 命令
- **超时**: 60 秒
- **详细文档**: [03-Actor层完整分析.md](./03-Actor层完整分析.md#27-executecmdactor---命令执行)

#### LogActor.java
- **路径**: `com/datasophon/worker/actor/LogActor.java`
- **行数**: 74 行
- **职责**: 查询服务日志文件
- **功能**: 读取最后 N 行（默认 500 行）
- **详细文档**: [03-Actor层完整分析.md](./03-Actor层完整分析.md#28-logactor---日志查询)

#### FileOperateActor.java
- **路径**: `com/datasophon/worker/actor/FileOperateActor.java`
- **行数**: 52 行
- **职责**: 文件写入操作
- **支持**: 单行/多行写入
- **详细文档**: [03-Actor层完整分析.md](./03-Actor层完整分析.md#29-fileoperateactor---文件操作)

### 3.4 系统管理 Actor (4个)

#### UnixUserActor.java
- **路径**: `com/datasophon/worker/actor/UnixUserActor.java`
- **行数**: 51 行
- **职责**: 创建/删除 Unix 用户

#### UnixGroupActor.java
- **路径**: `com/datasophon/worker/actor/UnixGroupActor.java`
- **行数**: 52 行
- **职责**: 创建/删除 Unix 组

#### KerberosActor.java
- **路径**: `com/datasophon/worker/actor/KerberosActor.java`
- **行数**: 61 行
- **职责**: 生成 Kerberos keytab 文件

#### AlertConfigActor.java
- **路径**: `com/datasophon/worker/actor/AlertConfigActor.java`
- **行数**: 50 行
- **职责**: 配置 Prometheus 告警规则

### 3.5 状态监控 Actor (3个)

#### NMStateActor.java
- **路径**: `com/datasophon/worker/actor/NMStateActor.java`
- **行数**: 21 行
- **职责**: 查询 YARN NodeManager 状态

#### RMStateActor.java
- **路径**: `com/datasophon/worker/actor/RMStateActor.java`
- **行数**: 21 行
- **职责**: 查询 YARN ResourceManager 状态

#### PingActor.java
- **路径**: `com/datasophon/worker/actor/PingActor.java`
- **行数**: 49 行
- **职责**: 响应心跳检测请求
- **响应**: "pong"
- **详细文档**: [03-Actor层完整分析.md](./03-Actor层完整分析.md#210-pingactor---心跳检测)

### 3.6 网络事件 Actor (2个)

#### RemoteEventActor.java
- **路径**: `com/datasophon/worker/actor/RemoteEventActor.java`
- **行数**: 45 行
- **职责**: 监听 Akka Remoting 网络事件
- **监听事件**: Associated, Disassociated, AssociationError
- **详细文档**: [03-Actor层完整分析.md](./03-Actor层完整分析.md#211-remoteeventactor---网络事件监听)

#### SupervisorFunction.java
- **路径**: `com/datasophon/worker/actor/SupervisorFunction.java`
- **行数**: 43 行
- **职责**: Actor 监督策略函数

## 四、Handler 层 (3个文件)

### InstallServiceHandler.java
- **路径**: `com/datasophon/worker/handler/InstallServiceHandler.java`
- **行数**: 244 行
- **职责**: 服务安装业务逻辑
- **核心功能**:
  - 下载安装包（MD5 校验）
  - 解压安装包
  - 执行资源策略 (5种)
  - 设置权限
  - Hadoop/Prometheus 特殊处理
- **详细文档**: [04-Handler处理器分析.md](./04-Handler处理器分析.md#二installservicehandler---服务安装处理器)

### ConfigureServiceHandler.java
- **路径**: `com/datasophon/worker/handler/ConfigureServiceHandler.java`
- **行数**: 299 行
- **职责**: 配置文件生成业务逻辑
- **核心功能**:
  - FreeMarker 模板渲染
  - 配置项处理（INPUT、MULTIPLE、PATH、CUSTOM）
  - ZooKeeper myid 文件生成
  - RangerAdmin 初始化
- **支持格式**: XML, Properties, Prometheus, Custom
- **详细文档**: [04-Handler处理器分析.md](./04-Handler处理器分析.md#三configureservicehandler---配置生成处理器)

### ServiceHandler.java
- **路径**: `com/datasophon/worker/handler/ServiceHandler.java`
- **行数**: 184 行
- **职责**: 服务操作业务逻辑
- **核心功能**:
  - 启动服务（状态验证）
  - 停止服务（状态验证）
  - 重启服务
  - Shell 脚本执行
- **Shell 检测**: 自动识别 bash/sh
- **详细文档**: [04-Handler处理器分析.md](./04-Handler处理器分析.md#四servicehandler---服务操作处理器)

## 五、Strategy 层 (31个文件)

### 5.1 策略基类和上下文 (3个)

#### ServiceRoleStrategy.java
- **路径**: `com/datasophon/worker/strategy/ServiceRoleStrategy.java`
- **行数**: 28 行
- **类型**: 接口
- **职责**: 定义策略接口
- **详细文档**: [05-Strategy策略模式详解.md](./05-Strategy策略模式详解.md#22-核心类分析)

#### AbstractHandlerStrategy.java
- **路径**: `com/datasophon/worker/strategy/AbstractHandlerStrategy.java`
- **行数**: 43 行
- **类型**: 抽象基类
- **职责**: 提供公共的日志记录器
- **详细文档**: [05-Strategy策略模式详解.md](./05-Strategy策略模式详解.md#22-核心类分析)

#### ServiceRoleStrategyContext.java
- **路径**: `com/datasophon/worker/strategy/ServiceRoleStrategyContext.java`
- **行数**: 68 行
- **类型**: 工厂类
- **职责**: 策略注册和查找
- **注册数量**: 24个策略
- **详细文档**: [05-Strategy策略模式详解.md](./05-Strategy策略模式详解.md#22-核心类分析)

### 5.2 HDFS 服务策略 (4个)

#### NameNodeHandlerStrategy.java
- **路径**: `com/datasophon/worker/strategy/NameNodeHandlerStrategy.java`
- **行数**: 115 行
- **职责**: NameNode 启动策略
- **特殊处理**: format、bootstrapStandby、Ranger 插件、Kerberos
- **详细文档**: [05-Strategy策略模式详解.md](./05-Strategy策略模式详解.md#23-hdfs-服务策略-4个)

#### DataNodeHandlerStrategy.java
- **路径**: `com/datasophon/worker/strategy/DataNodeHandlerStrategy.java`
- **行数**: 80 行
- **职责**: DataNode 启动策略
- **特殊处理**: Ranger 插件、Kerberos

#### JournalNodeHandlerStrategy.java
- **路径**: `com/datasophon/worker/strategy/JournalNodeHandlerStrategy.java`
- **行数**: 79 行
- **职责**: JournalNode 启动策略
- **特殊处理**: zkfc -formatZK、Kerberos

#### ZKFCHandlerStrategy.java
- **路径**: `com/datasophon/worker/strategy/ZKFCHandlerStrategy.java`
- **行数**: 63 行
- **职责**: ZKFC 启动策略
- **特殊处理**: Kerberos

### 5.3 YARN 服务策略 (3个)

#### ResourceManagerHandlerStrategy.java
- **路径**: `com/datasophon/worker/strategy/ResourceManagerHandlerStrategy.java`
- **行数**: 60 行
- **职责**: ResourceManager 启动策略

#### NodeManagerHandlerStrategy.java
- **路径**: `com/datasophon/worker/strategy/NodeManagerHandlerStrategy.java`
- **行数**: 56 行
- **职责**: NodeManager 启动策略

#### HistoryServerHandlerStrategy.java
- **路径**: `com/datasophon/worker/strategy/HistoryServerHandlerStrategy.java`
- **行数**: 60 行
- **职责**: MapReduce HistoryServer 启动策略

### 5.4 其他大数据服务策略 (17个)

| 文件名 | 行数 | 服务 | 核心职责 |
|--------|------|------|---------|
| ZkServerHandlerStrategy.java | 58 | ZooKeeper | ZK 启动，myid 配置 |
| KafkaHandlerStrategy.java | 51 | Kafka | Kafka 启动，Kerberos JAAS |
| HbaseHandlerStrategy.java | 83 | HBase | HBase 启动，HDFS 依赖检查 |
| HiveServer2HandlerStrategy.java | 115 | Hive | HiveServer2 启动，Metastore、Ranger |
| RangerAdminHandlerStrategy.java | 56 | Ranger | Ranger 初始化，数据库 setup |
| Krb5KdcHandlerStrategy.java | 61 | Kerberos | KDC 启动，kdc.conf 配置 |
| KAdminHandlerStrategy.java | 51 | Kerberos | KAdmin 启动 |
| FEHandlerStrategy.java | 88 | StarRocks/Doris | FE 启动，元数据初始化 |
| FEObserverHandlerStrategy.java | 83 | Doris | FEObserver 启动 |
| BEHandlerStrategy.java | 70 | StarRocks/Doris | BE 启动 |
| TezServerHandlerStrategy.java | 127 | TEZ | Tez 启动，YARN 集成 |
| KyuubiServerHandlerStrategy.java | 61 | Kyuubi | Kyuubi 启动，Spark/Hive 集成 |
| FlinkHandlerStrategy.java | 46 | Flink | Flink 客户端配置 |
| DSMasterHandlerStrategy.java | 120 | DolphinScheduler | DS Master 启动，数据库初始化 |

**详细文档**: [05-Strategy策略模式详解.md](./05-Strategy策略模式详解.md#25-其他服务策略概览)

### 5.5 资源操作策略 (7个)

#### ResourceStrategy.java
- **路径**: `com/datasophon/worker/strategy/resource/ResourceStrategy.java`
- **行数**: 20 行
- **类型**: 抽象基类
- **职责**: 定义资源策略接口
- **详细文档**: [05-Strategy策略模式详解.md](./05-Strategy策略模式详解.md#三资源操作策略详解)

#### ReplaceStrategy.java
- **路径**: `com/datasophon/worker/strategy/resource/ReplaceStrategy.java`
- **行数**: 35 行
- **职责**: 替换文件中的占位符
- **使用场景**: 替换 ${JAVA_HOME}、${USER} 等

#### DownloadStrategy.java
- **路径**: `com/datasophon/worker/strategy/resource/DownloadStrategy.java`
- **行数**: 51 行
- **职责**: 从 HTTP URL 下载文件
- **使用场景**: 下载 MySQL JDBC 驱动、第三方 JAR

#### LinkStrategy.java
- **路径**: `com/datasophon/worker/strategy/resource/LinkStrategy.java`
- **行数**: 39 行
- **职责**: 创建符号链接
- **使用场景**: ln -s /opt/app /opt/link

#### AppendLineStrategy.java
- **路径**: `com/datasophon/worker/strategy/resource/AppendLineStrategy.java`
- **行数**: 40 行
- **职责**: 向文件追加内容
- **使用场景**: 添加环境变量到 bashrc

#### ShellStrategy.java
- **路径**: `com/datasophon/worker/strategy/resource/ShellStrategy.java`
- **行数**: 28 行
- **职责**: 执行 Shell 脚本
- **使用场景**: 初始化数据库、创建配置文件

#### EmptyStrategy.java
- **路径**: `com/datasophon/worker/strategy/resource/EmptyStrategy.java`
- **行数**: 14 行
- **职责**: 空操作（占位符）

## 六、Utils 层 (6个文件)

### ActorUtils.java
- **路径**: `com/datasophon/worker/utils/ActorUtils.java`
- **行数**: 36 行
- **职责**: Akka Actor 工具类
- **核心功能**: 获取远程 Actor 引用
- **超时**: 30 秒
- **详细文档**: [06-Utils工具类详解.md](./06-Utils工具类详解.md#二actorutils---actor-工具类)

### FreemakerUtils.java
- **路径**: `com/datasophon/worker/utils/FreemakerUtils.java`
- **行数**: 188 行
- **职责**: FreeMarker 模板渲染工具
- **支持格式**: XML, Properties, Prometheus, Custom
- **核心功能**: 生成配置文件、支持扩展模板路径
- **详细文档**: [06-Utils工具类详解.md](./06-Utils工具类详解.md#三freemakerutils---模板渲染工具)

### KerberosUtils.java
- **路径**: `com/datasophon/worker/utils/KerberosUtils.java`
- **行数**: 70 行
- **职责**: Kerberos 认证工具
- **核心功能**: 
  - 下载 keytab 文件
  - 创建 keytab 目录
  - 设置权限 (770)
- **详细文档**: [06-Utils工具类详解.md](./06-Utils工具类详解.md#四kerberosutils---kerberos-工具类)

### UnixUtils.java
- **路径**: `com/datasophon/worker/utils/UnixUtils.java`
- **行数**: 97 行
- **职责**: Unix 用户/组管理工具
- **核心功能**:
  - 创建/删除用户
  - 创建/删除组
  - 检查用户/组是否存在
- **详细文档**: [06-Utils工具类详解.md](./06-Utils工具类详解.md#五unixutils---unix-用户管理工具)

### FileUtils.java
- **路径**: `com/datasophon/worker/utils/FileUtils.java`
- **行数**: 63 行
- **职责**: 文件操作工具
- **核心功能**: 读取文件最后 N 行
- **算法**: RandomAccessFile 反向扫描
- **详细文档**: [06-Utils工具类详解.md](./06-Utils工具类详解.md#六fileutils---文件工具类)

### TaskConstants.java
- **路径**: `com/datasophon/worker/utils/TaskConstants.java`
- **行数**: 111 行
- **职责**: 任务相关常量定义
- **包含**: 正则表达式、特殊符号、日志常量、退出码等
- **详细文档**: [06-Utils工具类详解.md](./06-Utils工具类详解.md#七taskconstants---任务常量)

## 七、Log 层 (2个文件)

### TaskLogDiscriminator.java
- **路径**: `com/datasophon/worker/log/TaskLogDiscriminator.java`
- **行数**: 83 行
- **类型**: Logback Discriminator
- **职责**: 日志分类器，根据 Logger 名称路由日志
- **核心功能**: 动态创建日志文件，任务级别隔离
- **详细文档**: [07-Log日志系统分析.md](./07-Log日志系统分析.md#二tasklogdiscriminator---日志分类器)

### TaskLogFilter.java
- **路径**: `com/datasophon/worker/log/TaskLogFilter.java`
- **行数**: 64 行
- **类型**: Logback Filter
- **职责**: 日志过滤器，控制日志级别
- **核心功能**: 任务日志全量记录，系统日志精简
- **详细文档**: [07-Log日志系统分析.md](./07-Log日志系统分析.md#三tasklogfilter---日志过滤器)

## 八、快速导航

### 按职责分类查找

**① 服务安装相关**:
- InstallServiceActor
- InstallServiceHandler
- ResourceStrategy 系列 (7个)

**② 服务配置相关**:
- ConfigureServiceActor
- ConfigureServiceHandler
- FreemakerUtils

**③ 服务启动相关**:
- StartServiceActor
- ServiceHandler
- ServiceRoleStrategy 系列 (24个)

**④ 日志管理相关**:
- LogActor
- TaskLogDiscriminator
- TaskLogFilter
- FileUtils

**⑤ 用户管理相关**:
- UnixUserActor
- UnixGroupActor
- UnixUtils

**⑥ Kerberos 认证相关**:
- KerberosActor
- KerberosUtils
- 各个 Strategy 中的 Kerberos 处理

### 按技术栈分类查找

**Akka Actor**:
- 所有 Actor 文件 (19个)
- ActorUtils
- SupervisorFunction

**FreeMarker 模板**:
- ConfigureServiceHandler
- FreemakerUtils

**Logback 日志**:
- TaskLogDiscriminator
- TaskLogFilter

**Shell 脚本**:
- ServiceHandler
- ExecuteCmdActor
- 各个 Strategy

## 九、文档导航

### 核心文档

1. [Worker模块概述.md](./01-Worker模块概述.md) - 模块整体介绍
2. [WorkerApplicationServer启动类详解.md](./02-WorkerApplicationServer启动类详解.md) - 启动流程
3. [Actor层完整分析.md](./03-Actor层完整分析.md) - Actor 层详解
4. [Handler处理器分析.md](./04-Handler处理器分析.md) - Handler 层详解
5. [Strategy策略模式详解.md](./05-Strategy策略模式详解.md) - Strategy 层详解
6. [Utils工具类详解.md](./06-Utils工具类详解.md) - Utils 层详解
7. [Log日志系统分析.md](./07-Log日志系统分析.md) - 日志系统详解
8. [Worker模块速查手册.md](./09-Worker模块速查手册.md) - 快速参考

### 相关文档

- [整体架构设计](../overview/03-整体架构设计.md)
- [核心业务流程](../overview/04-核心业务流程.md)
- [Common模块速查手册](../common/10-Common模块速查手册.md)

## 十、总结

### 文件覆盖率
- **启动类**: 1/1 (100%)
- **Actor 层**: 19/19 (100%)
- **Handler 层**: 3/3 (100%)
- **Strategy 层**: 31/31 (100%)
- **Utils 层**: 6/6 (100%)
- **Log 层**: 2/2 (100%)
- **总计**: 62/62 (100%)

### 文档特色
- ✅ 每个文件都有详细说明
- ✅ 完整的职责描述
- ✅ 详细文档链接
- ✅ 快速导航支持
- ✅ 多维度分类

---

**维护说明**: 本索引文档会随着 Worker 模块的变更而更新，请保持同步。
