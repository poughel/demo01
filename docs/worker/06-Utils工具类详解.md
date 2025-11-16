# DataSophon Worker Utils 工具类完整分析

## 一、Utils 层概述

### 1.1 工具类职责

Worker 的 Utils 层提供了一组通用的工具类，用于简化常见操作，避免代码重复。

### 1.2 Utils 类列表

| 工具类 | 文件名 | 行数 | 核心职责 |
|--------|--------|------|---------|
| `ActorUtils` | ActorUtils.java | 36 行 | Akka Actor 工具类，获取远程 Actor 引用 |
| `FreemakerUtils` | FreemakerUtils.java | 188 行 | FreeMarker 模板渲染工具 |
| `KerberosUtils` | KerberosUtils.java | 70 行 | Kerberos 认证工具，keytab 下载和目录创建 |
| `UnixUtils` | UnixUtils.java | 97 行 | Unix 用户/组管理工具 |
| `FileUtils` | FileUtils.java | 63 行 | 文件操作工具，读取文件最后 N 行 |
| `TaskConstants` | TaskConstants.java | 111 行 | 任务相关常量定义 |

## 二、ActorUtils - Actor 工具类

### 2.1 类基本信息

**文件路径**: `com/datasophon/worker/utils/ActorUtils.java`  
**行数**: 36 行  
**职责**: 管理 ActorSystem 实例，获取远程 Actor 引用

### 2.2 源码分析

```java
public class ActorUtils {
    
    private static ActorSystem actorSystem;
    
    /**
     * 设置全局 ActorSystem 实例
     * @param actorSystem Akka ActorSystem
     */
    public static void setActorSystem(ActorSystem actorSystem) {
        ActorUtils.actorSystem = actorSystem;
    }
    
    /**
     * 获取远程 Actor 引用
     * @param hostname 远程主机名
     * @param actorName Actor 名称
     * @return ActorRef，如果获取失败返回 null
     */
    public static ActorRef getRemoteActor(String hostname, String actorName) {
        // 构造 Actor 路径
        String actorPath = "akka.tcp://datasophon@" + hostname + ":2551/user/" + actorName;
        ActorSelection actorSelection = actorSystem.actorSelection(actorPath);
        
        // 设置 30 秒超时
        Timeout timeout = new Timeout(Duration.create(30, TimeUnit.SECONDS));
        
        // 解析 ActorRef
        Future<ActorRef> future = actorSelection.resolveOne(timeout);
        ActorRef actorRef = null;
        
        try {
            // 等待解析完成
            actorRef = Await.result(future, Duration.create(30, TimeUnit.SECONDS));
        } catch (Exception e) {
            e.printStackTrace();
        }
        
        return actorRef;
    }
}
```

### 2.3 使用场景

**① Worker 向 Master 发送消息**:
```java
// 获取 Master 的 Actor 引用
ActorRef masterActor = ActorUtils.getRemoteActor("master-host", "masterActor");
if (masterActor != null) {
    masterActor.tell(message, ActorRef.noSender());
}
```

**② Actor 路径解析**:
```
akka.tcp://datasophon@master-host:2551/user/workerStartActor
│          │           │               │    │
│          │           │               │    └─ Actor 名称
│          │           │               └────── user 创建的 Actor 路径
│          │           └─────────────────────── 端口号
│          └─────────────────────────────────── 主机名
└────────────────────────────────────────────── 协议和 ActorSystem 名称
```

### 2.4 设计思想

**单例模式**: ActorSystem 全局唯一，通过 static 字段共享。

**超时处理**: 设置 30 秒超时，避免长时间等待。

## 三、FreemakerUtils - 模板渲染工具

### 3.1 类基本信息

**文件路径**: `com/datasophon/worker/utils/FreemakerUtils.java`  
**行数**: 188 行  
**职责**: 使用 FreeMarker 模板引擎生成配置文件

### 3.2 核心方法

#### generateConfigFile() - 生成配置文件

```java
public static void generateConfigFile(
    Generators generators,
    List<ServiceConfig> configs,
    String decompressPackageName,
    String extPath
) throws IOException, TemplateException {
    
    // ① 创建 FreeMarker 配置对象
    Configuration config = new Configuration(Configuration.getVersion());
    
    // ② 设置模板加载路径
    List<TemplateLoader> loaderList = new ArrayList<>();
    // 默认路径：classpath:/templates
    loaderList.add(new ClassTemplateLoader(FreemakerUtils.class, "/templates"));
    
    // 扩展路径：/opt/datasophon/hadoop-3.3.3/templates
    if (StringUtils.isNotBlank(extPath) && new File(extPath).exists()) {
        loaderList.add(new FileTemplateLoader(new File(extPath)));
    }
    
    config.setTemplateLoader(new MultiTemplateLoader(
        loaderList.toArray(new TemplateLoader[0])
    ));
    
    // ③ 加载模板
    Map<String, Object> data = new HashMap<>();
    String configFormat = generators.getConfigFormat();
    Template template = null;
    
    if (Constants.XML.equals(configFormat)) {
        template = config.getTemplate("xml.ftl");
    } else if (Constants.PROPERTIES.equals(configFormat)) {
        template = config.getTemplate("properties.ftl");
    } else if (Constants.PROPERTIES2.equals(configFormat)) {
        template = config.getTemplate("properties2.ftl");
    } else if (Constants.PROPERTIES3.equals(configFormat)) {
        template = config.getTemplate("properties3.ftl");
    } else if (Constants.PROMETHEUS.equals(configFormat)) {
        template = config.getTemplate("alert.yml");
    } else if (Constants.CUSTOM.equals(configFormat)) {
        template = config.getTemplate(generators.getTemplateName());
        // 提取 map 类型的配置
        data = configs.stream()
            .filter(e -> "map".equals(e.getConfigType()))
            .collect(Collectors.toMap(
                key -> key.getName(),
                value -> value.getValue()
            ));
        // 过滤掉 map 类型的配置
        configs = configs.stream()
            .filter(e -> !"map".equals(e.getConfigType()))
            .collect(Collectors.toList());
    }
    
    logger.info("load template: {} success.", template.getSourceName());
    
    // ④ 准备数据模型
    data.put("itemList", configs);
    
    // ⑤ 渲染并写入文件
    processOut(generators, template, data, decompressPackageName);
}
```

### 3.3 支持的配置格式

| 格式 | 模板文件 | 使用场景 |
|------|---------|---------|
| `xml` | xml.ftl | Hadoop 配置文件 (hdfs-site.xml, core-site.xml) |
| `properties` | properties.ftl | Java Properties 格式 |
| `properties2` | properties2.ftl | 带注释的 Properties |
| `properties3` | properties3.ftl | 带分隔符的 Properties |
| `prometheus` | alert.yml | Prometheus 告警规则 |
| `custom` | 自定义 | 用户自定义模板 |

### 3.4 模板示例

**xml.ftl**:
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

**properties.ftl**:
```properties
<#list itemList as item>
${item.name}=${item.value}
</#list>
```

### 3.5 文件输出处理

```java
private static void processOut(
    Generators generators,
    Template template,
    Map<String, Object> data,
    String decompressPackageName
) throws IOException, TemplateException {
    
    String packagePath = Constants.INSTALL_PATH + Constants.SLASH + 
                         decompressPackageName + Constants.SLASH;
    String outputDirectory = generators.getOutputDirectory();
    
    if (outputDirectory.contains(Constants.COMMA)) {
        // 多个输出目录，以逗号分隔
        for (String outPutDir : outputDirectory.split(StrUtil.COMMA)) {
            String outputFile = packagePath + outPutDir + Constants.SLASH + 
                                generators.getFilename();
            writeToTemplate(template, data, outputFile);
        }
    } else if (outputDirectory.startsWith(Constants.SLASH)) {
        // 绝对路径
        String outputFile = outputDirectory + Constants.SLASH + 
                            generators.getFilename();
        writeToTemplate(template, data, outputFile);
    } else {
        // 相对路径
        String outputFile = packagePath + outputDirectory + Constants.SLASH + 
                            generators.getFilename();
        writeToTemplate(template, data, outputFile);
    }
}
```

**支持的路径类型**:
1. **多个目录**: `etc/hadoop,etc/yarn` - 同一文件写入多个目录
2. **绝对路径**: `/etc/prometheus/rules` - 系统级配置
3. **相对路径**: `conf` - 相对于安装目录

## 四、KerberosUtils - Kerberos 工具类

### 4.1 类基本信息

**文件路径**: `com/datasophon/worker/utils/KerberosUtils.java`  
**行数**: 70 行  
**职责**: Kerberos 认证相关操作，keytab 文件下载和目录创建

### 4.2 核心方法

#### downloadKeytabFromMaster() - 下载 keytab 文件

```java
public static void downloadKeytabFromMaster(String principal, String keytabName) {
    String masterHost = PropertyUtils.getString(Constants.MASTER_HOST);
    String masterPort = PropertyUtils.getString(Constants.MASTER_WEB_PORT);
    Integer clusterId = PropertyUtils.getInt("clusterId");
    String hostname = CacheUtils.getString("hostname");
    
    // 构造下载 URL
    String downloadUrl = "http://" + masterHost + ":" + masterPort + 
        "/ddh/cluster/kerberos/downloadKeytab?clusterId=" + clusterId +
        "&principal=" + principal +
        "&keytabName=" + keytabName +
        "&hostname=" + hostname;
    
    String dest = "/etc/security/keytab/";
    
    // 下载文件，显示进度
    HttpUtil.downloadFile(downloadUrl, FileUtil.file(dest), new StreamProgress() {
        @Override
        public void start() {
            Console.log("start to install。。。。");
        }
        
        @Override
        public void progress(long progressSize, long totalSize) {
            Console.log("installed：{}", FileUtil.readableFileSize(progressSize));
        }
        
        @Override
        public void finish() {
            Console.log("install success！");
        }
    });
}
```

#### createKeytabDir() - 创建 keytab 目录

```java
public static void createKeytabDir() {
    // 创建目录
    if (!FileUtil.exist("/etc/security/keytab/")) {
        FileUtil.mkdir("/etc/security/keytab/");
    }
    
    // 设置所有者和权限
    ShellUtils.exceShell("chown -R root:hadoop /etc/security/keytab/");
    ShellUtils.exceShell("chmod -R 770 /etc/security/keytab/");
}
```

### 4.3 Kerberos 工作流程

```
┌─────────────────────────────────────────────────────────────┐
│              Kerberos 认证配置流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ① 创建 keytab 目录                                         │
│     - mkdir /etc/security/keytab                            │
│     - chown root:hadoop /etc/security/keytab                │
│     - chmod 770 /etc/security/keytab                        │
│                                                              │
│  ② 下载 keytab 文件                                         │
│     - 从 Master 下载 nn.service.keytab                      │
│     - 从 Master 下载 spnego.service.keytab                  │
│     - 保存到 /etc/security/keytab/                          │
│                                                              │
│  ③ 配置服务使用 keytab                                      │
│     - 在 hdfs-site.xml 中配置 keytab 路径                   │
│     - 在 core-site.xml 中配置 principal                     │
│                                                              │
│  ④ 启动服务                                                 │
│     - 服务自动使用 keytab 进行认证                          │
│     - 无需手动 kinit                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.4 Principal 命名规范

**格式**: `{service}/{hostname}@{REALM}`

**示例**:
- `nn/node1.example.com@HADOOP.COM` - NameNode
- `dn/node2.example.com@HADOOP.COM` - DataNode
- `HTTP/node1.example.com@HADOOP.COM` - Web UI (SPNEGO)

## 五、UnixUtils - Unix 用户管理工具

### 5.1 类基本信息

**文件路径**: `com/datasophon/worker/utils/UnixUtils.java`  
**行数**: 97 行  
**职责**: Unix/Linux 系统用户和组管理

### 5.2 核心方法

#### createUnixUser() - 创建用户

```java
public static ExecResult createUnixUser(
    String username,
    String mainGroup,
    String otherGroups
) {
    ArrayList<String> commands = new ArrayList<>();
    
    if (isUserExists(username)) {
        // 用户已存在，使用 usermod 修改
        commands.add("usermod");
    } else {
        // 用户不存在，使用 useradd 创建
        commands.add("useradd");
    }
    
    commands.add(username);
    
    // 主组
    if (StringUtils.isNotBlank(mainGroup)) {
        commands.add("-g");
        commands.add(mainGroup);
    }
    
    // 附加组
    if (StringUtils.isNotBlank(otherGroups)) {
        commands.add("-G");
        commands.add(otherGroups);
    }
    
    return ShellUtils.execWithStatus(Constants.INSTALL_PATH, commands, TIME_OUT);
}
```

**等效命令**:
```bash
# 创建新用户
useradd hdfs -g hadoop

# 修改已存在的用户
usermod hdfs -g hadoop -G supergroup
```

#### isUserExists() - 检查用户是否存在

```java
public static boolean isUserExists(String username) {
    ArrayList<String> commands = new ArrayList<>();
    commands.add("id");
    commands.add(username);
    
    ExecResult execResult = ShellUtils.execWithStatus(
        Constants.INSTALL_PATH, commands, TIME_OUT
    );
    
    return execResult.getExecResult();
}
```

**id 命令输出**:
```bash
$ id hdfs
uid=1001(hdfs) gid=1000(hadoop) groups=1000(hadoop)
```

#### createUnixGroup() - 创建组

```java
public static ExecResult createUnixGroup(String groupName) {
    if (isGroupExists(groupName)) {
        // 组已存在，返回成功
        ExecResult execResult = new ExecResult();
        execResult.setExecResult(true);
        return execResult;
    }
    
    ArrayList<String> commands = new ArrayList<>();
    commands.add("groupadd");
    commands.add(groupName);
    
    return ShellUtils.execWithStatus(Constants.INSTALL_PATH, commands, TIME_OUT);
}
```

#### isGroupExists() - 检查组是否存在

```java
public static boolean isGroupExists(String groupName) {
    ExecResult execResult = ShellUtils.exceShell(
        "egrep \"" + groupName + "\" /etc/group >& /dev/null"
    );
    return execResult.getExecResult();
}
```

**/etc/group 文件格式**:
```
hadoop:x:1000:hdfs,yarn,hive
│      │ │    └─ 组成员列表
│      │ └────── GID (组 ID)
│      └──────── 密码 (通常为 x)
└─────────────── 组名
```

### 5.3 用户管理最佳实践

**① 最小权限原则**:
- 每个服务使用独立的用户运行
- 避免使用 root 运行服务

**② 统一的组管理**:
- 大数据服务统一归属 hadoop 组
- 便于文件权限管理

**③ 幂等性设计**:
- 创建前先检查是否存在
- 避免重复创建导致错误

## 六、FileUtils - 文件工具类

### 6.1 类基本信息

**文件路径**: `com/datasophon/worker/utils/FileUtils.java`  
**行数**: 63 行  
**职责**: 文件操作，读取文件最后 N 行

### 6.2 readLastRows() - 读取最后N行

```java
public static String readLastRows(
    String filename,
    Charset charset,
    int rows
) throws IOException {
    
    charset = charset == null ? Charset.defaultCharset() : charset;
    byte[] lineSeparator = System.getProperty("line.separator").getBytes();
    
    try (RandomAccessFile rf = new RandomAccessFile(filename, "r")) {
        byte[] c = new byte[lineSeparator.length];
        
        // 从文件末尾向前扫描
        for (long pointer = rf.length(), lineSeparatorNum = 0;
             pointer >= 0 && lineSeparatorNum < rows; ) {
            
            // 移动指针
            rf.seek(pointer--);
            
            // 读取数据
            int readLength = rf.read(c);
            
            // 检查是否为换行符
            if (readLength != -1 && Arrays.equals(lineSeparator, c)) {
                lineSeparatorNum++;
            }
            
            // 扫描完依然没有找到足够的行数，将指针归0
            if (pointer == -1 && lineSeparatorNum < rows) {
                rf.seek(0);
            }
        }
        
        // 读取剩余内容
        byte[] tempbytes = new byte[(int) (rf.length() - rf.getFilePointer())];
        rf.readFully(tempbytes);
        return new String(tempbytes, charset);
    }
}
```

### 6.3 工作原理

```
┌─────────────────────────────────────────────────────────────┐
│            读取文件最后 N 行的算法                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  文件内容：                                                 │
│  ┌────────────────────────────────────┐                    │
│  │ Line 1                             │                     │
│  │ Line 2                             │                     │
│  │ Line 3                             │                     │
│  │ Line 4                             │                     │
│  │ Line 5  ← 要读取的开始位置 (N=3)   │                     │
│  │ Line 6                             │                     │
│  │ Line 7  ← 文件末尾                 │                     │
│  └────────────────────────────────────┘                    │
│                                                              │
│  算法步骤：                                                 │
│  ① 从文件末尾开始向前扫描                                   │
│  ② 每次移动一个字节                                         │
│  ③ 检查是否为换行符                                         │
│  ④ 找到 N 个换行符后停止                                    │
│  ⑤ 读取当前位置到文件末尾的内容                             │
│                                                              │
│  时间复杂度： O(文件大小)                                   │
│  空间复杂度： O(读取的行数大小)                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 6.4 使用场景

**① 日志查询**:
```java
// 读取服务日志的最后 500 行
String logContent = FileUtils.readLastRows(
    "/opt/datasophon/hadoop-3.3.3/logs/namenode.log",
    Charset.defaultCharset(),
    500
);
```

**② 错误排查**:
- 快速查看最新的错误信息
- 避免读取整个日志文件

**③ 性能优势**:
- 只读取需要的部分，不加载整个文件到内存
- 适合处理大型日志文件（GB 级别）

## 七、TaskConstants - 任务常量

### 7.1 类基本信息

**文件路径**: `com/datasophon/worker/utils/TaskConstants.java`  
**行数**: 111 行  
**职责**: 定义任务相关的常量

### 7.2 常量分类

#### 正则表达式常量

```java
// YARN 应用 ID 正则
public static final String YARN_APPLICATION_REGEX = "application_\\d+_\\d+";
// 示例: application_1234567890_0001

// Flink Job ID 正则
public static final String FLINK_APPLICATION_REGEX = "JobID \\w+";
// 示例: JobID abc123def456

// setValue 占位符正则
public static final String SETVALUE_REGEX = "[\\$#]\\{setValue\\(([^)]*)\\)}";
// 示例: ${setValue(key=value)}
```

#### 特殊符号常量

```java
public static final String COMMA = ",";
public static final String HYPHEN = "-";
public static final String SLASH = "/";
public static final String COLON = ":";
public static final String SPACE = " ";
```

#### 日志相关常量

```java
// 任务日志记录器名称
public static final String TASK_LOG_LOGGER_NAME = "TaskLogLogger";

// 任务日志记录器名称格式
public static final String TASK_LOG_LOGGER_NAME_FORMAT = TASK_LOG_LOGGER_NAME + "-%s";

// 任务日志前缀
public static final String TASK_LOGGER_INFO_PREFIX = "TASK";
```

#### 其他常量

```java
// 退出码 - 被 kill
public static final int EXIT_CODE_KILL = 137;

// 文件权限
public static final String RWXR_XR_X = "rwxr-xr-x";
```

### 7.3 使用示例

```java
// 构造任务日志记录器名称
String loggerName = String.format(
    TaskConstants.TASK_LOG_LOGGER_NAME_FORMAT,
    "HDFS-NameNode"
);
// 结果: TaskLogLogger-HDFS-NameNode
```

## 八、工具类设计模式

### 8.1 工具类的特点

**① 静态方法**: 所有方法都是 static，无需实例化

**② 无状态**: 工具类本身不保存状态（除 ActorUtils 的 ActorSystem）

**③ 私有构造函数**: 防止实例化（TaskConstants）

```java
private TaskConstants() {
    throw new IllegalStateException("Utility class");
}
```

### 8.2 依赖关系图

```
Handler 层
    │
    ├─────> FreemakerUtils (生成配置文件)
    ├─────> KerberosUtils (下载 keytab)
    └─────> UnixUtils (创建用户)

Strategy 层
    │
    ├─────> KerberosUtils (Kerberos 认证)
    └─────> UnixUtils (用户管理)

Actor 层
    │
    ├─────> ActorUtils (获取远程 Actor)
    └─────> FileUtils (读取日志)
```

## 九、性能优化建议

### 9.1 FreemakerUtils 优化

**缓存模板配置**:
```java
private static Configuration configCache;

public static Configuration getConfiguration() {
    if (configCache == null) {
        synchronized (FreemakerUtils.class) {
            if (configCache == null) {
                configCache = new Configuration(Configuration.getVersion());
                // 配置初始化...
            }
        }
    }
    return configCache;
}
```

### 9.2 FileUtils 优化

**NIO 优化**:
```java
// 使用 MappedByteBuffer 提升大文件读取性能
FileChannel channel = FileChannel.open(Paths.get(filename), StandardOpenOption.READ);
MappedByteBuffer buffer = channel.map(FileChannel.MapMode.READ_ONLY, 0, channel.size());
```

## 十、总结

### 10.1 Utils 层的价值

1. **代码复用**: 避免重复编写相同逻辑
2. **降低耦合**: Handler 和 Strategy 不直接操作底层 API
3. **易于测试**: 工具类可以独立测试
4. **统一标准**: 统一的文件操作、用户管理方式

### 10.2 设计原则

- **单一职责**: 每个工具类职责明确
- **高内聚低耦合**: 工具类之间相互独立
- **易于扩展**: 可以随时添加新的工具类

### 10.3 最佳实践

- **幂等性**: UnixUtils 创建用户前先检查是否存在
- **容错性**: ActorUtils 超时机制避免永久阻塞
- **性能**: FileUtils 只读取需要的部分，不加载整个文件

---

**相关文档**:
- [04-Handler处理器分析.md](./04-Handler处理器分析.md)
- [05-Strategy策略模式详解.md](./05-Strategy策略模式详解.md)
- [07-Log日志系统分析.md](./07-Log日志系统分析.md)
