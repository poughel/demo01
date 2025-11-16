# DataSophon Worker Strategy 策略模式完整分析

## 一、Strategy 层架构概述

### 1.1 策略模式简介

**策略模式** (Strategy Pattern) 是一种行为设计模式，它定义了一系列算法，将每个算法封装起来，并使它们可以互相替换。

**在 Worker 中的应用**:
- 不同的大数据服务有不同的启动逻辑
- 使用策略模式避免在 Actor 中写大量 if-else
- 每个服务的特殊处理逻辑封装在独立的 Strategy 类中

### 1.2 Strategy 类分类

Worker 模块共有 **31 个 Strategy 类**，分为两大类：

**① 服务角色策略 (24个)**:
- 处理特定服务角色的启动、停止、配置等操作
- 继承 `AbstractHandlerStrategy`
- 实现 `ServiceRoleStrategy` 接口

**② 资源操作策略 (7个)**:
- 处理安装过程中的资源操作
- 继承 `ResourceStrategy` 抽象类
- 在 InstallServiceHandler 中被调用

## 二、服务角色策略详解

### 2.1 策略模式架构

```
ServiceRoleStrategy (接口)
        ▲
        │ implements
        │
AbstractHandlerStrategy (抽象类)
        ▲
        │ extends
        ├─────────────────────────┬─────────────────────────┐
        │                         │                         │
NameNodeHandlerStrategy  ResourceManagerHandlerStrategy  KafkaHandlerStrategy
DataNodeHandlerStrategy  NodeManagerHandlerStrategy      ZkServerHandlerStrategy
JournalNodeHandlerStrategy  HistoryServerHandlerStrategy ...
        │                         │                         │
        └─────────────────────────┴─────────────────────────┘
                            │
                ServiceRoleStrategyContext (工厂类)
                            │
                     ┌──────┴──────┐
                     │   Map       │
                     │  String →   │
                     │  Strategy   │
                     └─────────────┘
```

### 2.2 核心类分析

#### ServiceRoleStrategy - 策略接口

**文件路径**: `strategy/ServiceRoleStrategy.java`  
**行数**: 28 行

```java
public interface ServiceRoleStrategy {
    /**
     * 处理服务角色操作
     * @param command 服务角色操作命令
     * @return 执行结果
     */
    ExecResult handler(ServiceRoleOperateCommand command) 
        throws SQLException, ClassNotFoundException;
}
```

**设计思想**:
- 统一的接口，所有策略类必须实现 `handler()` 方法
- 接收 `ServiceRoleOperateCommand`，包含启动所需的所有参数
- 返回 `ExecResult`，包含执行结果和详细信息

#### AbstractHandlerStrategy - 抽象基类

**文件路径**: `strategy/AbstractHandlerStrategy.java`  
**行数**: 43 行

```java
@Data
public class AbstractHandlerStrategy {
    public String serviceName;       // 服务名
    public String serviceRoleName;   // 角色名
    public Logger logger;            // 日志记录器
    
    public AbstractHandlerStrategy(String serviceName, String serviceRoleName) {
        this.serviceName = serviceName;
        this.serviceRoleName = serviceRoleName;
        
        // 创建任务专属日志记录器
        String loggerName = String.format("%s-%s-%s", 
            TaskConstants.TASK_LOG_LOGGER_NAME, 
            serviceName, 
            serviceRoleName
        );
        logger = LoggerFactory.getLogger(loggerName);
    }
}
```

**设计优点**:
- 提供公共的日志记录器
- 子类可以直接使用 logger 记录日志
- 统一的日志命名规则

#### ServiceRoleStrategyContext - 策略工厂

**文件路径**: `strategy/ServiceRoleStrategyContext.java`  
**行数**: 68 行

```java
public class ServiceRoleStrategyContext {
    
    private static final Map<String, ServiceRoleStrategy> map = new ConcurrentHashMap<>();
    
    static {
        // HDFS 服务
        map.put("NameNode", new NameNodeHandlerStrategy("HDFS", "NameNode"));
        map.put("ZKFC", new ZKFCHandlerStrategy("HDFS", "ZKFC"));
        map.put("JournalNode", new JournalNodeHandlerStrategy("HDFS", "JournalNode"));
        map.put("DataNode", new DataNodeHandlerStrategy("HDFS", "DataNode"));
        
        // YARN 服务
        map.put("ResourceManager", new ResourceManagerHandlerStrategy("YARN", "ResourceManager"));
        map.put("NodeManager", new NodeManagerHandlerStrategy("YARN", "NodeManager"));
        map.put("HistoryServer", new HistoryServerHandlerStrategy("YARN", "HistoryServer"));
        
        // ZooKeeper
        map.put("ZkServer", new ZkServerHandlerStrategy("ZOOKEEPER", "ZkServer"));
        
        // Kafka
        map.put("KafkaBroker", new KafkaHandlerStrategy("KAFKA", "KafkaBroker"));
        
        // HBase
        map.put("HbaseMaster", new HbaseHandlerStrategy("HBASE", "HbaseMaster"));
        map.put("RegionServer", new HbaseHandlerStrategy("HBASE", "RegionServer"));
        
        // Hive
        map.put("HiveServer2", new HiveServer2HandlerStrategy("HIVE", "HiveServer2"));
        
        // Ranger
        map.put("RangerAdmin", new RangerAdminHandlerStrategy("RANGER", "RangerAdmin"));
        
        // Kerberos
        map.put("Krb5Kdc", new Krb5KdcHandlerStrategy("KERBEROS", "Krb5Kdc"));
        map.put("KAdmin", new KAdminHandlerStrategy("KERBEROS", "KAdmin"));
        
        // StarRocks
        map.put("SRFE", new FEHandlerStrategy("STARROCKS", "SRFE"));
        map.put("SRBE", new BEHandlerStrategy("STARROCKS", "SRBE"));
        
        // Doris
        map.put("DorisFE", new FEHandlerStrategy("DORIS", "DorisFE"));
        map.put("DorisFEObserver", new FEObserverHandlerStrategy("DORIS", "DorisFEObserver"));
        map.put("DorisBE", new BEHandlerStrategy("DORIS", "DorisBE"));
        
        // TEZ
        map.put("TezServer", new TezServerHandlerStrategy("TEZ", "TezServer"));
        
        // Kyuubi
        map.put("KyuubiServer", new KyuubiServerHandlerStrategy("KYUUBI", "KyuubiServer"));
        
        // Flink
        map.put("FlinkClient", new FlinkHandlerStrategy("FLINK", "FlinkClient"));
        
        // DolphinScheduler
        map.put("MasterServer", new DSMasterHandlerStrategy("DS", "MasterServer"));
    }
    
    /**
     * 获取服务角色对应的策略处理器
     * @param type 服务角色名称
     * @return 策略处理器，如果不存在返回 null
     */
    public static ServiceRoleStrategy getServiceRoleHandler(String type) {
        if (StringUtils.isBlank(type)) {
            return null;
        }
        return map.get(type);
    }
}
```

**工厂模式优势**:
1. **集中管理**: 所有策略在一处注册
2. **线程安全**: 使用 ConcurrentHashMap
3. **懒加载**: static 块在类加载时初始化
4. **类型安全**: 返回 ServiceRoleStrategy 接口

**使用方式**:
```java
// 在 StartServiceActor 中使用
ServiceRoleStrategy strategy = ServiceRoleStrategyContext.getServiceRoleHandler("NameNode");
if (strategy != null) {
    ExecResult result = strategy.handler(command);
}
```

### 2.3 HDFS 服务策略 (4个)

#### NameNodeHandlerStrategy - NameNode 启动策略

**文件路径**: `strategy/NameNodeHandlerStrategy.java`  
**行数**: 115 行  
**职责**: 处理 NameNode 的格式化、HA 配置、Kerberos 认证、Ranger 插件集成

```java
public class NameNodeHandlerStrategy extends AbstractHandlerStrategy implements ServiceRoleStrategy {
    
    @Override
    public ExecResult handler(ServiceRoleOperateCommand command) {
        ServiceHandler serviceHandler = new ServiceHandler(
            command.getServiceName(), 
            command.getServiceRoleName()
        );
        String workPath = Constants.INSTALL_PATH + Constants.SLASH + command.getDecompressPackageName();
        
        // ① 安装阶段：格式化 NameNode
        if (command.getCommandType().equals(CommandType.INSTALL_SERVICE)) {
            if (command.isSlave()) {
                // Standby NameNode：从 Active 同步元数据
                logger.info("Start to execute hdfs namenode -bootstrapStandby");
                ArrayList<String> commands = new ArrayList<>();
                commands.add(workPath + "/bin/hdfs");
                commands.add("namenode");
                commands.add("-bootstrapStandby");
                commands.add("-nonInteractive");
                commands.add("-force");
                
                ExecResult execResult = ShellUtils.execWithStatus(workPath, commands, 30L, logger);
                if (!execResult.getExecResult()) {
                    logger.error("Namenode standby failed");
                    return execResult;
                }
            } else {
                // Active NameNode：格式化元数据目录
                logger.info("Start to execute format namenode");
                ArrayList<String> commands = new ArrayList<>();
                commands.add(workPath + "/bin/hdfs");
                commands.add("namenode");
                commands.add("-format");
                commands.add("-nonInteractive");
                commands.add("-force");
                commands.add("smhadoop");
                
                // 清空旧的元数据
                FileUtil.del("/data/dfs/nn/current");
                
                ExecResult execResult = ShellUtils.execWithStatus(workPath, commands, 180L, logger);
                if (!execResult.getExecResult()) {
                    logger.error("Namenode format failed");
                    return execResult;
                }
            }
        }
        
        // ② 启用 Ranger HDFS 插件
        if (command.getEnableRangerPlugin()) {
            logger.info("Start to enable ranger hdfs plugin");
            ArrayList<String> commands = new ArrayList<>();
            commands.add("sh");
            commands.add(workPath + "/ranger-hdfs-plugin/enable-hdfs-plugin.sh");
            
            if (!FileUtil.exist(workPath + "/ranger-hdfs-plugin/success.id")) {
                ExecResult execResult = ShellUtils.execWithStatus(
                    workPath + "/ranger-hdfs-plugin", commands, 30L, logger
                );
                
                if (execResult.getExecResult()) {
                    logger.info("Enable ranger hdfs plugin success");
                    // 写入成功标识，避免重复执行
                    FileUtil.writeUtf8String("success", workPath + "/ranger-hdfs-plugin/success.id");
                } else {
                    logger.info("Enable ranger hdfs plugin failed");
                    return execResult;
                }
            }
        }
        
        // ③ 配置 Kerberos 认证
        if (command.getEnableKerberos()) {
            logger.info("Start to get namenode keytab file");
            String hostname = CacheUtils.getString(Constants.HOSTNAME);
            KerberosUtils.createKeytabDir();
            
            // 下载 NameNode principal 的 keytab
            if (!FileUtil.exist("/etc/security/keytab/nn.service.keytab")) {
                KerberosUtils.downloadKeytabFromMaster("nn/" + hostname, "nn.service.keytab");
            }
            
            // 下载 HTTP principal 的 keytab (用于 WebUI 认证)
            if (!FileUtil.exist("/etc/security/keytab/spnego.service.keytab")) {
                KerberosUtils.downloadKeytabFromMaster("HTTP/" + hostname, "spnego.service.keytab");
            }
        }
        
        // ④ 启动 NameNode
        ExecResult startResult = serviceHandler.start(
            command.getStartRunner(),
            command.getStatusRunner(),
            command.getDecompressPackageName(),
            command.getRunAs()
        );
        
        return startResult;
    }
}
```

**NameNode HA 启动流程**:
```
初始化 HDFS HA 集群
        │
        ├─────────────────┬─────────────────┐
        │                 │                 │
  Node1 (Active)    Node2 (Standby)   Node3 (Standby)
        │                 │                 │
        ▼                 ▼                 ▼
   format NameNode   bootstrapStandby   bootstrapStandby
        │                 │                 │
        │                 │                 │
  格式化元数据目录      从 Active 同步     从 Active 同步
  /data/dfs/nn         元数据             元数据
        │                 │                 │
        └─────────────────┴─────────────────┘
                        │
                        ▼
                启动 3 个 NameNode
                        │
                        ▼
                ZKFC 选举 Active
```

**Ranger 插件工作原理**:
1. 执行 `enable-hdfs-plugin.sh` 脚本
2. 将 Ranger 的 JAR 包复制到 HDFS 的 lib 目录
3. 修改 HDFS 配置，启用 Ranger 授权
4. 创建 success.id 标识文件，避免重复执行

#### JournalNodeHandlerStrategy - JournalNode 启动策略

**文件路径**: `strategy/JournalNodeHandlerStrategy.java`  
**行数**: 79 行  
**职责**: JournalNode 是 HDFS HA 的共享存储，需要先于 NameNode 启动

```java
public class JournalNodeHandlerStrategy extends AbstractHandlerStrategy implements ServiceRoleStrategy {
    
    @Override
    public ExecResult handler(ServiceRoleOperateCommand command) {
        ServiceHandler serviceHandler = new ServiceHandler(
            command.getServiceName(),
            command.getServiceRoleName()
        );
        String workPath = Constants.INSTALL_PATH + Constants.SLASH + command.getDecompressPackageName();
        
        // ① 安装阶段：格式化 ZooKeeper 状态
        if (command.getCommandType().equals(CommandType.INSTALL_SERVICE)) {
            if (!command.isSlave()) {
                logger.info("Start to execute hdfs zkfc -formatZK");
                ArrayList<String> commands = new ArrayList<>();
                commands.add(workPath + "/bin/hdfs");
                commands.add("zkfc");
                commands.add("-formatZK");
                commands.add("-force");
                
                ExecResult execResult = ShellUtils.execWithStatus(workPath, commands, 30L, logger);
                if (!execResult.getExecResult()) {
                    logger.error("Format zk failed");
                    return execResult;
                }
            }
        }
        
        // ② 配置 Kerberos
        if (command.getEnableKerberos()) {
            logger.info("Start to get jn keytab file");
            String hostname = CacheUtils.getString(Constants.HOSTNAME);
            KerberosUtils.createKeytabDir();
            
            if (!FileUtil.exist("/etc/security/keytab/jn.service.keytab")) {
                KerberosUtils.downloadKeytabFromMaster("jn/" + hostname, "jn.service.keytab");
            }
            if (!FileUtil.exist("/etc/security/keytab/spnego.service.keytab")) {
                KerberosUtils.downloadKeytabFromMaster("HTTP/" + hostname, "spnego.service.keytab");
            }
        }
        
        // ③ 启动 JournalNode
        ExecResult startResult = serviceHandler.start(
            command.getStartRunner(),
            command.getStatusRunner(),
            command.getDecompressPackageName(),
            command.getRunAs()
        );
        
        return startResult;
    }
}
```

**JournalNode 作用**:
- **共享存储**: 存储 NameNode 的 EditLog
- **HA 保证**: Active NameNode 写入，Standby NameNode 读取
- **Quorum 机制**: 至少 3 个节点，过半数可用即可

#### DataNodeHandlerStrategy - DataNode 启动策略

**文件路径**: `strategy/DataNodeHandlerStrategy.java`  
**行数**: 80 行  
**职责**: DataNode 的 Kerberos 认证和 Ranger 插件配置

```java
public class DataNodeHandlerStrategy extends AbstractHandlerStrategy implements ServiceRoleStrategy {
    
    @Override
    public ExecResult handler(ServiceRoleOperateCommand command) {
        ServiceHandler serviceHandler = new ServiceHandler(
            command.getServiceName(),
            command.getServiceRoleName()
        );
        String workPath = Constants.INSTALL_PATH + Constants.SLASH + command.getDecompressPackageName();
        
        // ① 启用 Ranger 插件
        if (command.getEnableRangerPlugin()) {
            logger.info("Start to enable ranger hdfs plugin");
            ArrayList<String> commands = new ArrayList<>();
            commands.add("sh");
            commands.add(workPath + "/ranger-hdfs-plugin/enable-hdfs-plugin.sh");
            
            if (!FileUtil.exist(workPath + "/ranger-hdfs-plugin/success.id")) {
                ExecResult execResult = ShellUtils.execWithStatus(
                    workPath + "/ranger-hdfs-plugin", commands, 30L, logger
                );
                
                if (execResult.getExecResult()) {
                    logger.info("Enable ranger hdfs plugin success");
                    FileUtil.writeUtf8String("success", workPath + "/ranger-hdfs-plugin/success.id");
                } else {
                    logger.info("Enable ranger hdfs plugin failed");
                    return execResult;
                }
            }
        }
        
        // ② 配置 Kerberos
        if (command.getEnableKerberos()) {
            logger.info("Start to get datanode keytab file");
            String hostname = CacheUtils.getString(Constants.HOSTNAME);
            KerberosUtils.createKeytabDir();
            
            if (!FileUtil.exist("/etc/security/keytab/dn.service.keytab")) {
                KerberosUtils.downloadKeytabFromMaster("dn/" + hostname, "dn.service.keytab");
            }
            if (!FileUtil.exist("/etc/security/keytab/spnego.service.keytab")) {
                KerberosUtils.downloadKeytabFromMaster("HTTP/" + hostname, "spnego.service.keytab");
            }
        }
        
        // ③ 启动 DataNode
        ExecResult startResult = serviceHandler.start(
            command.getStartRunner(),
            command.getStatusRunner(),
            command.getDecompressPackageName(),
            command.getRunAs()
        );
        
        return startResult;
    }
}
```

#### ZKFCHandlerStrategy - ZKFC 启动策略

**文件路径**: `strategy/ZKFCHandlerStrategy.java`  
**行数**: 63 行  
**职责**: ZKFC (ZooKeeper Failover Controller) 负责 NameNode 的自动故障转移

```java
public class ZKFCHandlerStrategy extends AbstractHandlerStrategy implements ServiceRoleStrategy {
    
    @Override
    public ExecResult handler(ServiceRoleOperateCommand command) {
        ServiceHandler serviceHandler = new ServiceHandler(
            command.getServiceName(),
            command.getServiceRoleName()
        );
        
        // 配置 Kerberos
        if (command.getEnableKerberos()) {
            logger.info("Start to get zkfc keytab file");
            String hostname = CacheUtils.getString(Constants.HOSTNAME);
            KerberosUtils.createKeytabDir();
            
            if (!FileUtil.exist("/etc/security/keytab/nn.service.keytab")) {
                KerberosUtils.downloadKeytabFromMaster("nn/" + hostname, "nn.service.keytab");
            }
        }
        
        // 启动 ZKFC
        ExecResult startResult = serviceHandler.start(
            command.getStartRunner(),
            command.getStatusRunner(),
            command.getDecompressPackageName(),
            command.getRunAs()
        );
        
        return startResult;
    }
}
```

**ZKFC 工作原理**:
```
ZKFC1 (Node1)          ZKFC2 (Node2)          ZKFC3 (Node3)
    │                      │                      │
    └──────────────────────┴──────────────────────┘
                           │
                           ▼
                    ZooKeeper 集群
                           │
                  选举 Active ZKFC
                           │
                           ▼
            Active ZKFC 监控 NameNode 健康状态
                           │
                ┌──────────┴──────────┐
                │                     │
           NameNode 健康          NameNode 故障
                │                     │
           保持 Active          触发 Failover
                                      │
                            ┌─────────┴─────────┐
                            │                   │
                    将当前 Active NN      将 Standby NN
                      切换为 Standby       切换为 Active
```

### 2.4 YARN 服务策略 (3个)

#### ResourceManagerHandlerStrategy

**职责**: ResourceManager 的 Kerberos 认证配置

#### NodeManagerHandlerStrategy

**职责**: NodeManager 的 Kerberos 认证和本地目录配置

#### HistoryServerHandlerStrategy

**职责**: MapReduce History Server 的启动和 Kerberos 配置

### 2.5 其他服务策略概览

| 服务 | Strategy 类 | 特殊处理 |
|------|-----------|---------|
| **ZooKeeper** | ZkServerHandlerStrategy | 写入 myid 文件 |
| **Kafka** | KafkaHandlerStrategy | Kerberos JAAS 配置 |
| **HBase** | HbaseHandlerStrategy | HDFS 依赖检查 |
| **Hive** | HiveServer2HandlerStrategy | Metastore 初始化、Kerberos、Ranger |
| **Ranger** | RangerAdminHandlerStrategy | 数据库初始化 |
| **Kerberos** | Krb5KdcHandlerStrategy, KAdminHandlerStrategy | KDC 初始化 |
| **StarRocks** | FEHandlerStrategy, BEHandlerStrategy | 元数据初始化 |
| **Doris** | FEHandlerStrategy, FEObserverHandlerStrategy, BEHandlerStrategy | 集群拓扑配置 |
| **TEZ** | TezServerHandlerStrategy | YARN 集成配置 |
| **Kyuubi** | KyuubiServerHandlerStrategy | Spark/Hive 集成 |
| **Flink** | FlinkHandlerStrategy | YARN/K8s 模式配置 |
| **DolphinScheduler** | DSMasterHandlerStrategy | 数据库初始化 |

## 三、资源操作策略详解

### 3.1 ResourceStrategy 架构

```
ResourceStrategy (抽象类)
        │
        ├─────────────────────────────────────────┐
        │                                         │
  ReplaceStrategy                         DownloadStrategy
  (文件内容替换)                           (下载外部文件)
        │                                         │
  LinkStrategy                            AppendLineStrategy
  (创建软链接)                             (追加文本行)
        │                                         │
  ShellStrategy                           EmptyStrategy
  (执行脚本)                               (空操作)
```

### 3.2 ResourceStrategy 基类

**文件路径**: `strategy/resource/ResourceStrategy.java`  
**行数**: 20 行

```java
@Data
public abstract class ResourceStrategy {
    
    public static final String TYPE_KEY = "type";
    
    String frameCode;       // 框架代码
    String service;         // 服务名
    String serviceRole;     // 角色名
    String basePath;        // 基础路径
    
    /**
     * 执行策略
     */
    public abstract void exec();
}
```

### 3.3 ReplaceStrategy - 文件内容替换

**文件路径**: `strategy/resource/ReplaceStrategy.java`  
**行数**: 35 行  
**职责**: 替换文件中的占位符

```java
@Data
@EqualsAndHashCode(callSuper = true)
public class ReplaceStrategy extends ResourceStrategy {
    
    public static final String REPLACE_TYPE = "replace";
    
    private String srcFile;        // 源文件路径
    private String replaceKey;     // 要替换的键
    private String replaceValue;   // 替换的值
    
    @Override
    public void exec() {
        String filePath = basePath + File.separator + srcFile;
        if (FileUtil.exist(filePath)) {
            String content = FileUtil.readUtf8String(filePath);
            content = content.replace(replaceKey, replaceValue);
            FileUtil.writeUtf8String(content, filePath);
        }
    }
}
```

**使用示例**:
```json
{
    "type": "replace",
    "srcFile": "conf/hadoop-env.sh",
    "replaceKey": "${JAVA_HOME}",
    "replaceValue": "/usr/local/jdk1.8.0_211"
}
```

### 3.4 DownloadStrategy - 下载外部文件

**文件路径**: `strategy/resource/DownloadStrategy.java`  
**行数**: 51 行  
**职责**: 从 HTTP URL 下载文件

```java
@Data
@EqualsAndHashCode(callSuper = true)
public class DownloadStrategy extends ResourceStrategy {
    
    public static final String DOWNLOAD_TYPE = "download";
    
    private String url;            // 下载 URL
    private String targetPath;     // 目标路径
    
    @Override
    public void exec() {
        String filePath = basePath + File.separator + targetPath;
        HttpUtil.downloadFile(url, FileUtil.file(filePath));
    }
}
```

**使用场景**:
```json
{
    "type": "download",
    "url": "http://master:8081/packages/mysql-connector-java-8.0.28.jar",
    "targetPath": "lib/mysql-connector-java.jar"
}
```

### 3.5 LinkStrategy - 创建软链接

**文件路径**: `strategy/resource/LinkStrategy.java`  
**行数**: 39 行  
**职责**: 创建符号链接

```java
@Data
@EqualsAndHashCode(callSuper = true)
public class LinkStrategy extends ResourceStrategy {
    
    public static final String LINK_TYPE = "link";
    
    private String srcPath;        // 源路径
    private String targetPath;     // 目标路径
    
    @Override
    public void exec() {
        String src = basePath + File.separator + srcPath;
        String target = basePath + File.separator + targetPath;
        
        if (FileUtil.exist(src) && !FileUtil.exist(target)) {
            ShellUtils.exceShell("ln -s " + src + " " + target);
        }
    }
}
```

### 3.6 AppendLineStrategy - 追加文本行

**文件路径**: `strategy/resource/AppendLineStrategy.java`  
**行数**: 40 行  
**职责**: 向文件追加内容

```java
@Data
@EqualsAndHashCode(callSuper = true)
public class AppendLineStrategy extends ResourceStrategy {
    
    public static final String APPEND_LINE_TYPE = "append";
    
    private String targetFile;     // 目标文件
    private String content;        // 追加的内容
    
    @Override
    public void exec() {
        String filePath = basePath + File.separator + targetFile;
        if (FileUtil.exist(filePath)) {
            FileUtil.appendUtf8String(content + "\n", filePath);
        }
    }
}
```

### 3.7 ShellStrategy - 执行脚本

**文件路径**: `strategy/resource/ShellStrategy.java`  
**行数**: 28 行  
**职责**: 执行 Shell 脚本

```java
@Data
@EqualsAndHashCode(callSuper = true)
public class ShellStrategy extends ResourceStrategy {
    
    public static final String SHELL_TYPE = "shell";
    
    private String script;         // 脚本路径
    
    @Override
    public void exec() {
        String scriptPath = basePath + File.separator + script;
        if (FileUtil.exist(scriptPath)) {
            ShellUtils.exceShell("sh " + scriptPath);
        }
    }
}
```

### 3.8 EmptyStrategy - 空操作

**文件路径**: `strategy/resource/EmptyStrategy.java`  
**行数**: 14 行  
**职责**: 占位符，不执行任何操作

```java
public class EmptyStrategy extends ResourceStrategy {
    @Override
    public void exec() {
        // 空实现
    }
}
```

## 四、策略模式的优势

### 4.1 代码组织清晰

**不使用策略模式** (if-else 地狱):
```java
public ExecResult start(ServiceRoleOperateCommand command) {
    if ("NameNode".equals(command.getServiceRoleName())) {
        // NameNode 特殊处理
        if (command.isSlave()) {
            // bootstrapStandby
        } else {
            // format
        }
        if (command.getEnableKerberos()) {
            // 下载 keytab
        }
        if (command.getEnableRangerPlugin()) {
            // 启用插件
        }
    } else if ("DataNode".equals(command.getServiceRoleName())) {
        // DataNode 特殊处理
        ...
    } else if ("ResourceManager".equals(command.getServiceRoleName())) {
        // ResourceManager 特殊处理
        ...
    }
    // 还有 20+ 个服务...
}
```

**使用策略模式** (简洁明了):
```java
public ExecResult start(ServiceRoleOperateCommand command) {
    ServiceRoleStrategy strategy = ServiceRoleStrategyContext.getServiceRoleHandler(
        command.getServiceRoleName()
    );
    
    if (strategy != null) {
        return strategy.handler(command);
    } else {
        return serviceHandler.start(...);  // 默认启动逻辑
    }
}
```

### 4.2 易于扩展

**添加新服务**只需 3 步：
1. 创建新的 Strategy 类
2. 在 ServiceRoleStrategyContext 中注册
3. 无需修改其他代码

### 4.3 单一职责原则

每个 Strategy 类只负责一个服务角色的启动逻辑，符合单一职责原则。

### 4.4 开闭原则

对扩展开放（可以添加新 Strategy），对修改关闭（不需要修改现有代码）。

## 五、Strategy 使用流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                   策略模式使用流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ① Master 发送启动命令                                           │
│     ServiceRoleOperateCommand {                                 │
│         serviceRoleName: "NameNode",                            │
│         commandType: INSTALL_SERVICE,                           │
│         enableKerberos: true,                                   │
│         enableRangerPlugin: true,                               │
│         ...                                                      │
│     }                                                            │
│                                                                  │
│  ② StartServiceActor 接收命令                                    │
│     - 调用 ServiceRoleStrategyContext.getServiceRoleHandler()   │
│     - 传入 serviceRoleName: "NameNode"                          │
│                                                                  │
│  ③ ServiceRoleStrategyContext 查找策略                          │
│     - 从 Map<String, ServiceRoleStrategy> 中查找                │
│     - 返回 NameNodeHandlerStrategy 实例                         │
│                                                                  │
│  ④ NameNodeHandlerStrategy 执行处理逻辑                         │
│     ┌────────────────────────────────────────┐                 │
│     │ handler(command)                       │                 │
│     ├────────────────────────────────────────┤                 │
│     │ ① 格式化 NameNode                       │                 │
│     │    hdfs namenode -format                │                 │
│     │                                         │                 │
│     │ ② 启用 Ranger 插件                      │                 │
│     │    sh enable-hdfs-plugin.sh             │                 │
│     │                                         │                 │
│     │ ③ 配置 Kerberos                         │                 │
│     │    下载 nn.service.keytab               │                 │
│     │    下载 spnego.service.keytab           │                 │
│     │                                         │                 │
│     │ ④ 启动 NameNode                         │                 │
│     │    sudo -u hdfs sh hadoop-daemon.sh ... │                 │
│     └────────────────────────────────────────┘                 │
│                                                                  │
│  ⑤ 返回执行结果                                                  │
│     ExecResult {                                                │
│         execResult: true,                                       │
│         execOut: "NameNode started successfully"               │
│     }                                                            │
│                                                                  │
│  ⑥ StartServiceActor 将结果返回给 Master                        │
│     - getSender().tell(execResult, getSelf())                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 六、性能优化建议

### 6.1 缓存策略实例

**当前实现**: 每次调用 `getServiceRoleHandler()` 都从 Map 中查找
**优化方案**: 使用缓存避免频繁 Map 查找

```java
// 使用 Guava Cache
private static final LoadingCache<String, ServiceRoleStrategy> cache = 
    CacheBuilder.newBuilder()
        .maximumSize(100)
        .build(new CacheLoader<String, ServiceRoleStrategy>() {
            @Override
            public ServiceRoleStrategy load(String key) {
                return map.get(key);
            }
        });
```

### 6.2 延迟初始化

**当前实现**: static 块一次性创建所有 Strategy 实例
**优化方案**: 首次使用时才创建实例

```java
public static ServiceRoleStrategy getServiceRoleHandler(String type) {
    return map.computeIfAbsent(type, key -> {
        switch (key) {
            case "NameNode":
                return new NameNodeHandlerStrategy("HDFS", "NameNode");
            // ...
            default:
                return null;
        }
    });
}
```

## 七、总结

### 7.1 Strategy 数量统计

- **服务角色策略**: 24 个
- **资源操作策略**: 7 个
- **总计**: 31 个

### 7.2 设计模式应用

1. **策略模式**: ServiceRoleStrategy
2. **工厂模式**: ServiceRoleStrategyContext
3. **模板方法模式**: AbstractHandlerStrategy

### 7.3 核心优势

1. **解耦**: 服务特定逻辑与通用逻辑分离
2. **扩展性**: 添加新服务无需修改现有代码
3. **可维护性**: 每个 Strategy 类职责单一
4. **可测试性**: 每个 Strategy 可以独立测试

### 7.4 适用场景

- 需要根据不同条件执行不同算法
- 多个类只在行为上有差异
- 需要在运行时动态选择算法

---

**相关文档**:
- [04-Handler处理器分析.md](./04-Handler处理器分析.md)
- [06-Utils工具类详解.md](./06-Utils工具类详解.md)
- [整体架构设计](../overview/03-整体架构设计.md)
