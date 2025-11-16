# DataSophon API 模块速查手册

## 一、速查手册说明

本手册提供 DataSophon API 模块的快速参考，包含常用 API、配置、代码片段等，方便快速查找和使用。

**适用人群**: 开发人员、运维人员  
**更新日期**: 2025-11-16

## 二、常用 API 速查

### 2.1 集群管理 API

```bash
# 获取集群列表
GET /api/cluster/list

# 获取集群详情
GET /api/cluster/info/{clusterId}

# 创建集群
POST /api/cluster/save
Content-Type: application/json
{
  "clusterName": "生产集群",
  "clusterCode": "prod-cluster",
  "clusterFrame": "DDP-1.2.2"
}

# 更新集群
POST /api/cluster/update
{
  "id": 1,
  "clusterName": "新名称"
}

# 删除集群
POST /api/cluster/delete
[1, 2, 3]

# 更新集群状态
POST /api/cluster/updateClusterState?clusterId=1&clusterState=1
```

### 2.2 服务管理 API

```bash
# 获取服务列表
GET /api/service/list?clusterId=1

# 安装服务
POST /api/service/install
{
  "clusterId": 1,
  "serviceName": "HDFS",
  "serviceVersion": "3.3.4"
}

# 启动服务
POST /api/service/start?serviceId=1

# 停止服务
POST /api/service/stop?serviceId=1

# 重启服务
POST /api/service/restart?serviceId=1

# 获取服务配置
GET /api/service/config/list?serviceInstanceId=1

# 更新服务配置
POST /api/service/config/update
{
  "serviceInstanceId": 1,
  "configs": {
    "dfs.replication": "3",
    "dfs.blocksize": "134217728"
  }
}
```

### 2.3 主机管理 API

```bash
# 获取主机列表
GET /api/host/list?clusterId=1

# 添加主机
POST /api/host/add
{
  "clusterId": 1,
  "hostname": "node01",
  "ip": "192.168.1.101",
  "rackId": 1
}

# 删除主机
POST /api/host/delete
[1, 2, 3]

# 获取主机详情
GET /api/host/info/{hostId}

# 检查主机连接
POST /api/host/checkConnection
{
  "ip": "192.168.1.101",
  "sshPort": 22,
  "username": "root",
  "password": "******"
}
```

### 2.4 监控大盘 API

```bash
# 获取集群概览
GET /api/dashboard/clusterOverview?clusterId=1

# 获取服务监控指标
GET /api/dashboard/serviceMetrics
  ?clusterId=1
  &serviceName=HDFS
  &metricName=cpu_usage_percent
  &startTime=1700100000000
  &endTime=1700200000000

# 获取主机监控指标
GET /api/dashboard/hostMetrics
  ?clusterId=1
  &hostname=node01
  &metricNames=cpu,memory,disk
  &startTime=1700100000000
  &endTime=1700200000000

# 获取活跃告警
GET /api/dashboard/activeAlerts?clusterId=1&page=1&pageSize=20

# 获取实时指标
GET /api/dashboard/realtimeMetrics?clusterId=1
```

### 2.5 告警管理 API

```bash
# 获取告警组列表
GET /api/alert/group/list?clusterId=1

# 创建告警组
POST /api/alert/group/save
{
  "clusterId": 1,
  "alertGroupName": "生产告警组",
  "alertGroupCategory": "email,dingtalk"
}

# 获取告警历史
GET /api/alert/history/list?clusterId=1&page=1&pageSize=20

# 获取告警规则列表
GET /api/alert/rule/list?clusterId=1
```

### 2.6 用户权限 API

```bash
# 用户登录
POST /api/login/signin
{
  "username": "admin",
  "password": "admin123",
  "captchaId": "abc123",
  "captchaCode": "1234"
}

# 用户登出
POST /api/login/signout

# 刷新 Token
POST /api/login/refreshToken

# 获取验证码
GET /api/login/captcha

# 获取用户列表
GET /api/user/list?page=1&pageSize=20

# 创建用户
POST /api/user/save
{
  "username": "testuser",
  "password": "Pass@123",
  "email": "test@example.com"
}

# 修改密码
POST /api/user/changePassword
{
  "oldPassword": "Pass@123",
  "newPassword": "NewPass@456"
}
```

## 三、配置速查

### 3.1 应用配置

```yaml
# application.yml

# 服务器配置
server:
  port: 8080
  tomcat:
    max-threads: 200
    max-connections: 10000

# 数据源配置
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/datasophon
    username: root
    password: ******
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5

# Redis 配置
spring:
  redis:
    host: localhost
    port: 6379
    database: 0
    lettuce:
      pool:
        max-active: 20

# MyBatis 配置
mybatis:
  mapper-locations: classpath*:mapper/**/*.xml
  type-aliases-package: com.datasophon.domain.entity
  configuration:
    map-underscore-to-camel-case: true
    cache-enabled: true
```

### 3.2 日志配置

```xml
<!-- logback-spring.xml -->
<configuration>
    <property name="LOG_PATH" value="/opt/datasophon/logs/api"/>
    
    <!-- 控制台输出 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-DD HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- 文件输出 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/datasophon-api.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/datasophon-api.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

### 3.3 Akka 配置

```yaml
# Akka Actor 系统配置
akka:
  actor:
    provider: remote
    default-dispatcher:
      type: Dispatcher
      executor: fork-join-executor
      fork-join-executor:
        parallelism-min: 8
        parallelism-max: 64
  
  remote:
    artery:
      enabled: on
      transport: tcp
      canonical:
        hostname: localhost
        port: 2551
```

## 四、代码片段速查

### 4.1 Controller 模板

```java
@RestController
@RequestMapping("api/resource")
@Slf4j
public class ResourceController {
    
    @Autowired
    private ResourceService resourceService;
    
    /**
     * 获取列表
     */
    @GetMapping("/list")
    public Result list(
        @RequestParam(defaultValue = "1") Integer page,
        @RequestParam(defaultValue = "20") Integer pageSize
    ) {
        Page<Resource> result = resourceService.list(page, pageSize);
        return Result.success(result);
    }
    
    /**
     * 获取详情
     */
    @GetMapping("/info/{id}")
    public Result info(@PathVariable Integer id) {
        Resource resource = resourceService.getById(id);
        if (resource == null) {
            throw new NotFoundException("资源", id);
        }
        return Result.success(resource);
    }
    
    /**
     * 创建
     */
    @PostMapping("/save")
    @RequiresPermissions("resource:create")
    public Result save(@Valid @RequestBody ResourceCreateRequest request) {
        resourceService.create(request);
        return Result.success();
    }
    
    /**
     * 更新
     */
    @PutMapping("/update")
    @RequiresPermissions("resource:update")
    public Result update(@Valid @RequestBody ResourceUpdateRequest request) {
        resourceService.update(request);
        return Result.success();
    }
    
    /**
     * 删除
     */
    @DeleteMapping("/delete")
    @RequiresPermissions("resource:delete")
    public Result delete(@RequestBody Integer[] ids) {
        resourceService.deleteByIds(Arrays.asList(ids));
        return Result.success();
    }
}
```

### 4.2 Service 模板

```java
@Service
@Slf4j
public class ResourceService {
    
    @Autowired
    private ResourceMapper resourceMapper;
    
    /**
     * 分页查询
     */
    public Page<Resource> list(Integer page, Integer pageSize) {
        Page<Resource> pageObj = new Page<>(page, pageSize);
        return resourceMapper.selectPage(pageObj, null);
    }
    
    /**
     * 根据 ID 查询
     */
    public Resource getById(Integer id) {
        return resourceMapper.selectById(id);
    }
    
    /**
     * 创建资源
     */
    @Transactional(rollbackFor = Exception.class)
    public void create(ResourceCreateRequest request) {
        // 1. 参数验证
        validateCreateRequest(request);
        
        // 2. 检查是否存在
        Resource exists = resourceMapper.selectByName(request.getName());
        if (exists != null) {
            throw new BusinessException("资源名称已存在");
        }
        
        // 3. 创建资源
        Resource resource = new Resource();
        BeanUtils.copyProperties(request, resource);
        resource.setCreateTime(LocalDateTime.now());
        resourceMapper.insert(resource);
    }
    
    /**
     * 更新资源
     */
    @Transactional(rollbackFor = Exception.class)
    public void update(ResourceUpdateRequest request) {
        // 1. 检查资源是否存在
        Resource resource = resourceMapper.selectById(request.getId());
        if (resource == null) {
            throw new NotFoundException("资源", request.getId());
        }
        
        // 2. 更新资源
        BeanUtils.copyProperties(request, resource);
        resource.setUpdateTime(LocalDateTime.now());
        resourceMapper.updateById(resource);
    }
    
    /**
     * 批量删除
     */
    @Transactional(rollbackFor = Exception.class)
    public void deleteByIds(List<Integer> ids) {
        resourceMapper.deleteBatchIds(ids);
    }
    
    /**
     * 验证创建请求
     */
    private void validateCreateRequest(ResourceCreateRequest request) {
        if (StringUtils.isBlank(request.getName())) {
            throw new ValidationException("资源名称不能为空");
        }
    }
}
```

### 4.3 自定义异常

```java
// API 异常
throw new ApiException("操作失败");
throw new ApiException(500, "系统错误");

// 未认证异常
throw new UnauthorizedException("未登录");

// 无权限异常
throw new ForbiddenException("权限不足");

// 资源不存在异常
throw new NotFoundException("集群", clusterId);

// 业务异常
throw new BusinessException("集群名称已存在");
throw new BusinessException(ErrorCode.CLUSTER_NAME_EXISTS);

// 参数验证异常
throw new ValidationException("参数验证失败", errors);

// 限流异常
throw new RateLimitException("请求过于频繁");
```

### 4.4 权限注解

```java
// 需要单个权限
@RequiresPermissions("cluster:create")
public Result create() { ... }

// 需要多个权限（AND）
@RequiresPermissions({"cluster:create", "cluster:manage"})
public Result create() { ... }

// 需要任意一个权限（OR）
@RequiresPermissions(
    value = {"cluster:create", "cluster:manage"},
    logical = Logical.OR
)
public Result create() { ... }
```

### 4.5 参数验证

```java
@Data
public class Request {
    
    @NotBlank(message = "名称不能为空")
    private String name;
    
    @NotNull(message = "ID 不能为空")
    @Min(value = 1, message = "ID 必须大于 0")
    private Integer id;
    
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String phone;
    
    @Size(min = 1, max = 100, message = "列表大小必须在 1-100 之间")
    private List<String> items;
}
```

### 4.6 事务处理

```java
// 声明式事务
@Transactional(rollbackFor = Exception.class)
public void create(Request request) {
    // 业务逻辑
    mapper.insert(entity);
}

// 编程式事务
@Autowired
private TransactionTemplate transactionTemplate;

public void execute() {
    transactionTemplate.execute(status -> {
        try {
            // 业务逻辑
            mapper.insert(entity);
            return true;
        } catch (Exception e) {
            status.setRollbackOnly();
            throw e;
        }
    });
}
```

## 五、常用工具类

### 5.1 Result 工具类

```java
// 成功响应
return Result.success();
return Result.success(data);
return Result.success("操作成功", data);

// 失败响应
return Result.error("操作失败");
return Result.error(500, "系统错误");
return Result.error(ErrorCode.CLUSTER_NOT_FOUND);
```

### 5.2 UserContext 工具类

```java
// 获取当前用户 ID
Integer userId = UserContext.getCurrentUserId();

// 获取当前用户名
String username = UserContext.getCurrentUsername();

// 设置当前用户
UserContext.setCurrentUser(userId, username);

// 清理上下文
UserContext.clear();
```

### 5.3 JsonUtils 工具类

```java
// 对象转 JSON
String json = JsonUtils.toJson(object);

// JSON 转对象
User user = JsonUtils.fromJson(json, User.class);

// JSON 转列表
List<User> users = JsonUtils.fromJson(json, new TypeReference<List<User>>() {});
```

### 5.4 BeanUtils 工具类

```java
// 对象复制
Target target = new Target();
BeanUtils.copyProperties(source, target);

// 忽略 null 值复制
BeanUtils.copyProperties(source, target, "id", "createTime");
```

## 六、常用 SQL 片段

### 6.1 集群查询

```sql
-- 查询集群列表
SELECT * FROM t_cluster_info WHERE state = 1;

-- 查询集群及服务数量
SELECT 
    c.id,
    c.cluster_name,
    COUNT(s.id) as service_count
FROM t_cluster_info c
LEFT JOIN t_cluster_service_instance s ON c.id = s.cluster_id
GROUP BY c.id;

-- 查询集群及主机数量
SELECT 
    c.id,
    c.cluster_name,
    COUNT(h.id) as host_count
FROM t_cluster_info c
LEFT JOIN t_cluster_host h ON c.id = h.cluster_id
GROUP BY c.id;
```

### 6.2 服务查询

```sql
-- 查询服务列表
SELECT * FROM t_cluster_service_instance WHERE cluster_id = 1;

-- 查询服务及角色实例
SELECT 
    s.id,
    s.service_name,
    COUNT(r.id) as role_count
FROM t_cluster_service_instance s
LEFT JOIN t_cluster_service_role_instance r ON s.id = r.service_id
WHERE s.cluster_id = 1
GROUP BY s.id;

-- 查询服务状态统计
SELECT 
    service_state,
    COUNT(*) as count
FROM t_cluster_service_instance
WHERE cluster_id = 1
GROUP BY service_state;
```

### 6.3 告警查询

```sql
-- 查询活跃告警
SELECT * FROM t_cluster_alert_history 
WHERE cluster_id = 1 AND alert_state = 'FIRING'
ORDER BY create_time DESC;

-- 查询告警统计
SELECT 
    severity,
    COUNT(*) as count
FROM t_cluster_alert_history
WHERE cluster_id = 1 AND alert_state = 'FIRING'
GROUP BY severity;

-- 查询告警趋势
SELECT 
    DATE(create_time) as date,
    COUNT(*) as count
FROM t_cluster_alert_history
WHERE cluster_id = 1
  AND create_time >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY DATE(create_time);
```

## 七、常见问题快速解决

### 7.1 登录问题

**问题**: Token 无效或已过期

**解决方案**:
```java
// 1. 刷新 Token
POST /api/login/refreshToken

// 2. 重新登录
POST /api/login/signin
```

### 7.2 权限问题

**问题**: 权限不足

**解决方案**:
```java
// 1. 检查用户权限
SELECT p.permission_code 
FROM t_user_role ur
JOIN t_role_permission rp ON ur.role_id = rp.role_id
JOIN t_permission p ON rp.permission_id = p.id
WHERE ur.user_id = ?;

// 2. 分配权限
INSERT INTO t_user_role (user_id, role_id) VALUES (?, ?);
```

### 7.3 配置问题

**问题**: 数据库连接失败

**解决方案**:
```yaml
# 检查 application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/datasophon
    username: root
    password: ******
    
# 测试数据库连接
mysql -h localhost -u root -p datasophon
```

### 7.4 性能问题

**问题**: 接口响应慢

**解决方案**:
```java
// 1. 检查慢查询日志
SELECT * FROM slow_query_log;

// 2. 添加索引
CREATE INDEX idx_cluster_id ON t_cluster_service_instance(cluster_id);

// 3. 开启缓存
@Cacheable(value = "cluster", key = "#id")
public Cluster getById(Integer id) { ... }

// 4. 使用异步处理
@Async
public CompletableFuture<Result> process() { ... }
```

### 7.5 异常问题

**问题**: 系统异常

**解决方案**:
```bash
# 1. 查看日志
tail -f /opt/datasophon/logs/api/datasophon-api.log

# 2. 查看错误日志
tail -f /opt/datasophon/logs/api/datasophon-api-error.log

# 3. 查看异常日志表
SELECT * FROM t_exception_log ORDER BY create_time DESC LIMIT 10;
```

## 八、性能优化技巧

### 8.1 数据库优化

```sql
-- 1. 添加索引
CREATE INDEX idx_cluster_id ON t_cluster_service_instance(cluster_id);
CREATE INDEX idx_create_time ON t_cluster_alert_history(create_time);

-- 2. 优化查询
-- 避免 SELECT *
SELECT id, name, state FROM t_cluster_info;

-- 使用 LIMIT
SELECT * FROM t_cluster_info LIMIT 100;

-- 使用 JOIN 代替子查询
SELECT c.*, s.service_name
FROM t_cluster_info c
LEFT JOIN t_cluster_service_instance s ON c.id = s.cluster_id;
```

### 8.2 缓存优化

```java
// 1. 启用 Spring Cache
@EnableCaching
public class CacheConfig { ... }

// 2. 方法级缓存
@Cacheable(value = "cluster", key = "#id")
public Cluster getById(Integer id) { ... }

// 3. 清除缓存
@CacheEvict(value = "cluster", key = "#id")
public void update(Integer id) { ... }

// 4. Redis 缓存
redisTemplate.opsForValue().set(key, value, 30, TimeUnit.MINUTES);
```

### 8.3 并发优化

```java
// 1. 使用线程池
@Async("taskExecutor")
public CompletableFuture<Result> process() { ... }

// 2. 使用并行流
list.parallelStream().forEach(item -> { ... });

// 3. 使用 CompletableFuture
CompletableFuture.supplyAsync(() -> { ... })
    .thenApply(result -> { ... })
    .join();
```

## 九、调试技巧

### 9.1 日志调试

```yaml
# 开启 SQL 日志
logging:
  level:
    com.datasophon: DEBUG
    org.mybatis: DEBUG

# MyBatis SQL 日志
mybatis:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

### 9.2 远程调试

```bash
# 启动参数添加
-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=5005

# IDEA 配置远程调试
Run > Edit Configurations > Remote JVM Debug
Host: localhost
Port: 5005
```

### 9.3 性能分析

```bash
# JVM 参数
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-Xloggc:/opt/datasophon/logs/gc.log

# 使用 jstack 查看线程
jstack <pid> > thread.dump

# 使用 jmap 查看堆内存
jmap -heap <pid>
jmap -dump:format=b,file=heap.bin <pid>
```

## 十、常用命令

### 10.1 应用管理

```bash
# 启动应用
java -jar datasophon-api.jar

# 后台启动
nohup java -jar datasophon-api.jar > /dev/null 2>&1 &

# 查看进程
ps aux | grep datasophon-api

# 停止应用
kill -15 <pid>

# 查看日志
tail -f /opt/datasophon/logs/api/datasophon-api.log
```

### 10.2 数据库管理

```bash
# 连接数据库
mysql -h localhost -u root -p datasophon

# 备份数据库
mysqldump -u root -p datasophon > backup.sql

# 恢复数据库
mysql -u root -p datasophon < backup.sql

# 查看表结构
DESCRIBE t_cluster_info;

# 查看索引
SHOW INDEX FROM t_cluster_info;
```

### 10.3 Redis 管理

```bash
# 连接 Redis
redis-cli -h localhost -p 6379

# 查看所有键
KEYS *

# 查看键值
GET session:token123

# 删除键
DEL session:token123

# 清空数据库
FLUSHDB
```

---

**文档版本**: v1.0  
**最后更新**: 2025-11-16  
**维护团队**: DataSophon 源码分析团队

**使用建议**: 建议将本手册保存为书签，以便快速查阅常用 API 和代码片段。
