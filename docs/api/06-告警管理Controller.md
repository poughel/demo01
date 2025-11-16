# AlertGroupController 告警组管理控制器详解

## 一、控制器概述

### 1.1 基本信息

**文件位置**: `datasophon-api/src/main/java/com/datasophon/api/controller/AlertGroupController.java`

**职责定位**:
- 管理告警组配置
- 告警组的 CRUD 操作
- 告警组与告警指标的关联管理
- 告警通知的分组管理

**核心概念**:
- **告警组**: 用于组织和管理告警规则的逻辑分组
- **告警指标**: 具体的监控指标阈值配置
- **告警通知**: 告警触发后的通知方式和接收人

### 1.2 类结构

```java
@RestController
@RequestMapping("alert/group")
public class AlertGroupController {
    @Autowired
    private AlertGroupService alertGroupService;
    
    @Autowired
    private ClusterAlertQuotaService alertQuotaService;
}
```

**关键特征**:
- **请求前缀**: `/alert/group`
- **双依赖注入**: AlertGroupService 和 ClusterAlertQuotaService
- **关联验证**: 删除前检查告警指标绑定

## 二、API 接口详解

### 2.1 查询告警组列表

#### 接口定义
```java
@RequestMapping("/list")
public Result list(Integer clusterId, String alertGroupName, 
                   Integer page, Integer pageSize) {
    return alertGroupService.getAlertGroupList(
        clusterId, alertGroupName, page, pageSize
    );
}
```

**接口信息**:
- **URL**: `GET /alert/group/list`
- **请求参数**: 
  - `clusterId`: 集群ID
  - `alertGroupName`: 告警组名称（模糊查询）
  - `page`: 页码
  - `pageSize`: 每页大小
- **返回值**: Result 包装的分页数据
- **权限要求**: 登录用户

**功能描述**:
- 分页查询告警组列表
- 支持按集群ID过滤
- 支持按告警组名称模糊搜索
- 返回告警组基本信息和通知配置

**使用场景**:
1. 告警配置页面的告警组列表
2. 告警组选择下拉框
3. 告警组管理和维护

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "total": 10,
    "pageSize": 10,
    "currentPage": 1,
    "list": [
      {
        "id": 1,
        "alertGroupName": "核心服务告警组",
        "alertGroupCategory": "SERVICE",
        "clusterId": 1,
        "alertLevel": "WARNING",
        "noticeUserIds": "1,2,3",
        "noticeGroupUserIds": "1,2",
        "noticeWay": "EMAIL,DINGDING",
        "createTime": "2025-01-01 10:00:00"
      }
    ]
  }
}
```

### 2.2 获取告警组详情

#### 接口定义
```java
@RequestMapping("/info/{id}")
public Result info(@PathVariable("id") Integer id) {
    AlertGroupEntity alertGroup = alertGroupService.getById(id);
    return Result.success().put("alertGroup", alertGroup);
}
```

**接口信息**:
- **URL**: `GET /alert/group/info/{id}`
- **请求参数**: 
  - `id`: 告警组ID（路径参数）
- **返回值**: Result 包装的告警组详情
- **权限要求**: 登录用户

**功能描述**:
- 查询告警组的完整配置信息
- 包含通知人员、通知方式等详细配置
- 用于编辑时回显数据

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "success",
  "alertGroup": {
    "id": 1,
    "alertGroupName": "核心服务告警组",
    "alertGroupCategory": "SERVICE",
    "clusterId": 1,
    "alertLevel": "WARNING",
    "noticeUserIds": "1,2,3",
    "noticeGroupUserIds": "1,2",
    "noticeWay": "EMAIL,DINGDING",
    "emailConfig": {
      "host": "smtp.example.com",
      "port": 465,
      "ssl": true
    },
    "dingdingConfig": {
      "webhook": "https://oapi.dingtalk.com/robot/send?access_token=xxx",
      "atMobiles": "13800138000"
    },
    "createTime": "2025-01-01 10:00:00",
    "updateTime": "2025-01-10 15:30:00"
  }
}
```

**告警级别枚举**:
```java
public enum AlertLevel {
    INFO("INFO", "提示"),
    WARNING("WARNING", "警告"),
    ERROR("ERROR", "错误"),
    CRITICAL("CRITICAL", "严重");
}
```

**通知方式枚举**:
```java
public enum NoticeWay {
    EMAIL("EMAIL", "邮件"),
    DINGDING("DINGDING", "钉钉"),
    WECHAT("WECHAT", "企业微信"),
    SMS("SMS", "短信"),
    WEBHOOK("WEBHOOK", "自定义Webhook");
}
```

### 2.3 保存告警组

#### 接口定义
```java
@RequestMapping("/save")
public Result save(@RequestBody AlertGroupEntity alertGroup) {
    alertGroup.setCreateTime(new Date());
    return alertGroupService.saveAlertGroup(alertGroup);
}
```

**接口信息**:
- **URL**: `POST /alert/group/save`
- **请求参数**: 
  - `alertGroup`: 告警组对象（JSON Body）
- **返回值**: Result 表示操作结果
- **权限要求**: 登录用户（应该添加 @UserPermission）

**功能描述**:
- 创建新的告警组
- 配置告警通知人员
- 配置告警通知方式
- 自动设置创建时间

**使用场景**:
1. 新建告警组
2. 配置告警策略
3. 设置通知规则

**请求数据示例**:
```json
{
  "alertGroupName": "数据库告警组",
  "alertGroupCategory": "DATABASE",
  "clusterId": 1,
  "alertLevel": "ERROR",
  "noticeUserIds": "1,2,3",
  "noticeGroupUserIds": "1",
  "noticeWay": "EMAIL,DINGDING",
  "emailConfig": {
    "host": "smtp.example.com",
    "port": 465,
    "ssl": true,
    "from": "alert@example.com"
  },
  "dingdingConfig": {
    "webhook": "https://oapi.dingtalk.com/robot/send?access_token=xxx"
  }
}
```

**业务逻辑**:
1. 参数验证
   - 告警组名称不能为空
   - 告警组名称不能重复
   - 至少配置一种通知方式
   - 至少指定一个通知人员
2. 配置验证
   - 邮件配置有效性
   - 钉钉webhook有效性
   - 用户ID存在性
3. 保存到数据库
4. 返回操作结果

### 2.4 更新告警组

#### 接口定义
```java
@RequestMapping("/update")
public Result update(@RequestBody AlertGroupEntity alertGroup) {
    alertGroupService.updateById(alertGroup);
    return Result.success();
}
```

**接口信息**:
- **URL**: `POST /alert/group/update`
- **请求参数**: 
  - `alertGroup`: 告警组对象（JSON Body）
- **返回值**: Result 表示操作结果
- **权限要求**: 登录用户（应该添加 @UserPermission）

**功能描述**:
- 修改告警组配置
- 更新通知人员
- 更新通知方式
- 修改告警级别

**可修改字段**:
- 告警组名称
- 告警级别
- 通知人员
- 通知方式
- 通知配置

**不可修改字段**:
- 告警组ID
- 集群ID
- 创建时间

### 2.5 删除告警组

#### 接口定义
```java
@RequestMapping("/delete")
public Result delete(@RequestBody Integer[] ids) {
    // 校验是否绑定告警指标
    List<ClusterAlertQuota> list = alertQuotaService.lambdaQuery()
        .in(ClusterAlertQuota::getAlertGroupId, ids)
        .list();
    if (list.size() > 0) {
        return Result.error(Status.ALERT_GROUP_TIPS_ONE.getMsg());
    }
    alertGroupService.removeByIds(Arrays.asList(ids));
    return Result.success();
}
```

**接口信息**:
- **URL**: `POST /alert/group/delete`
- **请求参数**: 
  - `ids`: 告警组ID数组（JSON Body）
- **返回值**: Result 表示操作结果
- **权限要求**: 登录用户（应该添加 @UserPermission）

**功能描述**:
- 批量删除告警组
- 删除前检查关联的告警指标
- 防止删除正在使用的告警组

**使用场景**:
1. 清理不再使用的告警组
2. 批量删除测试告警组

**删除前检查**:
1. 检查是否有告警指标绑定到该告警组
2. 如果有绑定，返回错误信息
3. 如果无绑定，执行删除

**错误处理**:
```java
if (list.size() > 0) {
    return Result.error("该告警组已绑定告警指标，无法删除！");
}
```

**级联删除**:
- 删除告警组与用户的关联
- 删除告警组与用户组的关联
- 清理缓存数据

## 三、告警组数据模型

### 3.1 AlertGroupEntity

```java
public class AlertGroupEntity {
    private Integer id;                    // 主键ID
    private String alertGroupName;         // 告警组名称
    private String alertGroupCategory;     // 告警组类别
    private Integer clusterId;             // 集群ID
    private String alertLevel;             // 告警级别
    private String noticeUserIds;          // 通知用户ID列表
    private String noticeGroupUserIds;     // 通知用户组ID列表
    private String noticeWay;              // 通知方式
    private String emailConfig;            // 邮件配置(JSON)
    private String dingdingConfig;         // 钉钉配置(JSON)
    private String wechatConfig;           // 企业微信配置(JSON)
    private String webhookConfig;          // Webhook配置(JSON)
    private Date createTime;               // 创建时间
    private Date updateTime;               // 更新时间
}
```

### 3.2 告警组类别

```java
public enum AlertGroupCategory {
    SERVICE("SERVICE", "服务告警"),
    HOST("HOST", "主机告警"),
    DATABASE("DATABASE", "数据库告警"),
    NETWORK("NETWORK", "网络告警"),
    CUSTOM("CUSTOM", "自定义告警");
}
```

## 四、告警通知机制

### 4.1 邮件通知

**配置示例**:
```json
{
  "host": "smtp.example.com",
  "port": 465,
  "ssl": true,
  "username": "alert@example.com",
  "password": "xxx",
  "from": "alert@example.com",
  "to": "admin@example.com",
  "subject": "[DataSophon] 告警通知"
}
```

**邮件模板**:
```html
<h2>告警通知</h2>
<p><strong>集群:</strong> bigdata-cluster</p>
<p><strong>服务:</strong> HDFS</p>
<p><strong>告警级别:</strong> WARNING</p>
<p><strong>告警内容:</strong> NameNode堆内存使用率超过80%</p>
<p><strong>触发时间:</strong> 2025-01-15 10:00:00</p>
```

### 4.2 钉钉通知

**配置示例**:
```json
{
  "webhook": "https://oapi.dingtalk.com/robot/send?access_token=xxx",
  "secret": "xxx",
  "atMobiles": "13800138000,13900139000",
  "isAtAll": false
}
```

**消息格式**:
```json
{
  "msgtype": "markdown",
  "markdown": {
    "title": "DataSophon 告警通知",
    "text": "## 告警通知\n**集群:** bigdata-cluster\n**服务:** HDFS\n**告警级别:** WARNING\n**告警内容:** NameNode堆内存使用率超过80%\n**触发时间:** 2025-01-15 10:00:00"
  },
  "at": {
    "atMobiles": ["13800138000"],
    "isAtAll": false
  }
}
```

### 4.3 企业微信通知

**配置示例**:
```json
{
  "webhook": "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx",
  "mentionedList": ["user1", "user2"]
}
```

### 4.4 自定义Webhook

**配置示例**:
```json
{
  "url": "https://your-api.com/alert",
  "method": "POST",
  "headers": {
    "Content-Type": "application/json",
    "Authorization": "Bearer xxx"
  },
  "bodyTemplate": {
    "cluster": "${cluster}",
    "service": "${service}",
    "level": "${level}",
    "message": "${message}",
    "time": "${time}"
  }
}
```

## 五、告警组与告警指标关联

### 5.1 关联关系

```
告警组 (AlertGroup)
  ├─ 核心服务告警组
  │   ├─ HDFS NameNode 内存告警
  │   ├─ HDFS DataNode 磁盘告警
  │   └─ YARN ResourceManager CPU告警
  └─ 数据库告警组
      ├─ MySQL 连接数告警
      └─ MySQL 慢查询告警
```

### 5.2 删除约束

**检查逻辑**:
```java
List<ClusterAlertQuota> list = alertQuotaService.lambdaQuery()
    .in(ClusterAlertQuota::getAlertGroupId, ids)
    .list();
if (list.size() > 0) {
    return Result.error("该告警组已绑定告警指标，无法删除！");
}
```

**解除绑定流程**:
1. 查找绑定的告警指标
2. 修改告警指标，解除与告警组的关联
3. 重新删除告警组

## 六、设计模式与架构

### 6.1 告警通知策略模式

```java
public interface NotificationStrategy {
    void send(AlertMessage message);
}

public class EmailNotificationStrategy implements NotificationStrategy {
    @Override
    public void send(AlertMessage message) {
        // 发送邮件
    }
}

public class DingDingNotificationStrategy implements NotificationStrategy {
    @Override
    public void send(AlertMessage message) {
        // 发送钉钉消息
    }
}
```

### 6.2 告警分组策略

**按服务分组**:
- HDFS 告警组
- YARN 告警组
- Kafka 告警组

**按级别分组**:
- 严重告警组（CRITICAL）
- 错误告警组（ERROR）
- 警告告警组（WARNING）

**按业务分组**:
- 核心业务告警组
- 数据仓库告警组
- 实时计算告警组

## 七、性能优化

### 7.1 缓存告警组配置

```java
@Cacheable(value = "alert:group", key = "#id")
public AlertGroupEntity getById(Integer id) {
    // ...
}

@CacheEvict(value = "alert:group:*", allEntries = true)
public void update(AlertGroupEntity alertGroup) {
    // ...
}
```

### 7.2 批量通知优化

```java
// 批量发送通知，减少网络请求
public void batchNotify(List<AlertMessage> messages) {
    // 按通知方式分组
    Map<String, List<AlertMessage>> groupedMessages = 
        messages.stream()
            .collect(Collectors.groupingBy(AlertMessage::getNoticeWay));
    
    // 并发发送
    groupedMessages.forEach((way, msgs) -> {
        executor.submit(() -> sendNotifications(way, msgs));
    });
}
```

## 八、使用示例

### 8.1 前端调用示例

**创建告警组**:
```javascript
const alertGroupData = {
  alertGroupName: '核心服务告警组',
  alertGroupCategory: 'SERVICE',
  clusterId: 1,
  alertLevel: 'WARNING',
  noticeUserIds: '1,2,3',
  noticeWay: 'EMAIL,DINGDING',
  emailConfig: {
    host: 'smtp.example.com',
    port: 465,
    ssl: true
  }
};

axios.post('/alert/group/save', alertGroupData)
  .then(response => {
    console.log('告警组创建成功');
  });
```

**删除告警组**:
```javascript
axios.post('/alert/group/delete', [1, 2, 3])
  .then(response => {
    if (response.data.code === 0) {
      console.log('删除成功');
    } else {
      console.error('删除失败:', response.data.msg);
    }
  });
```

## 九、总结

### 9.1 核心功能

AlertGroupController 提供告警组管理功能：
1. 告警组的 CRUD 操作
2. 多种通知方式配置
3. 删除前关联检查
4. 批量操作支持

### 9.2 设计特点

- **多通知方式**: 支持邮件、钉钉、企业微信等
- **关联检查**: 防止误删除正在使用的告警组
- **灵活配置**: 支持详细的通知参数配置
- **批量操作**: 提高管理效率

### 9.3 改进建议

1. 添加权限控制注解
2. 完善参数验证
3. 增加通知测试接口
4. 优化批量通知性能
5. 添加告警静默功能
6. 支持告警组继承
7. 增加通知日志记录

---

**文档版本**: v1.0  
**最后更新**: 2025-11-15  
**维护者**: DataSophon 源码分析团队
