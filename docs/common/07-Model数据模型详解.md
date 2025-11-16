# DataSophon Common 模块 - Model 数据模型详解

## 一、模块概述

**模块路径**: `datasophon-common/src/main/java/com/datasophon/common/model/`

**功能定位**: 定义系统中使用的数据模型和值对象，提供数据传输和业务逻辑的基础结构

**核心特点**:
- **POJO 设计**: 简单的 Java 对象，无业务逻辑
- **序列化支持**: 大部分类实现 `Serializable`，支持网络传输和持久化
- **Lombok 注解**: 使用 `@Data` 简化代码
- **类型安全**: 强类型约束，避免类型错误

**文件统计**: 共 31 个模型类文件

## 二、模型类分类

### 分类概览

| 分类 | 文件数 | 主要用途 |
|------|-------|---------|
| 服务相关模型 | 10 | 服务、服务角色、服务配置 |
| DAG 图模型 | 3 | 依赖关系管理 |
| 主机相关模型 | 3 | 主机信息、映射关系 |
| 监控相关模型 | 5 | Prometheus 监控数据 |
| 消息相关模型 | 7 | Akka 消息传递 |
| 其他模型 | 3 | 检查结果、配置写入等 |

## 三、服务相关模型

### 3.1 ServiceInfo - 服务信息模型

**功能**: 封装大数据组件服务的完整信息

**完整源码分析**:
```java
@Data
public class ServiceInfo {
    // 服务名称（如 HDFS, YARN, KAFKA）
    private String name;
    
    // 服务显示标签
    private String label;
    
    // 服务版本
    private String version;
    
    // 服务描述
    private String description;
    
    // 服务角色列表（如 NameNode, DataNode）
    private List<ServiceRoleInfo> roles;
    
    // 服务参数配置列表
    private List<ServiceConfig> parameters;
    
    // 依赖的其他服务列表
    private List<String> dependencies;
    
    // 配置文件生成器
    private ConfigWriter configWriter;
    
    // 安装包名称
    private String packageName;
    
    // 解压后的包名称
    private String decompressPackageName;
    
    // 外部链接（WebUI等）
    private ExternalLink externalLink;
    
    // 排序号
    private Integer sortNum;
}
```

**核心功能**:
1. **服务描述**: 完整描述一个大数据服务组件
2. **角色管理**: 包含该服务的所有角色信息
3. **依赖管理**: 声明服务依赖关系，用于安装顺序控制
4. **配置管理**: 定义服务的可配置参数

**使用场景**:
```java
// HDFS服务定义
ServiceInfo hdfsService = new ServiceInfo();
hdfsService.setName("HDFS");
hdfsService.setLabel("Hadoop分布式文件系统");
hdfsService.setVersion("3.3.4");
hdfsService.setPackageName("hadoop-3.3.4.tar.gz");

// 添加角色
List<ServiceRoleInfo> roles = new ArrayList<>();
roles.add(nameNodeRole);
roles.add(dataNodeRole);
hdfsService.setRoles(roles);

// 声明依赖（HDFS无依赖）
hdfsService.setDependencies(Collections.emptyList());
```

**设计亮点**:
- **元数据驱动**: 通过配置描述服务特性，无需硬编码
- **可扩展性**: 易于添加新的服务组件
- **依赖图**: `dependencies` 字段支持构建服务依赖 DAG

### 3.2 ServiceRoleInfo - 服务角色信息模型

**功能**: 描述服务中的特定角色（如 NameNode、DataNode）

**完整源码分析**:
```java
@Data
public class ServiceRoleInfo implements Serializable, Comparable<ServiceRoleInfo> {
    private static final long serialVersionUID = 1L;
    
    // 角色ID
    private Integer id;
    
    // 角色名称
    private String name;
    
    // 角色类型（MASTER/WORKER/CLIENT）
    private ServiceRoleType roleType;
    
    // 基数约束（如 "1", "1+", "1-2"）
    private String cardinality;
    
    // 排序号
    private Integer sortNum;
    
    // 启动运行器
    private ServiceRoleRunner startRunner;
    
    // 停止运行器
    private ServiceRoleRunner stopRunner;
    
    // 状态检查运行器
    private ServiceRoleRunner statusRunner;
    
    // 外部链接
    private ExternalLink externalLink;
    
    // 部署主机名
    private String hostname;
    
    // 主机命令ID
    private String hostCommandId;
    
    // 集群ID
    private Integer clusterId;
    
    // 父服务名称
    private String parentName;
    
    // 框架代码
    private String frameCode;
    
    // 安装包名称
    private String packageName;
    
    // 解压包名称
    private String decompressPackageName;
    
    // 配置文件映射
    private Map<Generators, List<ServiceConfig>> configFileMap;
    
    // 日志文件路径
    private String logFile;
    
    // JMX端口
    private String jmxPort;
    
    // 资源策略
    private List<Map<String, Object>> resourceStrategies;
    
    // 是否为从节点
    private boolean isSlave = false;
    
    // 命令类型
    private CommandType commandType;
    
    // 主节点主机名
    private String masterHost;
    
    // 是否启用Ranger插件
    private Boolean enableRangerPlugin;
    
    // 服务实例ID
    private Integer serviceInstanceId;
    
    // 运行用户
    private RunAs runAs;
    
    @Override
    public int compareTo(ServiceRoleInfo serviceRoleInfo) {
        if (Objects.nonNull(serviceRoleInfo.getSortNum()) 
                && Objects.nonNull(this.getSortNum())) {
            return this.sortNum - serviceRoleInfo.getSortNum();
        }
        return 0;
    }
}
```

**核心功能**:
1. **角色定义**: 完整描述服务中的一个角色
2. **基数约束**: 通过 `cardinality` 控制角色实例数量
3. **运行器管理**: 定义启动、停止、状态检查的执行方式
4. **排序支持**: 实现 `Comparable`，支持启动顺序控制

**基数约束示例**:
- `"1"`: 必须且只能有 1 个实例（如 NameNode Active）
- `"1+"`: 至少 1 个实例（如 DataNode）
- `"0+"`: 可选，0 个或多个（如 HttpFS）
- `"1-2"`: 1 到 2 个实例（如 NameNode HA）

**使用场景**:
```java
// HDFS NameNode角色
ServiceRoleInfo nameNode = new ServiceRoleInfo();
nameNode.setName("NameNode");
nameNode.setRoleType(ServiceRoleType.MASTER);
nameNode.setCardinality("1-2"); // HA模式支持2个

// 设置启动运行器
ServiceRoleRunner startRunner = new ServiceRoleRunner();
startRunner.setProgram("bin/hdfs");
startRunner.setArgs(Arrays.asList("--daemon", "start", "namenode"));
startRunner.setTimeout("30");
nameNode.setStartRunner(startRunner);
```

### 3.3 ServiceConfig - 服务配置模型

**功能**: 定义服务的可配置参数

**完整源码分析**:
```java
@Data
public class ServiceConfig implements Serializable {
    // 配置项名称（如 dfs.replication）
    private String name;
    
    // 配置项值
    private Object value;
    
    // 显示标签
    private String label;
    
    // 配置描述
    private String description;
    
    // 是否必填
    private boolean required;
    
    // 值类型（string/number/boolean）
    private String type;
    
    // 安装向导中是否可配置
    private boolean configurableInWizard;
    
    // 默认值
    private Object defaultValue;
    
    // 最小值（数值型）
    private Integer minValue;
    
    // 最大值（数值型）
    private Integer maxValue;
    
    // 单位（如 MB, GB）
    private String unit;
    
    // 是否隐藏
    private boolean hidden;
    
    // 可选值列表（下拉框）
    private List<String> selectValue;
    
    // 配置类型（自定义分类）
    private String configType;
    
    // 是否与Kerberos相关
    private boolean configWithKerberos;
    
    // 是否与机架感知相关
    private boolean configWithRack;
    
    // 是否与HA相关
    private boolean configWithHA;
    
    // 分隔符（多值配置）
    private String separator;
}
```

**配置类型示例**:
```java
// HDFS副本数配置
ServiceConfig replication = new ServiceConfig();
replication.setName("dfs.replication");
replication.setLabel("副本数");
replication.setType("number");
replication.setDefaultValue(3);
replication.setMinValue(1);
replication.setMaxValue(10);
replication.setRequired(true);
replication.setConfigurableInWizard(true);

// Kerberos相关配置
ServiceConfig kerberosConfig = new ServiceConfig();
kerberosConfig.setName("hadoop.security.authentication");
kerberosConfig.setSelectValue(Arrays.asList("simple", "kerberos"));
kerberosConfig.setConfigWithKerberos(true);
kerberosConfig.setHidden(false); // 启用Kerberos时显示
```

**设计亮点**:
- **UI 友好**: `configurableInWizard` 控制界面显示
- **条件显示**: `configWithKerberos/HA/Rack` 实现条件配置
- **类型约束**: 支持数值范围、可选值限制
- **灵活性**: 支持简单值和复杂对象

### 3.4 SimpleServiceConfig - 简化服务配置

**功能**: 精简版的服务配置模型，用于轻量级场景

**核心属性**:
```java
@Data
public class SimpleServiceConfig implements Serializable {
    // 配置名称
    private String name;
    
    // 配置值
    private String value;
    
    // 配置类型
    private String configType;
}
```

**使用场景**: 快速传递配置，不需要完整的元数据信息

### 3.5 ServiceRoleRunner - 服务角色运行器

**功能**: 定义服务角色的执行方式（启动、停止、状态检查等）

**完整源码分析**:
```java
@Data
public class ServiceRoleRunner implements Serializable {
    // 超时时间（秒）
    private String timeout;
    
    // 执行程序路径
    private String program;
    
    // 程序参数列表
    private List<String> args;
}
```

**使用示例**:
```java
// HDFS NameNode启动运行器
ServiceRoleRunner startRunner = new ServiceRoleRunner();
startRunner.setTimeout("30");
startRunner.setProgram("${HADOOP_HOME}/bin/hdfs");
startRunner.setArgs(Arrays.asList("--daemon", "start", "namenode"));

// HDFS NameNode停止运行器
ServiceRoleRunner stopRunner = new ServiceRoleRunner();
stopRunner.setTimeout("30");
stopRunner.setProgram("${HADOOP_HOME}/bin/hdfs");
stopRunner.setArgs(Arrays.asList("--daemon", "stop", "namenode"));

// HDFS NameNode状态检查运行器
ServiceRoleRunner statusRunner = new ServiceRoleRunner();
statusRunner.setTimeout("10");
statusRunner.setProgram("${HADOOP_HOME}/bin/hdfs");
statusRunner.setArgs(Arrays.asList("haadmin", "-getServiceState", "nn1"));
```

**设计模式**: 策略模式，不同的运行器实现不同的执行策略

### 3.6 ServiceNode - 服务节点（DAG）

**功能**: DAG 图中的服务节点，用于依赖管理

**完整源码分析**:
```java
@Data
public class ServiceNode {
    // 服务名称
    private String serviceName;
    
    // Master角色列表
    private List<ServiceRoleInfo> masterRoles;
    
    // 其他角色列表（Worker/Client）
    private List<ServiceRoleInfo> elseRoles;
    
    // 命令ID
    private String commandId;
    
    // 服务实例ID
    private Integer serviceInstanceId;
}
```

**应用场景**: 批量安装/启动服务时，构建依赖图

**示例**:
```
ZooKeeper (无依赖)
    ↓
HDFS (依赖ZooKeeper)
    ↓
YARN (依赖HDFS)
    ↓
Spark (依赖YARN和HDFS)
```

### 3.7 ServiceNodeEdge - 服务节点边

**功能**: DAG 图中的边，表示服务间的依赖关系

**源码**:
```java
public class ServiceNodeEdge {
    // 空类，作为占位符
}
```

**说明**: 当前版本为空实现，边的信息存储在 DAG 图中

### 3.8 ServiceRoleHostMapping - 服务角色主机映射

**功能**: 映射服务角色到主机的关系

**核心属性**:
```java
@Data
public class ServiceRoleHostMapping implements Serializable {
    // 服务角色名称
    private String serviceRoleName;
    
    // 主机名列表
    private List<String> hostnames;
}
```

**使用场景**:
```java
// DataNode部署到多个主机
ServiceRoleHostMapping mapping = new ServiceRoleHostMapping();
mapping.setServiceRoleName("DataNode");
mapping.setHostnames(Arrays.asList("node1", "node2", "node3"));
```

### 3.9 HostServiceRoleMapping - 主机服务角色映射

**功能**: 映射主机到服务角色的关系（反向映射）

**核心属性**:
```java
@Data
public class HostServiceRoleMapping implements Serializable {
    // 主机名
    private String hostname;
    
    // 服务角色列表
    private List<String> serviceRoles;
}
```

**使用场景**:
```java
// node1主机上运行的服务角色
HostServiceRoleMapping mapping = new HostServiceRoleMapping();
mapping.setHostname("node1");
mapping.setServiceRoles(Arrays.asList("NameNode", "ResourceManager"));
```

### 3.10 ExternalLink - 外部链接

**功能**: 定义服务的 Web UI 等外部访问链接

**核心属性**:
```java
@Data
public class ExternalLink implements Serializable {
    // 链接名称
    private String name;
    
    // 链接标签
    private String label;
    
    // URL模板
    private String url;
    
    // 是否需要认证
    private Boolean requireAuth;
}
```

**使用示例**:
```java
// HDFS NameNode WebUI
ExternalLink webui = new ExternalLink();
webui.setName("namenode-ui");
webui.setLabel("NameNode WebUI");
webui.setUrl("http://${namenode_host}:9870");
webui.setRequireAuth(false);
```

## 四、DAG 图模型

### 4.1 DAG - 有向无环图接口

**功能**: 定义有向无环图的基本操作

**核心方法**:
```java
public interface DAG<Node, NodeInfo, EdgeInfo> {
    // 添加节点
    void addNode(Node node, NodeInfo nodeInfo);
    
    // 添加边
    boolean addEdge(Node fromNode, Node toNode, boolean createNode);
    
    // 检查是否有环
    boolean hasCycle();
    
    // 拓扑排序
    List<Node> topologicalSort() throws Exception;
    
    // 获取起始节点
    Collection<Node> getBeginNode();
    
    // 获取结束节点
    Collection<Node> getEndNode();
    
    // 获取前驱节点
    Set<Node> getPreviousNodes(Node node);
    
    // 获取后继节点
    Set<Node> getSubsequentNodes(Node node);
    
    // 获取入度
    int getIndegree(Node node);
}
```

**应用场景**:
1. **服务依赖管理**: 确定服务安装/启动顺序
2. **任务调度**: 按依赖关系调度任务执行
3. **并发控制**: 无依赖的任务可并行执行

### 4.2 DAGGraph - DAG 图实现类

**功能**: 实现有向无环图的完整功能

**核心数据结构**:
```java
public class DAGGraph<Node, NodeInfo, EdgeInfo> {
    private static final Logger logger = LoggerFactory.getLogger(DAGGraph.class);
    
    // 节点映射：节点 -> 节点信息
    private Map<Node, NodeInfo> nodesMap;
    
    // 边映射：起点 -> (终点 -> 边信息)
    private Map<Node, Map<Node, EdgeInfo>> edgesMap;
    
    // 反向边映射：终点 -> (起点 -> 边信息)
    private Map<Node, Map<Node, EdgeInfo>> reverseEdgesMap;
    
    public DAGGraph() {
        nodesMap = new HashMap<>();
        edgesMap = new HashMap<>();
        reverseEdgesMap = new HashMap<>();
    }
}
```

**核心算法**:

#### 添加边（防止成环）
```java
public boolean addEdge(Node fromNode, Node toNode, boolean createNode) {
    // 1. 检查自环
    if (fromNode.equals(toNode)) {
        logger.error("edge fromNode({}) can't equals toNode({})", 
                     fromNode, toNode);
        return false;
    }
    
    // 2. 检查节点存在性
    if (!createNode) {
        if (!containsNode(fromNode) || !containsNode(toNode)) {
            return false;
        }
    }
    
    // 3. 检查是否会形成环
    // 使用BFS从toNode开始搜索，如果能到达fromNode，则会形成环
    Queue<Node> queue = new LinkedList<>();
    queue.add(toNode);
    int verticesCount = getNodesCount();
    
    while (!queue.isEmpty() && (--verticesCount > 0)) {
        Node key = queue.poll();
        for (Node subsequentNode : getSubsequentNodes(key)) {
            if (subsequentNode.equals(fromNode)) {
                // 会形成环，拒绝添加
                return false;
            }
            queue.add(subsequentNode);
        }
    }
    
    // 4. 可以安全添加边
    return true;
}
```

#### 拓扑排序（Kahn 算法）
```java
public List<Node> topologicalSort() throws Exception {
    // 1. 找出所有入度为0的节点
    Queue<Node> zeroIndegreeQueue = new LinkedList<>();
    Map<Node, Integer> indegreeMap = new HashMap<>();
    
    for (Node node : nodesMap.keySet()) {
        int indegree = getIndegree(node);
        if (indegree == 0) {
            zeroIndegreeQueue.add(node);
        } else {
            indegreeMap.put(node, indegree);
        }
    }
    
    // 2. 如果没有入度为0的节点，说明有环
    if (zeroIndegreeQueue.isEmpty()) {
        throw new Exception("Graph has cycle!");
    }
    
    // 3. Kahn算法：逐步移除入度为0的节点
    List<Node> result = new ArrayList<>();
    while (!zeroIndegreeQueue.isEmpty()) {
        Node node = zeroIndegreeQueue.poll();
        result.add(node);
        
        // 将后继节点的入度减1
        for (Node successor : getSubsequentNodes(node)) {
            int degree = indegreeMap.get(successor) - 1;
            if (degree == 0) {
                zeroIndegreeQueue.add(successor);
                indegreeMap.remove(successor);
            } else {
                indegreeMap.put(successor, degree);
            }
        }
    }
    
    // 4. 如果还有节点未访问，说明有环
    if (!indegreeMap.isEmpty()) {
        throw new Exception("Graph has cycle!");
    }
    
    return result;
}
```

**使用示例**:
```java
// 构建服务依赖图
DAGGraph<String, ServiceNode, String> dag = new DAGGraph<>();

// 添加节点
dag.addNode("ZooKeeper", zkNode);
dag.addNode("HDFS", hdfsNode);
dag.addNode("YARN", yarnNode);
dag.addNode("Spark", sparkNode);

// 添加依赖边
dag.addEdge("ZooKeeper", "HDFS", false); // HDFS依赖ZooKeeper
dag.addEdge("HDFS", "YARN", false);      // YARN依赖HDFS
dag.addEdge("HDFS", "Spark", false);     // Spark依赖HDFS
dag.addEdge("YARN", "Spark", false);     // Spark依赖YARN

// 获取安装顺序（拓扑排序）
List<String> installOrder = dag.topologicalSort();
// 结果: [ZooKeeper, HDFS, YARN, Spark]

// 检查是否有环
if (dag.hasCycle()) {
    logger.error("服务依赖存在循环！");
}
```

**算法复杂度**:
- **添加边**: O(V) - V 为节点数（需要BFS检测环）
- **拓扑排序**: O(V + E) - V 为节点数，E 为边数
- **检查环**: O(V + E)

### 4.3 DAGGraph 的并发安全

**问题**: 原始代码未实现线程安全

**建议优化**:
```java
public class DAGGraph<Node, NodeInfo, EdgeInfo> {
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    public boolean addEdge(Node fromNode, Node toNode, boolean createNode) {
        lock.writeLock().lock();
        try {
            // 添加边的逻辑
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    public List<Node> topologicalSort() throws Exception {
        lock.readLock().lock();
        try {
            // 拓扑排序逻辑
        } finally {
            lock.readLock().unlock();
        }
    }
}
```

## 五、主机相关模型

### 5.1 HostInfo - 主机信息模型

**功能**: 封装集群中主机的完整信息

**完整源码分析**:
```java
@Data
public class HostInfo {
    // 主机名
    private String hostname;
    
    // IP地址
    private String ip;
    
    // 是否受管（是否由DataSophon管理）
    private boolean managed;
    
    // 主机检测结果
    private CheckResult checkResult;
    
    // SSH登录用户
    private String sshUser;
    
    // SSH端口
    private Integer sshPort;
    
    // 安装进度（0-100）
    private Integer progress;
    
    // 集群代码
    private String clusterCode;
    
    // 安装状态（INSTALLING/SUCCESS/FAILED）
    private InstallState installState;
    
    // 安装状态码
    private Integer installStateCode;
    
    // 错误信息
    private String errMsg;
    
    // 提示信息
    private String message;
    
    // 创建时间
    private Date createTime;
    
    // CPU架构（x86_64/aarch64）
    private String cpuArchitecture;
}
```

**状态流转**:
```
未检测 → 检测中 → 检测通过 → 安装中 → 安装成功
                ↓          ↓          ↓
              检测失败  →  安装失败
```

**使用场景**:
```java
// 添加主机到集群
HostInfo host = new HostInfo();
host.setHostname("node1.example.com");
host.setIp("192.168.1.101");
host.setSshUser("root");
host.setSshPort(22);
host.setManaged(true);
host.setCpuArchitecture("x86_64");

// 检测主机
CheckResult checkResult = hostChecker.check(host);
host.setCheckResult(checkResult);

// 安装Agent
if (checkResult.getCode() == 0) {
    host.setInstallState(InstallState.INSTALLING);
    agentInstaller.install(host);
}
```

### 5.2 CheckResult - 检查结果模型

**功能**: 封装主机检查的结果

**完整源码分析**:
```java
@Data
public class CheckResult {
    // 结果代码（0-成功，非0-失败）
    private Integer code;
    
    // 结果消息
    private String msg;
    
    public CheckResult(Integer code, String msg) {
        this.code = code;
        this.msg = msg;
    }
}
```

**常见检查项**:
1. **SSH 连通性**: 能否SSH登录
2. **系统资源**: CPU、内存、磁盘
3. **系统软件**: JDK、Python等
4. **网络**: 端口占用、防火墙
5. **文件系统**: 权限、磁盘空间

**使用示例**:
```java
// 检查通过
CheckResult success = new CheckResult(0, "检查通过");

// 检查失败
CheckResult failed = new CheckResult(1, 
    "JDK未安装或版本不符合要求，需要JDK 1.8+");
```

### 5.3 ProcInfo - 进程信息模型

**功能**: 封装进程信息

**核心属性**:
```java
@Data
public class ProcInfo implements Serializable {
    // 进程ID
    private Integer pid;
    
    // 进程名称
    private String name;
    
    // 进程状态（RUNNING/STOPPED）
    private String status;
    
    // CPU使用率
    private Double cpuUsage;
    
    // 内存使用（MB）
    private Long memoryUsage;
}
```

## 六、监控相关模型

### 6.1 PromResponceInfo - Prometheus 响应信息

**功能**: 封装 Prometheus 查询的响应

**核心属性**:
```java
@Data
public class PromResponceInfo implements Serializable {
    // 响应状态（success/error）
    private String status;
    
    // 数据部分
    private PromDataInfo data;
    
    // 错误类型
    private String errorType;
    
    // 错误信息
    private String error;
}
```

### 6.2 PromDataInfo - Prometheus 数据信息

**功能**: 封装 Prometheus 响应的数据部分

**核心属性**:
```java
@Data
public class PromDataInfo implements Serializable {
    // 结果类型（matrix/vector/scalar/string）
    private String resultType;
    
    // 结果列表
    private List<PromResultInfo> result;
}
```

### 6.3 PromResultInfo - Prometheus 结果信息

**功能**: 封装单个查询结果

**核心属性**:
```java
@Data
public class PromResultInfo implements Serializable {
    // 指标信息
    private PromMetricInfo metric;
    
    // 值（时间戳和值的数组）
    private Object[] value;
    
    // 时间序列值（多个时间点）
    private List<Object[]> values;
}
```

### 6.4 PromMetricInfo - Prometheus 指标信息

**功能**: 封装监控指标的元数据

**核心属性**:
```java
@Data
public class PromMetricInfo implements Serializable {
    // 指标名称
    private String __name__;
    
    // 任务名称
    private String job;
    
    // 实例
    private String instance;
    
    // 其他标签（动态）
    private Map<String, String> labels;
}
```

### 6.5 AlertItem - 告警项模型

**功能**: 定义告警规则

**核心属性**:
```java
@Data
public class AlertItem implements Serializable {
    // 告警名称
    private String alertName;
    
    // 告警表达式（PromQL）
    private String expr;
    
    // 持续时间
    private String forDuration;
    
    // 告警级别（warning/critical）
    private String severity;
    
    // 告警摘要
    private String summary;
    
    // 告警描述
    private String description;
}
```

**使用示例**:
```java
// HDFS NameNode宕机告警
AlertItem alert = new AlertItem();
alert.setAlertName("HDFSNameNodeDown");
alert.setExpr("up{job=\"namenode\"} == 0");
alert.setForDuration("1m");
alert.setSeverity("critical");
alert.setSummary("HDFS NameNode宕机");
alert.setDescription("NameNode在{{$labels.instance}}上已停止运行超过1分钟");
```

## 七、消息相关模型

### 7.1 StartWorkerMessage - 启动 Worker 消息

**功能**: Master 向 Worker 发送的启动消息

**核心属性**:
```java
@Data
public class StartWorkerMessage implements Serializable {
    // 集群ID
    private Integer clusterId;
    
    // 集群代码
    private String clusterCode;
    
    // Master主机名
    private String masterHost;
    
    // Master端口
    private Integer masterPort;
    
    // Worker主机名
    private String workerHost;
}
```

### 7.2 StartWorkerMessageConfirmed - Worker 启动确认消息

**功能**: Worker 向 Master 发送的启动确认

**核心属性**:
```java
@Data
public class StartWorkerMessageConfirmed implements Serializable {
    // Worker主机名
    private String hostname;
    
    // 确认时间
    private Long confirmTime;
    
    // Worker版本
    private String version;
}
```

### 7.3 WorkerServiceMessage - Worker 服务消息

**功能**: Worker 定期向 Master 报告的心跳消息

**核心属性**:
```java
@Data
public class WorkerServiceMessage implements Serializable {
    // Worker主机名
    private String hostname;
    
    // 集群ID
    private Integer clusterId;
    
    // 心跳时间
    private Long heartbeatTime;
    
    // 运行的服务角色列表
    private List<ServiceRoleInfo> runningRoles;
    
    // CPU使用率
    private Double cpuUsage;
    
    // 内存使用率
    private Double memoryUsage;
    
    // 磁盘使用率
    private Double diskUsage;
}
```

### 7.4 ServiceExecuteResultMessage - 服务执行结果消息

**功能**: Worker 向 Master 报告服务执行结果

**核心属性**:
```java
@Data
public class ServiceExecuteResultMessage implements Serializable {
    // 服务名称
    private String serviceName;
    
    // 服务角色名称
    private String serviceRoleName;
    
    // 执行结果（SUCCESS/FAILED）
    private String result;
    
    // 错误信息
    private String errorMsg;
    
    // 执行耗时
    private Long duration;
}
```

### 7.5 AkkaRemoteReply - Akka 远程应答

**功能**: 通用的 Akka 远程调用应答

**核心属性**:
```java
@Data
public class AkkaRemoteReply implements Serializable {
    // 应答类型
    private String replyType;
    
    // 应答数据
    private Object data;
    
    // 错误信息
    private String error;
}
```

### 7.6 UpdateCommandMessage - 更新命令消息

**功能**: 通知更新命令状态

**核心属性**:
```java
@Data
public class UpdateCommandMessage implements Serializable {
    // 命令ID
    private String commandId;
    
    // 新状态
    private String newState;
    
    // 更新时间
    private Long updateTime;
}
```

### 7.7 UpdateCommandHostMessage - 更新命令主机消息

**功能**: 更新命令在特定主机上的执行状态

**核心属性**:
```java
@Data
public class UpdateCommandHostMessage implements Serializable {
    // 主机命令ID
    private String hostCommandId;
    
    // 主机名
    private String hostname;
    
    // 命令状态
    private String commandState;
    
    // 进度
    private Integer progress;
}
```

## 八、其他模型

### 8.1 ConfigWriter - 配置写入器

**功能**: 定义配置文件的写入方式

**核心属性**:
```java
@Data
public class ConfigWriter implements Serializable {
    // 配置生成器列表
    private List<Generators> generators;
}
```

### 8.2 Generators - 配置生成器

**功能**: 定义如何生成配置文件

**完整源码分析**:
```java
@Data
public class Generators implements Serializable {
    // 文件名
    private String filename;
    
    // 配置格式（properties/xml/yaml）
    private String configFormat;
    
    // 输出目录
    private String outputDirectory;
    
    // 包含的参数列表
    private List<String> includeParams;
    
    // 模板名称
    private String templateName;
    
    // 条件属性（满足条件才生成）
    private String conditionalOnProperty;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Generators that = (Generators) o;
        return filename.equals(that.filename);
    }
    
    @Override
    public int hashCode() {
        return filename.hashCode();
    }
}
```

**使用示例**:
```java
// 生成 hdfs-site.xml
Generators hdfsSiteGen = new Generators();
hdfsSiteGen.setFilename("hdfs-site.xml");
hdfsSiteGen.setConfigFormat("xml");
hdfsSiteGen.setOutputDirectory("${HADOOP_CONF_DIR}");
hdfsSiteGen.setIncludeParams(Arrays.asList(
    "dfs.replication",
    "dfs.namenode.name.dir",
    "dfs.datanode.data.dir"
));

// 条件生成（仅HA模式）
Generators haConfigGen = new Generators();
haConfigGen.setConditionalOnProperty("hadoop.ha.enabled=true");
haConfigGen.setFilename("hdfs-site-ha.xml");
```

### 8.3 ExecCmdResult - 命令执行结果

**功能**: 封装 Shell 命令的执行结果

**完整源码分析**:
```java
public class ExecCmdResult {
    // 命令执行是否成功
    private boolean success;
    
    // 输出结果
    private String result;
    
    public boolean isSuccess() {
        return success;
    }
    
    public void setSuccess(boolean success) {
        this.success = success;
    }
    
    public String getResult() {
        return result;
    }
    
    public void setResult(String result) {
        this.result = result;
    }
}
```

**使用场景**:
```java
// 执行Shell命令
ExecCmdResult result = ShellUtils.exec("free -m");
if (result.isSuccess()) {
    String memoryInfo = result.getResult();
    // 解析内存信息
}
```

### 8.4 RunAs - 运行用户

**功能**: 指定服务运行的系统用户和组

**完整源码分析**:
```java
@Data
public class RunAs implements Serializable {
    // 用户名
    private String user;
    
    // 用户组
    private String group;
}
```

**使用示例**:
```java
// HDFS以hdfs用户运行
RunAs hdfsUser = new RunAs();
hdfsUser.setUser("hdfs");
hdfsUser.setGroup("hadoop");

// Kafka以kafka用户运行
RunAs kafkaUser = new RunAs();
kafkaUser.setUser("kafka");
kafkaUser.setGroup("kafka");
```

**重要性**: 
- **安全隔离**: 不同服务使用不同用户，避免权限泄露
- **文件权限**: 确保服务能访问所需的文件和目录
- **审计追踪**: 便于审计和追踪操作

## 九、模型设计模式

### 9.1 POJO 模式

所有模型类都采用 POJO（Plain Old Java Object）设计：
- 无业务逻辑，只有数据
- 使用 Lombok `@Data` 简化代码
- 实现 `Serializable` 支持序列化

### 9.2 值对象模式

部分模型采用值对象（Value Object）模式：
- 不可变性（通过 final 字段）
- 基于值的相等性（重写 equals/hashCode）
- 无标识符

**示例**: `Generators` 类重写 `equals` 和 `hashCode`，基于 `filename` 判断相等

### 9.3 数据传输对象（DTO）模式

模型类主要用作 DTO，在不同层之间传递数据：
```
API层 → Service层 → Worker层
  ↓        ↓          ↓
 DTO     Domain     DTO
```

## 十、序列化设计

### 10.1 为什么需要序列化

1. **网络传输**: Master 和 Worker 之间通过 Akka 传递对象
2. **持久化**: 命令和状态可能需要保存到数据库
3. **缓存**: 对象可能被缓存到 Redis 等

### 10.2 序列化最佳实践

```java
@Data
public class MyModel implements Serializable {
    // 1. 定义版本号
    private static final long serialVersionUID = 1L;
    
    // 2. 所有字段都应该是可序列化的
    private String name;
    private Integer id;
    private List<String> items; // List/Map等集合类型也是可序列化的
    
    // 3. 不可序列化的字段使用transient
    private transient Thread workThread; // 不会被序列化
}
```

## 十一、最佳实践

### 11.1 模型类设计原则

1. **单一职责**: 每个模型类只表示一个概念
2. **完整性**: 包含该概念的所有必要属性
3. **一致性**: 相关模型使用一致的命名和结构
4. **可扩展性**: 便于添加新字段，保持向后兼容

### 11.2 字段命名规范

- 使用驼峰命名: `serviceName`, `clusterId`
- 布尔字段使用 `is/has/enable` 前缀: `isSlave`, `enableKerberos`
- 集合字段使用复数: `roles`, `parameters`

### 11.3 空值处理

```java
@Data
public class ServiceInfo {
    // 使用默认值避免NPE
    private List<ServiceRoleInfo> roles = new ArrayList<>();
    
    // 明确可空字段使用包装类型
    private Integer sortNum; // 可能为null
    
    // 基本类型用于必填字段
    private int clusterId; // 不应为null
}
```

## 十二、性能优化建议

### 12.1 减少对象大小

1. **按需传递**: 不要传递不必要的字段
2. **使用简化模型**: 对于轻量级场景使用 `SimpleXxx` 类
3. **引用共享**: 共用的配置对象可以共享引用

### 12.2 集合优化

```java
// 好的实践：预分配容量
List<ServiceRoleInfo> roles = new ArrayList<>(expectedSize);

// 避免：频繁扩容
List<ServiceRoleInfo> roles = new ArrayList<>();
for (int i = 0; i < 1000; i++) {
    roles.add(...); // 会导致多次扩容
}
```

### 12.3 不可变对象

对于不会修改的配置对象，考虑使用不可变设计：
```java
@Value // Lombok注解，生成不可变对象
public class ImmutableConfig {
    String name;
    String value;
}
```

## 十三、总结

Model 模块是 DataSophon 的数据基础，提供了：

### 核心价值

1. **类型安全**: 强类型约束，编译期检查
2. **自文档化**: 通过类和字段名表达语义
3. **可维护性**: 统一的数据结构便于理解和维护
4. **可扩展性**: 易于添加新字段和新模型

### 技术亮点

1. **DAG 图**: 优雅的依赖管理实现
2. **序列化支持**: 完整的网络传输支持
3. **分层设计**: 清晰的模型分类
4. **Lombok 集成**: 简洁的代码风格

### 应用场景

- 服务定义和配置
- DAG 依赖管理
- 主机信息管理
- 监控数据传递
- Akka 消息通信

Model 模块的设计体现了 DataSophon 对数据建模的深入思考和领域驱动设计的最佳实践。
