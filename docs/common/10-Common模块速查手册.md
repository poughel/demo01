# Common 模块速查手册

## 📘 手册说明

本手册是 DataSophon Common 模块的快速参考指南，提供常用类、方法和最佳实践的简明查询，适合日常开发快速查阅。

**目标读者**: 正在使用 Common 模块进行开发的工程师

**使用方法**: 
- 按 Ctrl+F 搜索关键字快速定位
- 查看示例代码快速上手
- 参考最佳实践避免常见问题

---

## 🔍 一、常用常量速查

### 1.1 路径常量

```java
// 安装路径
Constants.INSTALL_PATH              // "/opt/datasophon"
Constants.SERVICE_DIR               // "/opt/datasophon"

// 包路径
Constants.PACKAGE_PATH             // "/opt/datasophon/DDP/packages"

// Worker 路径
Constants.WORKER_PACKAGE_PATH      // "/opt/datasophon/worker"

// 配置路径
Constants.PROMETHEUS_CONFIG_PATH   // "/opt/datasophon/prometheus/conf"
Constants.ALERT_CONFIG_PATH        // "/opt/datasophon/prometheus/alertmanager"
```

**使用场景**: 文件操作、服务安装、配置管理

### 1.2 命令相关常量

```java
// Shell 命令
Constants.SHELL_COMMAND            // "bash"
Constants.COMMAND_TIMEOUT          // 300000 (5分钟)

// 分隔符
Constants.COMMA                    // ","
Constants.SEMICOLON               // ";"
Constants.EQUAL_SIGN              // "="
```

**使用场景**: 命令执行、字符串处理

### 1.3 服务状态常量

```java
// 服务状态
Constants.RUNNING                  // "运行中"
Constants.STOP                     // "已停止"
Constants.EXISTS                   // "已存在"
Constants.NOT_EXISTS              // "不存在"

// 安装状态
Constants.INSTALLING              // "正在安装"
Constants.INSTALLED               // "已安装"
```

**使用场景**: 状态判断、UI展示

---

## 🛠️ 二、常用工具类速查

### 2.1 Result - 统一结果封装

```java
// 成功返回
Result.success();
Result.success(data);
Result.success("操作成功", data);

// 失败返回
Result.failed();
Result.failed("错误信息");
Result.failed(errorCode, "错误信息");

// 判断结果
if (result.isSuccess()) {
    Object data = result.getData();
}

// 获取错误信息
String msg = result.getMsg();
```

**最佳实践**:
- ✅ 所有 API 返回统一使用 Result
- ✅ 成功时设置 data，失败时设置 msg
- ✅ 使用有意义的错误消息
- ❌ 避免在 Result 中嵌套 Result

### 2.2 ShellUtils - Shell 命令执行

```java
// 执行命令（同步）
ExecResult result = ShellUtils.execWithStatus(
    "/home/user",           // 工作目录
    null,                   // 环境变量
    10000L,                 // 超时时间（毫秒）
    "ls -l"                // 命令
);

// 检查结果
if (result.getExitCode() == 0) {
    String output = result.getOutput();
}

// 异步执行
Future<ExecResult> future = ShellUtils.execAsync(command);
```

**最佳实践**:
- ✅ 始终设置合理的超时时间
- ✅ 检查 exitCode 判断成功/失败
- ✅ 捕获并处理异常
- ✅ 长时间运行的命令使用异步执行
- ❌ 避免在循环中频繁执行 shell 命令

### 2.3 FileUtils - 文件操作

```java
// 读取文件内容
String content = FileUtils.readFile("/path/to/file");

// 写入文件
FileUtils.writeFile(content, "/path/to/file", false); // 覆盖
FileUtils.writeFile(content, "/path/to/file", true);  // 追加

// 复制文件
FileUtils.copy(sourceFile, destFile, true);  // 覆盖

// 创建目录
FileUtils.createDirectory("/path/to/dir");

// 删除文件/目录
FileUtils.delete(file);

// 检查文件存在
if (FileUtils.exists(path)) {
    // ...
}
```

**最佳实践**:
- ✅ 操作前检查文件是否存在
- ✅ 使用 try-catch 捕获 IOException
- ✅ 关闭文件流（使用 try-with-resources）
- ✅ 处理大文件时使用流式读取
- ❌ 避免频繁的小文件操作

### 2.4 CollectionUtils - 集合工具

```java
// 判空
if (CollectionUtils.isEmpty(list)) {
    // 集合为 null 或空
}

if (CollectionUtils.isNotEmpty(list)) {
    // 集合不为空
}

// 判断相等
if (CollectionUtils.isEqualCollection(list1, list2)) {
    // 两个集合相等（忽略顺序）
}

// 获取集合大小
int size = CollectionUtils.size(collection);  // null 安全
```

**最佳实践**:
- ✅ 使用 isEmpty/isNotEmpty 替代 list==null || list.size()==0
- ✅ 优先使用工具方法，避免 NPE
- ❌ 不要对 null 集合直接调用方法

### 2.5 EncryptionUtils - 加密工具

```java
// MD5 加密
String md5 = EncryptionUtils.md5("password");

// SHA-256 加密
String sha256 = EncryptionUtils.sha256("password");

// AES 加密/解密
String encrypted = EncryptionUtils.aesEncrypt("data", "key");
String decrypted = EncryptionUtils.aesDecrypt(encrypted, "key");
```

**最佳实践**:
- ✅ 密码必须加密存储
- ✅ 使用强加密算法（SHA-256, AES）
- ✅ 密钥妥善保管，不要硬编码
- ❌ 不要使用 MD5 加密敏感数据

### 2.6 PlaceholderUtils - 占位符替换

```java
// 定义模板
String template = "Hello ${name}, your age is ${age}";

// 准备数据
Map<String, String> params = new HashMap<>();
params.put("name", "Alice");
params.put("age", "25");

// 替换占位符
String result = PlaceholderUtils.replacePlaceholders(template, params);
// 结果: "Hello Alice, your age is 25"
```

**使用场景**:
- 配置文件模板替换
- SQL 语句参数化
- 消息模板渲染

**最佳实践**:
- ✅ 使用 ${} 作为占位符格式
- ✅ 确保所有占位符都有对应的值
- ✅ 对用户输入进行验证
- ❌ 避免嵌套占位符

### 2.7 HostUtils - 主机工具

```java
// 获取本机 IP
String ip = HostUtils.getLocalIP();

// 获取主机名
String hostname = HostUtils.getHostname();

// 检查主机是否可达
boolean reachable = HostUtils.isReachable("192.168.1.100", 22, 3000);

// 解析主机名到 IP
String ip = HostUtils.resolveHostname("www.example.com");
```

**使用场景**: 网络配置、主机管理、连接检查

---

## 📦 三、命令类速查

### 3.1 基础命令类

所有命令类都继承自 `BaseCommand`，实现序列化接口：

```java
public abstract class BaseCommand implements Serializable {
    private static final long serialVersionUID = 1L;
}
```

### 3.2 常用命令类型

#### 服务操作命令

```java
// 安装服务角色
InstallServiceRoleCommand installCmd = new InstallServiceRoleCommand();
installCmd.setServiceName("HDFS");
installCmd.setServiceRoleName("NameNode");
installCmd.setHostname("node1");

// 服务角色操作（启动/停止/重启）
ServiceRoleOperateCommand operateCmd = new ServiceRoleOperateCommand();
operateCmd.setCommandType(CommandType.START_SERVICE);
operateCmd.setServiceRoleName("NameNode");

// 检查服务状态
ServiceRoleCheckCommand checkCmd = new ServiceRoleCheckCommand();
checkCmd.setServiceRoleName("NameNode");
```

#### 配置管理命令

```java
// 生成服务配置
GenerateServiceConfigCommand configCmd = new GenerateServiceConfigCommand();
configCmd.setServiceName("HDFS");
configCmd.setConfigMap(configMap);

// 生成 Prometheus 配置
GeneratePrometheusConfigCommand promCmd = new GeneratePrometheusConfigCommand();
promCmd.setTargets(targets);
```

#### 主机管理命令

```java
// 主机检查
HostCheckCommand hostCheck = new HostCheckCommand();

// Ping 测试
PingCommand ping = new PingCommand();
```

#### 远程操作命令

```java
// 创建 Unix 用户
CreateUnixUserCommand createUser = new CreateUnixUserCommand();
createUser.setUsername("hadoop");
createUser.setGroup("hadoop");

// 创建 Unix 组
CreateUnixGroupCommand createGroup = new CreateUnixGroupCommand();
createGroup.setGroupname("hadoop");
```

**最佳实践**:
- ✅ 所有命令类必须可序列化
- ✅ 使用有意义的命令名称
- ✅ 设置合理的超时时间
- ✅ 命令执行后检查结果
- ❌ 避免命令参数过于复杂

---

## 📊 四、枚举类型速查

### 4.1 CommandType - 命令类型

```java
CommandType.INSTALL_SERVICE          // 安装服务
CommandType.START_SERVICE           // 启动服务
CommandType.STOP_SERVICE            // 停止服务
CommandType.RESTART_SERVICE         // 重启服务
CommandType.CHECK_SERVICE           // 检查服务
CommandType.GENERATE_CONFIG         // 生成配置
```

**使用场景**: 命令分发、操作日志、权限控制

### 4.2 InstallState - 安装状态

```java
InstallState.INIT                   // 初始化
InstallState.INSTALLING             // 安装中
InstallState.INSTALLED              // 已安装
InstallState.INSTALL_FAILED         // 安装失败
InstallState.UNINSTALLING           // 卸载中
InstallState.UNINSTALLED            // 已卸载
```

**状态流转**:
```
INIT → INSTALLING → INSTALLED
         ↓
    INSTALL_FAILED

INSTALLED → UNINSTALLING → UNINSTALLED
```

### 4.3 ServiceExecuteState - 服务执行状态

```java
ServiceExecuteState.RUNNING         // 运行中
ServiceExecuteState.STOP            // 已停止
ServiceExecuteState.FAIL            // 执行失败
ServiceExecuteState.WAITING         // 等待中
ServiceExecuteState.SUCCESS         // 执行成功
```

### 4.4 OperateType - 操作类型

```java
OperateType.START                   // 启动
OperateType.STOP                    // 停止
OperateType.RESTART                 // 重启
OperateType.UPDATE                  // 更新
OperateType.DELETE                  // 删除
```

**最佳实践**:
- ✅ 使用枚举代替字符串常量
- ✅ switch 语句处理所有枚举值
- ✅ 为枚举添加描述信息
- ❌ 避免使用 ordinal() 判断

---

## 🔄 五、DAG模型速查

### 5.1 DAGGraph - DAG图操作

```java
// 创建 DAG 图
DAGGraph<String, ServiceNode, ServiceNodeEdge> dag = new DAGGraph<>();

// 添加节点
ServiceNode node1 = new ServiceNode("HDFS", "NameNode");
ServiceNode node2 = new ServiceNode("HDFS", "DataNode");
dag.addNode("node1", node1);
dag.addNode("node2", node2);

// 添加边（依赖关系）
dag.addEdge("node1", "node2");  // node2 依赖 node1

// 拓扑排序（获取执行顺序）
List<String> sortedNodes = dag.topologicalSort();

// 检查是否有环
if (dag.hasCycle()) {
    throw new Exception("存在循环依赖");
}

// 获取所有依赖
List<String> dependencies = dag.getSubsequentNodes("node1");

// 获取所有被依赖
List<String> dependents = dag.getPreviousNodes("node2");
```

**使用场景**:
- 服务依赖管理
- 安装顺序计算
- 启停顺序控制
- 任务调度

**最佳实践**:
- ✅ 添加边前先添加节点
- ✅ 执行前检查是否有环
- ✅ 使用拓扑排序获取执行顺序
- ✅ 处理异常情况（如环检测失败）
- ❌ 避免动态修改正在执行的 DAG

### 5.2 ServiceNode - 服务节点

```java
ServiceNode node = new ServiceNode();
node.setName("HDFS");
node.setLabel("NameNode");
node.setServiceRoleName("NameNode");
```

### 5.3 ServiceNodeEdge - 节点边

```java
ServiceNodeEdge edge = new ServiceNodeEdge();
edge.setFromNode("node1");
edge.setToNode("node2");
```

---

## 🔐 六、缓存使用速查

### 6.1 CacheUtils - 缓存操作

```java
// 获取缓存实例
CacheUtils cache = CacheUtils.getInstance();

// 放入缓存
cache.put("key", value);
cache.put("cluster:1", clusterInfo);

// 从缓存获取
Object value = cache.get("key");
ClusterInfo cluster = (ClusterInfo) cache.get("cluster:1");

// 删除缓存
cache.remove("key");

// 清空缓存
cache.clear();

// 检查缓存是否存在
if (cache.containsKey("key")) {
    // ...
}

// 获取缓存大小
int size = cache.size();
```

**缓存特性**:
- LRU（最近最少使用）淘汰策略
- 线程安全（synchronized）
- 单例模式
- 基于 LinkedHashMap 实现

**最佳实践**:
- ✅ 缓存热点数据，减少数据库访问
- ✅ 设置合理的缓存大小
- ✅ 及时清理过期数据
- ✅ 注意缓存一致性
- ❌ 不要缓存大对象
- ❌ 不要缓存频繁变化的数据

**使用场景**:
- 配置信息缓存
- 集群信息缓存
- 用户会话缓存
- 服务状态缓存

---

## ⚙️ 七、生命周期管理速查

### 7.1 ServerLifeCycleManager - 生命周期管理器

```java
// 获取实例
ServerLifeCycleManager manager = ServerLifeCycleManager.getInstance();

// 启动服务器
manager.start();

// 停止服务器
manager.stop();

// 获取当前状态
ServerStatus status = manager.getServerStatus();

// 检查状态
if (manager.isRunning()) {
    // 服务器运行中
}

if (manager.isStopped()) {
    // 服务器已停止
}
```

### 7.2 ServerStatus - 服务器状态

```java
ServerStatus.STARTING      // 启动中
ServerStatus.RUNNING       // 运行中
ServerStatus.WAITING       // 等待中
ServerStatus.STOPPING      // 停止中
ServerStatus.STOPPED       // 已停止
```

**状态流转**:
```
STOPPED → STARTING → RUNNING → STOPPING → STOPPED
            ↓                      ↑
         WAITING ----------------+
```

**最佳实践**:
- ✅ 使用优雅启动和停机
- ✅ 在停止前完成正在进行的任务
- ✅ 释放所有资源
- ✅ 记录状态变更日志
- ❌ 避免强制终止

---

## 🎯 八、常见使用场景

### 8.1 场景1：执行 Shell 命令并处理结果

```java
public Result executeCommand(String command) {
    try {
        ExecResult result = ShellUtils.execWithStatus(
            "/opt/datasophon",
            null,
            30000L,
            command
        );
        
        if (result.getExitCode() == 0) {
            return Result.success("命令执行成功", result.getOutput());
        } else {
            return Result.failed("命令执行失败: " + result.getOutput());
        }
    } catch (Exception e) {
        return Result.failed("命令执行异常: " + e.getMessage());
    }
}
```

### 8.2 场景2：操作配置文件

```java
public Result updateConfig(String serviceName, Map<String, String> configs) {
    try {
        // 1. 读取配置文件
        String configPath = Constants.SERVICE_DIR + "/" + serviceName + "/conf/config.xml";
        String content = FileUtils.readFile(configPath);
        
        // 2. 替换占位符
        String newContent = PlaceholderUtils.replacePlaceholders(content, configs);
        
        // 3. 写入配置文件
        FileUtils.writeFile(newContent, configPath, false);
        
        // 4. 重启服务
        ServiceRoleOperateCommand cmd = new ServiceRoleOperateCommand();
        cmd.setCommandType(CommandType.RESTART_SERVICE);
        cmd.setServiceName(serviceName);
        
        return Result.success("配置更新成功");
    } catch (IOException e) {
        return Result.failed("配置更新失败: " + e.getMessage());
    }
}
```

### 8.3 场景3：管理服务依赖和安装顺序

```java
public List<String> calculateInstallOrder(List<ServiceInfo> services) {
    // 1. 创建 DAG 图
    DAGGraph<String, ServiceNode, ServiceNodeEdge> dag = new DAGGraph<>();
    
    // 2. 添加节点
    for (ServiceInfo service : services) {
        ServiceNode node = new ServiceNode(
            service.getName(),
            service.getLabel()
        );
        dag.addNode(service.getName(), node);
    }
    
    // 3. 添加依赖关系
    for (ServiceInfo service : services) {
        List<String> dependencies = service.getDependencies();
        if (CollectionUtils.isNotEmpty(dependencies)) {
            for (String dep : dependencies) {
                dag.addEdge(dep, service.getName());
            }
        }
    }
    
    // 4. 检查循环依赖
    if (dag.hasCycle()) {
        throw new RuntimeException("服务存在循环依赖");
    }
    
    // 5. 拓扑排序获取安装顺序
    return dag.topologicalSort();
}
```

### 8.4 场景4：缓存集群信息

```java
public ClusterInfo getClusterInfo(Integer clusterId) {
    // 1. 尝试从缓存获取
    String cacheKey = "cluster:" + clusterId;
    CacheUtils cache = CacheUtils.getInstance();
    
    ClusterInfo clusterInfo = (ClusterInfo) cache.get(cacheKey);
    if (clusterInfo != null) {
        return clusterInfo;
    }
    
    // 2. 缓存未命中，从数据库查询
    clusterInfo = clusterMapper.selectById(clusterId);
    
    // 3. 放入缓存
    if (clusterInfo != null) {
        cache.put(cacheKey, clusterInfo);
    }
    
    return clusterInfo;
}

public void updateClusterInfo(ClusterInfo clusterInfo) {
    // 1. 更新数据库
    clusterMapper.updateById(clusterInfo);
    
    // 2. 清除缓存
    String cacheKey = "cluster:" + clusterInfo.getId();
    CacheUtils.getInstance().remove(cacheKey);
}
```

### 8.5 场景5：服务生命周期管理

```java
public class WorkerServer {
    private ServerLifeCycleManager lifeCycleManager;
    
    public void start() {
        lifeCycleManager = ServerLifeCycleManager.getInstance();
        
        try {
            // 1. 启动服务器
            lifeCycleManager.start();
            
            // 2. 初始化资源
            initResources();
            
            // 3. 启动监听
            startListening();
            
            logger.info("Worker 服务器启动成功");
        } catch (ServerLifeCycleException e) {
            logger.error("Worker 服务器启动失败", e);
            throw e;
        }
    }
    
    public void stop() {
        try {
            // 1. 停止接收新任务
            stopListening();
            
            // 2. 等待当前任务完成
            waitForTasksComplete();
            
            // 3. 释放资源
            releaseResources();
            
            // 4. 停止服务器
            lifeCycleManager.stop();
            
            logger.info("Worker 服务器停止成功");
        } catch (ServerLifeCycleException e) {
            logger.error("Worker 服务器停止失败", e);
        }
    }
}
```

---

## 💡 九、最佳实践总结

### 9.1 代码规范

1. **异常处理**
   ```java
   // ✅ 好的做法
   try {
       // 操作
   } catch (SpecificException e) {
       logger.error("具体错误描述", e);
       return Result.failed(e.getMessage());
   }
   
   // ❌ 避免
   try {
       // 操作
   } catch (Exception e) {
       // 吞掉异常
   }
   ```

2. **资源管理**
   ```java
   // ✅ 使用 try-with-resources
   try (InputStream is = new FileInputStream(file);
        OutputStream os = new FileOutputStream(dest)) {
       // 操作
   }
   
   // ❌ 避免手动关闭
   InputStream is = new FileInputStream(file);
   // ... 如果出现异常，流可能没有关闭
   is.close();
   ```

3. **空值检查**
   ```java
   // ✅ 使用工具类
   if (CollectionUtils.isNotEmpty(list)) {
       // 操作
   }
   
   // ❌ 避免
   if (list != null && list.size() > 0) {
       // 操作
   }
   ```

### 9.2 性能优化

1. **缓存使用**
   - 缓存热点数据
   - 及时清理过期缓存
   - 注意缓存一致性

2. **批量操作**
   - 批量插入数据库
   - 批量文件操作
   - 减少网络往返

3. **异步处理**
   - 长时间任务异步执行
   - 使用线程池管理线程
   - 避免阻塞主线程

### 9.3 安全性

1. **输入验证**
   ```java
   // ✅ 验证输入
   if (StringUtils.isEmpty(username) || username.length() > 50) {
       return Result.failed("用户名无效");
   }
   ```

2. **密码加密**
   ```java
   // ✅ 加密存储
   String encryptedPassword = EncryptionUtils.sha256(password);
   ```

3. **权限检查**
   ```java
   // ✅ 执行前检查权限
   if (!hasPermission(user, operation)) {
       return Result.failed("权限不足");
   }
   ```

### 9.4 可维护性

1. **日志记录**
   ```java
   // ✅ 记录关键操作
   logger.info("开始安装服务: {}", serviceName);
   logger.error("服务安装失败: {}", serviceName, exception);
   ```

2. **代码注释**
   ```java
   // ✅ 复杂逻辑添加注释
   /**
    * 计算服务安装顺序
    * 使用拓扑排序算法，基于服务依赖关系
    */
   ```

3. **单元测试**
   ```java
   @Test
   public void testCalculateInstallOrder() {
       // 测试代码
   }
   ```

---

## 🔗 十、相关文档链接

### 详细文档

- [01-Common模块概述](./01-Common模块概述.md) - 模块整体介绍
- [02-Constants常量定义分析](./02-Constants常量定义分析.md) - 97个常量详解
- [03-缓存工具类分析](./03-缓存工具类分析.md) - LRU缓存实现
- [04-工具类详解](./04-工具类详解.md) - 13个工具类完整分析
- [05-枚举类型详解](./05-枚举类型详解.md) - 9个枚举类型详解
- [06-Command命令封装详解](./06-Command命令封装详解.md) - 38个命令类分析
- [07-Model数据模型详解](./07-Model数据模型详解.md) - 31个模型类和DAG图
- [08-Lifecycle与Exception详解](./08-Lifecycle与Exception详解.md) - 生命周期管理
- [09-Common模块文件覆盖索引](./09-Common模块文件覆盖索引.md) - 98个文件索引

### 其他模块

- [../overview/03-整体架构设计](../overview/03-整体架构设计.md) - 系统架构
- [../api/02-Controller层分析](../api/02-Controller层分析.md) - API接口
- [../service/00-Service模块总览](../service/00-Service模块总览.md) - 业务逻辑
- [../worker/01-Worker模块概述](../worker/01-Worker模块概述.md) - Worker节点

---

## 📞 反馈与支持

### 常见问题

**Q: 如何快速查找某个类的用法？**
A: 使用 Ctrl+F 搜索类名，或查看对应的详细文档。

**Q: 代码示例在哪里？**
A: 每个详细文档都包含完整的代码示例和使用场景。

**Q: 如何贡献文档？**
A: 发现错误或有改进建议，请提交 Issue 或 Pull Request。

### 版本信息

- **手册版本**: v1.0
- **创建日期**: 2025-11-15
- **适用版本**: DataSophon 1.x
- **维护团队**: DataSophon 源码分析团队

---

**提示**: 本手册是快速参考指南，详细的设计原理和实现细节请参考对应的详细分析文档。
