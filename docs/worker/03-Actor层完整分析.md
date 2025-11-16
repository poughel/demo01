# DataSophon Worker Actor 层完整分析

## 一、Actor 层架构概述

### 1.1 什么是 Actor 模型

**Actor 模型**是一种并发编程模型，由 Carl Hewitt 在 1973 年提出。在 DataSophon Worker 中，使用 Akka 框架实现 Actor 模型。

**核心特点**:
1. **消息驱动**: Actor 之间通过消息传递通信，避免共享状态
2. **单线程处理**: 每个 Actor 串行处理消息，无需考虑并发安全
3. **位置透明**: Actor 可以在本地或远程，使用方式相同
4. **容错性**: Actor 可以监督子 Actor，实现自动重启

### 1.2 Worker Actor 层次结构

```
ActorSystem: datasophon (Worker 端口: 2552)
    │
    └── WorkerActor (worker) ← 根 Actor，负责创建和监督所有子 Actor
            │
            ├── InstallServiceActor         ← 服务安装
            ├── ConfigureServiceActor        ← 配置生成
            ├── StartServiceActor            ← 服务启动
            ├── StopServiceActor             ← 服务停止
            ├── RestartServiceActor          ← 服务重启
            ├── CheckServiceStatusActor      ← 状态检查
            │
            ├── ExecuteCmdActor              ← 命令执行
            ├── LogActor                     ← 日志查询
            ├── FileOperateActor             ← 文件操作
            │
            ├── AlertConfigActor             ← 告警配置
            ├── UnixUserActor                ← 用户管理
            ├── UnixGroupActor               ← 组管理
            ├── KerberosActor                ← Kerberos 认证
            │
            ├── NMStateActor                 ← NodeManager 状态
            ├── RMStateActor                 ← ResourceManager 状态
            │
            └── PingActor                    ← 心跳检测

RemoteEventActor (独立) ← 监听 Akka 网络事件
```

### 1.3 Actor 分类

| 类别 | Actor 列表 | 职责说明 |
|------|-----------|----------|
| **服务生命周期管理** | InstallServiceActor<br>ConfigureServiceActor<br>StartServiceActor<br>StopServiceActor<br>RestartServiceActor<br>CheckServiceStatusActor | 负责大数据服务的安装、配置、启动、停止、重启、状态检查 |
| **通用操作** | ExecuteCmdActor<br>LogActor<br>FileOperateActor | 执行 Shell 命令、查询日志、文件读写操作 |
| **系统管理** | UnixUserActor<br>UnixGroupActor<br>KerberosActor<br>AlertConfigActor | 系统用户/组管理、Kerberos 认证、告警配置 |
| **状态监控** | NMStateActor<br>RMStateActor<br>PingActor | YARN 组件状态监控、心跳检测 |
| **网络事件** | RemoteEventActor | 监听 Akka Remoting 的网络事件 |

## 二、核心 Actor 详细分析

### 2.1 WorkerActor - 根 Actor

**文件路径**: `com/datasophon/worker/actor/WorkerActor.java`  
**行数**: 108 行  
**职责**: Worker 的根 Actor，负责创建和监督所有子 Actor

#### 源码分析

```java
public class WorkerActor extends UntypedActor {
    
    private static final Logger logger = LoggerFactory.getLogger(WorkerActor.class);
    
    @Override
    public void preStart() throws IOException {
        // 创建所有子 Actor
        ActorRef installServiceActor = getContext().actorOf(
            Props.create(InstallServiceActor.class),
            getActorRefName(InstallServiceActor.class)
        );
        // ... 创建其他 Actor
        
        // 监听所有子 Actor（监督模式）
        getContext().watch(installServiceActor);
        // ... 监听其他 Actor
    }
    
    @Override
    public void onReceive(Object message) throws Throwable {
        if (message instanceof String) {
            // 处理字符串消息
        } else if (message instanceof Terminated) {
            // 子 Actor 终止事件
            Terminated t = (Terminated) message;
            logger.info("find actor {} terminated", t.getActor());
        } else {
            unhandled(message);
        }
    }
    
    @Override
    public void preRestart(Throwable reason, Option<Object> message) {
        logger.info("worker actor restart by reason {}", reason.getMessage());
    }
}
```

#### 关键方法

**preStart()**: Actor 生命周期钩子，在 Actor 首次启动时调用
- **时机**: ActorSystem 创建 WorkerActor 时
- **作用**: 创建所有子 Actor，建立监督关系
- **监督**: 通过 `watch()` 监听子 Actor，当子 Actor 异常终止时接收 `Terminated` 消息

**onReceive(Object message)**: 消息处理方法
- **String**: 保留的字符串消息处理（当前为空实现）
- **Terminated**: 子 Actor 终止通知，记录日志

**preRestart()**: Actor 重启前钩子
- **时机**: Actor 因异常需要重启时
- **作用**: 记录重启原因，清理资源

#### Actor 命名规则

```java
private String getActorRefName(Class clazz) {
    return StringUtils.uncapitalize(clazz.getSimpleName());
}
```

**示例**:
- `InstallServiceActor.class` → `installServiceActor`
- `StartServiceActor.class` → `startServiceActor`

**Actor 路径**:
```
akka://datasophon/user/worker/installServiceActor
```

#### 监督策略

WorkerActor 使用 Akka 的默认监督策略：
- **Restart**: 重启失败的子 Actor
- **Resume**: 忽略异常，继续处理
- **Stop**: 停止子 Actor
- **Escalate**: 将异常上报给父 Actor

### 2.2 InstallServiceActor - 服务安装

**文件路径**: `com/datasophon/worker/actor/InstallServiceActor.java`  
**行数**: 87 行  
**职责**: 处理服务安装请求，下载、解压、配置服务包

#### 源码分析

```java
public class InstallServiceActor extends UntypedActor {
    
    private static final Logger logger = LoggerFactory.getLogger(InstallServiceActor.class);
    
    @Override
    public void onReceive(Object msg) throws Throwable {
        if (msg instanceof InstallServiceRoleCommand) {
            InstallServiceRoleCommand command = (InstallServiceRoleCommand) msg;
            ExecResult installResult = new ExecResult();
            
            InstallServiceHandler serviceHandler = new InstallServiceHandler(
                command.getFrameCode(),
                command.getServiceName(),
                command.getServiceRoleName()
            );
            
            logger.info("Start install package {}", command.getPackageName());
            
            // 特殊处理: Kerberos 需要通过 yum 安装
            if (command.getDecompressPackageName().contains("kerberos")) {
                ArrayList<String> commands = new ArrayList<>();
                commands.add("yum");
                commands.add("install");
                commands.add("-y");
                
                if (ServiceRoleType.MASTER == command.getServiceRoleType()) {
                    // KDC 服务端
                    commands.add("krb5-server");
                    commands.add("krb5-workstation");
                    commands.add("krb5-libs");
                } else {
                    // Kerberos 客户端
                    commands.add("krb5-workstation");
                    commands.add("krb5-libs");
                }
                
                ExecResult execResult = ShellUtils.execWithStatus(
                    Constants.INSTALL_PATH, commands, 180, logger
                );
                
                if (execResult.getExecResult()) {
                    installResult = serviceHandler.install(command);
                }
            } else {
                // 常规服务安装
                installResult = serviceHandler.install(command);
                
                // 创建软链接
                String appHome = Constants.INSTALL_PATH + Constants.SLASH + 
                                 command.getDecompressPackageName();
                String appLinkHome = Constants.INSTALL_PATH + Constants.SLASH + 
                                     StringUtils.lowerCase(command.getServiceName());
                
                if (!new File(appLinkHome).exists()) {
                    ShellUtils.exceShell("ln -s " + appHome + " " + appLinkHome);
                    logger.info("Create symbolic dir: {}", appLinkHome);
                }
            }
            
            // 返回安装结果
            getSender().tell(installResult, getSelf());
            logger.info("Install {} {}", 
                command.getPackageName(),
                installResult.getExecResult() ? "success" : "failed"
            );
        } else {
            unhandled(msg);
        }
    }
}
```

#### 安装流程

```
┌─────────────────────────────────────────────────────────────┐
│                    服务安装流程                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ① 接收 InstallServiceRoleCommand                           │
│     - packageName: hadoop-3.3.3.tar.gz                      │
│     - decompressPackageName: hadoop-3.3.3                   │
│     - packageMd5: abc123def456                              │
│                                                              │
│  ② 判断是否需要下载                                          │
│     - 检查本地文件是否存在                                   │
│     - 比对 MD5 值                                            │
│     - 如果需要，从 Master 下载                               │
│                                                              │
│  ③ 解压安装包                                                │
│     - tar -zxvf hadoop-3.3.3.tar.gz -C /opt/datasophon     │
│     - 解压到 /opt/datasophon/hadoop-3.3.3                   │
│                                                              │
│  ④ 执行资源策略                                              │
│     - ReplaceStrategy: 替换配置文件占位符                    │
│     - DownloadStrategy: 下载依赖文件                         │
│     - LinkStrategy: 创建软链接                               │
│     - ShellStrategy: 执行自定义脚本                          │
│                                                              │
│  ⑤ 设置权限                                                  │
│     - chown -R hdfs:hadoop /opt/datasophon/hadoop-3.3.3    │
│     - chmod -R 775 /opt/datasophon/hadoop-3.3.3            │
│                                                              │
│  ⑥ 创建软链接                                                │
│     - ln -s /opt/datasophon/hadoop-3.3.3 /opt/datasophon/hadoop │
│                                                              │
│  ⑦ 返回安装结果                                              │
│     - ExecResult.execResult = true/false                    │
│     - ExecResult.execOut = 详细信息                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### Kerberos 特殊处理

**为什么 Kerberos 需要 yum 安装？**
- Kerberos 依赖系统库，tar 包无法包含所有依赖
- 需要与系统的 PAM、GSSAPI 集成
- yum 安装可以自动处理依赖关系

**安装的包**:
- **krb5-server**: KDC 服务端（仅 Master 节点）
- **krb5-workstation**: Kerberos 客户端工具（kadmin、kinit）
- **krb5-libs**: Kerberos 库文件

### 2.3 ConfigureServiceActor - 配置生成

**文件路径**: `com/datasophon/worker/actor/ConfigureServiceActor.java`  
**行数**: 55 行  
**职责**: 生成服务配置文件，替换配置占位符

#### 源码分析

```java
public class ConfigureServiceActor extends UntypedActor {
    
    private static final Logger logger = LoggerFactory.getLogger(ConfigureServiceActor.class);
    
    @Override
    public void onReceive(Object msg) throws Throwable {
        if (msg instanceof GenerateServiceConfigCommand) {
            GenerateServiceConfigCommand command = (GenerateServiceConfigCommand) msg;
            
            logger.info("start configure {}", command.getServiceName());
            
            ConfigureServiceHandler serviceHandler = new ConfigureServiceHandler(
                command.getServiceName(),
                command.getServiceRoleName()
            );
            
            ExecResult startResult = serviceHandler.configure(
                command.getConfigFileMap(),      // 配置文件模板
                command.getDecompressPackageName(), // 解压目录
                command.getClusterId(),          // 集群 ID
                command.getMyid(),               // ZooKeeper myid
                command.getServiceRoleName(),    // 服务角色名
                command.getRunAs()               // 运行用户
            );
            
            getSender().tell(startResult, getSelf());
            
            logger.info("{} configure result {}", 
                command.getServiceName(),
                startResult.getExecResult() ? "success" : "failed"
            );
        } else {
            unhandled(msg);
        }
    }
}
```

#### 配置生成流程

**GenerateServiceConfigCommand 结构**:
```java
public class GenerateServiceConfigCommand {
    private String serviceName;              // 服务名: HDFS
    private String serviceRoleName;          // 角色名: NameNode
    private String decompressPackageName;    // hadoop-3.3.3
    private Integer clusterId;               // 集群 ID
    private String myid;                     // ZooKeeper 节点 ID
    private RunAs runAs;                     // 运行用户信息
    private Map<String, String> configFileMap; // 配置文件映射
}
```

**configFileMap 示例**:
```java
{
    "core-site.xml": "<configuration>...</configuration>",
    "hdfs-site.xml": "<configuration>...</configuration>",
    "hadoop-env.sh": "export JAVA_HOME=${JAVA_HOME}..."
}
```

**配置文件模板示例** (使用 FreeMarker):
```xml
<!-- hdfs-site.xml.ftl -->
<configuration>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>${nameNodeDataDir}</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>${replicationFactor}</value>
    </property>
</configuration>
```

### 2.4 StartServiceActor - 服务启动

**文件路径**: `com/datasophon/worker/actor/StartServiceActor.java`  
**行数**: 62 行  
**职责**: 启动大数据服务，支持自定义启动策略

#### 源码分析

```java
public class StartServiceActor extends UntypedActor {
    
    private static final Logger logger = LoggerFactory.getLogger(StartServiceActor.class);
    
    @Override
    public void onReceive(Object msg) throws Throwable {
        if (msg instanceof ServiceRoleOperateCommand) {
            ServiceRoleOperateCommand command = (ServiceRoleOperateCommand) msg;
            
            logger.info("start to start service role {}", command.getServiceRoleName());
            
            ExecResult startResult = new ExecResult();
            ServiceHandler serviceHandler = new ServiceHandler(
                command.getServiceName(),
                command.getServiceRoleName()
            );
            
            // 尝试获取自定义启动策略
            ServiceRoleStrategy serviceRoleHandler = 
                ServiceRoleStrategyContext.getServiceRoleHandler(command.getServiceRoleName());
            
            if (Objects.nonNull(serviceRoleHandler)) {
                // 使用自定义策略
                startResult = serviceRoleHandler.handler(command);
            } else {
                // 使用默认启动逻辑
                startResult = serviceHandler.start(
                    command.getStartRunner(),
                    command.getStatusRunner(),
                    command.getDecompressPackageName(),
                    command.getRunAs()
                );
            }
            
            getSender().tell(startResult, getSelf());
            
            logger.info("service role {} start result {}", 
                command.getServiceRoleName(),
                startResult.getExecResult() ? "success" : "failed"
            );
        } else {
            unhandled(msg);
        }
    }
}
```

#### 启动策略模式

**为什么需要自定义策略？**
某些服务的启动需要特殊处理：
- **NameNode**: 首次启动需要格式化
- **JournalNode**: HA 模式需要特定启动顺序
- **Kafka**: 需要等待 ZooKeeper 连接成功
- **HBase**: 需要检查 HDFS 可用性

**策略模式实现**:
```java
// 接口定义
public interface ServiceRoleStrategy {
    ExecResult handler(ServiceRoleOperateCommand command);
}

// 具体策略 - NameNode
public class NameNodeHandlerStrategy implements ServiceRoleStrategy {
    @Override
    public ExecResult handler(ServiceRoleOperateCommand command) {
        // 1. 检查是否需要格式化
        if (needFormat()) {
            formatNameNode();
        }
        // 2. 启动 NameNode
        startNameNode();
        // 3. 等待安全模式退出
        waitForSafeMode();
        return result;
    }
}
```

**自定义策略列表** (31 个):
- NameNodeHandlerStrategy
- DataNodeHandlerStrategy
- ResourceManagerHandlerStrategy
- NodeManagerHandlerStrategy
- JournalNodeHandlerStrategy
- ZKFCHandlerStrategy
- KafkaHandlerStrategy
- ZkServerHandlerStrategy
- HbaseHandlerStrategy
- ...

### 2.5 StopServiceActor - 服务停止

**文件路径**: `com/datasophon/worker/actor/StopServiceActor.java`  
**行数**: 50 行  
**职责**: 停止运行中的服务

#### 源码分析

```java
public class StopServiceActor extends UntypedActor {
    
    private static final Logger logger = LoggerFactory.getLogger(StopServiceActor.class);
    
    @Override
    public void onReceive(Object msg) throws Throwable {
        if (msg instanceof ServiceRoleOperateCommand) {
            ServiceRoleOperateCommand command = (ServiceRoleOperateCommand) msg;
            
            logger.info("start to stop service role {}", command.getServiceRoleName());
            
            ServiceHandler serviceHandler = new ServiceHandler(
                command.getServiceName(),
                command.getServiceRoleName()
            );
            
            ExecResult stopResult = serviceHandler.stop(
                command.getStopRunner(),
                command.getStatusRunner(),
                command.getDecompressPackageName(),
                command.getRunAs()
            );
            
            getSender().tell(stopResult, getSelf());
            
            logger.info("service role {} stop result {}", 
                command.getServiceRoleName(),
                stopResult.getExecResult() ? "success" : "failed"
            );
        } else {
            unhandled(msg);
        }
    }
}
```

#### 停止流程

**ServiceHandler.stop() 逻辑**:
```java
public ExecResult stop(
    ServiceRoleRunner stopRunner,
    ServiceRoleRunner statusRunner,
    String decompressPackageName,
    RunAs runAs
) {
    // 1. 检查服务状态
    ExecResult statusResult = execRunner(statusRunner, decompressPackageName, runAs);
    
    if (!statusResult.getExecResult()) {
        // 服务已经停止，直接返回成功
        logger.info("{} already stopped", decompressPackageName);
        ExecResult result = new ExecResult();
        result.setExecResult(true);
        return result;
    }
    
    // 2. 执行停止命令
    ExecResult stopResult = execRunner(stopRunner, decompressPackageName, runAs);
    
    // 3. 轮询检查是否停止成功（最多检查 times 次）
    if (stopResult.getExecResult()) {
        int times = PropertyUtils.getInt("times");  // 默认 10 次
        int count = 0;
        
        while (count < times) {
            logger.info("check stop result at times {}", count + 1);
            ExecResult result = execRunner(statusRunner, decompressPackageName, runAs);
            
            if (!result.getExecResult()) {
                // 服务已停止
                logger.info("stop success in {}", decompressPackageName);
                break;
            }
            
            Thread.sleep(5 * 1000);  // 等待 5 秒
            count++;
        }
        
        if (count == times) {
            // 超时，停止失败
            logger.info("stop {} timeout", decompressPackageName);
            stopResult.setExecResult(false);
        }
    }
    
    return stopResult;
}
```

**停止验证机制**:
- 每 5 秒检查一次服务状态
- 最多检查 10 次（50 秒）
- 如果超时，标记为停止失败

### 2.6 RestartServiceActor - 服务重启

**文件路径**: `com/datasophon/worker/actor/RestartServiceActor.java`  
**行数**: 40 行  
**职责**: 重启服务（调用服务自带的 restart 脚本）

#### 源码分析

```java
public class RestartServiceActor extends UntypedActor {
    
    @Override
    public void onReceive(Object msg) throws Throwable {
        if (msg instanceof ServiceRoleOperateCommand) {
            ServiceRoleOperateCommand command = (ServiceRoleOperateCommand) msg;
            
            ServiceHandler serviceHandler = new ServiceHandler(
                command.getServiceName(),
                command.getServiceRoleName()
            );
            
            ExecResult startResult = serviceHandler.reStart(
                command.getRestartRunner(),
                command.getDecompressPackageName()
            );
            
            getSender().tell(startResult, getSelf());
        } else {
            unhandled(msg);
        }
    }
}
```

**重启 vs 停止+启动**:
- **重启**: 调用服务的 `restart` 脚本，由服务自己处理
- **停止+启动**: 先调用 StopServiceActor，再调用 StartServiceActor

**适用场景**:
- 配置变更后生效
- 服务异常需要重启
- 滚动重启集群节点

### 2.7 ExecuteCmdActor - 命令执行

**文件路径**: `com/datasophon/worker/actor/ExecuteCmdActor.java`  
**行数**: 39 行  
**职责**: 执行任意 Shell 命令

#### 源码分析

```java
public class ExecuteCmdActor extends UntypedActor {
    
    @Override
    public void onReceive(Object msg) throws Throwable {
        if (msg instanceof ExecuteCmdCommand) {
            ExecuteCmdCommand command = (ExecuteCmdCommand) msg;
            
            ExecResult execResult = ShellUtils.execWithStatus(
                Constants.INSTALL_PATH,
                command.getCommands(),
                60L
            );
            
            getSender().tell(execResult, getSelf());
        } else {
            unhandled(msg);
        }
    }
}
```

**ExecuteCmdCommand 结构**:
```java
public class ExecuteCmdCommand {
    private ArrayList<String> commands;  // 命令列表
}
```

**使用场景**:
- 执行自定义脚本
- 清理临时文件
- 创建目录
- 修改文件权限

**安全考虑**:
- 命令在 `/opt/datasophon` 目录下执行
- 超时时间固定为 60 秒
- 不建议执行危险命令（rm -rf /）

### 2.8 LogActor - 日志查询

**文件路径**: `com/datasophon/worker/actor/LogActor.java`  
**行数**: 74 行  
**职责**: 查询服务日志文件的最后 N 行

#### 源码分析

```java
public class LogActor extends UntypedActor {
    
    private static final Logger logger = LoggerFactory.getLogger(LogActor.class);
    
    @Override
    public void onReceive(Object msg) throws Throwable {
        if (msg instanceof GetLogCommand) {
            logger.info("get query log command");
            GetLogCommand command = (GetLogCommand) msg;
            
            HashMap<String, String> paramMap = new HashMap<>();
            String hostName = InetAddress.getLocalHost().getHostName();
            paramMap.put("${user}", "root");
            paramMap.put("${host}", hostName);
            
            // 替换日志文件路径中的占位符
            String logFileName = PlaceholderUtils.replacePlaceholders(
                command.getLogFile(),
                paramMap,
                Constants.REGEX_VARIABLE
            );
            
            ExecResult execResult = new ExecResult();
            String logStr = "can not find log file";
            
            // 1. 检查绝对路径
            if (logFileName.startsWith(StrUtil.SLASH) && FileUtil.exist(logFileName)) {
                logStr = FileUtils.readLastRows(
                    logFileName,
                    Charset.defaultCharset(),
                    PropertyUtils.getInt("rows")  // 默认 500 行
                );
            }
            // 2. 检查相对路径
            else if (FileUtil.exist(Constants.INSTALL_PATH + Constants.SLASH + 
                    command.getDecompressPackageName() + Constants.SLASH + logFileName)) {
                logStr = FileUtils.readLastRows(
                    Constants.INSTALL_PATH + Constants.SLASH + 
                    command.getDecompressPackageName() + Constants.SLASH + logFileName,
                    Charset.defaultCharset(),
                    PropertyUtils.getInt("rows")
                );
            }
            
            execResult.setExecResult(true);
            execResult.setExecOut(logStr);
            getSender().tell(execResult, getSelf());
        } else {
            unhandled(msg);
        }
    }
}
```

**日志文件路径示例**:
```
# 绝对路径
/var/log/hadoop/hdfs/hadoop-hdfs-namenode-${host}.log

# 相对路径
logs/namenode.log

# 解析后
/opt/datasophon/hadoop-3.3.3/logs/namenode.log
```

**占位符替换**:
- `${user}`: 当前用户（默认 root）
- `${host}`: 主机名

**读取策略**:
- 只读取最后 N 行（默认 500 行）
- 避免读取超大日志文件导致内存溢出

### 2.9 FileOperateActor - 文件操作

**文件路径**: `com/datasophon/worker/actor/FileOperateActor.java`  
**行数**: 52 行  
**职责**: 写入文件内容（覆盖或追加）

#### 源码分析

```java
public class FileOperateActor extends UntypedActor {
    
    @Override
    public void onReceive(Object msg) throws Throwable {
        if (msg instanceof FileOperateCommand) {
            ExecResult execResult = new ExecResult();
            FileOperateCommand fileOperateCommand = (FileOperateCommand) msg;
            
            TreeSet<String> lines = fileOperateCommand.getLines();
            
            if (Objects.nonNull(lines) && lines.size() > 0) {
                // 多行写入
                File file = FileUtil.writeLines(
                    lines,
                    fileOperateCommand.getPath(),
                    Charset.defaultCharset()
                );
                
                if (file.exists()) {
                    execResult.setExecResult(true);
                }
            } else {
                // 单一内容写入
                FileUtil.writeUtf8String(
                    fileOperateCommand.getContent(),
                    fileOperateCommand.getPath()
                );
            }
            
            getSender().tell(execResult, getSelf());
        } else {
            unhandled(msg);
        }
    }
}
```

**FileOperateCommand 结构**:
```java
public class FileOperateCommand {
    private String path;           // 文件路径
    private String content;        // 文件内容（单一内容）
    private TreeSet<String> lines; // 文件内容（多行）
}
```

**使用场景**:
- 生成配置文件
- 写入脚本文件
- 更新 hosts 文件
- 写入 Kerberos keytab

**为什么使用 TreeSet？**
- 自动去重
- 自动排序
- 适合写入需要排序的配置（如 hosts）

### 2.10 PingActor - 心跳检测

**文件路径**: `com/datasophon/worker/actor/PingActor.java`  
**行数**: 49 行  
**职责**: 响应 Master 的心跳检测请求

#### 源码分析

```java
public class PingActor extends UntypedActor {
    
    private static final Logger logger = LoggerFactory.getLogger(PingActor.class);
    
    @Override
    public void onReceive(Object msg) throws Throwable {
        if (msg instanceof PingCommand) {
            PingCommand command = (PingCommand) msg;
            
            ExecResult execResult = new ExecResult();
            execResult.setExecResult(true);
            execResult.setExecOut("pong");
            
            getSender().tell(execResult, getSelf());
        } else {
            unhandled(msg);
        }
    }
}
```

**Ping-Pong 机制**:
```
Master                          Worker
  │                               │
  ├─────── PingCommand ──────────>│
  │                               │
  │<────── ExecResult("pong") ────┤
  │                               │
```

**作用**:
- 检测 Worker 是否在线
- 监控 Worker 响应时间
- 发现网络分区问题

**心跳频率**:
- 建议每 30 秒一次
- 超时时间 5 秒
- 连续 3 次失败标记为离线

### 2.11 RemoteEventActor - 网络事件监听

**文件路径**: `com/datasophon/worker/actor/RemoteEventActor.java`  
**行数**: 45 行  
**职责**: 监听 Akka Remoting 的网络事件

#### 源码分析

```java
public class RemoteEventActor extends UntypedActor {
    
    private static final Logger logger = LoggerFactory.getLogger(RemoteEventActor.class);
    
    @Override
    public void onReceive(Object msg) throws Throwable {
        if (msg instanceof AssociationErrorEvent) {
            // 连接错误
            AssociationErrorEvent aee = (AssociationErrorEvent) msg;
            logger.info("{} --> {}: {}", 
                aee.getLocalAddress(),
                aee.getRemoteAddress(),
                aee.getCause()
            );
        } else if (msg instanceof AssociatedEvent) {
            // 连接建立
            AssociatedEvent ae = (AssociatedEvent) msg;
            logger.info("{} --> {} associated", 
                ae.getLocalAddress(),
                ae.getRemoteAddress()
            );
        } else if (msg instanceof DisassociatedEvent) {
            // 连接断开
            DisassociatedEvent de = (DisassociatedEvent) msg;
            logger.info("{} --> {} disassociated", 
                de.getLocalAddress(),
                de.getRemoteAddress()
            );
        }
    }
}
```

**监听的事件**:

| 事件 | 说明 | 日志示例 |
|------|------|---------|
| `AssociatedEvent` | Worker 与 Master 建立连接 | `akka.tcp://datasophon@worker1:2552 --> akka.tcp://datasophon@master:2551 associated` |
| `DisassociatedEvent` | Worker 与 Master 断开连接 | `akka.tcp://datasophon@worker1:2552 --> akka.tcp://datasophon@master:2551 disassociated` |
| `AssociationErrorEvent` | 连接过程中发生错误 | `Connection refused: master/192.168.1.100:2551` |

**作用**:
- 监控网络连接状态
- 记录连接/断连日志
- 辅助排查网络问题

## 三、其他 Actor 简要说明

### 3.1 系统管理类 Actor

#### UnixUserActor
**职责**: 创建/删除 Unix 用户
```java
// 创建用户
CreateUnixUserCommand command = new CreateUnixUserCommand();
command.setUser("hdfs");
command.setGroup("hadoop");
unixUserActor.tell(command, ActorRef.noSender());
```

#### UnixGroupActor
**职责**: 创建/删除 Unix 组
```java
// 创建组
CreateUnixGroupCommand command = new CreateUnixGroupCommand();
command.setGroup("hadoop");
unixGroupActor.tell(command, ActorRef.noSender());
```

#### KerberosActor
**职责**: 生成 Kerberos keytab 文件
```java
// 生成 keytab
GenerateKeytabCommand command = new GenerateKeytabCommand();
command.setPrincipal("hdfs/hostname@REALM");
command.setKeytabFile("/etc/security/keytabs/hdfs.keytab");
kerberosActor.tell(command, ActorRef.noSender());
```

#### AlertConfigActor
**职责**: 配置 Prometheus 告警规则
```java
// 更新告警规则
UpdateAlertConfigCommand command = new UpdateAlertConfigCommand();
command.setAlertRules(alertRulesYaml);
alertConfigActor.tell(command, ActorRef.noSender());
```

### 3.2 状态监控类 Actor

#### NMStateActor
**职责**: 查询 YARN NodeManager 状态
```java
// 返回 NodeManager 的内存、CPU、容器数等信息
```

#### RMStateActor
**职责**: 查询 YARN ResourceManager 状态
```java
// 返回 ResourceManager 的 HA 状态、Active/Standby
```

#### CheckServiceStatusActor
**职责**: 检查服务运行状态
```java
// 执行 status 命令，返回服务是否运行
```

## 四、Actor 消息传递机制

### 4.1 消息流向

```
Master                                    Worker
  │                                         │
  │  ① 发送安装命令                         │
  ├──── InstallServiceRoleCommand ────────>│
  │                                         │
  │                                         ├─> InstallServiceActor
  │                                         │    ├─ 下载安装包
  │                                         │    ├─ 解压
  │                                         │    ├─ 配置权限
  │                                         │    └─ 创建软链接
  │                                         │
  │  ② 返回安装结果                         │
  │<──────── ExecResult ────────────────────┤
  │   {execResult: true, execOut: "success"}│
```

### 4.2 Ask vs Tell 模式

**Tell 模式**（Fire and Forget）:
```java
workerActor.tell(command, ActorRef.noSender());
```
- 发送消息后立即返回
- 不等待响应
- 适合异步操作

**Ask 模式**（Request-Response）:
```java
Future<Object> future = Patterns.ask(workerActor, command, timeout);
ExecResult result = (ExecResult) Await.result(future, duration);
```
- 发送消息后等待响应
- 返回 Future 对象
- 适合需要结果的操作

**Master 使用 Ask 模式的原因**:
- 需要知道操作结果（成功/失败）
- 需要获取返回数据（日志内容、状态信息）
- 需要设置超时时间

### 4.3 消息序列化

Akka 使用 Java 序列化或自定义序列化器:
```java
// Command 对象必须实现 Serializable
public class InstallServiceRoleCommand implements Serializable {
    private static final long serialVersionUID = 1L;
    // ...
}
```

**为什么需要序列化？**
- Actor 可能在不同 JVM 进程中
- 消息需要通过网络传输
- 序列化后的数据可以持久化

## 五、Actor 的容错机制

### 5.1 监督策略

**默认监督策略**:
```java
@Override
public SupervisorStrategy supervisorStrategy() {
    return new OneForOneStrategy(
        10,                              // 最多重启 10 次
        Duration.create(1, TimeUnit.MINUTES),  // 1 分钟内
        DeciderBuilder
            .match(Exception.class, e -> SupervisorStrategy.restart())
            .build()
    );
}
```

**重启流程**:
```
Actor 抛出异常
     │
     ▼
preRestart() 钩子
     │
     ▼
停止 Actor
     │
     ▼
创建新 Actor 实例
     │
     ▼
postRestart() 钩子
     │
     ▼
继续处理消息
```

### 5.2 失败恢复

**Actor 重启不影响消息队列**:
- 未处理的消息会保留
- 重启后继续处理
- 不会丢失消息

**状态恢复**:
- Actor 字段会重置
- 如果需要恢复状态，使用 Akka Persistence

### 5.3 日志记录

**关键日志**:
```java
logger.info("start to install service role {}", command.getServiceRoleName());
logger.info("service role {} start result {}", serviceName, success ? "success" : "failed");
logger.error("install failed: {}", e.getMessage());
```

**日志级别**:
- **INFO**: 正常操作（安装、启动、停止）
- **WARN**: 可恢复的错误
- **ERROR**: 严重错误

## 六、性能优化建议

### 6.1 消息批处理

**当前实现**: 每个操作一个消息
**优化方案**: 批量操作打包成一个消息

```java
// 批量安装
public class BatchInstallCommand implements Serializable {
    private List<InstallServiceRoleCommand> commands;
}
```

### 6.2 异步执行

**当前实现**: Actor 内部同步执行 Shell 命令
**优化方案**: 使用 CompletableFuture 异步执行

```java
CompletableFuture.supplyAsync(() -> {
    return ShellUtils.execWithStatus(...);
}).thenAccept(result -> {
    getSender().tell(result, getSelf());
});
```

### 6.3 并发控制

**问题**: 同时安装多个服务可能导致资源竞争
**解决方案**: 使用 Akka Router 限制并发度

```java
ActorRef router = getContext().actorOf(
    new RoundRobinPool(5).props(Props.create(InstallServiceActor.class)),
    "installRouter"
);
```

## 七、总结

### 7.1 Actor 层的优势

1. **解耦**: 每个 Actor 职责单一，易于维护
2. **并发**: Actor 模型天然支持并发，无需手动管理线程
3. **容错**: 监督机制提供自动故障恢复
4. **分布式**: 位置透明，易于扩展到多机部署

### 7.2 设计模式应用

- **策略模式**: ServiceRoleStrategy
- **工厂模式**: ServiceRoleStrategyContext
- **观察者模式**: RemoteEventActor 监听网络事件
- **命令模式**: 各种 Command 对象

### 7.3 改进空间

1. **消息持久化**: 使用 Akka Persistence 持久化消息
2. **集群支持**: 使用 Akka Cluster 实现 Worker 集群
3. **背压处理**: 当 Worker 负载过高时，拒绝新任务
4. **监控指标**: 暴露 Actor 邮箱大小、处理时间等指标

---

**相关文档**:
- [02-WorkerApplicationServer启动类详解.md](./02-WorkerApplicationServer启动类详解.md)
- [04-Handler处理器分析.md](./04-Handler处理器分析.md)
- [05-Strategy策略模式详解.md](./05-Strategy策略模式详解.md)
