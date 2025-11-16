# ClusterInfoController 集群管理控制器详解

## 一、控制器概述

### 1.1 基本信息

**文件位置**: `datasophon-api/src/main/java/com/datasophon/api/controller/ClusterInfoController.java`

**职责定位**: 
- 提供集群管理的 RESTful API 接口
- 处理集群的 CRUD 操作
- 管理集群状态变更
- 集群列表查询和筛选

**技术特点**:
- 使用 Spring MVC 注解映射
- 基于 Service 层进行业务处理
- 统一的 Result 响应封装
- 权限控制注解 @UserPermission

### 1.2 类结构

```java
@RestController
@RequestMapping("api/cluster")
public class ClusterInfoController {
    @Autowired
    private ClusterInfoService clusterInfoService;
}
```

**关键特征**:
- **@RestController**: 自动将返回值序列化为 JSON
- **请求前缀**: `/api/cluster`
- **依赖注入**: ClusterInfoService 业务服务
- **无状态设计**: RESTful 风格，不保存会话状态

## 二、API 接口详解

### 2.1 查询集群列表

#### 接口定义
```java
@RequestMapping("/list")
public Result list() {
    return clusterInfoService.getClusterList();
}
```

**接口信息**:
- **URL**: `GET /api/cluster/list`
- **请求参数**: 无
- **返回值**: Result 包装的集群列表
- **权限要求**: 无特殊权限（登录即可）

**功能描述**:
- 查询当前用户可见的所有集群
- 返回集群基本信息列表
- 包含集群名称、状态、创建时间等

**使用场景**:
1. 首页展示所有集群
2. 集群选择下拉框
3. 集群总览页面

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "data": [
    {
      "id": 1,
      "clusterName": "bigdata-cluster",
      "clusterCode": "cluster001",
      "clusterState": 1,
      "frameCode": "HADOOP",
      "createTime": "2025-01-01 10:00:00"
    }
  ]
}
```

### 2.2 查询运行中的集群

#### 接口定义
```java
@RequestMapping("/runningClusterList")
public Result runningClusterList() {
    return clusterInfoService.runningClusterList();
}
```

**接口信息**:
- **URL**: `GET /api/cluster/runningClusterList`
- **请求参数**: 无
- **返回值**: Result 包装的运行中集群列表
- **权限要求**: 无特殊权限

**功能描述**:
- 查询已配置完成且正在运行的集群
- 过滤掉未配置或停止的集群
- 通常用于需要选择可用集群的场景

**使用场景**:
1. 服务安装时选择目标集群
2. 监控系统选择监控对象
3. 操作任务选择执行集群

**与 list 接口的区别**:
| 特性 | list | runningClusterList |
|-----|------|-------------------|
| 状态过滤 | 所有状态 | 仅运行中 |
| 配置要求 | 包含未配置 | 仅已配置 |
| 使用场景 | 管理页面 | 操作选择 |

### 2.3 获取集群详情

#### 接口定义
```java
@RequestMapping("/info/{id}")
public Result info(@PathVariable("id") Integer id) {
    ClusterInfoEntity clusterInfo = clusterInfoService.getById(id);
    return Result.success().put(Constants.DATA, clusterInfo);
}
```

**接口信息**:
- **URL**: `GET /api/cluster/info/{id}`
- **请求参数**: 
  - `id`: 集群ID（路径参数）
- **返回值**: Result 包装的集群详细信息
- **权限要求**: 无特殊权限

**功能描述**:
- 根据集群 ID 查询完整的集群信息
- 包含集群配置、状态、元数据等
- 支持编辑时回显数据

**使用场景**:
1. 集群详情页面展示
2. 集群编辑前获取当前数据
3. 集群状态监控

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "id": 1,
    "clusterName": "bigdata-cluster",
    "clusterCode": "cluster001",
    "clusterState": 1,
    "frameCode": "HADOOP",
    "frameVersion": "3.3.4",
    "createTime": "2025-01-01 10:00:00",
    "updateTime": "2025-01-10 15:30:00",
    "description": "生产环境大数据集群"
  }
}
```

**设计要点**:
- 使用 `@PathVariable` 注解获取 URL 路径参数
- 直接调用 Service 的 getById 方法
- 通过 Constants.DATA 作为统一的数据键名

### 2.4 保存新集群

#### 接口定义
```java
@RequestMapping("/save")
@UserPermission
public Result save(@RequestBody ClusterInfoEntity clusterInfo) {
    return clusterInfoService.saveCluster(clusterInfo);
}
```

**接口信息**:
- **URL**: `POST /api/cluster/save`
- **请求参数**: 
  - `clusterInfo`: 集群信息对象（JSON Body）
- **返回值**: Result 表示操作结果
- **权限要求**: @UserPermission 注解控制

**功能描述**:
- 创建新的集群
- 验证集群名称唯一性
- 初始化集群基础配置
- 生成集群编码

**使用场景**:
1. 首次创建集群
2. 添加新的业务集群
3. 集群初始化流程

**请求数据示例**:
```json
{
  "clusterName": "test-cluster",
  "clusterCode": "cluster002",
  "frameCode": "HADOOP",
  "frameVersion": "3.3.4",
  "description": "测试环境集群"
}
```

**业务逻辑**:
1. 参数验证
   - 集群名称不能为空
   - 集群名称不能重复
   - 框架版本必须合法
2. 数据初始化
   - 生成集群 ID
   - 设置创建时间
   - 初始化状态为"未配置"
3. 保存到数据库
4. 返回操作结果

**权限说明**:
- `@UserPermission` 注解表示需要管理员权限
- 普通用户只能查看，不能创建集群
- 防止未授权用户创建资源

### 2.5 更新集群状态

#### 接口定义
```java
@RequestMapping("/updateClusterState")
public Result updateClusterState(Integer clusterId, Integer clusterState) {
    return clusterInfoService.updateClusterState(clusterId, clusterState);
}
```

**接口信息**:
- **URL**: `POST /api/cluster/updateClusterState`
- **请求参数**: 
  - `clusterId`: 集群ID
  - `clusterState`: 目标状态
- **返回值**: Result 表示操作结果
- **权限要求**: 无特殊权限（但通常由系统内部调用）

**功能描述**:
- 修改集群的运行状态
- 状态转换验证
- 记录状态变更历史

**集群状态枚举**:
```
0 - 未配置 (WAIT_CONFIG)
1 - 运行中 (RUNNING)
2 - 已停止 (STOPPED)
3 - 配置中 (CONFIGURING)
4 - 异常 (ERROR)
```

**使用场景**:
1. 服务安装完成后，状态从"配置中"变为"运行中"
2. 集群停止后，状态变为"已停止"
3. 出现异常时，状态变为"异常"

**状态转换规则**:
```
未配置 → 配置中 → 运行中
运行中 → 已停止
任意状态 → 异常
```

**注意事项**:
- 不是所有状态转换都合法
- 需要验证前置状态
- 某些状态转换需要检查集群内服务状态

### 2.6 更新集群信息

#### 接口定义
```java
@RequestMapping("/update")
@UserPermission
public Result update(@RequestBody ClusterInfoEntity clusterInfo) {
    return clusterInfoService.updateCluster(clusterInfo);
}
```

**接口信息**:
- **URL**: `POST /api/cluster/update`
- **请求参数**: 
  - `clusterInfo`: 集群信息对象（JSON Body）
- **返回值**: Result 表示操作结果
- **权限要求**: @UserPermission 注解控制

**功能描述**:
- 修改集群的配置信息
- 更新集群描述、版本等
- 验证修改的合法性

**可修改字段**:
- 集群名称（需验证唯一性）
- 集群描述
- 框架版本（需谨慎）
- 其他元数据

**不可修改字段**:
- 集群 ID
- 集群编码
- 创建时间

**使用场景**:
1. 修改集群描述信息
2. 升级集群框架版本
3. 调整集群配置参数

**请求数据示例**:
```json
{
  "id": 1,
  "clusterName": "bigdata-cluster-updated",
  "description": "更新后的描述信息",
  "frameVersion": "3.3.5"
}
```

**业务逻辑**:
1. 验证集群是否存在
2. 验证修改权限
3. 验证字段合法性
4. 更新数据库
5. 清理相关缓存

### 2.7 删除集群

#### 接口定义
```java
@RequestMapping("/delete")
@UserPermission
public Result delete(@RequestBody Integer[] ids) {
    clusterInfoService.deleteCluster(Arrays.asList(ids));
    return Result.success();
}
```

**接口信息**:
- **URL**: `POST /api/cluster/delete`
- **请求参数**: 
  - `ids`: 集群ID数组（JSON Body）
- **返回值**: Result 表示操作结果
- **权限要求**: @UserPermission 注解控制

**功能描述**:
- 批量删除集群
- 级联删除相关数据
- 删除前检查约束

**使用场景**:
1. 清理测试集群
2. 删除废弃集群
3. 批量清理操作

**请求数据示例**:
```json
[1, 2, 3]
```

**删除前检查**:
1. 集群是否存在
2. 集群是否有正在运行的服务
3. 是否有关联的任务正在执行
4. 用户是否有删除权限

**级联删除内容**:
- 集群下的所有主机
- 集群下的所有服务实例
- 集群相关的配置数据
- 集群相关的监控数据
- 集群相关的告警规则

**安全措施**:
- 需要管理员权限
- 删除前二次确认
- 记录删除操作日志
- 支持软删除（可选）

## 三、设计模式与架构

### 3.1 RESTful 设计

#### 资源定位
- **资源**: Cluster（集群）
- **集合**: `/api/cluster/list`
- **单个**: `/api/cluster/info/{id}`

#### HTTP 方法映射
虽然代码中都使用 `@RequestMapping`，但实际应该按照 RESTful 规范：
```
GET    /api/cluster/list      - 列表查询
GET    /api/cluster/info/{id} - 详情查询
POST   /api/cluster/save      - 创建资源
PUT    /api/cluster/update    - 更新资源
DELETE /api/cluster/delete    - 删除资源
```

**改进建议**:
```java
@GetMapping("/list")
@GetMapping("/info/{id}")
@PostMapping("/save")
@PutMapping("/update")
@DeleteMapping("/delete")
```

### 3.2 分层架构

```
┌─────────────────────────┐
│   ClusterInfoController  │ ← HTTP 请求入口
├─────────────────────────┤
│   ClusterInfoService     │ ← 业务逻辑层
├─────────────────────────┤
│   ClusterInfoMapper      │ ← 数据访问层
├─────────────────────────┤
│   MySQL Database         │ ← 数据存储
└─────────────────────────┘
```

**职责分离**:
- **Controller**: 仅负责请求响应，不包含业务逻辑
- **Service**: 实现核心业务逻辑，事务控制
- **Mapper**: 数据库操作，SQL 执行

### 3.3 依赖注入

```java
@Autowired
private ClusterInfoService clusterInfoService;
```

**优点**:
- 解耦 Controller 和 Service
- 方便单元测试（可 Mock Service）
- Spring 容器统一管理生命周期

**最佳实践**:
```java
// 推荐使用构造器注入
private final ClusterInfoService clusterInfoService;

public ClusterInfoController(ClusterInfoService clusterInfoService) {
    this.clusterInfoService = clusterInfoService;
}
```

### 3.4 统一响应封装

```java
return Result.success().put(Constants.DATA, clusterInfo);
```

**Result 结构**:
```java
{
  "code": 0,      // 状态码
  "msg": "success", // 提示信息
  "data": {...}   // 数据内容
}
```

**优点**:
- 统一的返回格式
- 前端统一处理
- 便于错误处理

## 四、安全与权限

### 4.1 @UserPermission 注解

```java
@RequestMapping("/save")
@UserPermission
public Result save(@RequestBody ClusterInfoEntity clusterInfo) {
    // ...
}
```

**功能**:
- 标记需要特殊权限的接口
- 通过 AOP 拦截器实现权限检查
- 未授权用户访问会返回 403

**实现原理**:
```java
@Aspect
public class UserPermissionAspect {
    @Around("@annotation(UserPermission)")
    public Object checkPermission(ProceedingJoinPoint point) {
        // 获取当前用户
        // 检查权限
        // 有权限则继续执行，否则抛出异常
    }
}
```

### 4.2 接口权限分级

| 接口 | 权限要求 | 说明 |
|-----|---------|------|
| list | 登录用户 | 查看集群列表 |
| info | 登录用户 | 查看集群详情 |
| save | 管理员 | 创建新集群 |
| update | 管理员 | 修改集群信息 |
| delete | 管理员 | 删除集群 |

### 4.3 数据权限

- 用户只能看到自己有权限的集群
- 多租户场景下的数据隔离
- 通过 SQL 过滤实现

## 五、性能优化

### 5.1 缓存策略

**列表缓存**:
```java
@Cacheable(value = "cluster:list", key = "#userId")
public Result getClusterList() {
    // ...
}
```

**详情缓存**:
```java
@Cacheable(value = "cluster:info", key = "#id")
public ClusterInfoEntity getById(Integer id) {
    // ...
}
```

**缓存失效**:
```java
@CacheEvict(value = "cluster:*", allEntries = true)
public Result updateCluster(ClusterInfoEntity clusterInfo) {
    // ...
}
```

### 5.2 分页查询

虽然当前 list 接口没有分页，但建议添加：
```java
@RequestMapping("/list")
public Result list(Integer page, Integer pageSize) {
    return clusterInfoService.getClusterList(page, pageSize);
}
```

### 5.3 批量操作

删除接口已支持批量操作：
```java
public Result delete(@RequestBody Integer[] ids) {
    clusterInfoService.deleteCluster(Arrays.asList(ids));
    return Result.success();
}
```

## 六、异常处理

### 6.1 全局异常拦截

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public Result handleException(Exception e) {
        log.error("系统异常", e);
        return Result.error("系统异常: " + e.getMessage());
    }
}
```

### 6.2 业务异常

Service 层抛出的业务异常会被统一捕获：
```java
if (clusterExists) {
    throw new ServiceException("集群名称已存在");
}
```

## 七、最佳实践

### 7.1 参数验证

**添加 JSR-303 验证**:
```java
public Result save(@Valid @RequestBody ClusterInfoEntity clusterInfo) {
    // Spring 自动验证参数
}
```

**实体类添加验证注解**:
```java
public class ClusterInfoEntity {
    @NotBlank(message = "集群名称不能为空")
    private String clusterName;
    
    @NotNull(message = "框架代码不能为空")
    private String frameCode;
}
```

### 7.2 日志记录

```java
@Slf4j
public class ClusterInfoController {
    @RequestMapping("/save")
    public Result save(@RequestBody ClusterInfoEntity clusterInfo) {
        log.info("创建集群: {}", clusterInfo.getClusterName());
        return clusterInfoService.saveCluster(clusterInfo);
    }
}
```

### 7.3 API 文档

使用 Swagger 生成接口文档：
```java
@Api(tags = "集群管理")
@RestController
@RequestMapping("api/cluster")
public class ClusterInfoController {
    
    @ApiOperation("查询集群列表")
    @GetMapping("/list")
    public Result list() {
        // ...
    }
}
```

## 八、使用示例

### 8.1 前端调用示例

**查询集群列表**:
```javascript
axios.get('/api/cluster/list')
  .then(response => {
    const clusters = response.data.data;
    console.log(clusters);
  });
```

**创建集群**:
```javascript
const clusterData = {
  clusterName: 'new-cluster',
  frameCode: 'HADOOP',
  frameVersion: '3.3.4'
};

axios.post('/api/cluster/save', clusterData)
  .then(response => {
    console.log('集群创建成功');
  });
```

**删除集群**:
```javascript
axios.post('/api/cluster/delete', [1, 2, 3])
  .then(response => {
    console.log('集群删除成功');
  });
```

### 8.2 集成测试示例

```java
@SpringBootTest
@AutoConfigureMockMvc
public class ClusterInfoControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    public void testGetClusterList() throws Exception {
        mockMvc.perform(get("/api/cluster/list"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(0));
    }
}
```

## 九、总结

### 9.1 核心功能

ClusterInfoController 是 DataSophon 集群管理的核心入口，提供：
1. 集群的完整 CRUD 操作
2. 集群状态管理
3. 权限控制
4. 统一的响应格式

### 9.2 设计特点

- **RESTful 风格**: 符合 REST 规范
- **分层清晰**: Controller 只负责请求响应
- **权限控制**: 基于注解的权限管理
- **统一封装**: Result 统一返回格式

### 9.3 改进建议

1. **使用明确的 HTTP 方法**: @GetMapping、@PostMapping 等
2. **添加参数验证**: @Valid 注解 + JSR-303
3. **完善日志记录**: 记录关键操作
4. **添加接口文档**: Swagger 或 OpenAPI
5. **实现分页查询**: 支持大数据量场景
6. **优化批量操作**: 提高效率
7. **增强安全性**: 输入过滤、SQL 注入防护

---

**文档版本**: v1.0  
**最后更新**: 2025-11-15  
**维护者**: DataSophon 源码分析团队
