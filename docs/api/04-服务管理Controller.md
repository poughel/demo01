# ClusterServiceInstanceController 服务管理控制器详解

## 一、控制器概述

### 1.1 基本信息

**文件位置**: `datasophon-api/src/main/java/com/datasophon/api/controller/ClusterServiceInstanceController.java`

**职责定位**:
- 管理集群中的服务实例（如 HDFS、YARN、Kafka 等）
- 提供服务实例的 CRUD 操作
- 服务角色类型查询
- 服务配置版本对比
- 客户端配置文件下载

**核心概念**:
- **服务实例**: 部署在集群中的大数据组件实例
- **服务角色**: 服务包含的各种角色（如 HDFS 的 NameNode、DataNode）
- **角色组**: 同一角色的实例分组管理

### 1.2 类结构

```java
@RestController
@RequestMapping("cluster/service/instance")
public class ClusterServiceInstanceController {
    @Autowired
    private ClusterServiceInstanceService clusterServiceInstanceService;
}
```

**关键特征**:
- **请求前缀**: `/cluster/service/instance`
- **依赖服务**: ClusterServiceInstanceService
- **无权限注解**: 大部分接口登录即可访问

## 二、API 接口详解

### 2.1 查询服务实例列表

#### 接口定义
```java
@RequestMapping("/list")
public Result list(Integer clusterId) {
    return Result.success(clusterServiceInstanceService.listAll(clusterId));
}
```

**接口信息**:
- **URL**: `GET /cluster/service/instance/list`
- **请求参数**: 
  - `clusterId`: 集群ID
- **返回值**: Result 包装的服务实例列表
- **权限要求**: 登录用户

**功能描述**:
- 查询指定集群下的所有服务实例
- 返回服务名称、版本、状态等基本信息
- 用于服务管理页面的列表展示

**使用场景**:
1. 集群服务管理页面加载
2. 服务状态总览
3. 服务选择下拉框

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "data": [
    {
      "id": 1,
      "clusterId": 1,
      "serviceName": "HDFS",
      "serviceState": 1,
      "serviceVersion": "3.3.4",
      "label": "存储服务",
      "createTime": "2025-01-01 10:00:00"
    },
    {
      "id": 2,
      "clusterId": 1,
      "serviceName": "YARN",
      "serviceState": 1,
      "serviceVersion": "3.3.4",
      "label": "资源管理",
      "createTime": "2025-01-01 10:30:00"
    }
  ]
}
```

**服务状态枚举**:
```
0 - 未安装 (UNINSTALLED)
1 - 运行中 (RUNNING)
2 - 已停止 (STOPPED)
3 - 安装中 (INSTALLING)
4 - 异常 (ERROR)
```

### 2.2 获取服务角色类型列表

#### 接口定义
```java
@RequestMapping("/getServiceRoleType")
public Result getServiceRoleType(Integer serviceInstanceId) {
    return clusterServiceInstanceService.getServiceRoleType(serviceInstanceId);
}
```

**接口信息**:
- **URL**: `GET /cluster/service/instance/getServiceRoleType`
- **请求参数**: 
  - `serviceInstanceId`: 服务实例ID
- **返回值**: Result 包装的角色类型列表
- **权限要求**: 登录用户

**功能描述**:
- 查询服务包含的所有角色类型
- 不同服务有不同的角色类型
- 用于服务角色实例的分类展示

**使用场景**:
1. 服务详情页面展示角色分类
2. 添加服务角色实例时选择角色类型
3. 服务拓扑图展示

**角色类型示例**:

**HDFS 服务的角色**:
```json
{
  "code": 0,
  "msg": "success",
  "data": [
    {
      "roleType": "NameNode",
      "roleFullName": "HDFS NameNode",
      "cardinality": "1",
      "roleGroup": "主节点"
    },
    {
      "roleType": "DataNode",
      "roleFullName": "HDFS DataNode",
      "cardinality": "1+",
      "roleGroup": "工作节点"
    },
    {
      "roleType": "JournalNode",
      "roleFullName": "HDFS JournalNode",
      "cardinality": "3+",
      "roleGroup": "主节点"
    }
  ]
}
```

**YARN 服务的角色**:
```json
{
  "data": [
    {
      "roleType": "ResourceManager",
      "roleFullName": "YARN ResourceManager",
      "cardinality": "1-2"
    },
    {
      "roleType": "NodeManager",
      "roleFullName": "YARN NodeManager",
      "cardinality": "1+"
    }
  ]
}
```

**基数说明** (cardinality):
- `1`: 必须且只能有1个实例
- `1-2`: 可以有1个或2个实例（高可用）
- `1+`: 至少1个，可以有多个
- `3+`: 至少3个（如 ZooKeeper、JournalNode）

### 2.3 配置版本对比

#### 接口定义
```java
@RequestMapping("/configVersionCompare")
public Result configVersionCompare(Integer serviceInstanceId, Integer roleGroupId) {
    return clusterServiceInstanceService.configVersionCompare(serviceInstanceId, roleGroupId);
}
```

**接口信息**:
- **URL**: `GET /cluster/service/instance/configVersionCompare`
- **请求参数**: 
  - `serviceInstanceId`: 服务实例ID
  - `roleGroupId`: 角色组ID（可选）
- **返回值**: Result 包装的配置差异对比
- **权限要求**: 登录用户

**功能描述**:
- 对比服务配置的不同版本
- 展示配置项的变更历史
- 支持配置回滚

**使用场景**:
1. 配置变更前预览
2. 配置变更后对比
3. 问题排查时查看配置历史
4. 配置回滚

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "currentVersion": "v3",
    "compareVersion": "v2",
    "diffItems": [
      {
        "configKey": "dfs.replication",
        "currentValue": "3",
        "compareValue": "2",
        "changeType": "MODIFIED",
        "updateTime": "2025-01-10 15:30:00"
      },
      {
        "configKey": "dfs.namenode.handler.count",
        "currentValue": "20",
        "compareValue": null,
        "changeType": "ADDED",
        "updateTime": "2025-01-10 15:30:00"
      }
    ]
  }
}
```

**变更类型**:
- `ADDED`: 新增配置项
- `MODIFIED`: 修改配置值
- `DELETED`: 删除配置项
- `UNCHANGED`: 未变更

### 2.4 获取服务实例详情

#### 接口定义
```java
@RequestMapping("/info/{id}")
public Result info(@PathVariable("id") Integer id) {
    ClusterServiceInstanceEntity clusterServiceInstance = 
        clusterServiceInstanceService.getById(id);
    return Result.success().put("clusterServiceInstance", clusterServiceInstance);
}
```

**接口信息**:
- **URL**: `GET /cluster/service/instance/info/{id}`
- **请求参数**: 
  - `id`: 服务实例ID（路径参数）
- **返回值**: Result 包装的服务实例详情
- **权限要求**: 登录用户

**功能描述**:
- 查询服务实例的完整信息
- 包含配置、依赖、状态等详细数据
- 用于服务详情页面展示

**使用场景**:
1. 服务详情页面
2. 服务编辑前数据回显
3. 服务状态监控

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "clusterServiceInstance": {
    "id": 1,
    "clusterId": 1,
    "serviceName": "HDFS",
    "serviceState": 1,
    "serviceVersion": "3.3.4",
    "label": "存储服务",
    "sortNum": 1,
    "dependencies": "ZOOKEEPER",
    "packageName": "hadoop-3.3.4.tar.gz",
    "alertQuota": 80,
    "createTime": "2025-01-01 10:00:00",
    "updateTime": "2025-01-10 15:30:00"
  }
}
```

### 2.5 下载客户端配置

#### 接口定义
```java
@RequestMapping("/downloadClientConfig")
public Result downloadClientConfig(Integer clusterId, String serviceName) {
    return clusterServiceInstanceService.downloadClientConfig(clusterId, serviceName);
}
```

**接口信息**:
- **URL**: `GET /cluster/service/instance/downloadClientConfig`
- **请求参数**: 
  - `clusterId`: 集群ID
  - `serviceName`: 服务名称
- **返回值**: Result 包装的配置文件下载信息
- **权限要求**: 登录用户

**功能描述**:
- 生成客户端配置文件
- 打包成 tar.gz 或 zip
- 提供下载链接

**使用场景**:
1. 开发人员需要连接集群
2. 应用程序集成大数据组件
3. 第三方工具配置

**支持的服务**:
- **HDFS**: core-site.xml, hdfs-site.xml
- **YARN**: yarn-site.xml, mapred-site.xml
- **Kafka**: producer.properties, consumer.properties
- **HBase**: hbase-site.xml
- **Hive**: hive-site.xml

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "downloadUrl": "/api/download/hdfs-client-config-20250115.tar.gz",
    "fileName": "hdfs-client-config-20250115.tar.gz",
    "fileSize": "12.5KB",
    "expiresAt": "2025-01-16 10:00:00"
  }
}
```

**配置文件内容示例** (HDFS):
```xml
<!-- core-site.xml -->
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://namenode:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/data/hadoop/tmp</value>
    </property>
</configuration>
```

### 2.6 保存服务实例

#### 接口定义
```java
@RequestMapping("/save")
public Result save(@RequestBody ClusterServiceInstanceEntity clusterServiceInstance) {
    clusterServiceInstanceService.save(clusterServiceInstance);
    return Result.success();
}
```

**接口信息**:
- **URL**: `POST /cluster/service/instance/save`
- **请求参数**: 
  - `clusterServiceInstance`: 服务实例对象（JSON Body）
- **返回值**: Result 表示操作结果
- **权限要求**: 登录用户（应该添加 @UserPermission）

**功能描述**:
- 创建新的服务实例
- 初始化服务配置
- 建立服务依赖关系

**使用场景**:
1. 安装新服务
2. 服务初始化

**请求数据示例**:
```json
{
  "clusterId": 1,
  "serviceName": "HDFS",
  "serviceVersion": "3.3.4",
  "label": "存储服务",
  "sortNum": 1
}
```

**业务逻辑**:
1. 验证服务名称唯一性
2. 检查服务依赖是否已安装
3. 初始化服务配置
4. 创建默认角色组
5. 保存到数据库

### 2.7 更新服务实例

#### 接口定义
```java
@RequestMapping("/update")
public Result update(@RequestBody ClusterServiceInstanceEntity clusterServiceInstance) {
    clusterServiceInstanceService.updateById(clusterServiceInstance);
    return Result.success();
}
```

**接口信息**:
- **URL**: `POST /cluster/service/instance/update`
- **请求参数**: 
  - `clusterServiceInstance`: 服务实例对象（JSON Body）
- **返回值**: Result 表示操作结果
- **权限要求**: 登录用户（应该添加 @UserPermission）

**功能描述**:
- 修改服务实例信息
- 更新服务配置
- 调整服务参数

**可修改字段**:
- 服务标签
- 排序号
- 告警阈值
- 配置参数

**不可修改字段**:
- 服务ID
- 服务名称
- 集群ID

### 2.8 删除服务实例

#### 接口定义
```java
@RequestMapping("/delete")
public Result delete(Integer serviceInstanceId) {
    return clusterServiceInstanceService.delServiceInstance(serviceInstanceId);
}
```

**接口信息**:
- **URL**: `POST /cluster/service/instance/delete`
- **请求参数**: 
  - `serviceInstanceId`: 服务实例ID
- **返回值**: Result 表示操作结果
- **权限要求**: 登录用户（应该添加 @UserPermission）

**功能描述**:
- 删除服务实例
- 清理服务相关数据
- 检查依赖关系

**使用场景**:
1. 卸载不需要的服务
2. 重新安装服务
3. 清理测试环境

**删除前检查**:
1. 服务是否正在运行（需先停止）
2. 是否有其他服务依赖
3. 是否有正在执行的任务

**级联删除内容**:
- 所有服务角色实例
- 服务配置数据
- 服务监控数据
- 服务告警规则

**依赖检查示例**:
```
删除 ZOOKEEPER 失败：
- HDFS 依赖 ZOOKEEPER
- KAFKA 依赖 ZOOKEEPER
请先卸载依赖服务
```

## 三、服务实例数据模型

### 3.1 ClusterServiceInstanceEntity

```java
public class ClusterServiceInstanceEntity {
    private Integer id;                // 主键ID
    private Integer clusterId;         // 集群ID
    private String serviceName;        // 服务名称
    private String label;              // 服务标签
    private String serviceVersion;     // 服务版本
    private Integer serviceState;      // 服务状态
    private Integer sortNum;           // 排序号
    private String dependencies;       // 依赖服务
    private String packageName;        // 安装包名称
    private Integer alertQuota;        // 告警配额
    private Date createTime;           // 创建时间
    private Date updateTime;           // 更新时间
}
```

### 3.2 服务与角色关系

```
服务实例 (ClusterServiceInstance)
    ├─ 角色组 (RoleGroup)
    │   ├─ 默认组
    │   └─ 自定义组
    └─ 角色实例 (ServiceRoleInstance)
        ├─ NameNode
        ├─ DataNode
        └─ ...
```

## 四、设计模式与架构

### 4.1 服务依赖管理

**依赖关系图**:
```
HIVE → HDFS, YARN, MySQL
SPARK → HDFS, YARN
FLINK → HDFS, YARN
KAFKA → ZOOKEEPER
HDFS → ZOOKEEPER
YARN → HDFS, ZOOKEEPER
```

**依赖检查**:
```java
// 安装 HIVE 前检查
if (!isServiceInstalled(clusterId, "HDFS")) {
    throw new ServiceException("请先安装 HDFS");
}
if (!isServiceInstalled(clusterId, "YARN")) {
    throw new ServiceException("请先安装 YARN");
}
```

### 4.2 服务状态机

```
未安装 → 安装中 → 运行中 → 已停止
   ↓         ↓        ↓
  异常 ← ─ ─ ─ ─ ─ ─ ─
```

**状态转换规则**:
```java
// 合法的状态转换
UNINSTALLED → INSTALLING → RUNNING
RUNNING → STOPPED → RUNNING
任意状态 → ERROR
```

### 4.3 配置版本管理

**版本演进**:
```
v1 (初始版本)
  ↓ (修改 dfs.replication)
v2
  ↓ (添加 dfs.namenode.handler.count)
v3 (当前版本)
```

**版本对比**:
- 支持任意两个版本对比
- 显示新增、修改、删除的配置项
- 支持配置回滚到历史版本

## 五、性能优化

### 5.1 服务列表缓存

```java
@Cacheable(value = "service:list", key = "#clusterId")
public List<ClusterServiceInstanceEntity> listAll(Integer clusterId) {
    // ...
}
```

### 5.2 配置文件缓存

```java
// 客户端配置文件生成后缓存
@Cacheable(value = "service:client:config", 
           key = "#clusterId + ':' + #serviceName")
public String downloadClientConfig(Integer clusterId, String serviceName) {
    // ...
}
```

## 六、安全性考虑

### 6.1 权限控制建议

```java
// 建议添加权限控制
@RequestMapping("/save")
@UserPermission  // 添加此注解
public Result save(@RequestBody ClusterServiceInstanceEntity clusterServiceInstance) {
    // ...
}
```

### 6.2 参数验证

```java
@Valid @RequestBody ClusterServiceInstanceEntity clusterServiceInstance
```

## 七、使用示例

### 7.1 前端调用示例

**查询服务列表**:
```javascript
axios.get('/cluster/service/instance/list', {
  params: { clusterId: 1 }
})
.then(response => {
  const services = response.data.data;
  console.log(services);
});
```

**下载客户端配置**:
```javascript
axios.get('/cluster/service/instance/downloadClientConfig', {
  params: {
    clusterId: 1,
    serviceName: 'HDFS'
  }
})
.then(response => {
  const downloadUrl = response.data.data.downloadUrl;
  window.open(downloadUrl);
});
```

## 八、总结

### 8.1 核心功能

ClusterServiceInstanceController 是服务管理的核心入口：
1. 服务实例的生命周期管理
2. 服务角色类型查询
3. 配置版本对比
4. 客户端配置下载

### 8.2 设计亮点

- **依赖管理**: 自动检查服务依赖
- **版本控制**: 配置版本追踪和对比
- **配置下载**: 方便客户端集成

### 8.3 改进建议

1. 添加权限控制注解
2. 完善参数验证
3. 增加批量操作接口
4. 优化配置文件缓存
5. 添加服务健康检查接口

---

**文档版本**: v1.0  
**最后更新**: 2025-11-15  
**维护者**: DataSophon 源码分析团队
