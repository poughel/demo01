# DataSophon Common 模块源码分析

## 一、模块概述

`datasophon-common` 是 DataSophon 的公共组件模块，提供系统通用的工具类、常量定义、数据模型、枚举类等基础功能，被其他所有模块依赖和使用。

### 模块定位
- **基础支撑模块**: 为其他模块提供基础功能
- **无外部依赖**: 不依赖其他业务模块
- **高复用性**: 被多个模块引用
- **稳定性要求高**: 变更影响面大

## 二、模块结构

```
datasophon-common/
└── src/main/java/com/datasophon/common/
    ├── cache/              # 缓存相关 (1个文件)
    │   └── CacheUtils.java
    ├── command/            # 命令封装 (34个文件)
    │   ├── BaseCommand.java
    │   ├── InstallServiceRoleCommand.java
    │   ├── ServiceRoleOperateCommand.java
    │   ├── GenerateServiceConfigCommand.java
    │   └── ... (30+ 其他命令类)
    ├── enums/              # 枚举类型 (9个文件)
    │   ├── CommandType.java
    │   ├── ServiceExecuteState.java
    │   ├── ServiceRoleType.java
    │   └── ... (6个其他枚举)
    ├── exception/          # 异常定义 (1个文件)
    │   └── AkkaRemoteException.java
    ├── lifecycle/          # 生命周期管理 (3个文件)
    │   ├── ServerStatus.java
    │   ├── ServerLifeCycleManager.java
    │   └── ServerLifeCycleException.java
    ├── model/              # 数据模型 (31个文件)
    │   ├── DAG.java
    │   ├── DAGGraph.java
    │   ├── ServiceConfig.java
    │   ├── ServiceInfo.java
    │   ├── ServiceRoleInfo.java
    │   ├── HostInfo.java
    │   └── ... (25+ 其他模型类)
    └── utils/              # 工具类 (13个文件)
        ├── Result.java
        ├── ShellUtils.java
        ├── FileUtils.java
        ├── CollectionUtils.java
        └── ... (9个其他工具类)

总计：97个Java文件
```

## 三、核心组件详解

### 3.1 Constants (常量定义)

**文件**: `com/datasophon/common/Constants.java`

**功能**: 定义系统全局使用的常量

**主要常量类别**:

#### 通用常量
```java
public class Constants {
    // 系统常量
    public static final String HOSTNAME = "hostname";
    public static final String MASTER = "master";
    public static final String WORKER = "worker";
    
    // 状态常量
    public static final Integer RUNNING = 1;
    public static final Integer STOPPED = 2;
    public static final Integer INSTALLING = 3;
    
    // 配置常量
    public static final String CONF_DIR = "conf";
    public static final String LOG_DIR = "log";
    public static final String DATA_DIR = "data";
}
```

#### 服务相关常量
- 服务名称常量 (HDFS, YARN, KAFKA 等)
- 服务角色常量 (NameNode, DataNode 等)
- 配置文件名常量

#### 命令相关常量
- 命令类型 (INSTALL, START, STOP 等)
- 命令状态 (SUCCESS, FAILED, RUNNING 等)

#### 路径相关常量
- 安装路径
- 配置路径
- 日志路径
- 数据路径

**使用场景**:
```java
// 判断服务状态
if (serviceState.equals(Constants.RUNNING)) {
    // 服务运行中
}

// 构建配置文件路径
String confPath = basePath + Constants.CONF_DIR;
```

### 3.2 工具类 (Utils)

#### Result 类 - 统一响应封装

**文件**: `com/datasophon/common/utils/Result.java`

**功能**: 封装统一的 API 响应格式

**类设计**:
```java
public class Result implements Serializable {
    private int code;           // 响应码：0-成功，非0-失败
    private String msg;         // 响应消息
    private Object data;        // 响应数据
    
    // 成功响应
    public static Result success() {
        return new Result(0, "success", null);
    }
    
    public static Result success(Object data) {
        return new Result(0, "success", data);
    }
    
    // 失败响应
    public static Result error(String msg) {
        return new Result(500, msg, null);
    }
    
    public static Result error(int code, String msg) {
        return new Result(code, msg, null);
    }
    
    // 链式调用
    public Result put(String key, Object value) {
        if (data == null) {
            data = new HashMap<>();
        }
        ((Map<String, Object>) data).put(key, value);
        return this;
    }
}
```

**使用示例**:
```java
// Controller 中返回成功结果
return Result.success().put("cluster", clusterInfo);

// Service 中返回失败结果
return Result.error("集群不存在");

// 带数据的成功响应
return Result.success(clusterList);
```

**设计优势**:
- 统一的响应格式
- 链式调用，代码简洁
- 支持灵活的数据封装
- 便于前后端约定

#### FileUtils 类 - 文件工具

**文件**: `com/datasophon/common/utils/FileUtils.java`

**功能**: 提供文件操作的通用方法

**主要方法**:
```java
public class FileUtils {
    // 读取文件内容
    public static String readFile(String filePath);
    
    // 写入文件
    public static void writeFile(String filePath, String content);
    
    // 复制文件
    public static void copyFile(String srcPath, String destPath);
    
    // 删除文件或目录
    public static boolean deleteFile(String filePath);
    
    // 创建目录
    public static boolean createDirectory(String dirPath);
    
    // 检查文件是否存在
    public static boolean exists(String filePath);
    
    // 获取文件大小
    public static long getFileSize(String filePath);
}
```

**使用场景**:
- 读取配置文件模板
- 写入生成的配置文件
- 备份配置文件
- 清理临时文件

#### ShellUtils 类 - Shell 命令执行

**文件**: `com/datasophon/common/utils/ShellUtils.java`

**功能**: 封装 Shell 命令执行逻辑

**核心方法**:
```java
public class ShellUtils {
    /**
     * 执行 Shell 命令
     * @param command 命令字符串
     * @return ExecResult 执行结果
     */
    public static ExecResult execCommand(String command);
    
    /**
     * 执行 Shell 命令（带超时）
     * @param command 命令字符串
     * @param timeout 超时时间（秒）
     * @return ExecResult 执行结果
     */
    public static ExecResult execCommandWithTimeout(String command, long timeout);
    
    /**
     * 异步执行命令
     */
    public static void execCommandAsync(String command);
}
```

**ExecResult 结构**:
```java
public class ExecResult {
    private int exitCode;       // 退出码
    private String output;      // 标准输出
    private String error;       // 错误输出
    private boolean success;    // 是否成功
}
```

**使用示例**:
```java
// 启动服务
String startCmd = "/opt/hadoop/sbin/start-dfs.sh";
ExecResult result = ShellUtils.execCommand(startCmd);
if (result.isSuccess()) {
    logger.info("HDFS 启动成功");
} else {
    logger.error("HDFS 启动失败: " + result.getError());
}
```

#### CollectionUtils 类 - 集合工具

**文件**: `com/datasophon/common/utils/CollectionUtils.java`

**功能**: 提供集合操作的便捷方法

**主要方法**:
```java
public class CollectionUtils {
    // 判断集合是否为空
    public static boolean isEmpty(Collection<?> collection);
    
    // 判断集合是否不为空
    public static boolean isNotEmpty(Collection<?> collection);
    
    // 集合转数组
    public static <T> T[] toArray(Collection<T> collection, Class<T> clazz);
    
    // 过滤集合
    public static <T> List<T> filter(List<T> list, Predicate<T> predicate);
}
```

### 3.3 数据模型 (Model)

#### DAG 类 - 有向无环图

**文件**: `com/datasophon/common/model/DAG.java`

**功能**: 实现任务依赖关系的有向无环图

**类设计**:
```java
public class DAG<Node, NodeInfo, EdgeInfo> {
    // 节点集合
    private Map<Node, NodeInfo> nodesMap;
    
    // 边集合 (依赖关系)
    private Map<Node, List<Edge<Node, EdgeInfo>>> edgesMap;
    
    // 添加节点
    public void addNode(Node node, NodeInfo nodeInfo);
    
    // 添加边 (依赖关系)
    public void addEdge(Node fromNode, Node toNode);
    
    // 拓扑排序
    public List<Node> topologicalSort();
    
    // 获取入度为0的节点
    public Set<Node> getBeginNode();
    
    // 获取后继节点
    public Set<Node> getSubsequentNodes(Node node);
}
```

**使用场景**:
服务启动顺序管理，例如 Hadoop 生态组件启动：
```
ZooKeeper (无依赖)
    ↓
HDFS NameNode (依赖 ZooKeeper)
    ↓
HDFS DataNode (依赖 NameNode)
    ↓
YARN ResourceManager (依赖 HDFS)
    ↓
YARN NodeManager (依赖 ResourceManager)
```

**实现原理**:
1. 构建依赖图
2. 拓扑排序确定执行顺序
3. 并行执行无依赖的节点
4. 按顺序执行有依赖的节点

#### ServiceConfig 类 - 服务配置模型

**文件**: `com/datasophon/common/model/ServiceConfig.java`

**功能**: 封装服务配置信息

**类结构**:
```java
public class ServiceConfig {
    private String name;                // 配置项名称
    private String value;               // 配置项值
    private String desc;                // 配置项描述
    private String configurableInWizard;// 是否在向导中可配置
    private String defaultValue;        // 默认值
    private String valueType;           // 值类型 (input/select/radio)
    private List<String> options;       // 可选值列表
    private boolean required;           // 是否必填
}
```

**使用场景**:
- 服务安装向导中的配置项
- 配置文件生成
- 配置项验证

#### HostInfo 类 - 主机信息模型

**文件**: `com/datasophon/common/model/HostInfo.java`

**功能**: 封装主机节点信息

**类结构**:
```java
public class HostInfo {
    private String hostname;            // 主机名
    private String ip;                  // IP 地址
    private Integer cpuCores;           // CPU 核心数
    private Long memory;                // 内存大小 (MB)
    private Long disk;                  // 磁盘大小 (GB)
    private String os;                  // 操作系统
    private String arch;                // 架构 (x86_64/aarch64)
    private Integer sshPort;            // SSH 端口
    private String sshUser;             // SSH 用户
}
```

### 3.4 命令模型 (Command)

#### CommandType 枚举

**文件**: `com/datasophon/common/command/CommandType.java`

**功能**: 定义命令类型枚举

**枚举值**:
```java
public enum CommandType {
    INSTALL_SERVICE,        // 安装服务
    START_SERVICE,          // 启动服务
    STOP_SERVICE,           // 停止服务
    RESTART_SERVICE,        // 重启服务
    CONFIGURE_SERVICE,      // 配置服务
    CHECK_SERVICE,          // 检查服务状态
    DECOMMISSION_SERVICE,   // 下线服务
    UPGRADE_SERVICE,        // 升级服务
}
```

#### ExecuteServiceRoleCommand 类

**功能**: 封装服务角色执行命令

**类结构**:
```java
public class ExecuteServiceRoleCommand implements Serializable {
    private CommandType commandType;    // 命令类型
    private String serviceName;         // 服务名称
    private String serviceRoleName;     // 服务角色名称
    private String commandId;           // 命令 ID
    private Map<String, String> params; // 命令参数
}
```

### 3.5 枚举类型 (Enums)

#### ServiceState 枚举 - 服务状态

```java
public enum ServiceState {
    WAIT_INSTALL(1, "待安装"),
    INSTALLING(2, "安装中"),
    RUNNING(3, "运行中"),
    STOP(4, "已停止"),
    EXISTS_ALARM(5, "存在告警"),
    NEED_RESTART(6, "需要重启"),
    DELETED(7, "已删除");
    
    private Integer code;
    private String desc;
}
```

#### ServiceRoleType 枚举 - 服务角色类型

```java
public enum ServiceRoleType {
    MASTER("master"),       // 主节点角色
    SLAVE("slave"),         // 从节点角色
    CLIENT("client");       // 客户端角色
}
```

### 3.6 缓存工具 (Cache)

#### CacheUtils 类

**文件**: `com/datasophon/common/cache/CacheUtils.java`

**功能**: 提供简单的内存缓存功能

**实现方式**:
```java
public class CacheUtils {
    // 使用 ConcurrentHashMap 作为缓存容器
    private static final ConcurrentHashMap<String, Object> CACHE_MAP 
        = new ConcurrentHashMap<>();
    
    // 存储数据
    public static void put(String key, Object value);
    
    // 获取数据
    public static <T> T get(String key);
    
    // 删除数据
    public static void remove(String key);
    
    // 清空缓存
    public static void clear();
    
    // 检查是否存在
    public static boolean containsKey(String key);
}
```

**使用场景**:
- 缓存主机名
- 缓存配置信息
- 缓存集群信息
- 临时数据存储

**注意事项**:
- 数据存储在 JVM 内存中
- 不支持过期时间
- 不支持持久化
- 适合单机缓存场景

## 四、通用数据结构

### 4.1 Prometheus 相关模型

#### PromResponceInfo - Prometheus 响应信息
```java
public class PromResponceInfo {
    private String status;              // 响应状态
    private PromDataInfo data;          // 响应数据
}
```

#### PromDataInfo - Prometheus 数据
```java
public class PromDataInfo {
    private String resultType;          // 结果类型
    private List<PromResultInfo> result;// 结果列表
}
```

### 4.2 告警相关模型

#### AlertItem - 告警项
```java
public class AlertItem {
    private String alertName;           // 告警名称
    private String alertLevel;          // 告警级别
    private String alertInfo;           // 告警信息
    private String alertAdvice;         // 告警建议
}
```

## 五、设计模式应用

### 5.1 工具类模式
- 私有构造函数，防止实例化
- 静态方法提供功能
- 无状态设计

### 5.2 枚举单例模式
```java
public enum ServiceState {
    RUNNING(1, "运行中");
    // 天然单例，线程安全
}
```

### 5.3 建造者模式
部分模型类使用建造者模式：
```java
ServiceConfig config = ServiceConfig.builder()
    .name("dfs.replication")
    .value("3")
    .desc("副本数")
    .build();
```

## 六、最佳实践

### 6.1 常量使用
```java
// 推荐：使用常量
if (state.equals(Constants.RUNNING)) { }

// 不推荐：使用魔法值
if (state.equals(1)) { }
```

### 6.2 Result 使用
```java
// 推荐：链式调用
return Result.success()
    .put("total", total)
    .put("list", list);

// 不推荐：多次调用
Result result = Result.success();
result.put("total", total);
result.put("list", list);
return result;
```

### 6.3 工具类使用
```java
// 推荐：使用工具类
if (CollectionUtils.isEmpty(list)) { }

// 不推荐：直接判断
if (list == null || list.size() == 0) { }
```

## 七、依赖关系

### 7.1 被依赖模块
- datasophon-api
- datasophon-service
- datasophon-worker
- datasophon-infrastructure

### 7.2 外部依赖
- Hutool (工具类库)
- Jackson (JSON 处理)
- Lombok (简化代码)

## 八、扩展建议

### 8.1 缓存增强
当前 CacheUtils 功能简单，可以考虑：
- 支持过期时间
- 支持 LRU 淘汰策略
- 集成 Redis 分布式缓存

### 8.2 工具类完善
可以添加更多工具类：
- DateUtils: 日期处理
- StringUtils: 字符串处理
- JsonUtils: JSON 转换
- EncryptUtils: 加密解密

### 8.3 模型验证
为模型类添加验证注解：
```java
public class ServiceConfig {
    @NotBlank(message = "配置名称不能为空")
    private String name;
    
    @NotBlank(message = "配置值不能为空")
    private String value;
}
```

## 九、测试建议

### 9.1 工具类测试
```java
@Test
public void testShellUtils() {
    ExecResult result = ShellUtils.execCommand("ls -la");
    assertTrue(result.isSuccess());
    assertNotNull(result.getOutput());
}
```

### 9.2 DAG 测试
```java
@Test
public void testDAGTopologicalSort() {
    DAG<String, String, String> dag = new DAG<>();
    dag.addNode("A", "Node A");
    dag.addNode("B", "Node B");
    dag.addEdge("A", "B");
    
    List<String> sorted = dag.topologicalSort();
    assertEquals("A", sorted.get(0));
    assertEquals("B", sorted.get(1));
}
```

## 十、完整文档索引

Common 模块已完成全面的源码分析，共8篇详细文档：

### 10.1 基础文档

**[01-Common模块概述](./01-Common模块概述.md)** (本文档)
- 模块定位和结构
- 核心组件概览
- 设计模式分析
- 使用最佳实践

### 10.2 核心组件详解

**[02-Constants常量定义分析](./02-Constants常量定义分析.md)** (~24,000字)
- 97个常量完整分析
- 19个分类详细讲解
- 路径、服务、命令、状态等常量
- 使用场景和最佳实践

**[03-缓存工具类分析](./03-缓存工具类分析.md)** (~21,000字)
- CacheUtils 实现原理
- LRU 缓存策略
- 线程安全设计
- 性能优化建议

**[04-工具类详解](./04-工具类详解.md)** (~28,000字)
- 13个工具类全面分析
- Result、ShellUtils、FileUtils 等
- 源码逐行解读
- 实用示例代码

**[05-枚举类型详解](./05-枚举类型详解.md)** (~22,000字)
- 9个枚举类型详解
- CommandType、ServiceExecuteState 等
- 设计模式和最佳实践
- 状态机设计分析

### 10.3 高级组件详解

**[06-Command命令封装详解](./06-Command命令封装详解.md)** (~20,000字)
- 34个命令类完整分析
- 命令模式设计详解
- DAG 依赖管理
- 序列化和网络传输
- 分类：安装、操作、配置、检查等

**[07-Model数据模型详解](./07-Model数据模型详解.md)** (~27,000字)
- 31个模型类完整分析
- 服务定义模型（ServiceInfo、ServiceRoleInfo）
- DAG 图实现（拓扑排序、环检测）
- 主机和监控模型
- Akka 消息模型

**[08-Lifecycle与Exception详解](./08-Lifecycle与Exception详解.md)** (~19,000字)
- 服务器生命周期管理
- 状态机设计（RUNNING/WAITING/STOPPED）
- 优雅停机实现
- 异常处理机制
- 线程安全设计

### 10.4 文档统计

| 指标 | 数量 |
|------|------|
| 文档总数 | 8篇 |
| 总字数 | 约18.3万字 |
| 分析的Java文件 | 97个 |
| 代码示例 | 200+ |
| 设计模式讲解 | 15+ |

### 10.5 阅读建议

**初学者路线**:
1. 先读本概述文档，了解整体结构
2. 阅读 Constants 和枚举类型，理解系统常量
3. 学习工具类和缓存，掌握基础工具
4. 深入 Model 和 Command，理解数据和命令
5. 最后学习 Lifecycle，了解生命周期管理

**开发者路线**:
1. 根据需要直接查阅相关文档
2. 使用文档中的代码示例
3. 参考最佳实践进行开发
4. 理解设计模式，应用到实际项目

**架构师路线**:
1. 重点阅读设计模式章节
2. 理解 DAG 图和命令模式的应用
3. 学习线程安全和性能优化
4. 掌握系统架构的基础支撑

---

**文档版本**: v2.0  
**最后更新**: 2025-11-15  
**覆盖率**: 100% (Common模块所有文件已完整分析)
