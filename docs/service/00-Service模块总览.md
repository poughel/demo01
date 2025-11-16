# DataSophon Service 模块总览

## 一、模块简介

### 1.1 Service 层的作用

Service 层是 DataSophon 的核心业务逻辑层，位于 `datasophon-service` 模块中，负责实现平台的所有业务功能。它承担了以下职责：

- **业务逻辑封装**: 实现集群管理、服务部署、监控告警等核心业务
- **事务管理**: 确保数据操作的原子性和一致性
- **服务编排**: 协调多个服务和组件的交互
- **Actor 集成**: 使用 Akka Actor 实现分布式任务调度
- **数据转换**: 在 Controller 和 Mapper 层之间进行数据转换

### 1.2 模块结构

```
datasophon-service/
├── src/main/java/com/datasophon/api/
│   ├── master/                    # Actor 系统
│   │   ├── ClusterActor.java      # 集群 Actor
│   │   ├── ServiceActor.java      # 服务 Actor
│   │   ├── PrometheusActor.java   # 监控 Actor
│   │   └── handler/               # 命令处理器
│   ├── service/                   # 服务接口
│   │   ├── ClusterInfoService.java          # 集群服务
│   │   ├── ClusterServiceInstanceService.java # 服务实例
│   │   ├── host/ClusterHostService.java     # 主机服务
│   │   ├── AlertGroupService.java           # 告警组服务
│   │   ├── ServiceInstallService.java       # 服务安装
│   │   └── ...
│   └── impl/                      # 服务实现
│       └── ...
└── ...
```

## 二、核心服务分类

### 2.1 集群管理服务

#### ClusterInfoService - 集群信息服务
**职责**: 集群生命周期管理
- ✅ **已完成文档**: [01-集群服务.md](./01-集群服务.md)
- 集群创建、查询、更新、删除
- 集群状态管理（需要配置、运行中、已停止、删除中）
- 全局变量管理和缓存
- 默认资源初始化（YARN 调度器、节点标签、队列、机架）

**核心方法**:
```java
Result saveCluster(ClusterInfoEntity clusterInfo)           // 创建集群
Result getClusterList()                                      // 查询集群列表
Result updateClusterState(Integer clusterId, Integer state)  // 更新状态
void deleteCluster(List<Integer> ids)                       // 删除集群
```

#### ClusterHostService - 主机服务
**职责**: 主机节点管理
- 主机列表查询和分页
- 主机状态监控
- 机架感知（Rack Awareness）
- 节点标签管理
- 主机批量删除和服务清理

**核心方法**:
```java
Result listByPage(...)                               // 分页查询主机
Result getRoleListByHostname(Integer clusterId, String hostname)  // 查询主机上的角色
Result deleteHosts(String hostIds)                   // 批量删除主机
Result assignRack(Integer clusterId, String rack, String hostIds)  // 分配机架
```

#### ClusterRackService - 机架服务
**职责**: 数据中心机架管理
- 机架拓扑结构管理
- 机架容量规划
- 机架与主机映射

### 2.2 服务实例管理

#### ClusterServiceInstanceService - 服务实例服务
**职责**: 大数据组件实例管理
- ✅ **已完成文档**: [02-服务实例服务.md](./02-服务实例服务.md)
- 服务实例生命周期管理
- 实时状态监控和更新
- 配置版本管理和对比
- 监控大盘集成
- 告警信息统计
- 安全的级联删除

**核心方法**:
```java
List<ClusterServiceInstanceEntity> listAll(Integer clusterId)              // 查询所有服务
Result configVersionCompare(Integer serviceInstanceId, Integer roleGroupId) // 配置对比
Result delServiceInstance(Integer serviceInstanceId)                        // 删除服务
```

#### ClusterServiceRoleInstanceService - 服务角色实例服务
**职责**: 服务角色管理（如 NameNode、DataNode）
- 角色实例的创建和删除
- 角色状态管理
- 角色配置管理
- 角色依赖关系处理

**核心方法**:
```java
List<ClusterServiceRoleInstanceEntity> getServiceRoleInstanceListByServiceId(Integer serviceId)
Result startRole(String roleIds)                     // 启动角色
Result stopRole(String roleIds)                      // 停止角色
Result restartRole(String roleIds)                   // 重启角色
```

#### ClusterServiceInstanceRoleGroupService - 角色组服务
**职责**: 角色分组管理
- 角色组创建和管理
- 角色组配置隔离
- 灵活的角色分配策略

### 2.3 服务安装与部署

#### ServiceInstallService - 服务安装服务
**职责**: 服务安装流程控制
- 服务安装前置检查
- 依赖服务检查
- 安装任务调度
- DAG 任务图构建
- 安装进度跟踪

**核心方法**:
```java
Result installService(Integer clusterId, List<ServiceRoleInfo> serviceList)  // 安装服务
Result checkServiceDependency(...)                   // 检查依赖
```

#### InstallService - 通用安装服务
**职责**: 安装流程的底层实现
- 软件包分发
- 配置文件生成
- 服务进程启动
- 安装结果验证

#### ServiceOperateStrategy - 服务操作策略
**职责**: 服务操作的策略模式实现
- 不同服务的启停策略
- 特殊服务的定制化操作
- 操作前后的钩子函数

### 2.4 命令执行服务

#### ClusterServiceCommandService - 服务命令服务
**职责**: 服务级别的命令管理
- 命令创建和调度
- 命令状态跟踪
- 命令历史记录

**核心方法**:
```java
Result generateCommand(ServiceRoleHostCommand command)  // 生成命令
Result getCommandList(Integer clusterId)                // 查询命令列表
Result getCommandInfo(String commandId)                 // 查询命令详情
```

#### ClusterServiceCommandHostService - 主机命令服务
**职责**: 主机级别的命令执行
- 主机命令分发
- 命令执行状态追踪
- 命令执行结果收集

#### ClusterServiceCommandHostCommandService - 主机命令详情服务
**职责**: 单个主机命令的详细管理
- 命令执行日志
- 错误信息记录
- 重试机制

### 2.5 告警管理服务

#### AlertGroupService - 告警组服务
**职责**: 告警接收组管理
- 告警组的增删改查
- 告警接收人管理
- 多通知渠道支持（钉钉、企业微信、邮件、飞书）

**核心方法**:
```java
Result saveAlertGroup(AlertGroupEntity alertGroup)   // 创建告警组
Result listPage(String alertGroupName, Integer page, Integer pageSize)  // 分页查询
Result updateAlertGroup(AlertGroupEntity alertGroup) // 更新告警组
```

#### ClusterAlertRuleService - 告警规则服务
**职责**: 告警规则配置
- 告警规则定义
- 告警阈值设置
- 告警触发条件
- 规则与告警组关联

#### ClusterAlertHistoryService - 告警历史服务
**职责**: 告警记录管理
- 告警历史查询
- 告警趋势分析
- 告警统计报表

#### ClusterAlertQuotaService - 告警指标服务
**职责**: 监控指标管理
- 指标定义和采集
- 指标与规则关联
- 指标数据查询

### 2.6 配置管理服务

#### ClusterServiceInstanceConfigService - 服务配置服务
**职责**: 服务实例配置管理
- 配置项的增删改查
- 配置模板管理
- 配置参数校验

#### ClusterServiceRoleGroupConfigService - 角色组配置服务
**职责**: 角色组配置管理
- 配置版本控制
- 配置继承关系
- 配置历史追踪

#### ClusterVariableService - 集群变量服务
**职责**: 集群级别变量管理
- 全局变量定义
- 变量替换和引用
- 变量作用域控制

### 2.7 监控与大盘服务

#### ClusterServiceDashboardService - 监控大盘服务
**职责**: Grafana 大盘集成
- Dashboard 模板管理
- Dashboard URL 生成
- 监控指标映射

#### ClusterServiceRoleInstanceWebuisService - WebUI 服务
**职责**: 服务 WebUI 管理
- WebUI 地址配置
- WebUI 访问入口
- WebUI 状态检查

### 2.8 框架与服务元数据

#### FrameServiceService - 框架服务服务
**职责**: 大数据框架管理
- 支持的框架版本
- 框架包含的服务
- 框架兼容性

#### FrameServiceRoleService - 框架角色服务
**职责**: 框架服务角色定义
- 角色类型定义（Master、Worker、Client）
- 角色基数约束
- 角色依赖关系

#### FrameInfoService - 框架信息服务
**职责**: 框架基础信息
- 框架名称和版本
- 框架描述信息
- 框架适用场景

### 2.9 YARN 相关服务

#### ClusterYarnSchedulerService - YARN 调度器服务
**职责**: YARN 调度器配置
- 调度器类型选择（Capacity、Fair）
- 调度器参数配置
- 调度器状态监控

#### ClusterQueueCapacityService - 队列容量服务
**职责**: YARN 队列管理
- 队列创建和删除
- 队列容量配置
- 队列资源分配

#### ClusterNodeLabelService - 节点标签服务
**职责**: YARN 节点标签
- 标签定义和分配
- 标签与队列关联
- 标签资源隔离

### 2.10 用户与权限服务

#### ClusterRoleUserService - 集群角色用户服务
**职责**: 集群权限管理
- 集群管理员分配
- 用户权限控制
- 角色权限映射

#### UserInfoService - 用户信息服务
**职责**: 用户基础信息
- 用户认证
- 用户信息管理
- 密码管理

#### SessionService - 会话服务
**职责**: 用户会话管理
- 登录会话创建
- 会话状态维护
- 会话超时控制

## 三、Service 层设计模式

### 3.1 分层架构

```
┌─────────────────────────────────────┐
│        Controller 层                │
│    (处理 HTTP 请求，参数校验)          │
└──────────────┬──────────────────────┘
               │
               ↓
┌─────────────────────────────────────┐
│        Service 层                   │
│  (业务逻辑，事务管理，服务编排)         │
└──────────────┬──────────────────────┘
               │
               ↓
┌─────────────────────────────────────┐
│        Mapper 层                    │
│    (数据访问，SQL 执行)               │
└──────────────┬──────────────────────┘
               │
               ↓
┌─────────────────────────────────────┐
│        Database (MySQL)             │
└─────────────────────────────────────┘
```

### 3.2 事务管理

所有 Service 实现类都使用 `@Transactional` 注解：

```java
@Service("clusterInfoService")
@Transactional
public class ClusterInfoServiceImpl extends ServiceImpl<...> 
    implements ClusterInfoService {
    // 所有方法默认在事务中执行
}
```

**事务传播行为**:
- 默认使用 `REQUIRED` 传播行为
- 方法调用会加入到已有事务中
- 确保一组操作的原子性

### 3.3 依赖注入

使用 Spring 的依赖注入管理服务依赖：

```java
@Autowired
private ClusterInfoService clusterInfoService;

@Autowired
private ClusterHostService clusterHostService;

@Autowired
private AlertGroupService alertGroupService;
```

### 3.4 Actor 模型集成

部分服务使用 Akka Actor 实现异步处理：

```java
// 发送命令到 Actor
ActorUtils.getLocalActor(ClusterActor.class, "clusterActor")
    .tell(new ClusterCommand(ClusterCommandType.DELETE, clusterId), 
          ActorRef.noSender());
```

**Actor 的优势**:
- 异步非阻塞
- 高并发处理
- 容错和监督
- 分布式支持

### 3.5 策略模式

不同服务使用不同的操作策略：

```java
public interface ServiceOperateStrategy {
    void start(ServiceRoleInfo roleInfo);
    void stop(ServiceRoleInfo roleInfo);
    void restart(ServiceRoleInfo roleInfo);
}

// HDFS 策略
public class HDFSOperateStrategy implements ServiceOperateStrategy {
    // HDFS 特定的启停逻辑
}

// Kafka 策略
public class KafkaOperateStrategy implements ServiceOperateStrategy {
    // Kafka 特定的启停逻辑
}
```

## 四、Service 层关键特性

### 4.1 全局变量管理

使用 `GlobalVariables` 类管理集群级别的变量：

```java
// 存储变量
HashMap<String, String> variables = new HashMap<>();
variables.put("${HDFS_HOME}", "/opt/datasophon/hdfs");
variables.put("${YARN_HOME}", "/opt/datasophon/yarn");
GlobalVariables.put(clusterId, variables);

// 获取变量
Map<String, String> vars = GlobalVariables.get(clusterId);
```

**用途**:
- 配置文件模板变量替换
- Shell 脚本参数注入
- 服务间路径引用

### 4.2 状态管理

#### 服务状态
```java
public enum ServiceState {
    WAIT_INSTALL(0, "待安装"),
    RUNNING(1, "运行中"),
    ONLY_CLIENT(2, "仅客户端"),
    EXISTS_ALARM(3, "存在告警"),
    EXISTS_EXCEPTION(4, "存在异常");
}
```

#### 角色状态
```java
public enum ServiceRoleState {
    RUNNING(1, "运行中"),
    STOP(2, "已停止"),
    STARTING(3, "启动中"),
    STOPPING(4, "停止中"),
    EXISTS_ALARM(5, "存在告警");
}
```

#### 集群状态
```java
public enum ClusterState {
    NEED_CONFIG(0, "需要配置"),
    RUNNING(1, "运行中"),
    STOP(2, "已停止"),
    DELETING(3, "删除中");
}
```

### 4.3 配置版本管理

支持配置的版本控制和回滚：

```java
// 保存新版本配置
ClusterServiceRoleGroupConfig config = new ClusterServiceRoleGroupConfig();
config.setConfigVersion(newVersion);
config.setConfigJson(jsonString);
roleGroupConfigService.save(config);

// 查询历史版本
List<ClusterServiceRoleGroupConfig> history = 
    roleGroupConfigService.getConfigHistory(roleGroupId);

// 配置对比
Result result = serviceInstanceService.configVersionCompare(
    serviceInstanceId, roleGroupId);
```

### 4.4 DAG 任务调度

服务安装使用 DAG（有向无环图）管理依赖关系：

```
ZooKeeper
    ↓
HDFS NameNode
    ↓
HDFS DataNode
    ↓
YARN ResourceManager
    ↓
YARN NodeManager
```

### 4.5 缓存策略

多层次的缓存机制：

1. **JVM 内存缓存** (`GlobalVariables`)
2. **Redis 缓存** (`CacheUtils`)
3. **数据库缓存**

## 五、Service 层最佳实践

### 5.1 事务边界控制

```java
@Transactional
public Result complexOperation() {
    // 1. 数据验证
    validate();
    
    // 2. 业务逻辑
    processBusinessLogic();
    
    // 3. 数据持久化
    saveData();
    
    // 4. 发送事件（异步，事务提交后）
    applicationEventPublisher.publishEvent(new CustomEvent());
    
    return Result.success();
}
```

### 5.2 异常处理

```java
public Result serviceMethod() {
    try {
        // 业务逻辑
        doSomething();
        return Result.success();
    } catch (BusinessException e) {
        log.error("业务异常", e);
        return Result.error(e.getMessage());
    } catch (Exception e) {
        log.error("系统异常", e);
        return Result.error("系统异常，请联系管理员");
    }
}
```

### 5.3 参数校验

```java
public Result createCluster(ClusterInfoEntity cluster) {
    // 1. 参数校验
    if (StringUtils.isBlank(cluster.getClusterCode())) {
        return Result.error("集群编码不能为空");
    }
    
    // 2. 业务规则校验
    if (existsClusterCode(cluster.getClusterCode())) {
        return Result.error("集群编码已存在");
    }
    
    // 3. 执行业务逻辑
    // ...
}
```

### 5.4 日志记录

```java
@Slf4j
public class ClusterInfoServiceImpl {
    
    public Result saveCluster(ClusterInfoEntity cluster) {
        log.info("开始创建集群: {}", cluster.getClusterName());
        
        try {
            // 业务逻辑
            this.save(cluster);
            log.info("集群创建成功, ID: {}", cluster.getId());
            return Result.success();
        } catch (Exception e) {
            log.error("集群创建失败", e);
            return Result.error("创建失败");
        }
    }
}
```

### 5.5 异步处理

```java
// 使用 Actor 异步处理耗时操作
ActorRef actor = ActorUtils.getLocalActor(ServiceActor.class, "serviceActor");
actor.tell(new InstallCommand(serviceInfo), ActorRef.noSender());

// 或使用 Spring @Async
@Async
public CompletableFuture<Result> asyncOperation() {
    // 异步操作
    return CompletableFuture.completedFuture(Result.success());
}
```

## 六、性能优化建议

### 6.1 批量操作

```java
// 避免循环单条插入
for (ClusterHostDO host : hosts) {
    this.save(host);  // ❌ 不推荐
}

// 使用批量插入
this.saveBatch(hosts);  // ✅ 推荐
```

### 6.2 分页查询

```java
// 大数据量查询使用分页
Page<ClusterHostDO> page = new Page<>(pageNum, pageSize);
IPage<ClusterHostDO> result = this.page(page, queryWrapper);
```

### 6.3 查询优化

```java
// 只查询需要的字段
QueryWrapper<ClusterHostDO> wrapper = new QueryWrapper<>();
wrapper.select("id", "hostname", "ip");
List<ClusterHostDO> list = this.list(wrapper);
```

### 6.4 缓存使用

```java
// 缓存频繁访问的数据
@Cacheable(value = "clusterInfo", key = "#clusterId")
public ClusterInfoEntity getById(Integer clusterId) {
    return super.getById(clusterId);
}
```

## 七、文档完成情况

### 7.1 已完成文档

- ✅ [01-集群服务.md](./01-集群服务.md) - ClusterInfoService 详解
- ✅ [02-服务实例服务.md](./02-服务实例服务.md) - ClusterServiceInstanceService 详解

### 7.2 待完成文档

1. **03-主机服务.md** - ClusterHostService 详解
2. **04-告警服务.md** - AlertGroupService 详解
3. **05-服务安装服务.md** - ServiceInstallService 详解
4. **06-命令执行服务.md** - ClusterServiceCommandService 详解
5. **07-监控数据采集.md** - 监控与指标服务
6. **08-告警规则管理.md** - ClusterAlertRuleService 详解
7. **09-服务操作策略.md** - ServiceOperateStrategy 详解

## 八、总结

DataSophon 的 Service 层是一个设计良好的业务逻辑层，具有以下特点：

**优点**:
1. ✅ **清晰的分层架构**: Controller-Service-Mapper 三层结构
2. ✅ **完善的事务管理**: 使用 Spring 事务确保数据一致性
3. ✅ **Actor 模型集成**: 支持异步和分布式处理
4. ✅ **策略模式应用**: 灵活的服务操作策略
5. ✅ **版本控制**: 配置版本管理和对比
6. ✅ **状态机管理**: 清晰的状态流转逻辑
7. ✅ **缓存机制**: 多层次缓存提高性能

**改进空间**:
1. 🔄 批量操作优化（减少数据库访问）
2. 🔄 异步处理增强（更多使用 Actor 或 @Async）
3. 🔄 缓存策略完善（更广泛使用缓存）
4. 🔄 监控指标添加（方法级别的性能监控）
5. 🔄 错误处理统一（统一的异常处理机制）

---

**文档版本**: v1.0  
**创建日期**: 2025-11-15  
**维护者**: DataSophon 源码分析团队
