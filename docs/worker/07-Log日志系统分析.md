# DataSophon Worker 日志系统完整分析

## 一、日志系统概述

### 1.1 日志系统架构

Worker 的日志系统基于 **Logback**，实现了**任务级别的日志隔离**，每个任务有独立的日志文件。

**核心组件**:
- **TaskLogDiscriminator**: 日志分类器，根据 Logger 名称路由日志到不同文件
- **TaskLogFilter**: 日志过滤器，过滤任务日志和系统日志

### 1.2 日志类列表

| 类名 | 文件名 | 行数 | 职责 |
|------|--------|------|------|
| `TaskLogDiscriminator` | TaskLogDiscriminator.java | 83 行 | 根据 Logger 名称动态创建日志文件 |
| `TaskLogFilter` | TaskLogFilter.java | 64 行 | 过滤任务日志，控制日志级别 |

## 二、TaskLogDiscriminator - 日志分类器

### 2.1 类基本信息

**文件路径**: `com/datasophon/worker/log/TaskLogDiscriminator.java`  
**行数**: 83 行  
**继承**: `ch.qos.logback.core.sift.AbstractDiscriminator<ILoggingEvent>`  
**职责**: 根据 Logger 名称动态创建不同的日志文件

### 2.2 源码分析

```java
public class TaskLogDiscriminator extends AbstractDiscriminator<ILoggingEvent> {
    
    private static Logger logger = LoggerFactory.getLogger(TaskLogDiscriminator.class);
    
    private String key;        // 用于配置的 key
    private String logBase;    // 日志基础路径
    
    /**
     * 获取区分值
     * Logger 名称格式: TaskLogLogger-{frameCode}-{serviceName}-{serviceRoleName}
     * 
     * @param event 日志事件
     * @return 日志文件路径
     */
    @Override
    public String getDiscriminatingValue(ILoggingEvent event) {
        String loggerName = event.getLoggerName();
        String prefix = TaskConstants.TASK_LOG_LOGGER_NAME + "-";
        
        if (loggerName.startsWith(prefix)) {
            // 提取任务标识，替换 - 为 /
            return loggerName
                .substring(prefix.length(), loggerName.length())
                .replace("-", "/");
        } else {
            return "unknown_task";
        }
    }
    
    @Override
    public void start() {
        started = true;
    }
    
    // Getter 和 Setter...
}
```

### 2.3 工作原理

```
┌─────────────────────────────────────────────────────────────┐
│            TaskLogDiscriminator 工作流程                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ① 日志事件到达                                             │
│     Logger name: TaskLogLogger-DDP-HDFS-NameNode           │
│                                                              │
│  ② 检查前缀                                                 │
│     是否以 "TaskLogLogger-" 开头？                          │
│     ├─ 是 → 继续处理                                        │
│     └─ 否 → 返回 "unknown_task"                             │
│                                                              │
│  ③ 提取任务标识                                             │
│     去掉前缀: "DDP-HDFS-NameNode"                           │
│                                                              │
│  ④ 替换分隔符                                               │
│     将 "-" 替换为 "/": "DDP/HDFS/NameNode"                  │
│                                                              │
│  ⑤ 构造日志文件路径                                         │
│     ${logBase}/DDP/HDFS/NameNode.log                        │
│     示例: /opt/datasophon/logs/tasks/DDP/HDFS/NameNode.log │
│                                                              │
│  ⑥ Logback 使用该路径创建日志文件                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.4 日志文件结构

```
/opt/datasophon/logs/tasks/
├── DDP/
│   ├── HDFS/
│   │   ├── NameNode.log
│   │   ├── DataNode.log
│   │   └── JournalNode.log
│   ├── YARN/
│   │   ├── ResourceManager.log
│   │   └── NodeManager.log
│   └── HIVE/
│       └── HiveServer2.log
└── unknown_task.log  (未识别的日志)
```

### 2.5 Logback 配置示例

```xml
<!-- logback-worker.xml -->
<appender name="TASK_LOG" class="ch.qos.logback.classic.sift.SiftingAppender">
    <discriminator class="com.datasophon.worker.log.TaskLogDiscriminator">
        <key>taskAppId</key>
        <logBase>/opt/datasophon/logs/tasks</logBase>
    </discriminator>
    
    <sift>
        <appender name="TASK-${taskAppId}" class="ch.qos.logback.core.FileAppender">
            <file>${logBase}/${taskAppId}.log</file>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
    </sift>
</appender>
```

**SiftingAppender 工作原理**:
1. 接收日志事件
2. 调用 Discriminator 获取区分值
3. 根据区分值动态创建 FileAppender
4. 将日志写入对应的文件

## 三、TaskLogFilter - 日志过滤器

### 3.1 类基本信息

**文件路径**: `com/datasophon/worker/log/TaskLogFilter.java`  
**行数**: 64 行  
**继承**: `ch.qos.logback.core.filter.Filter<ILoggingEvent>`  
**职责**: 过滤任务日志，只允许任务日志或高级别日志通过

### 3.2 源码分析

```java
public class TaskLogFilter extends Filter<ILoggingEvent> {
    
    private static Logger logger = LoggerFactory.getLogger(TaskLogFilter.class);
    
    private Level level;  // 日志级别阈值
    
    public void setLevel(String level) {
        this.level = Level.toLevel(level);
    }
    
    /**
     * 根据线程名和日志级别决定是否接受日志
     * 
     * @param event 日志事件
     * @return FilterReply.ACCEPT 或 FilterReply.DENY
     */
    @Override
    public FilterReply decide(ILoggingEvent event) {
        FilterReply filterReply = FilterReply.DENY;
        
        // ① 检查是否为任务日志
        if (event.getLoggerName().startsWith(TaskConstants.TASK_LOG_LOGGER_NAME)) {
            filterReply = FilterReply.ACCEPT;
        }
        // ② 检查日志级别
        else if (event.getLevel().isGreaterOrEqual(level)) {
            filterReply = FilterReply.ACCEPT;
        }
        
        logger.debug("task log filter, thread name:{}, loggerName:{}, filterReply:{}, level:{}",
            event.getThreadName(),
            event.getLoggerName(),
            filterReply.name(),
            level
        );
        
        return filterReply;
    }
}
```

### 3.3 过滤逻辑

```
┌─────────────────────────────────────────────────────────────┐
│              TaskLogFilter 决策流程                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  日志事件到达                                               │
│       │                                                      │
│       ▼                                                      │
│  检查 Logger 名称                                           │
│       │                                                      │
│  ┌────┴────┐                                                │
│  │ 是否以   │                                                │
│  │ TaskLog  │ ─ 是 ──> ACCEPT (接受)                        │
│  │ Logger   │                                                │
│  │ 开头？   │                                                │
│  └────┬────┘                                                │
│       │ 否                                                   │
│       ▼                                                      │
│  检查日志级别                                               │
│       │                                                      │
│  ┌────┴────┐                                                │
│  │ 级别 >=  │                                                │
│  │ 阈值？   │ ─ 是 ──> ACCEPT (接受)                        │
│  └────┬────┘                                                │
│       │ 否                                                   │
│       ▼                                                      │
│  DENY (拒绝)                                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**示例**:

| Logger 名称 | 日志级别 | 阈值 | 结果 | 原因 |
|------------|---------|------|------|------|
| `TaskLogLogger-DDP-HDFS-NameNode` | INFO | WARN | ACCEPT | 任务日志，总是接受 |
| `com.datasophon.worker.actor.StartServiceActor` | DEBUG | WARN | DENY | 非任务日志且级别低于阈值 |
| `com.datasophon.worker.actor.StartServiceActor` | ERROR | WARN | ACCEPT | 非任务日志但级别高于阈值 |

### 3.4 Logback 配置示例

```xml
<!-- logback-worker.xml -->
<appender name="TASK_LOG" class="ch.qos.logback.classic.sift.SiftingAppender">
    <!-- 添加过滤器 -->
    <filter class="com.datasophon.worker.log.TaskLogFilter">
        <level>WARN</level>  <!-- 只允许 WARN 及以上级别的系统日志 -->
    </filter>
    
    <discriminator class="com.datasophon.worker.log.TaskLogDiscriminator">
        <key>taskAppId</key>
        <logBase>/opt/datasophon/logs/tasks</logBase>
    </discriminator>
    
    <sift>
        <appender name="TASK-${taskAppId}" class="ch.qos.logback.core.FileAppender">
            <file>${logBase}/${taskAppId}.log</file>
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
    </sift>
</appender>
```

## 四、日志系统的优势

### 4.1 任务级别隔离

**传统方式** (所有日志混在一起):
```
worker.log:
2025-01-15 10:00:01 Start to install HDFS NameNode
2025-01-15 10:00:02 Start to configure YARN ResourceManager
2025-01-15 10:00:03 HDFS NameNode install success
2025-01-15 10:00:04 Start to start HDFS NameNode
...
```

**DataSophon 方式** (每个任务独立文件):
```
/opt/datasophon/logs/tasks/
├── DDP/HDFS/NameNode.log:
│   2025-01-15 10:00:01 Start to install HDFS NameNode
│   2025-01-15 10:00:03 HDFS NameNode install success
│   2025-01-15 10:00:04 Start to start HDFS NameNode
│
└── DDP/YARN/ResourceManager.log:
    2025-01-15 10:00:02 Start to configure YARN ResourceManager
    2025-01-15 10:00:05 YARN ResourceManager configured
```

**优势**:
- **易于排查**: 只需查看特定任务的日志文件
- **并发友好**: 多个任务并发执行，日志不会混淆
- **归档方便**: 可以按任务归档日志

### 4.2 动态日志文件创建

**无需预先配置**: 
- 新增服务时，无需修改 Logback 配置
- 日志文件自动按照 Logger 名称创建

**自动目录创建**:
- 如果 `DDP/HDFS/` 目录不存在，自动创建

### 4.3 灵活的日志级别控制

**任务日志**: 所有级别都记录（DEBUG、INFO、WARN、ERROR）

**系统日志**: 只记录重要的日志（WARN、ERROR）

**好处**:
- 任务日志详细，便于调试
- 系统日志精简，不影响性能

## 五、使用示例

### 5.1 创建任务日志记录器

```java
public class InstallServiceHandler {
    private Logger logger;
    
    public InstallServiceHandler(String frameCode, String serviceName, String serviceRoleName) {
        // 构造 Logger 名称
        String loggerName = String.format("%s-%s-%s-%s",
            TaskConstants.TASK_LOG_LOGGER_NAME,
            frameCode,
            serviceName,
            serviceRoleName
        );
        
        // 获取 Logger 实例
        logger = LoggerFactory.getLogger(loggerName);
    }
    
    public ExecResult install(InstallServiceRoleCommand command) {
        logger.info("Start to install package {}", command.getPackageName());
        // ... 安装逻辑
        logger.info("Install {} {}", command.getPackageName(), "success");
    }
}
```

**日志输出路径**:
```
/opt/datasophon/logs/tasks/DDP/HDFS/NameNode.log
```

### 5.2 日志内容示例

```
2025-01-15 10:00:01.123 [worker-executor-1] INFO  TaskLogLogger-DDP-HDFS-NameNode - Start to install package hadoop-3.3.3.tar.gz
2025-01-15 10:00:02.456 [worker-executor-1] INFO  TaskLogLogger-DDP-HDFS-NameNode - Remote package md5 is abc123def456
2025-01-15 10:00:03.789 [worker-executor-1] INFO  TaskLogLogger-DDP-HDFS-NameNode - download url is http://master:8081/ddh/service/install/downloadPackage?packageName=hadoop-3.3.3.tar.gz
2025-01-15 10:00:45.012 [worker-executor-1] INFO  TaskLogLogger-DDP-HDFS-NameNode - download package hadoop-3.3.3.tar.gz success
2025-01-15 10:01:30.345 [worker-executor-1] INFO  TaskLogLogger-DDP-HDFS-NameNode - chown hadoop-3.3.3 success
2025-01-15 10:01:30.678 [worker-executor-1] INFO  TaskLogLogger-DDP-HDFS-NameNode - chmod hadoop-3.3.3 success
2025-01-15 10:01:30.901 [worker-executor-1] INFO  TaskLogLogger-DDP-HDFS-NameNode - Install hadoop-3.3.3.tar.gz success
```

## 六、日志文件管理

### 6.1 日志轮转配置

**建议配置** (logback-worker.xml):
```xml
<appender name="TASK_LOG" class="ch.qos.logback.classic.sift.SiftingAppender">
    <discriminator class="com.datasophon.worker.log.TaskLogDiscriminator">
        <key>taskAppId</key>
        <logBase>/opt/datasophon/logs/tasks</logBase>
    </discriminator>
    
    <sift>
        <appender name="TASK-${taskAppId}" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>${logBase}/${taskAppId}.log</file>
            
            <!-- 滚动策略 -->
            <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
                <!-- 每天滚动，保留 30 天 -->
                <fileNamePattern>${logBase}/${taskAppId}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
                <maxFileSize>100MB</maxFileSize>
                <maxHistory>30</maxHistory>
                <totalSizeCap>1GB</totalSizeCap>
            </rollingPolicy>
            
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
    </sift>
</appender>
```

**滚动规则**:
- 每个文件最大 100MB
- 每天创建新文件
- 保留 30 天的日志
- 总大小不超过 1GB

### 6.2 日志清理脚本

```bash
#!/bin/bash
# clean-old-logs.sh

LOG_BASE="/opt/datasophon/logs/tasks"
DAYS=30

# 删除 30 天前的日志文件
find $LOG_BASE -name "*.log" -mtime +$DAYS -delete
find $LOG_BASE -name "*.log.gz" -mtime +$DAYS -delete

echo "Old logs cleaned"
```

### 6.3 日志归档

```bash
#!/bin/bash
# archive-logs.sh

LOG_BASE="/opt/datasophon/logs/tasks"
ARCHIVE_BASE="/opt/datasophon/logs/archive"
DATE=$(date +%Y%m%d)

# 压缩并归档日志
tar -czf $ARCHIVE_BASE/tasks-$DATE.tar.gz $LOG_BASE
echo "Logs archived to $ARCHIVE_BASE/tasks-$DATE.tar.gz"
```

## 七、性能优化

### 7.1 异步日志

**配置异步 Appender**:
```xml
<appender name="ASYNC_TASK_LOG" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="TASK_LOG"/>
    <queueSize>512</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <includeCallerData>false</includeCallerData>
</appender>
```

**优势**:
- 日志写入不阻塞业务线程
- 提升任务执行性能

### 7.2 缓冲区优化

```xml
<appender name="TASK_LOG" class="ch.qos.logback.core.FileAppender">
    <file>${logBase}/${taskAppId}.log</file>
    
    <!-- 启用缓冲区 -->
    <encoder>
        <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        <immediateFlush>false</immediateFlush>
    </encoder>
</appender>
```

**immediateFlush=false**:
- 日志先写入缓冲区
- 批量刷盘，减少 I/O 次数

## 八、总结

### 8.1 日志系统的设计亮点

1. **任务级别隔离**: 每个任务有独立的日志文件
2. **动态文件创建**: 无需预先配置，自动创建日志文件
3. **灵活的过滤**: 任务日志详细，系统日志精简
4. **易于维护**: 日志文件按照层次结构组织

### 8.2 技术栈

- **Logback**: 日志框架
- **SiftingAppender**: 动态路由日志到不同文件
- **Discriminator**: 自定义分类逻辑
- **Filter**: 自定义过滤逻辑

### 8.3 最佳实践

1. **命名规范**: Logger 名称遵循统一的格式
2. **日志轮转**: 避免单个文件过大
3. **定期清理**: 删除过期的日志文件
4. **异步写入**: 提升性能，不阻塞业务

---

**相关文档**:
- [02-WorkerApplicationServer启动类详解.md](./02-WorkerApplicationServer启动类详解.md)
- [04-Handler处理器分析.md](./04-Handler处理器分析.md)
- [06-Utils工具类详解.md](./06-Utils工具类详解.md)
