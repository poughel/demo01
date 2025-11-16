# DataSophon Worker 启动类详解 - WorkerApplicationServer

## 一、文件基本信息

**文件路径**: `datasophon-worker/src/main/java/com/datasophon/worker/WorkerApplicationServer.java`  
**包名**: `com.datasophon.worker`  
**类型**: 主启动类  
**职责**: Worker 节点的应用程序启动入口，负责初始化 Actor 系统、启动监控、创建系统用户、向 Master 注册等核心功能

## 二、类结构概览

### 2.1 依赖导入

```java
// DataSophon 内部依赖
import com.datasophon.common.Constants;
import com.datasophon.common.cache.CacheUtils;
import com.datasophon.common.lifecycle.ServerLifeCycleManager;
import com.datasophon.common.model.StartWorkerMessage;
import com.datasophon.common.utils.*;
import com.datasophon.worker.actor.*;
import com.datasophon.worker.utils.*;

// 外部框架依赖
import akka.actor.*;              // Akka Actor 框架
import com.typesafe.config.*;     // Typesafe Config 配置库
import com.alibaba.fastjson.*;    // FastJSON 解析库
```

**依赖说明**:
- **Akka Actor**: 分布式并发编程模型，用于 Worker 与 Master 的通信
- **Typesafe Config**: 配置管理，支持 HOCON 格式
- **FastJSON**: JSON 序列化/反序列化工具

### 2.2 类常量定义

```java
private static final String USER_DIR = "user.dir";        // JVM 当前工作目录
private static final String MASTER_HOST = "masterHost";   // Master 主机地址配置键
private static final String WORKER = "worker";            // Worker Actor 名称
private static final String SH = "sh";                    // Shell 脚本执行器
private static final String NODE = "node";                // Node Exporter 服务标识
private static final String HADOOP = "hadoop";            // Hadoop 用户组名称
```

**设计思想**: 将魔法值（Magic String）提取为常量，提升代码可维护性和可读性。

## 三、核心方法详解

### 3.1 main() 方法 - 程序入口

```java
public static void main(String[] args) throws UnknownHostException {
    String hostname = InetAddress.getLocalHost().getHostName();
    String workDir = System.getProperty(USER_DIR);
    String masterHost = PropertyUtils.getString(MASTER_HOST);
    String cpuArchitecture = ShellUtils.getCpuArchitecture();
    
    CacheUtils.put(Constants.HOSTNAME, hostname);
    
    // 1. 初始化 Actor 系统
    ActorSystem system = initActor(hostname);
    ActorUtils.setActorSystem(system);
    
    // 2. 订阅远程事件
    subscribeRemoteEvent(system);
    
    // 3. 启动 Node Exporter 监控
    startNodeExporter(workDir, cpuArchitecture);
    
    // 4. 初始化用户映射
    Map<String, String> userMap = new HashMap(16);
    initUserMap(userMap);
    
    // 5. 创建系统用户
    createDefaultUser(userMap);
    
    // 6. 向 Master 注册
    tellToMaster(hostname, workDir, masterHost, cpuArchitecture, system);
    logger.info("start worker");
    
    // 7. 注册 JVM 关闭钩子
    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
        if (!ServerLifeCycleManager.isStopped()) {
            close("WorkerServer shutdown hook");
        }
    }));
}
```

**执行流程**:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Worker 启动流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ① 获取主机信息                                                  │
│     - hostname: 主机名                                           │
│     - workDir: 工作目录                                          │
│     - masterHost: Master 地址                                    │
│     - cpuArchitecture: CPU 架构 (x86_64/aarch64)                │
│                                                                  │
│  ② 初始化 Akka Actor 系统                                        │
│     - 创建 ActorSystem                                           │
│     - 创建 WorkerActor（主 Actor）                               │
│     - 设置全局 ActorSystem 引用                                  │
│                                                                  │
│  ③ 订阅 Akka 远程事件                                            │
│     - AssociationErrorEvent: 连接错误                            │
│     - AssociatedEvent: 连接建立                                  │
│     - DisassociatedEvent: 连接断开                               │
│                                                                  │
│  ④ 启动 Node Exporter 监控进程                                   │
│     - 根据 CPU 架构选择对应的二进制文件                          │
│     - 执行 control.sh restart node                               │
│                                                                  │
│  ⑤ 创建系统用户和组                                              │
│     - hadoop 组: hdfs, yarn, hive, mapred, hbase, kyuubi, flink │
│     - elastic 组: elastic                                        │
│                                                                  │
│  ⑥ 向 Master 注册                                                │
│     - 收集主机信息（CPU、内存、磁盘等）                          │
│     - 发送 StartWorkerMessage 到 Master                          │
│                                                                  │
│  ⑦ 注册 JVM 关闭钩子                                             │
│     - 确保进程退出时停止 Node Exporter                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**关键点**:
1. **主机名缓存**: 将 hostname 存入缓存，供其他组件使用
2. **CPU 架构感知**: 支持 x86_64 和 aarch64 架构，动态选择二进制文件
3. **优雅关闭**: 注册 Shutdown Hook 确保资源正确释放

### 3.2 initActor() - 初始化 Actor 系统

```java
private static ActorSystem initActor(String hostname) {
    Config config = ConfigFactory.parseString("akka.remote.netty.tcp.hostname=" + hostname);
    ActorSystem system = ActorSystem.create(
        Constants.DATASOPHON, 
        config.withFallback(ConfigFactory.load())
    );
    system.actorOf(Props.create(WorkerActor.class), WORKER);
    return system;
}
```

**功能说明**:
1. **动态配置绑定地址**: 将 Akka Remoting 绑定到当前主机的 hostname
2. **加载配置文件**: 通过 `ConfigFactory.load()` 加载 `application.conf`
3. **创建 WorkerActor**: 顶层 Actor，负责创建和管理子 Actor

**配置优先级**:
```
程序指定的配置 > application.conf > reference.conf
```

**Akka 配置示例** (application.yml):
```yaml
akka:
  remote:
    netty.tcp:
      hostname: ${hostname}  # 动态注入
      port: 2552             # Worker 端口
  actor:
    provider: remote
```

### 3.3 subscribeRemoteEvent() - 订阅远程事件

```java
private static void subscribeRemoteEvent(ActorSystem system) {
    ActorRef remoteEventActor = system.actorOf(
        Props.create(RemoteEventActor.class), 
        "remoteEventActor"
    );
    EventStream eventStream = system.eventStream();
    eventStream.subscribe(remoteEventActor, AssociationErrorEvent.class);
    eventStream.subscribe(remoteEventActor, AssociatedEvent.class);
    eventStream.subscribe(remoteEventActor, DisassociatedEvent.class);
}
```

**事件类型说明**:

| 事件类型 | 触发时机 | 处理意义 |
|---------|---------|---------|
| `AssociatedEvent` | Worker 与 Master 建立连接成功 | 记录连接成功日志，可触发健康检查 |
| `DisassociatedEvent` | Worker 与 Master 连接断开 | 尝试重连，记录断连日志 |
| `AssociationErrorEvent` | 连接建立过程中出现错误 | 记录错误信息，触发告警 |

**设计模式**: **观察者模式**，RemoteEventActor 作为观察者监听 Akka 的网络事件。

### 3.4 startNodeExporter() - 启动监控

```java
private static void startNodeExporter(String workDir, String cpuArchitecture) {
    operateNodeExporter(workDir, cpuArchitecture, "restart");
}

private static void operateNodeExporter(String workDir, String cpuArchitecture, String operate) {
    ArrayList<String> commands = new ArrayList<>();
    commands.add(SH);
    if (Constants.X86_64.equals(cpuArchitecture)) {
        commands.add(workDir + "/node/x86/control.sh");
    } else {
        commands.add(workDir + "/node/arm/control.sh");
    }
    commands.add(operate);
    commands.add(NODE);
    ShellUtils.execWithStatus(Constants.INSTALL_PATH, commands, 60L, logger);
}
```

**目录结构**:
```
${workDir}/
├── node/
│   ├── x86/
│   │   ├── control.sh          # x86_64 架构控制脚本
│   │   └── node_exporter       # x86_64 二进制文件
│   └── arm/
│       ├── control.sh          # aarch64 架构控制脚本
│       └── node_exporter       # aarch64 二进制文件
```

**Node Exporter 简介**:
- **作用**: Prometheus 官方的主机指标采集器
- **采集指标**: CPU、内存、磁盘、网络、文件系统等
- **默认端口**: 9100
- **数据格式**: Prometheus Text-Based Exposition Format

**执行命令示例**:
```bash
sh /opt/datasophon/worker/node/x86/control.sh restart node
```

### 3.5 initUserMap() - 初始化用户映射

```java
private static void initUserMap(Map<String, String> userMap) {
    userMap.put("hdfs", HADOOP);      // HDFS 文件系统
    userMap.put("yarn", HADOOP);      // YARN 资源管理
    userMap.put("hive", HADOOP);      // Hive 数据仓库
    userMap.put("mapred", HADOOP);    // MapReduce 计算
    userMap.put("hbase", HADOOP);     // HBase NoSQL
    userMap.put("kyuubi", HADOOP);    // Kyuubi SQL 引擎
    userMap.put("flink", HADOOP);     // Flink 流计算
    userMap.put("elastic", "elastic"); // Elasticsearch
}
```

**设计思想**:
1. **用户隔离**: 不同组件使用不同的系统用户运行，提升安全性
2. **权限最小化**: 每个用户仅拥有运行对应组件所需的最小权限
3. **统一归属**: 大数据组件统一归属 `hadoop` 组，便于权限管理

**用户组设计**:
```
hadoop 组
├── hdfs (HDFS 守护进程用户)
├── yarn (YARN 守护进程用户)
├── hive (Hive 服务用户)
├── mapred (MapReduce 历史服务用户)
├── hbase (HBase 服务用户)
├── kyuubi (Kyuubi 服务用户)
└── flink (Flink 服务用户)

elastic 组
└── elastic (Elasticsearch 用户)
```

### 3.6 createDefaultUser() - 创建系统用户

```java
private static void createDefaultUser(Map<String, String> userMap) {
    for (Map.Entry<String, String> entry : userMap.entrySet()) {
        String user = entry.getKey();
        String group = entry.getValue();
        if (!UnixUtils.isGroupExists(group)) {
            UnixUtils.createUnixGroup(group);
        }
        UnixUtils.createUnixUser(user, group, null);
    }
}
```

**执行的系统命令** (等效):
```bash
# 1. 创建 hadoop 组（如果不存在）
groupadd hadoop

# 2. 创建用户并加入组
useradd -g hadoop -m hdfs
useradd -g hadoop -m yarn
useradd -g hadoop -m hive
useradd -g hadoop -m mapred
useradd -g hadoop -m hbase
useradd -g hadoop -m kyuubi
useradd -g hadoop -m flink

# 3. 创建 elastic 组和用户
groupadd elastic
useradd -g elastic -m elastic
```

**安全考虑**:
- **幂等性**: `UnixUtils.createUnixUser()` 内部会检查用户是否已存在，避免重复创建
- **Home 目录**: 为每个用户创建独立的 Home 目录 (`/home/{username}`)
- **无密码登录**: 创建的系统用户默认不允许密码登录，只能通过 sudo 切换

### 3.7 tellToMaster() - 向 Master 注册

```java
private static void tellToMaster(
    String hostname, String workDir, String masterHost, 
    String cpuArchitecture, ActorSystem system
) {
    // 1. 获取 Master 的 workerStartActor 引用
    ActorSelection workerStartActor = system.actorSelection(
        "akka.tcp://datasophon@" + masterHost + ":2551/user/workerStartActor"
    );
    
    // 2. 执行主机信息采集脚本
    ExecResult result = ShellUtils.exceShell(workDir + "/script/host-info-collect.sh");
    if (!result.getExecResult()) {
        logger.error("host info collect error:{}", result.getExecErrOut());
    } else {
        logger.info("host info collect success:{}", result.getExecOut());
    }
    
    // 3. 解析脚本输出为消息对象
    StartWorkerMessage startWorkerMessage = 
        JSONObject.parseObject(result.getExecOut(), StartWorkerMessage.class);
    startWorkerMessage.setCpuArchitecture(cpuArchitecture);
    startWorkerMessage.setClusterId(PropertyUtils.getInt("clusterId"));
    startWorkerMessage.setHostname(hostname);
    
    // 4. 发送注册消息到 Master
    workerStartActor.tell(startWorkerMessage, ActorRef.noSender());
}
```

**Actor 路径解析**:
```
akka.tcp://datasophon@192.168.1.100:2551/user/workerStartActor
│          │           │                │    │
│          │           │                │    └─ Actor 名称
│          │           │                └────── 用户创建的 Actor 路径
│          │           └─────────────────────── 端口号
│          └─────────────────────────────────── Master 主机地址
└────────────────────────────────────────────── 协议和 ActorSystem 名称
```

**主机信息采集脚本** (host-info-collect.sh):
```bash
#!/bin/bash
# 输出 JSON 格式的主机信息
cat <<EOF
{
  "cpuCores": $(nproc),
  "totalMem": $(free -m | awk 'NR==2{print $2}'),
  "totalDisk": $(df -m / | awk 'NR==2{print $2}'),
  "usedMem": $(free -m | awk 'NR==2{print $3}'),
  "usedDisk": $(df -m / | awk 'NR==2{print $3}'),
  "averageLoad": $(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}'),
  "diskNum": $(lsblk -d | grep disk | wc -l),
  "cpuRate": $(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
}
EOF
```

**StartWorkerMessage 数据结构**:
```java
public class StartWorkerMessage {
    private String hostname;          // 主机名
    private Integer clusterId;        // 集群 ID
    private String cpuArchitecture;   // CPU 架构
    private Integer cpuCores;         // CPU 核心数
    private Long totalMem;            // 总内存 (MB)
    private Long totalDisk;           // 总磁盘 (MB)
    private Long usedMem;             // 已用内存 (MB)
    private Long usedDisk;            // 已用磁盘 (MB)
    private String averageLoad;       // 平均负载
    private Integer diskNum;          // 磁盘数量
    private Double cpuRate;           // CPU 使用率 (%)
}
```

### 3.8 close() - 关闭 Worker

```java
public static void close(String cause) {
    stopNodeExporter();
    logger.info("Worker server stopped");
}

private static void stopNodeExporter() {
    String workDir = System.getProperty(USER_DIR);
    String cpuArchitecture = ShellUtils.getCpuArchitecture();
    operateNodeExporter(workDir, cpuArchitecture, "stop");
}
```

**关闭流程**:
```
JVM Shutdown Hook 触发
         │
         ▼
检查 ServerLifeCycleManager.isStopped()
         │
         ▼ (未停止)
调用 close("WorkerServer shutdown hook")
         │
         ▼
停止 Node Exporter (control.sh stop node)
         │
         ▼
记录日志: "Worker server stopped"
```

**优雅关闭的重要性**:
1. **释放端口**: 停止 Node Exporter，释放 9100 端口
2. **保存状态**: 如果有未完成的任务，可以在这里保存状态
3. **通知 Master**: 可以扩展为向 Master 发送 Worker 下线通知

## 四、技术亮点与设计模式

### 4.1 Actor 模型

**优势**:
1. **消息驱动**: 通过消息传递避免共享状态，减少并发问题
2. **位置透明**: Actor 可以在本地或远程，代码无需改变
3. **容错性**: Actor 异常崩溃后可自动重启，提升系统稳定性

**Worker 中的 Actor 层次结构**:
```
ActorSystem (datasophon)
    │
    └── WorkerActor (worker)
            ├── InstallServiceActor
            ├── ConfigureServiceActor
            ├── StartServiceActor
            ├── StopServiceActor
            ├── RestartServiceActor
            ├── LogActor
            ├── ExecuteCmdActor
            ├── FileOperateActor
            ├── AlertConfigActor
            ├── UnixUserActor
            ├── UnixGroupActor
            ├── KerberosActor
            ├── NMStateActor
            ├── RMStateActor
            └── PingActor
```

### 4.2 配置管理

使用 **Typesafe Config** 的优势:
1. **HOCON 语法**: 比 JSON 更灵活，支持注释、变量替换
2. **配置合并**: 支持多个配置文件的层级合并
3. **类型安全**: 提供类型化的 API，减少配置错误

**配置示例** (application.conf):
```hocon
akka {
  remote {
    netty.tcp {
      hostname = ${hostname}  # 运行时注入
      port = 2552
      maximum-frame-size = 10485760  # 10MB
    }
  }
  
  actor {
    provider = remote
    default-dispatcher {
      type = Dispatcher
      executor = "fork-join-executor"
      fork-join-executor {
        parallelism-min = 2
        parallelism-max = 8
      }
    }
  }
}
```

### 4.3 资源初始化策略

**初始化顺序的设计考虑**:
```
1. Akka Actor 系统 → 确保通信能力可用
2. 远程事件订阅 → 监控网络状态
3. Node Exporter → 开始采集监控指标
4. 系统用户创建 → 为服务运行做准备
5. 向 Master 注册 → 告知 Master 我已就绪
```

**为什么先启动 Node Exporter？**
- Master 可能会立即下发监控配置任务
- 提前启动确保监控指标不缺失

### 4.4 错误处理与容错

**容错机制**:
1. **主机信息采集失败**: 记录错误但不中断启动，使用默认值或稍后重试
2. **用户创建失败**: `UnixUtils` 内部已经处理了用户已存在的情况
3. **Node Exporter 启动失败**: 记录日志但不影响 Worker 主流程

**日志记录策略**:
```java
// 成功日志
logger.info("start worker");
logger.info("host info collect success:{}", result.getExecOut());

// 错误日志
logger.error("host info collect error:{}", result.getExecErrOut());
```

## 五、性能考虑

### 5.1 内存优化

**HashMap 初始容量设置**:
```java
Map<String, String> userMap = new HashMap(16);
```
- 预设容量为 16，避免动态扩容带来的性能损耗
- 实际存储 8 个键值对，负载因子 0.75，16 容量刚好

### 5.2 并发处理

**Akka 的并发优势**:
- **非阻塞 I/O**: 所有操作基于消息传递，不会阻塞主线程
- **线程池管理**: Akka 内部使用线程池，避免频繁创建销毁线程

### 5.3 启动速度优化

**并行初始化潜力**:
- Node Exporter 启动和用户创建可以并行执行（当前串行）
- 主机信息采集脚本可以优化为异步执行

**优化建议**:
```java
// 使用 CompletableFuture 并行执行
CompletableFuture<Void> nodeExporterFuture = 
    CompletableFuture.runAsync(() -> startNodeExporter(workDir, cpuArchitecture));
CompletableFuture<Void> userCreationFuture = 
    CompletableFuture.runAsync(() -> createDefaultUser(userMap));

// 等待全部完成
CompletableFuture.allOf(nodeExporterFuture, userCreationFuture).join();
```

## 六、常见问题与排查

### 6.1 Worker 无法启动

**可能原因**:
1. **端口被占用**: 2552 端口已被其他进程使用
2. **Master 地址错误**: `masterHost` 配置错误
3. **权限不足**: 无法创建系统用户

**排查命令**:
```bash
# 检查端口占用
netstat -tuln | grep 2552

# 检查配置
cat conf/worker.properties | grep masterHost

# 检查日志
tail -f logs/worker.log
```

### 6.2 Node Exporter 启动失败

**可能原因**:
1. **架构不匹配**: ARM 机器使用了 x86 二进制文件
2. **权限不足**: control.sh 没有执行权限
3. **端口冲突**: 9100 端口被占用

**解决方案**:
```bash
# 添加执行权限
chmod +x node/x86/control.sh
chmod +x node/x86/node_exporter

# 检查端口
netstat -tuln | grep 9100

# 手动启动测试
./node/x86/control.sh start node
```

### 6.3 向 Master 注册失败

**可能原因**:
1. **网络不通**: Worker 无法连接到 Master 的 2551 端口
2. **ActorSystem 名称不匹配**: Master 的 ActorSystem 名称不是 `datasophon`
3. **Actor 路径错误**: Master 没有 `workerStartActor`

**排查方法**:
```bash
# 测试网络连通性
telnet ${masterHost} 2551

# 查看 Worker 日志中的注册信息
grep "tellToMaster" logs/worker.log

# 查看 Master 日志中是否收到注册消息
grep "StartWorkerMessage" logs/api-server.log
```

## 七、扩展建议

### 7.1 健康检查机制

建议添加定期健康检查，向 Master 发送心跳：

```java
// 使用 Akka Scheduler 定期发送心跳
system.scheduler().schedule(
    Duration.create(10, TimeUnit.SECONDS),    // 初始延迟
    Duration.create(30, TimeUnit.SECONDS),    // 间隔时间
    () -> {
        // 发送心跳消息
        workerStartActor.tell(new HeartbeatMessage(hostname), ActorRef.noSender());
    },
    system.dispatcher()
);
```

### 7.2 配置热更新

支持配置文件变更后无需重启：

```java
// 监听配置文件变化
WatchService watchService = FileSystems.getDefault().newWatchService();
Paths.get("conf").register(watchService, StandardWatchEventKinds.ENTRY_MODIFY);

// 配置变更后重新加载
PropertyUtils.reloadConfig();
```

### 7.3 优雅停机增强

完善关闭流程，通知 Master 下线：

```java
public static void close(String cause) {
    logger.info("Worker is shutting down, reason: {}", cause);
    
    // 1. 通知 Master 下线
    notifyMasterShutdown();
    
    // 2. 等待正在执行的任务完成（最多 30 秒）
    waitForTasksCompletion(30);
    
    // 3. 停止 Node Exporter
    stopNodeExporter();
    
    // 4. 关闭 ActorSystem
    ActorUtils.getActorSystem().terminate();
    
    logger.info("Worker server stopped gracefully");
}
```

## 八、总结

### 8.1 核心职责

`WorkerApplicationServer` 是 Worker 节点的启动入口，主要完成以下职责：

1. **初始化通信**: 创建 Akka ActorSystem，建立与 Master 的通信通道
2. **启动监控**: 启动 Node Exporter，为 Prometheus 提供监控数据
3. **环境准备**: 创建系统用户和组，为服务运行准备环境
4. **注册上线**: 向 Master 发送主机信息，完成 Worker 注册

### 8.2 设计优点

1. **清晰的启动流程**: 各初始化步骤独立封装，职责明确
2. **架构适配**: 支持 x86_64 和 aarch64 两种 CPU 架构
3. **容错设计**: 部分环节失败不影响整体启动
4. **优雅关闭**: 通过 Shutdown Hook 确保资源正确释放

### 8.3 改进空间

1. **启动速度**: 可以通过并行初始化提升启动速度
2. **健康检查**: 缺少定期心跳机制，无法及时发现 Worker 异常
3. **配置验证**: 启动前应该验证必要的配置项是否正确
4. **优雅停机**: 当前仅停止 Node Exporter，未处理正在执行的任务

### 8.4 学习要点

1. **Akka Actor 的初始化和使用**
2. **Typesafe Config 的配置管理**
3. **系统用户的创建和权限管理**
4. **JVM Shutdown Hook 的使用场景**
5. **分布式系统中 Worker 的注册机制**

---

**相关文档**:
- [01-Worker模块概述.md](./01-Worker模块概述.md)
- [03-Actor层完整分析.md](./03-Actor层完整分析.md)
- [整体架构设计](../overview/03-整体架构设计.md)
