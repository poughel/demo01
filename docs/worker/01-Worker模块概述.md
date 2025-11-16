# DataSophon Worker 模块源码分析

## 一、模块概述

`datasophon-worker` 是 DataSophon 的工作节点模块，部署在集群的各个节点上，负责执行 Master 下发的命令，管理本地服务进程，采集监控数据等任务。

### 模块定位
- **任务执行者**: 执行 Master 下发的各种命令
- **服务管理者**: 管理本地大数据组件进程
- **数据采集者**: 采集本地监控指标和日志
- **分布式节点**: 通过 Actor 模型与 Master 通信

## 二、模块结构

```
datasophon-worker/
└── src/main/java/com/datasophon/worker/
    ├── actor/                      # Actor 实现
    │   ├── WorkerActor.java       # Worker 主 Actor
    │   └── RemoteEventActor.java  # 远程事件 Actor
    ├── handler/                    # 命令处理器
    │   ├── InstallServiceHandler.java
    │   ├── StartServiceHandler.java
    │   ├── StopServiceHandler.java
    │   ├── ConfigureServiceHandler.java
    │   └── ...
    ├── strategy/                   # 执行策略
    │   ├── ServiceRoleStrategy.java
    │   └── ...
    ├── utils/                      # 工具类
    │   ├── ActorUtils.java
    │   ├── UnixUtils.java
    │   └── ...
    ├── WorkerApplicationServer.java  # Worker 启动类
    └── resources/
        ├── application.yml         # 应用配置
        └── logback-worker.xml     # 日志配置
```

## 三、WorkerApplicationServer 详细分析

### 文件路径
`com/datasophon/worker/WorkerApplicationServer.java`

### 类职责
Worker 服务的主启动类，负责初始化 Worker 节点的各种组件和服务。

### 主要功能

#### 3.1 main 方法执行流程

```java
public static void main(String[] args) throws UnknownHostException {
    // 1. 获取主机名和工作目录
    String hostname = InetAddress.getLocalHost().getHostName();
    String workDir = System.getProperty("user.dir");
    
    // 2. 获取 Master 主机地址
    String masterHost = PropertyUtils.getString("masterHost");
    
    // 3. 获取 CPU 架构
    String cpuArchitecture = ShellUtils.getCpuArchitecture();
    
    // 4. 缓存主机名
    CacheUtils.put(Constants.HOSTNAME, hostname);
    
    // 5. 初始化 Actor 系统
    ActorSystem system = initActor(hostname);
    ActorUtils.setActorSystem(system);
    
    // 6. 订阅远程事件
    subscribeRemoteEvent(system);
    
    // 7. 启动 Node Exporter (监控数据采集)
    startNodeExporter(workDir, cpuArchitecture);
    
    // 8. 初始化用户映射
    Map<String, String> userMap = new HashMap<>();
    initUserMap(userMap);
    
    // 9. 创建默认用户
    createDefaultUser(userMap);
    
    // 10. 向 Master 注册
    tellToMaster(hostname, workDir, masterHost, cpuArchitecture, system);
    
    // 11. 注册关闭钩子
    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
        shutdown(system);
    }));
}
```

#### 3.2 初始化 Actor 系统

```java
private static ActorSystem initActor(String hostname) {
    // 加载 Actor 配置
    Map<String, Object> configMap = new HashMap<>();
    configMap.put("akka.actor.provider", "akka.remote.RemoteActorRefProvider");
    configMap.put("akka.remote.netty.tcp.hostname", hostname);
    configMap.put("akka.remote.netty.tcp.port", 2552);
    
    Config config = ConfigFactory.parseMap(configMap)
        .withFallback(ConfigFactory.load());
    
    // 创建 Actor 系统
    ActorSystem system = ActorSystem.create("datasophon-worker", config);
    
    // 创建 WorkerActor
    system.actorOf(Props.create(WorkerActor.class), "workerActor");
    
    return system;
}
```

**Actor 配置说明**:
- **provider**: 使用远程 Actor 提供者，支持分布式通信
- **hostname**: Worker 节点的主机名
- **port**: Worker Actor 监听端口 (默认 2552)

#### 3.3 订阅远程事件

```java
private static void subscribeRemoteEvent(ActorSystem system) {
    // 创建远程事件监听 Actor
    ActorRef remoteEventActor = system.actorOf(
        Props.create(RemoteEventActor.class), 
        "remoteEventActor"
    );
    
    // 订阅 Akka 远程事件
    EventStream eventStream = system.eventStream();
    eventStream.subscribe(remoteEventActor, AssociatedEvent.class);
    eventStream.subscribe(remoteEventActor, DisassociatedEvent.class);
    eventStream.subscribe(remoteEventActor, AssociationErrorEvent.class);
}
```

**监听的事件类型**:
- **AssociatedEvent**: 与 Master 建立连接
- **DisassociatedEvent**: 与 Master 断开连接
- **AssociationErrorEvent**: 连接错误

#### 3.4 启动 Node Exporter

```java
private static void startNodeExporter(String workDir, String cpuArchitecture) {
    logger.info("Starting node exporter...");
    
    // 构建 Node Exporter 路径
    String nodeExporterPath = workDir + "/prometheus/node_exporter";
    
    // 根据 CPU 架构选择对应的 exporter
    if ("aarch64".equals(cpuArchitecture)) {
        nodeExporterPath += "_arm64";
    }
    
    // 检查是否已经运行
    String checkCmd = "ps -ef | grep node_exporter | grep -v grep";
    ExecResult checkResult = ShellUtils.execCommand(checkCmd);
    
    if (checkResult.getExitCode() == 0 && 
        checkResult.getOutput().contains("node_exporter")) {
        logger.info("Node exporter is already running");
        return;
    }
    
    // 启动 Node Exporter
    String startCmd = String.format("nohup %s > /dev/null 2>&1 &", nodeExporterPath);
    ExecResult startResult = ShellUtils.execCommand(startCmd);
    
    if (startResult.getExitCode() == 0) {
        logger.info("Node exporter started successfully");
    } else {
        logger.error("Failed to start node exporter: " + startResult.getError());
    }
}
```

**Node Exporter 说明**:
- **功能**: Prometheus 的主机监控数据采集器
- **监控指标**: CPU、内存、磁盘、网络等
- **端口**: 默认 9100
- **架构支持**: x86_64 和 aarch64 (ARM)

#### 3.5 初始化用户映射

```java
private static void initUserMap(Map<String, String> userMap) {
    // 从配置文件读取服务用户映射
    userMap.put("hdfs", "hdfs");
    userMap.put("yarn", "yarn");
    userMap.put("hive", "hive");
    userMap.put("spark", "spark");
    userMap.put("kafka", "kafka");
    userMap.put("zookeeper", "zookeeper");
    // ... 更多服务用户
}
```

**用户映射的作用**:
- 每个大数据组件使用独立的系统用户运行
- 提高安全性，隔离不同服务
- 符合大数据组件的最佳实践

#### 3.6 创建默认用户

```java
private static void createDefaultUser(Map<String, String> userMap) {
    for (Map.Entry<String, String> entry : userMap.entrySet()) {
        String user = entry.getValue();
        
        // 检查用户是否存在
        String checkCmd = "id " + user;
        ExecResult checkResult = ShellUtils.execCommand(checkCmd);
        
        if (checkResult.getExitCode() != 0) {
            // 用户不存在，创建用户
            String createCmd = String.format(
                "useradd -m -s /bin/bash %s", user
            );
            ExecResult createResult = ShellUtils.execCommand(createCmd);
            
            if (createResult.getExitCode() == 0) {
                logger.info("User {} created successfully", user);
            } else {
                logger.error("Failed to create user {}: {}", 
                    user, createResult.getError());
            }
        } else {
            logger.info("User {} already exists", user);
        }
    }
}
```

#### 3.7 向 Master 注册

```java
private static void tellToMaster(String hostname, String workDir, 
                                 String masterHost, String cpuArchitecture,
                                 ActorSystem system) {
    // 构建 Master Actor 地址
    String masterActorPath = String.format(
        "akka.tcp://datasophon-api@%s:2551/user/masterActor",
        masterHost
    );
    
    // 获取 Master Actor 引用
    ActorSelection masterActor = system.actorSelection(masterActorPath);
    
    // 构建注册消息
    StartWorkerMessage message = new StartWorkerMessage();
    message.setHostname(hostname);
    message.setWorkDir(workDir);
    message.setCpuArchitecture(cpuArchitecture);
    message.setWorkerActorPath(getWorkerActorPath(hostname));
    
    // 发送注册消息到 Master
    masterActor.tell(message, ActorRef.noSender());
    
    logger.info("Worker registration message sent to Master: {}", masterHost);
}
```

**注册流程**:
1. 构建 Master Actor 的网络地址
2. 通过 ActorSelection 获取 Master Actor 引用
3. 构建包含 Worker 信息的注册消息
4. 发送消息到 Master
5. Master 收到后将 Worker 加入可用节点列表

#### 3.8 关闭处理

```java
private static void shutdown(ActorSystem system) {
    logger.info("Shutting down worker...");
    
    try {
        // 停止 Node Exporter
        stopNodeExporter();
        
        // 关闭 Actor 系统
        system.terminate();
        
        // 等待 Actor 系统完全关闭
        system.getWhenTerminated().toCompletableFuture().get();
        
        logger.info("Worker shutdown completed");
    } catch (Exception e) {
        logger.error("Error during shutdown", e);
    }
}
```

## 四、WorkerActor 详细分析

### 类职责
Worker 的核心 Actor，接收并处理 Master 下发的各种命令。

### Actor 消息处理

```java
public class WorkerActor extends AbstractActor {
    
    @Override
    public Receive createReceive() {
        return receiveBuilder()
            .match(ExecuteServiceRoleCommand.class, this::handleCommand)
            .match(String.class, this::handleString)
            .matchAny(this::unhandled)
            .build();
    }
    
    private void handleCommand(ExecuteServiceRoleCommand command) {
        logger.info("Received command: {}", command.getCommandType());
        
        // 根据命令类型分发到不同的处理器
        CommandHandler handler = getCommandHandler(command.getCommandType());
        
        // 执行命令
        ExecuteResult result = handler.handle(command);
        
        // 返回执行结果
        getSender().tell(result, getSelf());
    }
}
```

### 命令处理器

#### InstallServiceHandler - 安装服务处理器

```java
public class InstallServiceHandler implements CommandHandler {
    
    @Override
    public ExecuteResult handle(ExecuteServiceRoleCommand command) {
        String serviceName = command.getServiceName();
        String serviceRoleName = command.getServiceRoleName();
        
        logger.info("Installing {} - {}", serviceName, serviceRoleName);
        
        try {
            // 1. 下载安装包
            downloadPackage(serviceName);
            
            // 2. 解压安装包
            extractPackage(serviceName);
            
            // 3. 生成配置文件
            generateConfig(serviceName, serviceRoleName);
            
            // 4. 设置权限
            setPermissions(serviceName);
            
            // 5. 创建必要的目录
            createDirectories(serviceName);
            
            return ExecuteResult.success("Installation completed");
        } catch (Exception e) {
            logger.error("Installation failed", e);
            return ExecuteResult.failed(e.getMessage());
        }
    }
}
```

#### StartServiceHandler - 启动服务处理器

```java
public class StartServiceHandler implements CommandHandler {
    
    @Override
    public ExecuteResult handle(ExecuteServiceRoleCommand command) {
        String serviceName = command.getServiceName();
        String serviceRoleName = command.getServiceRoleName();
        
        logger.info("Starting {} - {}", serviceName, serviceRoleName);
        
        try {
            // 1. 检查服务是否已经运行
            if (isServiceRunning(serviceName, serviceRoleName)) {
                return ExecuteResult.success("Service is already running");
            }
            
            // 2. 获取启动命令
            String startCommand = getStartCommand(serviceName, serviceRoleName);
            
            // 3. 以指定用户执行启动命令
            String user = getServiceUser(serviceName);
            ExecResult result = UnixUtils.runAsUser(user, startCommand);
            
            if (result.getExitCode() == 0) {
                // 4. 等待服务启动
                waitForServiceStart(serviceName, serviceRoleName);
                
                return ExecuteResult.success("Service started successfully");
            } else {
                return ExecuteResult.failed(result.getError());
            }
        } catch (Exception e) {
            logger.error("Service start failed", e);
            return ExecuteResult.failed(e.getMessage());
        }
    }
}
```

#### StopServiceHandler - 停止服务处理器

```java
public class StopServiceHandler implements CommandHandler {
    
    @Override
    public ExecuteResult handle(ExecuteServiceRoleCommand command) {
        String serviceName = command.getServiceName();
        String serviceRoleName = command.getServiceRoleName();
        
        logger.info("Stopping {} - {}", serviceName, serviceRoleName);
        
        try {
            // 1. 检查服务是否在运行
            if (!isServiceRunning(serviceName, serviceRoleName)) {
                return ExecuteResult.success("Service is not running");
            }
            
            // 2. 获取停止命令
            String stopCommand = getStopCommand(serviceName, serviceRoleName);
            
            // 3. 执行停止命令
            String user = getServiceUser(serviceName);
            ExecResult result = UnixUtils.runAsUser(user, stopCommand);
            
            if (result.getExitCode() == 0) {
                // 4. 等待服务停止
                waitForServiceStop(serviceName, serviceRoleName);
                
                return ExecuteResult.success("Service stopped successfully");
            } else {
                // 5. 如果优雅停止失败，强制 kill
                forceKillService(serviceName, serviceRoleName);
                
                return ExecuteResult.success("Service force stopped");
            }
        } catch (Exception e) {
            logger.error("Service stop failed", e);
            return ExecuteResult.failed(e.getMessage());
        }
    }
}
```

## 五、监控数据采集

### 5.1 Node Exporter 集成

Worker 启动时会自动启动 Node Exporter：

**采集的指标**:
- CPU 使用率
- 内存使用情况
- 磁盘 I/O
- 网络流量
- 进程信息
- 系统负载

### 5.2 JMX 监控

对于 Java 应用（如 Hadoop、Kafka），Worker 会配置 JMX Exporter：

```java
private void configureJmxExporter(String serviceName) {
    // 下载 JMX Exporter JAR
    String jmxExporterJar = downloadJmxExporter();
    
    // 生成 JMX 配置文件
    String jmxConfig = generateJmxConfig(serviceName);
    
    // 在启动命令中添加 JVM 参数
    String javaOpts = String.format(
        "-javaagent:%s=%d:%s",
        jmxExporterJar,
        getJmxPort(serviceName),
        jmxConfig
    );
    
    // 设置环境变量
    setEnvVariable("JAVA_OPTS", javaOpts);
}
```

## 六、配置文件管理

### 6.1 配置模板

Worker 从 Master 接收配置模板和变量：

```java
public void generateConfig(String serviceName, 
                          String templateContent,
                          Map<String, String> variables) {
    // 使用 FreeMarker 渲染模板
    Template template = new Template("config", 
        new StringReader(templateContent), 
        freemarkerConfig);
    
    // 渲染配置文件
    StringWriter writer = new StringWriter();
    template.process(variables, writer);
    
    // 写入配置文件
    String configPath = getConfigPath(serviceName);
    FileUtils.writeFile(configPath, writer.toString());
    
    // 设置文件权限
    setFilePermissions(configPath);
}
```

### 6.2 配置热更新

```java
public void updateConfig(String serviceName, 
                        String configFile,
                        String newContent) {
    // 备份原配置
    backupConfig(serviceName, configFile);
    
    // 写入新配置
    writeConfig(serviceName, configFile, newContent);
    
    // 通知服务重新加载配置
    reloadServiceConfig(serviceName);
}
```

## 七、进程管理

### 7.1 进程监控

```java
public class ProcessMonitor {
    
    public boolean isProcessRunning(String processName) {
        String cmd = String.format(
            "ps -ef | grep %s | grep -v grep", 
            processName
        );
        ExecResult result = ShellUtils.execCommand(cmd);
        return result.getExitCode() == 0 && 
               !result.getOutput().isEmpty();
    }
    
    public int getProcessPid(String processName) {
        String cmd = String.format(
            "ps -ef | grep %s | grep -v grep | awk '{print $2}'",
            processName
        );
        ExecResult result = ShellUtils.execCommand(cmd);
        if (result.getExitCode() == 0) {
            return Integer.parseInt(result.getOutput().trim());
        }
        return -1;
    }
}
```

### 7.2 进程控制

```java
public class ProcessController {
    
    // 优雅停止
    public void gracefulStop(int pid) {
        ShellUtils.execCommand("kill -15 " + pid);
        waitForProcessStop(pid, 30); // 等待30秒
    }
    
    // 强制停止
    public void forceKill(int pid) {
        ShellUtils.execCommand("kill -9 " + pid);
    }
    
    // 重启进程
    public void restart(String serviceName) {
        stop(serviceName);
        start(serviceName);
    }
}
```

## 八、容错机制

### 8.1 心跳机制

```java
// Worker 定期向 Master 发送心跳
scheduler.scheduleAtFixedRate(() -> {
    HeartbeatMessage heartbeat = new HeartbeatMessage();
    heartbeat.setHostname(hostname);
    heartbeat.setTimestamp(System.currentTimeMillis());
    
    masterActor.tell(heartbeat, self());
}, 0, 30, TimeUnit.SECONDS);
```

### 8.2 重连机制

```java
public void handleDisconnect() {
    logger.warn("Connection to Master lost, attempting to reconnect...");
    
    int retryCount = 0;
    int maxRetries = 10;
    
    while (retryCount < maxRetries) {
        try {
            Thread.sleep(5000); // 等待5秒
            reconnectToMaster();
            logger.info("Reconnected to Master successfully");
            return;
        } catch (Exception e) {
            retryCount++;
            logger.error("Reconnect attempt {} failed", retryCount);
        }
    }
    
    logger.error("Failed to reconnect after {} attempts", maxRetries);
}
```

### 8.3 任务重试

```java
public ExecuteResult executeWithRetry(ExecuteServiceRoleCommand command) {
    int maxRetries = 3;
    int retryCount = 0;
    
    while (retryCount < maxRetries) {
        try {
            return execute(command);
        } catch (Exception e) {
            retryCount++;
            logger.warn("Execution failed, retry {} of {}", 
                retryCount, maxRetries);
            
            if (retryCount >= maxRetries) {
                return ExecuteResult.failed(
                    "Failed after " + maxRetries + " retries: " + e.getMessage()
                );
            }
            
            // 指数退避
            Thread.sleep(1000 * (long)Math.pow(2, retryCount));
        }
    }
    
    return ExecuteResult.failed("Unexpected error");
}
```

## 九、日志管理

### 9.1 服务日志收集

```java
public void collectServiceLogs(String serviceName) {
    String logDir = getServiceLogDir(serviceName);
    File[] logFiles = new File(logDir).listFiles();
    
    for (File logFile : logFiles) {
        if (logFile.getName().endsWith(".log")) {
            // 读取最新的日志内容
            List<String> lines = tailFile(logFile, 100);
            
            // 发送到 Master
            sendLogsToMaster(serviceName, lines);
        }
    }
}
```

### 9.2 日志轮转

```java
public void rotateLog(String logFile) {
    File file = new File(logFile);
    if (file.length() > MAX_LOG_SIZE) {
        String rotatedName = logFile + "." + 
            new SimpleDateFormat("yyyyMMddHHmmss").format(new Date());
        file.renameTo(new File(rotatedName));
        
        // 压缩旧日志
        compressFile(rotatedName);
    }
}
```

## 十、安全机制

### 10.1 用户隔离

不同服务使用不同的系统用户运行，避免权限泄露。

### 10.2 命令验证

```java
public boolean validateCommand(ExecuteServiceRoleCommand command) {
    // 验证命令类型
    if (!isValidCommandType(command.getCommandType())) {
        return false;
    }
    
    // 验证服务名称
    if (!isValidServiceName(command.getServiceName())) {
        return false;
    }
    
    // 验证参数
    if (!validateParameters(command.getParams())) {
        return false;
    }
    
    return true;
}
```

### 10.3 文件权限控制

```java
public void setSecurePermissions(String path) {
    // 设置文件所有者
    ShellUtils.execCommand("chown " + user + ":" + group + " " + path);
    
    // 设置文件权限 (750)
    ShellUtils.execCommand("chmod 750 " + path);
}
```

---

**文档版本**: v1.0  
**最后更新**: 2025-11-15
