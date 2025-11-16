# DataSophon API Controller 层源码分析

## 一、Controller 层概述

Controller 层是 DataSophon API 服务的控制器层，负责处理 HTTP 请求，调用 Service 层业务逻辑，返回响应结果。

### 设计模式
- **RESTful API**: 遵循 REST 架构风格
- **MVC 模式**: Model-View-Controller 分层架构
- **统一响应**: 使用 Result 类封装统一的响应格式

### 技术栈
- **Spring MVC**: Web 框架
- **@RestController**: RESTful API 控制器
- **@RequestMapping**: 请求映射
- **@UserPermission**: 自定义权限注解

## 二、ClusterInfoController 详细分析

### 文件信息
- **文件路径**: `datasophon-api/src/main/java/com/datasophon/api/controller/ClusterInfoController.java`
- **功能**: 集群信息管理控制器
- **URL前缀**: `/api/cluster`

### 类定义
```java
@RestController
@RequestMapping("api/cluster")
public class ClusterInfoController {
    @Autowired
    private ClusterInfoService clusterInfoService;
}
```

### 依赖注入
- **ClusterInfoService**: 集群信息服务层，处理集群相关业务逻辑

### API 接口列表

#### 2.1 获取集群列表
```java
@RequestMapping("/list")
public Result list()
```

**功能**: 获取所有集群的列表  
**HTTP 方法**: GET/POST  
**URL**: `/api/cluster/list`  
**请求参数**: 无  
**返回值**: `Result` 对象，包含集群列表数据  
**业务逻辑**: 调用 `clusterInfoService.getClusterList()` 获取集群列表

#### 2.2 获取运行中的集群列表
```java
@RequestMapping("/runningClusterList")
public Result runningClusterList()
```

**功能**: 获取已配置且正在运行的集群列表  
**HTTP 方法**: GET/POST  
**URL**: `/api/cluster/runningClusterList`  
**请求参数**: 无  
**返回值**: `Result` 对象，包含运行中集群列表  
**业务逻辑**: 调用 `clusterInfoService.runningClusterList()` 获取运行中的集群

**使用场景**: 
- 监控大盘显示运行中的集群
- 部署服务时选择目标集群

#### 2.3 获取集群详细信息
```java
@RequestMapping("/info/{id}")
public Result info(@PathVariable("id") Integer id)
```

**功能**: 根据集群 ID 获取集群详细信息  
**HTTP 方法**: GET/POST  
**URL**: `/api/cluster/info/{id}`  
**路径参数**: 
- `id`: 集群 ID

**返回值**: 
```java
{
  "code": 0,
  "msg": "success",
  "data": {
    "id": 1,
    "clusterName": "cluster1",
    "clusterCode": "c001",
    ...
  }
}
```

**业务逻辑**: 
1. 通过 `@PathVariable` 获取路径参数 id
2. 调用 `clusterInfoService.getById(id)` 查询集群信息
3. 封装到 Result 对象返回

#### 2.4 保存新集群
```java
@RequestMapping("/save")
@UserPermission
public Result save(@RequestBody ClusterInfoEntity clusterInfo)
```

**功能**: 创建新的集群  
**HTTP 方法**: POST  
**URL**: `/api/cluster/save`  
**权限要求**: 需要用户权限（@UserPermission 注解）  
**请求体**: 
```json
{
  "clusterName": "新集群",
  "clusterCode": "cluster01",
  "clusterFrame": "DDP-1.2.2",
  ...
}
```

**返回值**: `Result` 对象，包含创建结果  
**业务逻辑**: 
1. 接收前端传递的 ClusterInfoEntity 对象
2. 调用 `clusterInfoService.saveCluster(clusterInfo)` 保存集群
3. 返回保存结果

**注意事项**:
- 使用 `@UserPermission` 注解进行权限控制
- 使用 `@RequestBody` 接收 JSON 格式的请求体

#### 2.5 更新集群状态
```java
@RequestMapping("/updateClusterState")
public Result updateClusterState(Integer clusterId, Integer clusterState)
```

**功能**: 更新集群的运行状态  
**HTTP 方法**: GET/POST  
**URL**: `/api/cluster/updateClusterState`  
**请求参数**: 
- `clusterId`: 集群 ID
- `clusterState`: 集群状态（枚举值）

**返回值**: `Result` 对象  
**业务逻辑**: 调用 `clusterInfoService.updateClusterState()` 更新状态

**使用场景**: 
- 启动/停止集群
- 标记集群为维护状态

#### 2.6 更新集群信息
```java
@RequestMapping("/update")
@UserPermission
public Result update(@RequestBody ClusterInfoEntity clusterInfo)
```

**功能**: 更新集群的配置信息  
**HTTP 方法**: POST  
**URL**: `/api/cluster/update`  
**权限要求**: 需要用户权限  
**请求体**: ClusterInfoEntity JSON 对象  
**返回值**: `Result` 对象  
**业务逻辑**: 调用 `clusterInfoService.updateCluster(clusterInfo)` 更新集群

#### 2.7 删除集群
```java
@RequestMapping("/delete")
@UserPermission
public Result delete(@RequestBody Integer[] ids)
```

**功能**: 批量删除集群  
**HTTP 方法**: POST  
**URL**: `/api/cluster/delete`  
**权限要求**: 需要用户权限  
**请求体**: 
```json
[1, 2, 3]
```

**返回值**: `Result` 对象  
**业务逻辑**: 
1. 接收集群 ID 数组
2. 转换为 List 后调用 `clusterInfoService.deleteCluster()` 删除
3. 返回成功结果

**注意事项**:
- 支持批量删除
- 删除前应检查集群是否有关联服务

## 三、其他 Controller 概览

### 3.1 服务管理相关

#### ClusterServiceInstanceController
**功能**: 管理集群中的服务实例  
**URL前缀**: `/api/service/instance`  
**主要接口**:
- 服务列表查询
- 服务安装
- 服务启动/停止
- 服务配置管理

#### ClusterServiceRoleInstanceController
**功能**: 管理服务角色实例（如 NameNode, DataNode）  
**URL前缀**: `/api/service/role/instance`  
**主要接口**:
- 角色实例列表
- 角色实例启动/停止
- 角色实例重启
- 角色实例删除

### 3.2 主机管理相关

#### ClusterHostController
**功能**: 管理集群中的主机节点  
**URL前缀**: `/api/host`  
**主要接口**:
- 主机列表查询
- 主机添加/删除
- 主机状态监控
- 主机资源信息

#### HostInstallController
**功能**: 主机安装和初始化  
**URL前缀**: `/api/host/install`  
**主要接口**:
- 主机环境检查
- Worker 安装
- SSH 连接测试

### 3.3 告警管理相关

#### ClusterAlertGroupController
**功能**: 告警组管理  
**URL前缀**: `/api/alert/group`  
**主要接口**:
- 告警组列表
- 创建/更新告警组
- 删除告警组
- 告警组配置

#### ClusterAlertHistoryController
**功能**: 告警历史记录  
**URL前缀**: `/api/alert/history`  
**主要接口**:
- 告警历史查询
- 告警详情
- 告警处理记录

### 3.4 监控大盘相关

#### ClusterServiceDashboardController
**功能**: 服务监控大盘数据  
**URL前缀**: `/api/dashboard`  
**主要接口**:
- 集群概览数据
- 服务监控指标
- 节点状态统计
- 性能数据查询

### 3.5 用户权限相关

#### UserInfoController
**功能**: 用户信息管理  
**URL前缀**: `/api/user`  
**主要接口**:
- 用户列表
- 创建/更新用户
- 用户密码修改
- 用户角色分配

#### LoginController
**功能**: 用户登录认证  
**URL前缀**: `/api/login`  
**主要接口**:
- 用户登录
- 用户登出
- Token 刷新
- 验证码生成

### 3.6 配置管理相关

#### ClusterServiceInstanceConfigController
**功能**: 服务实例配置管理  
**URL前缀**: `/api/service/config`  
**主要接口**:
- 配置项查询
- 配置更新
- 配置历史
- 配置对比

#### ClusterYarnSchedulerController
**功能**: YARN 调度器配置  
**URL前缀**: `/api/yarn/scheduler`  
**主要接口**:
- 调度器配置查询
- 调度器配置更新
- 队列配置管理

## 四、Controller 层设计模式

### 4.1 统一响应格式
```java
public class Result {
    private int code;       // 响应码：0-成功，非0-失败
    private String msg;     // 响应消息
    private Object data;    // 响应数据
}
```

### 4.2 权限控制
使用 `@UserPermission` 注解标记需要权限验证的接口：
```java
@RequestMapping("/save")
@UserPermission
public Result save(@RequestBody ClusterInfoEntity clusterInfo)
```

### 4.3 异常处理
通过 `@ApiExceptionHandler` 全局异常处理器统一处理异常：
- 业务异常：ApiException
- 系统异常：RuntimeException
- 参数验证异常：MethodArgumentNotValidException

### 4.4 参数接收方式
- **路径参数**: `@PathVariable`
- **查询参数**: 方法参数自动绑定
- **请求体**: `@RequestBody`
- **表单参数**: 自动绑定到对象

## 五、Controller 层最佳实践

### 5.1 职责单一
- Controller 只负责请求响应处理
- 业务逻辑委托给 Service 层
- 不直接操作数据库

### 5.2 统一返回格式
- 所有接口返回 `Result` 对象
- 成功：code=0, data 包含数据
- 失败：code≠0, msg 包含错误信息

### 5.3 权限控制
- 敏感操作添加 `@UserPermission` 注解
- 在拦截器中统一验证权限

### 5.4 参数验证
- 使用 Bean Validation 注解验证参数
- 统一的异常处理机制

### 5.5 RESTful 设计
- 资源命名清晰
- HTTP 方法语义明确
- URL 层次结构合理

## 六、完整 Controller 列表

| Controller | 功能 | URL前缀 |
|-----------|------|---------|
| ClusterInfoController | 集群管理 | /api/cluster |
| ClusterHostController | 主机管理 | /api/host |
| ClusterServiceInstanceController | 服务实例管理 | /api/service/instance |
| ClusterServiceRoleInstanceController | 服务角色管理 | /api/service/role |
| ClusterAlertGroupController | 告警组管理 | /api/alert/group |
| ClusterAlertHistoryController | 告警历史 | /api/alert/history |
| ClusterAlertQuotaController | 告警指标 | /api/alert/quota |
| ClusterServiceDashboardController | 监控大盘 | /api/dashboard |
| UserInfoController | 用户管理 | /api/user |
| LoginController | 登录认证 | /api/login |
| ClusterServiceInstanceConfigController | 服务配置 | /api/service/config |
| ClusterYarnSchedulerController | YARN调度器 | /api/yarn/scheduler |
| ClusterYarnQueueController | YARN队列 | /api/yarn/queue |
| ClusterKerberosController | Kerberos配置 | /api/kerberos |
| ClusterRackController | 机架管理 | /api/rack |
| ClusterGroupController | 集群分组 | /api/cluster/group |
| FrameServiceController | 框架服务 | /api/frame/service |
| ServiceInstallController | 服务安装 | /api/service/install |
| HostInstallController | 主机安装 | /api/host/install |

## 七、相关文件

- **Result类**: `com.datasophon.common.utils.Result`
- **权限注解**: `com.datasophon.api.security.UserPermission`
- **异常处理**: `com.datasophon.api.exceptions.ApiExceptionHandler`
- **拦截器**: `com.datasophon.api.interceptor.LoginHandlerInterceptor`

---

**文档版本**: v1.0  
**最后更新**: 2025-11-15
