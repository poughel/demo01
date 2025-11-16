# ClusterHostController 主机管理控制器详解

## 一、控制器概述

### 1.1 基本信息

**文件位置**: `datasophon-api/src/main/java/com/datasophon/api/controller/ClusterHostController.java`

**职责定位**:
- 管理集群中的物理主机节点
- 提供主机的 CRUD 操作
- 主机角色分配查询
- 机架感知管理
- 主机状态监控

**核心概念**:
- **主机节点**: 集群中的物理或虚拟机器
- **Worker 节点**: 安装了 Worker 程序的主机
- **机架感知**: 数据中心的机架拓扑结构
- **主机状态**: 主机的运行状态和健康度

### 1.2 类结构

```java
@Slf4j
@RestController
@RequestMapping("api/cluster/host")
public class ClusterHostController {
    @Autowired
    private ClusterHostService clusterHostService;
}
```

**关键特征**:
- **请求前缀**: `/api/cluster/host`
- **日志支持**: 使用 Lombok 的 @Slf4j
- **依赖服务**: ClusterHostService
- **实体类**: ClusterHostDO

## 二、API 接口详解

### 2.1 查询所有主机（简单列表）

#### 接口定义
```java
@RequestMapping("/all")
public Result all(Integer clusterId) {
    List<ClusterHostDO> list = clusterHostService.list(
        new QueryWrapper<ClusterHostDO>()
            .eq(Constants.CLUSTER_ID, clusterId)
            .eq(Constants.MANAGED, 1)
            .orderByAsc(Constants.HOSTNAME)
    );
    return Result.success(list);
}
```

**接口信息**:
- **URL**: `GET /api/cluster/host/all`
- **请求参数**: 
  - `clusterId`: 集群ID
- **返回值**: Result 包装的主机列表
- **权限要求**: 登录用户

**功能描述**:
- 查询集群下所有被管理的主机
- 按主机名升序排序
- 返回简单的主机信息
- 不分页，一次性返回所有数据

**使用场景**:
1. 主机选择下拉框
2. 服务角色分配时选择主机
3. 拓扑图展示所有节点

**查询条件**:
- `CLUSTER_ID = ?`: 指定集群
- `MANAGED = 1`: 仅查询被管理的主机
- `ORDER BY HOSTNAME ASC`: 按主机名排序

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "data": [
    {
      "id": 1,
      "hostname": "node01",
      "ip": "192.168.1.101",
      "cpuArchitecture": "x86_64",
      "hostState": 1,
      "managed": 1
    },
    {
      "id": 2,
      "hostname": "node02",
      "ip": "192.168.1.102",
      "cpuArchitecture": "x86_64",
      "hostState": 1,
      "managed": 1
    }
  ]
}
```

### 2.2 分页查询主机列表

#### 接口定义
```java
@RequestMapping("/list")
public Result list(Integer clusterId, String hostname, String ip, 
                   String cpuArchitecture, Integer hostState,
                   String orderField, String orderType, 
                   Integer page, Integer pageSize) {
    return clusterHostService.listByPage(
        clusterId, hostname, ip, cpuArchitecture, hostState, 
        orderField, orderType, page, pageSize
    );
}
```

**接口信息**:
- **URL**: `GET /api/cluster/host/list`
- **请求参数**: 
  - `clusterId`: 集群ID（必填）
  - `hostname`: 主机名（模糊查询）
  - `ip`: IP地址（模糊查询）
  - `cpuArchitecture`: CPU架构（x86_64, aarch64）
  - `hostState`: 主机状态
  - `orderField`: 排序字段
  - `orderType`: 排序方式（asc, desc）
  - `page`: 页码
  - `pageSize`: 每页大小
- **返回值**: Result 包装的分页数据
- **权限要求**: 登录用户

**功能描述**:
- 支持多条件组合查询
- 支持分页
- 支持自定义排序
- 返回详细的主机信息

**使用场景**:
1. 主机管理列表页面
2. 主机搜索和筛选
3. 主机状态监控列表

**查询条件组合**:
```sql
SELECT * FROM cluster_host
WHERE cluster_id = ?
  AND hostname LIKE '%?%'  -- 可选
  AND ip LIKE '%?%'        -- 可选
  AND cpu_architecture = ? -- 可选
  AND host_state = ?       -- 可选
ORDER BY ? ASC/DESC
LIMIT ?, ?
```

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "total": 100,
    "pageSize": 10,
    "currentPage": 1,
    "list": [
      {
        "id": 1,
        "hostname": "node01",
        "ip": "192.168.1.101",
        "rack": "/default-rack",
        "cpuArchitecture": "x86_64",
        "coreNum": 32,
        "totalMem": 128,
        "totalDisk": 1024,
        "usedMem": 64,
        "usedDisk": 512,
        "averageLoad": "2.5",
        "hostState": 1,
        "checkTime": "2025-01-15 10:00:00",
        "createTime": "2025-01-01 10:00:00"
      }
    ]
  }
}
```

**排序字段支持**:
- `hostname`: 主机名
- `ip`: IP地址
- `coreNum`: CPU核数
- `totalMem`: 总内存
- `averageLoad`: 平均负载
- `hostState`: 主机状态
- `createTime`: 创建时间

### 2.3 查询主机上的角色列表

#### 接口定义
```java
@RequestMapping("/getRoleListByHostname")
public Result getRoleListByHostname(Integer clusterId, String hostname) {
    return clusterHostService.getRoleListByHostname(clusterId, hostname);
}
```

**接口信息**:
- **URL**: `GET /api/cluster/host/getRoleListByHostname`
- **请求参数**: 
  - `clusterId`: 集群ID
  - `hostname`: 主机名
- **返回值**: Result 包装的角色列表
- **权限要求**: 登录用户

**功能描述**:
- 查询指定主机上部署的所有服务角色
- 显示角色名称、状态、进程信息
- 用于主机详情页面展示

**使用场景**:
1. 主机详情页面查看已部署角色
2. 主机资源使用分析
3. 角色迁移前查看当前分布

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "data": [
    {
      "serviceName": "HDFS",
      "serviceRoleName": "NameNode",
      "roleState": 1,
      "processId": 12345,
      "jmxPort": 50070,
      "webPort": 9870,
      "memoryUsage": "8GB",
      "cpuUsage": "15%"
    },
    {
      "serviceName": "YARN",
      "serviceRoleName": "ResourceManager",
      "roleState": 1,
      "processId": 12346,
      "jmxPort": 50088,
      "webPort": 8088,
      "memoryUsage": "4GB",
      "cpuUsage": "10%"
    }
  ]
}
```

**角色状态枚举**:
```
0 - 未安装 (UNINSTALLED)
1 - 运行中 (RUNNING)
2 - 已停止 (STOPPED)
3 - 启动中 (STARTING)
4 - 停止中 (STOPPING)
5 - 异常 (ERROR)
```

### 2.4 获取机架列表

#### 接口定义
```java
@RequestMapping("/getRack")
public Result getRack(Integer clusterId) {
    return clusterHostService.getRack(clusterId);
}
```

**接口信息**:
- **URL**: `GET /api/cluster/host/getRack`
- **请求参数**: 
  - `clusterId`: 集群ID
- **返回值**: Result 包装的机架列表
- **权限要求**: 登录用户

**功能描述**:
- 查询集群中的所有机架
- 返回机架名称和主机数量
- 支持机架感知配置

**使用场景**:
1. 配置机架拓扑
2. 主机分配机架
3. 机架感知调度配置

**机架感知概念**:
- **机架感知**: Hadoop 的数据本地性优化策略
- **副本放置**: 不同副本放置在不同机架
- **网络拓扑**: 减少跨机架网络传输

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "data": [
    {
      "rack": "/default-rack",
      "hostCount": 50,
      "isDefault": true
    },
    {
      "rack": "/rack1",
      "hostCount": 30,
      "isDefault": false
    },
    {
      "rack": "/rack2",
      "hostCount": 20,
      "isDefault": false
    }
  ]
}
```

**机架命名规范**:
```
/default-rack        # 默认机架
/rack1              # 机架1
/rack2              # 机架2
/dc1/rack1          # 数据中心1的机架1
/dc2/rack1          # 数据中心2的机架1
```

### 2.5 分配机架

#### 接口定义
```java
@RequestMapping("/assignRack")
public Result assignRack(Integer clusterId, String rack, String hostIds) {
    return clusterHostService.assignRack(clusterId, rack, hostIds);
}
```

**接口信息**:
- **URL**: `POST /api/cluster/host/assignRack`
- **请求参数**: 
  - `clusterId`: 集群ID
  - `rack`: 机架名称
  - `hostIds`: 主机ID列表（逗号分隔）
- **返回值**: Result 表示操作结果
- **权限要求**: 登录用户（应该添加 @UserPermission）

**功能描述**:
- 批量将主机分配到指定机架
- 更新机架拓扑配置
- 触发 HDFS 重新平衡

**使用场景**:
1. 新主机加入集群时分配机架
2. 调整机架拓扑结构
3. 优化数据分布

**请求示例**:
```
POST /api/cluster/host/assignRack
Content-Type: application/x-www-form-urlencoded

clusterId=1&rack=/rack1&hostIds=1,2,3,4,5
```

**业务逻辑**:
1. 验证机架名称格式
2. 验证主机ID有效性
3. 批量更新主机的机架属性
4. 更新 HDFS 的 `topology.script`
5. 触发数据重新平衡（可选）

**副作用**:
- HDFS 需要重新计算副本放置策略
- 可能触发数据块迁移
- 影响数据本地性

### 2.6 获取主机详情

#### 接口定义
```java
@RequestMapping("/info/{id}")
public Result info(@PathVariable("id") Integer id) {
    ClusterHostDO clusterHost = clusterHostService.getById(id);
    return Result.success().put(Constants.DATA, clusterHost);
}
```

**接口信息**:
- **URL**: `GET /api/cluster/host/info/{id}`
- **请求参数**: 
  - `id`: 主机ID（路径参数）
- **返回值**: Result 包装的主机详情
- **权限要求**: 登录用户

**功能描述**:
- 查询主机的完整信息
- 包含硬件、状态、统计数据
- 用于主机详情页面

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "id": 1,
    "clusterId": 1,
    "hostname": "node01",
    "ip": "192.168.1.101",
    "rack": "/rack1",
    "cpuArchitecture": "x86_64",
    "coreNum": 32,
    "totalMem": 128,
    "totalDisk": 1024,
    "usedMem": 64,
    "usedDisk": 512,
    "averageLoad": "2.5",
    "nodeLabel": "compute",
    "hostState": 1,
    "managed": 1,
    "checkTime": "2025-01-15 10:00:00",
    "createTime": "2025-01-01 10:00:00",
    "updateTime": "2025-01-10 15:30:00"
  }
}
```

### 2.7 保存主机

#### 接口定义
```java
@RequestMapping("/save")
public Result save(@RequestBody ClusterHostDO clusterHost) {
    clusterHostService.save(clusterHost);
    return Result.success();
}
```

**接口信息**:
- **URL**: `POST /api/cluster/host/save`
- **请求参数**: 
  - `clusterHost`: 主机对象（JSON Body）
- **返回值**: Result 表示操作结果
- **权限要求**: 登录用户（应该添加 @UserPermission）

**功能描述**:
- 添加新主机到集群
- 通常由主机安装流程调用
- 初始化主机基础信息

**使用场景**:
1. 主机安装 Worker 后自动注册
2. 手动添加主机
3. 批量导入主机

**请求数据示例**:
```json
{
  "clusterId": 1,
  "hostname": "node10",
  "ip": "192.168.1.110",
  "sshPort": 22,
  "sshUser": "root",
  "rack": "/default-rack"
}
```

### 2.8 更新主机

#### 接口定义
```java
@RequestMapping("/update")
public Result update(@RequestBody ClusterHostDO clusterHost) {
    clusterHostService.updateById(clusterHost);
    return Result.success();
}
```

**接口信息**:
- **URL**: `POST /api/cluster/host/update`
- **请求参数**: 
  - `clusterHost`: 主机对象（JSON Body）
- **返回值**: Result 表示操作结果
- **权限要求**: 登录用户（应该添加 @UserPermission）

**功能描述**:
- 修改主机信息
- 更新标签、机架等属性
- 调整主机配置

**可修改字段**:
- 主机标签
- 机架位置
- SSH配置
- 管理状态

**不可修改字段**:
- 主机ID
- 主机名
- IP地址（需谨慎）

### 2.9 删除主机

#### 接口定义
```java
@RequestMapping("/delete")
public Result delete(String hostIds) {
    if (StringUtils.isBlank(hostIds)) {
        return Result.error("请选择移除的主机!");
    }
    try {
        return clusterHostService.deleteHosts(hostIds);
    } catch (Exception e) {
        log.warn("移除主机异常.", e);
        return Result.error("移除主机异常, Cause: " + e.getMessage());
    }
}
```

**接口信息**:
- **URL**: `POST /api/cluster/host/delete`
- **请求参数**: 
  - `hostIds`: 主机ID列表（逗号分隔）
- **返回值**: Result 表示操作结果
- **权限要求**: 登录用户（应该添加 @UserPermission）

**功能描述**:
- 批量删除主机
- 移除主机前检查约束
- 清理相关数据

**使用场景**:
1. 主机下线
2. 集群缩容
3. 清理失效主机

**删除前检查**:
1. 主机上是否有运行中的服务角色
2. 是否有正在执行的任务
3. 数据副本是否充足

**错误处理**:
- 参数验证: hostIds 不能为空
- 异常捕获: try-catch 包裹
- 日志记录: log.warn 记录异常
- 友好提示: 返回具体错误信息

**级联操作**:
- 停止主机上的所有服务角色
- 删除角色实例记录
- 清理监控数据
- 更新机架拓扑

## 三、主机数据模型

### 3.1 ClusterHostDO

```java
public class ClusterHostDO {
    private Integer id;                // 主键ID
    private Integer clusterId;         // 集群ID
    private String hostname;           // 主机名
    private String ip;                 // IP地址
    private String rack;               // 机架位置
    private String cpuArchitecture;    // CPU架构
    private Integer coreNum;           // CPU核数
    private Integer totalMem;          // 总内存(GB)
    private Integer totalDisk;         // 总磁盘(GB)
    private Integer usedMem;           // 已用内存(GB)
    private Integer usedDisk;          // 已用磁盘(GB)
    private String averageLoad;        // 平均负载
    private String nodeLabel;          // 节点标签
    private Integer hostState;         // 主机状态
    private Integer managed;           // 是否被管理
    private Integer sshPort;           // SSH端口
    private String sshUser;            // SSH用户
    private Date checkTime;            // 检查时间
    private Date createTime;           // 创建时间
    private Date updateTime;           // 更新时间
}
```

### 3.2 主机状态枚举

```java
public enum HostState {
    UNKNOWN(0, "未知"),
    RUNNING(1, "运行中"),
    STOPPED(2, "已停止"),
    INSTALLING(3, "安装中"),
    ERROR(4, "异常"),
    LOST(5, "失联");
}
```

### 3.3 CPU架构支持

```java
public enum CpuArchitecture {
    X86_64("x86_64", "x86架构"),
    AARCH64("aarch64", "ARM架构"),
    LOONGARCH64("loongarch64", "龙芯架构");
}
```

## 四、设计模式与架构

### 4.1 机架感知架构

```
DataCenter
  ├─ Rack1
  │   ├─ node01 (NameNode)
  │   ├─ node02 (DataNode)
  │   └─ node03 (DataNode)
  ├─ Rack2
  │   ├─ node04 (DataNode)
  │   ├─ node05 (DataNode)
  │   └─ node06 (DataNode)
  └─ Rack3
      ├─ node07 (DataNode)
      ├─ node08 (DataNode)
      └─ node09 (DataNode)
```

**副本放置策略**:
- 第1副本: 本地节点
- 第2副本: 不同机架的节点
- 第3副本: 第2副本同机架的不同节点

### 4.2 主机状态监控

```
Worker 节点 → 心跳上报 → Master 节点 → 更新状态
     ↓
  采集指标
     ↓
 CPU、内存、磁盘
```

**监控指标**:
- CPU使用率
- 内存使用率
- 磁盘使用率
- 网络流量
- 平均负载
- 服务角色状态

### 4.3 主机生命周期

```
添加 → 安装Worker → 运行中 → 下线 → 删除
  ↓         ↓          ↓
 失败 ← ─ ─异常 ← ─ ─失联
```

## 五、性能优化

### 5.1 查询优化

**索引设计**:
```sql
CREATE INDEX idx_cluster_id ON cluster_host(cluster_id);
CREATE INDEX idx_hostname ON cluster_host(hostname);
CREATE INDEX idx_ip ON cluster_host(ip);
CREATE INDEX idx_host_state ON cluster_host(host_state);
```

### 5.2 分页查询

```java
// MyBatis-Plus 自动分页
Page<ClusterHostDO> page = new Page<>(pageNum, pageSize);
clusterHostService.page(page, queryWrapper);
```

### 5.3 缓存策略

```java
@Cacheable(value = "host:info", key = "#id")
public ClusterHostDO getById(Integer id) {
    // ...
}

@CacheEvict(value = "host:*", allEntries = true)
public void update(ClusterHostDO host) {
    // ...
}
```

## 六、安全性考虑

### 6.1 权限控制

```java
// 建议添加权限注解
@RequestMapping("/delete")
@UserPermission
public Result delete(String hostIds) {
    // ...
}
```

### 6.2 SSH安全

- SSH密钥认证优于密码
- 限制SSH访问IP
- 使用非标准端口
- 定期更新密钥

### 6.3 输入验证

```java
// 主机名格式验证
@Pattern(regexp = "^[a-z0-9-]+$")
private String hostname;

// IP地址格式验证
@Pattern(regexp = "^\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}$")
private String ip;
```

## 七、使用示例

### 7.1 前端调用示例

**分页查询主机**:
```javascript
axios.get('/api/cluster/host/list', {
  params: {
    clusterId: 1,
    page: 1,
    pageSize: 10,
    hostname: 'node',
    orderField: 'createTime',
    orderType: 'desc'
  }
})
.then(response => {
  const hosts = response.data.data.list;
  console.log(hosts);
});
```

**分配机架**:
```javascript
axios.post('/api/cluster/host/assignRack', null, {
  params: {
    clusterId: 1,
    rack: '/rack1',
    hostIds: '1,2,3,4,5'
  }
})
.then(response => {
  console.log('机架分配成功');
});
```

**删除主机**:
```javascript
axios.post('/api/cluster/host/delete', null, {
  params: {
    hostIds: '1,2,3'
  }
})
.then(response => {
  console.log('主机删除成功');
})
.catch(error => {
  console.error('删除失败:', error.response.data.msg);
});
```

## 八、总结

### 8.1 核心功能

ClusterHostController 提供主机管理的完整功能：
1. 主机的增删改查
2. 机架感知管理
3. 主机角色查询
4. 批量操作支持

### 8.2 设计特点

- **机架感知**: 支持数据本地性优化
- **批量操作**: 提高管理效率
- **状态监控**: 实时主机状态
- **灵活查询**: 多条件组合查询

### 8.3 改进建议

1. 添加权限控制注解
2. 完善参数验证
3. 增强错误处理
4. 优化查询性能
5. 添加主机健康检查接口
6. 支持主机标签管理
7. 增加主机资源使用趋势

---

**文档版本**: v1.0  
**最后更新**: 2025-11-15  
**维护者**: DataSophon 源码分析团队
