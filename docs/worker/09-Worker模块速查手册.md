# DataSophon Worker 模块速查手册

## 一、快速参考

本文档提供 Worker 模块的快速参考，包括常用 API、配置示例、故障排查等，适合快速查阅。

## 二、核心类速查

### 2.1 启动类

```java
// WorkerApplicationServer - Worker 启动入口
public class WorkerApplicationServer {
    public static void main(String[] args) {
        // ① 初始化 Actor 系统
        ActorSystem system = initActor(hostname);
        // ② 订阅远程事件
        subscribeRemoteEvent(system);
        // ③ 启动 Node Exporter
        startNodeExporter(workDir, cpuArchitecture);
        // ④ 创建系统用户
        createDefaultUser(userMap);
        // ⑤ 向 Master 注册
        tellToMaster(hostname, workDir, masterHost, cpuArchitecture, system);
    }
}
```

### 2.2 Actor 层常用 API

```java
// 获取远程 Actor 引用
ActorRef actor = ActorUtils.getRemoteActor("master-host", "workerStartActor");

// 发送消息 (Tell 模式 - 异步)
actor.tell(message, ActorRef.noSender());

// 发送消息 (Ask 模式 - 同步)
Future<Object> future = Patterns.ask(actor, message, timeout);
ExecResult result = (ExecResult) Await.result(future, duration);
```

### 2.3 Handler 层常用 API

```java
// 安装服务
InstallServiceHandler handler = new InstallServiceHandler(
    frameCode, serviceName, serviceRoleName
);
ExecResult result = handler.install(command);

// 生成配置文件
ConfigureServiceHandler handler = new ConfigureServiceHandler(
    serviceName, serviceRoleName
);
ExecResult result = handler.configure(
    configFileMap, decompressPackageName, clusterId, myid, serviceRoleName, runAs
);

// 启动服务
ServiceHandler handler = new ServiceHandler(serviceName, serviceRoleName);
ExecResult result = handler.start(
    startRunner, statusRunner, decompressPackageName, runAs
);
```

### 2.4 Strategy 层常用 API

```java
// 获取服务角色策略
ServiceRoleStrategy strategy = ServiceRoleStrategyContext.getServiceRoleHandler("NameNode");
if (strategy != null) {
    ExecResult result = strategy.handler(command);
}

// 执行资源策略
ReplaceStrategy strategy = new ReplaceStrategy();
strategy.setSrcFile("conf/hadoop-env.sh");
strategy.setReplaceKey("${JAVA_HOME}");
strategy.setReplaceValue("/usr/local/jdk1.8.0_211");
strategy.exec();
```

### 2.5 Utils 层常用 API

```java
// 创建 Unix 用户
UnixUtils.createUnixUser("hdfs", "hadoop", null);

// 检查用户是否存在
boolean exists = UnixUtils.isUserExists("hdfs");

// 下载 Kerberos keytab
KerberosUtils.downloadKeytabFromMaster("nn/hostname", "nn.service.keytab");

// 读取文件最后 N 行
String content = FileUtils.readLastRows("/path/to/file.log", Charset.defaultCharset(), 500);

// 生成配置文件
FreemakerUtils.generateConfigFile(generators, configs, decompressPackageName);
```

## 三、配置示例

### 3.1 Worker 配置文件

**application.yml**:
```yaml
# Master 配置
masterHost: master-hostname
masterWebPort: 8081
clusterId: 1

# Akka 配置
akka:
  remote:
    netty.tcp:
      hostname: worker-hostname
      port: 2552
      maximum-frame-size: 10485760  # 10MB
  
  actor:
    provider: remote
    default-dispatcher:
      type: Dispatcher
      executor: fork-join-executor
      fork-join-executor:
        parallelism-min: 2
        parallelism-max: 8

# 日志配置
logging:
  level:
    root: INFO
    com.datasophon: DEBUG
```

### 3.2 Logback 配置

**logback-worker.xml**:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 任务日志 Appender -->
    <appender name="TASK_LOG" class="ch.qos.logback.classic.sift.SiftingAppender">
        <filter class="com.datasophon.worker.log.TaskLogFilter">
            <level>WARN</level>
        </filter>
        
        <discriminator class="com.datasophon.worker.log.TaskLogDiscriminator">
            <key>taskAppId</key>
            <logBase>/opt/datasophon/logs/tasks</logBase>
        </discriminator>
        
        <sift>
            <appender name="TASK-${taskAppId}" class="ch.qos.logback.core.rolling.RollingFileAppender">
                <file>${logBase}/${taskAppId}.log</file>
                <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
                    <fileNamePattern>${logBase}/${taskAppId}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
                    <maxFileSize>100MB</maxFileSize>
                    <maxHistory>30</maxHistory>
                </rollingPolicy>
                <encoder>
                    <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
                </encoder>
            </appender>
        </sift>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="TASK_LOG"/>
    </root>
</configuration>
```

### 3.3 FreeMarker 模板示例

**xml.ftl** (Hadoop 配置文件):
```xml
<configuration>
<#list itemList as item>
    <property>
        <name>${item.name}</name>
        <value>${item.value}</value>
        <#if item.description??>
        <description>${item.description}</description>
        </#if>
    </property>
</#list>
</configuration>
```

**properties.ftl** (Properties 文件):
```properties
<#list itemList as item>
${item.name}=${item.value}
</#list>
```

## 四、常用命令

### 4.1 Worker 启动/停止

```bash
# 启动 Worker
cd /opt/datasophon/worker
sh bin/datasophon-worker.sh start

# 停止 Worker
sh bin/datasophon-worker.sh stop

# 重启 Worker
sh bin/datasophon-worker.sh restart

# 查看 Worker 状态
sh bin/datasophon-worker.sh status
```

### 4.2 日志查看

```bash
# 查看 Worker 主日志
tail -f /opt/datasophon/logs/worker.log

# 查看任务日志
tail -f /opt/datasophon/logs/tasks/DDP/HDFS/NameNode.log

# 查看最近的错误日志
grep -i error /opt/datasophon/logs/worker.log | tail -50

# 查看特定时间段的日志
sed -n '/2025-01-15 10:00/,/2025-01-15 11:00/p' /opt/datasophon/logs/worker.log
```

### 4.3 用户管理

```bash
# 创建 hadoop 组
groupadd hadoop

# 创建 hdfs 用户
useradd -g hadoop hdfs

# 检查用户是否存在
id hdfs

# 删除用户
userdel -r hdfs

# 查看用户所属组
groups hdfs
```

### 4.4 Kerberos 管理

```bash
# 列出 principal
kadmin.local -q "list_principals"

# 生成 keytab
kadmin.local -q "xst -k /etc/security/keytab/nn.service.keytab nn/hostname@REALM"

# 测试 keytab
kinit -kt /etc/security/keytab/nn.service.keytab nn/hostname@REALM
klist

# 删除 ticket
kdestroy
```

### 4.5 服务管理

```bash
# 查看服务状态
ps -ef | grep namenode

# 查看服务端口
netstat -tuln | grep 9000

# 查看服务日志
tail -f /opt/datasophon/hadoop-3.3.3/logs/hadoop-hdfs-namenode-*.log

# 手动启动服务
sudo -u hdfs sh /opt/datasophon/hadoop-3.3.3/sbin/hadoop-daemon.sh start namenode
```

## 五、故障排查

### 5.1 Worker 无法启动

**症状**: Worker 进程无法启动或启动后立即退出

**排查步骤**:
```bash
# ① 检查端口占用
netstat -tuln | grep 2552

# ② 检查配置文件
cat conf/worker.properties | grep -E "masterHost|clusterId"

# ③ 检查 JVM 内存
jps -lvm

# ④ 查看启动日志
tail -100 logs/worker.log

# ⑤ 检查 Akka 配置
grep -i "akka" conf/application.conf
```

**常见原因**:
- 端口 2552 被占用
- masterHost 配置错误
- JVM 内存不足
- 权限问题

### 5.2 服务安装失败

**症状**: InstallServiceActor 返回失败

**排查步骤**:
```bash
# ① 查看任务日志
tail -100 /opt/datasophon/logs/tasks/DDP/HDFS/NameNode.log

# ② 检查安装包
ls -lh /opt/datasophon/DDP/packages/

# ③ 检查 MD5
md5sum /opt/datasophon/DDP/packages/hadoop-3.3.3.tar.gz

# ④ 检查磁盘空间
df -h /opt/datasophon

# ⑤ 检查权限
ls -ld /opt/datasophon
```

**常见原因**:
- 安装包不存在或损坏
- 磁盘空间不足
- 权限不足
- 网络问题（下载失败）

### 5.3 服务启动失败

**症状**: StartServiceActor 返回失败或超时

**排查步骤**:
```bash
# ① 查看任务日志
tail -100 /opt/datasophon/logs/tasks/DDP/HDFS/NameNode.log

# ② 查看服务日志
tail -100 /opt/datasophon/hadoop-3.3.3/logs/hadoop-hdfs-namenode-*.log

# ③ 检查服务进程
ps -ef | grep namenode

# ④ 检查配置文件
cat /opt/datasophon/hadoop-3.3.3/etc/hadoop/hdfs-site.xml

# ⑤ 检查端口
netstat -tuln | grep 9000
```

**常见原因**:
- 配置文件错误
- 端口冲突
- 依赖服务未启动（如 HDFS 依赖 ZooKeeper）
- 数据目录权限问题

### 5.4 Kerberos 认证失败

**症状**: 服务启动后无法访问，提示认证失败

**排查步骤**:
```bash
# ① 检查 keytab 文件
ls -lh /etc/security/keytab/

# ② 测试 keytab
kinit -kt /etc/security/keytab/nn.service.keytab nn/hostname@REALM
klist

# ③ 检查 krb5.conf
cat /etc/krb5.conf

# ④ 检查 KDC 服务
systemctl status krb5-kdc

# ⑤ 查看 KDC 日志
tail -100 /var/log/krb5kdc.log
```

**常见原因**:
- keytab 文件不存在或权限错误
- principal 名称错误
- krb5.conf 配置错误
- KDC 服务未启动
- 时钟不同步

### 5.5 日志文件未生成

**症状**: 任务日志文件未创建或为空

**排查步骤**:
```bash
# ① 检查日志目录
ls -ld /opt/datasophon/logs/tasks/

# ② 检查 Logback 配置
cat conf/logback-worker.xml

# ③ 检查 Logger 名称
# 在代码中确认 Logger 名称格式是否正确

# ④ 检查日志级别
# 确认 TaskLogFilter 的 level 设置

# ⑤ 重启 Worker
sh bin/datasophon-worker.sh restart
```

**常见原因**:
- 日志目录不存在或权限不足
- Logback 配置错误
- Logger 名称格式不正确
- 日志级别过滤

## 六、性能调优

### 6.1 JVM 参数优化

**worker.sh**:
```bash
JAVA_OPTS="-Xms2g -Xmx4g \
           -XX:+UseG1GC \
           -XX:MaxGCPauseMillis=200 \
           -XX:+HeapDumpOnOutOfMemoryError \
           -XX:HeapDumpPath=/opt/datasophon/logs/heap_dump.hprof \
           -Dcom.sun.management.jmxremote \
           -Dcom.sun.management.jmxremote.port=9999 \
           -Dcom.sun.management.jmxremote.authenticate=false \
           -Dcom.sun.management.jmxremote.ssl=false"
```

**参数说明**:
- `-Xms2g -Xmx4g`: 堆内存 2GB - 4GB
- `-XX:+UseG1GC`: 使用 G1 垃圾收集器
- `-XX:MaxGCPauseMillis=200`: 最大 GC 停顿时间 200ms
- `-XX:+HeapDumpOnOutOfMemoryError`: OOM 时自动 dump 堆

### 6.2 Akka 调优

**application.conf**:
```hocon
akka {
  actor {
    default-dispatcher {
      type = Dispatcher
      executor = "fork-join-executor"
      fork-join-executor {
        parallelism-min = 4
        parallelism-factor = 2.0
        parallelism-max = 16
      }
      throughput = 100
    }
  }
  
  remote {
    netty.tcp {
      maximum-frame-size = 20971520  # 20MB
      send-buffer-size = 10485760     # 10MB
      receive-buffer-size = 10485760  # 10MB
    }
  }
}
```

### 6.3 日志性能优化

**异步日志**:
```xml
<appender name="ASYNC_TASK_LOG" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="TASK_LOG"/>
    <queueSize>1024</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <neverBlock>false</neverBlock>
</appender>
```

## 七、监控指标

### 7.1 Worker 健康检查

```bash
# 检查 Worker 进程
ps -ef | grep WorkerApplicationServer

# 检查 Akka 端口
netstat -tuln | grep 2552

# 检查 Node Exporter
curl -s http://localhost:9100/metrics | grep node_load

# 检查内存使用
jstat -gcutil $(jps | grep WorkerApplicationServer | awk '{print $1}') 1000 10
```

### 7.2 关键指标

| 指标 | 说明 | 正常范围 |
|------|------|---------|
| JVM Heap Used | 堆内存使用率 | < 80% |
| GC Time | GC 耗时 | < 1s |
| Actor Mailbox Size | Actor 邮箱消息数 | < 100 |
| Task Success Rate | 任务成功率 | > 95% |
| Response Time | 响应时间 | < 5s |

## 八、常见配置项

### 8.1 Worker 配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `masterHost` | localhost | Master 主机地址 |
| `masterWebPort` | 8081 | Master Web 端口 |
| `clusterId` | 1 | 集群 ID |
| `times` | 10 | 状态检查次数 |
| `rows` | 500 | 日志读取行数 |

### 8.2 Akka 配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `akka.remote.netty.tcp.hostname` | 动态 | Worker 绑定地址 |
| `akka.remote.netty.tcp.port` | 2552 | Worker 监听端口 |
| `akka.remote.netty.tcp.maximum-frame-size` | 10MB | 最大消息大小 |

## 九、开发指南

### 9.1 添加新的 Actor

```java
// ① 创建 Actor 类
public class MyCustomActor extends UntypedActor {
    @Override
    public void onReceive(Object msg) throws Throwable {
        if (msg instanceof MyCommand) {
            // 处理逻辑
            getSender().tell(result, getSelf());
        } else {
            unhandled(msg);
        }
    }
}

// ② 在 WorkerActor 中注册
ActorRef myActor = getContext().actorOf(
    Props.create(MyCustomActor.class),
    "myCustomActor"
);
getContext().watch(myActor);
```

### 9.2 添加新的 Strategy

```java
// ① 创建 Strategy 类
public class MyServiceHandlerStrategy extends AbstractHandlerStrategy 
    implements ServiceRoleStrategy {
    
    public MyServiceHandlerStrategy(String serviceName, String serviceRoleName) {
        super(serviceName, serviceRoleName);
    }
    
    @Override
    public ExecResult handler(ServiceRoleOperateCommand command) {
        // 特殊处理逻辑
        return startResult;
    }
}

// ② 在 ServiceRoleStrategyContext 中注册
map.put("MyService", new MyServiceHandlerStrategy("MY_SERVICE", "MyService"));
```

### 9.3 添加新的 ResourceStrategy

```java
// ① 创建 ResourceStrategy 类
public class MyResourceStrategy extends ResourceStrategy {
    public static final String MY_TYPE = "mytype";
    
    private String customParam;
    
    @Override
    public void exec() {
        // 资源操作逻辑
    }
}

// ② 在 InstallServiceHandler 中添加 case
case MyResourceStrategy.MY_TYPE:
    rs = BeanUtil.mapToBean(strategy, MyResourceStrategy.class, true,
        CopyOptions.create().ignoreError());
    break;
```

## 十、参考链接

### 10.1 详细文档

- [Worker模块概述](./01-Worker模块概述.md)
- [WorkerApplicationServer启动类详解](./02-WorkerApplicationServer启动类详解.md)
- [Actor层完整分析](./03-Actor层完整分析.md)
- [Handler处理器分析](./04-Handler处理器分析.md)
- [Strategy策略模式详解](./05-Strategy策略模式详解.md)
- [Utils工具类详解](./06-Utils工具类详解.md)
- [Log日志系统分析](./07-Log日志系统分析.md)
- [Worker模块文件索引](./08-Worker模块文件索引.md)

### 10.2 外部链接

- [Akka 官方文档](https://doc.akka.io/docs/akka/current/)
- [Logback 官方文档](https://logback.qos.ch/manual/)
- [FreeMarker 官方文档](https://freemarker.apache.org/docs/)

---

**维护说明**: 本速查手册提供快速参考，详细说明请查阅对应的详细文档。
