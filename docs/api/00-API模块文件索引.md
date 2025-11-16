# DataSophon API 模块文件索引

## 一、文档概述

本索引提供 DataSophon API 模块的完整文档导航，帮助快速定位所需信息。

**文档版本**: v1.0  
**最后更新**: 2025-11-16  
**文档总数**: 12个详细分析文档  
**文档总字数**: 约18万字  

## 二、文档列表

### 2.1 核心文档 (7个)

| 序号 | 文档名称 | 字符数 | 核心内容 |
|-----|---------|-------|---------|
| 01 | [DataSophonApplicationServer分析](./01-DataSophonApplicationServer分析.md) | ~15,000 | 主启动类、Spring Boot 配置、组件初始化 |
| 02 | [Controller层分析](./02-Controller层分析.md) | ~15,000 | RESTful API 设计、MVC 模式、统一响应 |
| 03 | [集群管理Controller](./03-集群管理Controller.md) | ~15,000 | 集群CRUD、状态管理、ClusterInfoController |
| 04 | [服务管理Controller](./04-服务管理Controller.md) | ~15,000 | 服务安装/启停、配置管理、ServiceInstanceController |
| 05 | [主机管理Controller](./05-主机管理Controller.md) | ~15,000 | 主机管理、机架感知、ClusterHostController |
| 06 | [告警管理Controller](./06-告警管理Controller.md) | ~15,000 | 告警组、告警规则、多通知渠道 |
| 07 | [用户权限Controller](./07-用户权限Controller.md) | ~15,000 | 登录认证、用户管理、LoginController |

### 2.2 扩展文档 (5个)

| 序号 | 文档名称 | 字符数 | 核心内容 |
|-----|---------|-------|---------|
| 08 | [监控大盘Controller](./08-监控大盘Controller.md) | 26,908 | 监控数据聚合、Prometheus集成、实时指标查询 |
| 09 | [配置管理](./09-配置管理.md) | 23,225 | 应用配置、多环境、MyBatis、Akka配置 |
| 10 | [安全认证](./10-安全认证.md) | 33,730 | JWT Token、RBAC权限、会话管理、安全防护 |
| 11 | [拦截器](./11-拦截器.md) | 30,880 | 认证拦截、权限验证、日志记录、限流控制 |
| 12 | [异常处理](./12-异常处理.md) | 28,118 | 全局异常处理、自定义异常、参数验证 |

### 2.3 索引文档 (本文档)

| 序号 | 文档名称 | 核心内容 |
|-----|---------|---------|
| 00 | [API模块文件索引](./00-API模块文件索引.md) | 完整文档导航、快速定位 |

## 三、按功能分类

### 3.1 基础架构类

这类文档介绍 API 模块的基础架构和设计模式。

**文档列表**:
- [01-DataSophonApplicationServer分析](./01-DataSophonApplicationServer分析.md) - 应用启动
- [02-Controller层分析](./02-Controller层分析.md) - Controller 设计
- [09-配置管理](./09-配置管理.md) - 配置体系
- [11-拦截器](./11-拦截器.md) - 拦截器链
- [12-异常处理](./12-异常处理.md) - 异常处理

**适合人群**: 架构师、新手开发者

**学习路径**:
```
01 启动类分析 → 02 Controller层设计 → 09 配置管理 → 11 拦截器 → 12 异常处理
```

### 3.2 业务功能类

这类文档介绍具体的业务功能模块和 API 接口。

**文档列表**:
- [03-集群管理Controller](./03-集群管理Controller.md) - 集群管理
- [04-服务管理Controller](./04-服务管理Controller.md) - 服务管理
- [05-主机管理Controller](./05-主机管理Controller.md) - 主机管理
- [06-告警管理Controller](./06-告警管理Controller.md) - 告警管理
- [08-监控大盘Controller](./08-监控大盘Controller.md) - 监控大盘

**适合人群**: 业务开发者、产品经理

**学习路径**:
```
03 集群管理 → 05 主机管理 → 04 服务管理 → 08 监控大盘 → 06 告警管理
```

### 3.3 安全认证类

这类文档介绍安全认证和权限控制机制。

**文档列表**:
- [07-用户权限Controller](./07-用户权限Controller.md) - 用户管理
- [10-安全认证](./10-安全认证.md) - 认证授权
- [11-拦截器](./11-拦截器.md) - 安全拦截

**适合人群**: 安全工程师、后端开发者

**学习路径**:
```
07 用户权限 → 10 安全认证 → 11 拦截器（认证部分）
```

## 四、按角色分类

### 4.1 新手开发者

**推荐阅读顺序**:
1. [01-DataSophonApplicationServer分析](./01-DataSophonApplicationServer分析.md) - 了解应用启动
2. [02-Controller层分析](./02-Controller层分析.md) - 理解 Controller 设计
3. [09-配置管理](./09-配置管理.md) - 学习配置方式
4. [03-集群管理Controller](./03-集群管理Controller.md) - 学习业务接口
5. [12-异常处理](./12-异常处理.md) - 掌握异常处理

### 4.2 后端开发者

**推荐阅读顺序**:
1. [02-Controller层分析](./02-Controller层分析.md) - Controller 设计模式
2. [10-安全认证](./10-安全认证.md) - 认证授权机制
3. [11-拦截器](./11-拦截器.md) - 拦截器实现
4. [12-异常处理](./12-异常处理.md) - 异常处理机制
5. [08-监控大盘Controller](./08-监控大盘Controller.md) - 监控集成

### 4.3 架构师

**推荐阅读顺序**:
1. [01-DataSophonApplicationServer分析](./01-DataSophonApplicationServer分析.md) - 整体架构
2. [09-配置管理](./09-配置管理.md) - 配置体系
3. [10-安全认证](./10-安全认证.md) - 安全架构
4. [11-拦截器](./11-拦截器.md) - 拦截器架构
5. [08-监控大盘Controller](./08-监控大盘Controller.md) - 监控架构

### 4.4 运维人员

**推荐阅读顺序**:
1. [03-集群管理Controller](./03-集群管理Controller.md) - 集群操作
2. [05-主机管理Controller](./05-主机管理Controller.md) - 主机管理
3. [04-服务管理Controller](./04-服务管理Controller.md) - 服务运维
4. [08-监控大盘Controller](./08-监控大盘Controller.md) - 监控查看
5. [06-告警管理Controller](./06-告警管理Controller.md) - 告警配置

## 五、核心概念索引

### 5.1 Spring Boot

| 概念 | 相关文档 | 章节 |
|-----|---------|------|
| 主启动类 | 01-DataSophonApplicationServer分析 | 一、二 |
| 配置文件 | 09-配置管理 | 二 |
| 自动配置 | 01-DataSophonApplicationServer分析 | 三 |
| Bean 管理 | 01-DataSophonApplicationServer分析 | 四 |

### 5.2 Spring MVC

| 概念 | 相关文档 | 章节 |
|-----|---------|------|
| Controller | 02-Controller层分析 | 全部 |
| @RestController | 02-Controller层分析 | 二 |
| @RequestMapping | 02-Controller层分析 | 二 |
| 参数绑定 | 12-异常处理 | 四 |
| 拦截器 | 11-拦截器 | 全部 |

### 5.3 安全认证

| 概念 | 相关文档 | 章节 |
|-----|---------|------|
| JWT Token | 10-安全认证 | 二.2 |
| Session 管理 | 10-安全认证 | 二.2.2 |
| RBAC 权限 | 10-安全认证 | 三.1 |
| 权限注解 | 10-安全认证 | 三.2 |
| 密码加密 | 10-安全认证 | 二.1.2 |

### 5.4 监控告警

| 概念 | 相关文档 | 章节 |
|-----|---------|------|
| Prometheus | 08-监控大盘Controller | 二.2 |
| 监控指标 | 08-监控大盘Controller | 六 |
| 告警规则 | 06-告警管理Controller | 全部 |
| 告警通知 | 06-告警管理Controller | 三 |

### 5.5 异常处理

| 概念 | 相关文档 | 章节 |
|-----|---------|------|
| 自定义异常 | 12-异常处理 | 二 |
| 全局异常处理器 | 12-异常处理 | 三 |
| 参数验证 | 12-异常处理 | 四 |
| 异常日志 | 12-异常处理 | 五 |

## 六、API 接口索引

### 6.1 集群管理 API

**文档**: [03-集群管理Controller](./03-集群管理Controller.md)

| API 路径 | 方法 | 功能 | 权限 |
|---------|------|------|------|
| /api/cluster/list | GET | 获取集群列表 | - |
| /api/cluster/save | POST | 创建集群 | cluster:create |
| /api/cluster/update | POST | 更新集群 | cluster:update |
| /api/cluster/delete | POST | 删除集群 | cluster:delete |
| /api/cluster/info/{id} | GET | 获取集群详情 | - |

### 6.2 服务管理 API

**文档**: [04-服务管理Controller](./04-服务管理Controller.md)

| API 路径 | 方法 | 功能 | 权限 |
|---------|------|------|------|
| /api/service/list | GET | 获取服务列表 | - |
| /api/service/install | POST | 安装服务 | service:install |
| /api/service/start | POST | 启动服务 | service:start |
| /api/service/stop | POST | 停止服务 | service:stop |
| /api/service/restart | POST | 重启服务 | service:restart |

### 6.3 主机管理 API

**文档**: [05-主机管理Controller](./05-主机管理Controller.md)

| API 路径 | 方法 | 功能 | 权限 |
|---------|------|------|------|
| /api/host/list | GET | 获取主机列表 | - |
| /api/host/add | POST | 添加主机 | host:add |
| /api/host/delete | POST | 删除主机 | host:delete |
| /api/host/info/{id} | GET | 获取主机详情 | - |

### 6.4 告警管理 API

**文档**: [06-告警管理Controller](./06-告警管理Controller.md)

| API 路径 | 方法 | 功能 | 权限 |
|---------|------|------|------|
| /api/alert/group/list | GET | 获取告警组列表 | - |
| /api/alert/group/save | POST | 创建告警组 | alert:create |
| /api/alert/history/list | GET | 获取告警历史 | - |

### 6.5 监控大盘 API

**文档**: [08-监控大盘Controller](./08-监控大盘Controller.md)

| API 路径 | 方法 | 功能 | 权限 |
|---------|------|------|------|
| /api/dashboard/clusterOverview | GET | 获取集群概览 | - |
| /api/dashboard/serviceMetrics | GET | 获取服务指标 | - |
| /api/dashboard/activeAlerts | GET | 获取活跃告警 | - |

### 6.6 用户权限 API

**文档**: [07-用户权限Controller](./07-用户权限Controller.md)

| API 路径 | 方法 | 功能 | 权限 |
|---------|------|------|------|
| /api/login/signin | POST | 用户登录 | - |
| /api/login/signout | POST | 用户登出 | - |
| /api/user/list | GET | 获取用户列表 | user:view |
| /api/user/save | POST | 创建用户 | user:create |

## 七、代码示例索引

### 7.1 Controller 示例

**基础 Controller**:
- 文档: [02-Controller层分析](./02-Controller层分析.md)
- 章节: 二、三
- 示例: CRUD 操作、参数绑定、响应封装

**监控 Controller**:
- 文档: [08-监控大盘Controller](./08-监控大盘Controller.md)
- 章节: 二
- 示例: 数据聚合、异步查询、缓存策略

### 7.2 配置示例

**Spring Boot 配置**:
- 文档: [09-配置管理](./09-配置管理.md)
- 章节: 二、三
- 示例: application.yml、多环境配置、MyBatis 配置

**Akka 配置**:
- 文档: [09-配置管理](./09-配置管理.md)
- 章节: 二.4
- 示例: Actor 系统配置、远程通信配置

### 7.3 安全示例

**认证实现**:
- 文档: [10-安全认证](./10-安全认证.md)
- 章节: 二
- 示例: 登录认证、Token 生成、Session 管理

**权限验证**:
- 文档: [10-安全认证](./10-安全认证.md)
- 章节: 三
- 示例: RBAC 实现、权限注解、权限验证

### 7.4 拦截器示例

**认证拦截器**:
- 文档: [11-拦截器](./11-拦截器.md)
- 章节: 二.3
- 示例: Token 验证、Session 检查

**日志拦截器**:
- 文档: [11-拦截器](./11-拦截器.md)
- 章节: 二.2
- 示例: 请求日志、响应日志、性能监控

### 7.5 异常处理示例

**自定义异常**:
- 文档: [12-异常处理](./12-异常处理.md)
- 章节: 二
- 示例: ApiException、BusinessException

**全局异常处理器**:
- 文档: [12-异常处理](./12-异常处理.md)
- 章节: 三
- 示例: @RestControllerAdvice、异常转换

## 八、快速导航

### 8.1 常见任务

| 任务 | 相关文档 | 关键章节 |
|-----|---------|---------|
| 创建新 Controller | 02-Controller层分析 | 二、三 |
| 添加新 API 接口 | 02-Controller层分析 | 四 |
| 配置数据库 | 09-配置管理 | 二.1.2 |
| 实现用户登录 | 10-安全认证 | 二.1 |
| 添加权限验证 | 10-安全认证 | 三.2 |
| 添加拦截器 | 11-拦截器 | 二 |
| 处理异常 | 12-异常处理 | 二、三 |
| 集成监控 | 08-监控大盘Controller | 二.2 |

### 8.2 常见问题

| 问题 | 相关文档 | 解决方案位置 |
|-----|---------|------------|
| 如何配置多环境? | 09-配置管理 | 三 |
| Token 如何生成? | 10-安全认证 | 二.2.1 |
| 权限如何验证? | 10-安全认证 | 三.3 |
| 拦截器顺序如何配置? | 11-拦截器 | 一.2 |
| 如何自定义异常? | 12-异常处理 | 二 |
| 如何记录审计日志? | 10-安全认证 | 五 |

## 九、技术亮点总结

### 9.1 架构设计

1. **分层架构**: Controller → Service → DAO 清晰分层
2. **RESTful API**: 遵循 REST 架构风格
3. **统一响应**: Result 对象封装统一响应格式
4. **拦截器链**: 多层拦截器协同工作
5. **全局异常处理**: 统一异常处理机制

### 9.2 安全机制

1. **JWT Token**: 无状态认证
2. **RBAC 权限**: 基于角色的访问控制
3. **会话管理**: Redis 分布式会话
4. **XSS 防护**: 请求参数过滤
5. **CSRF 防护**: Token 验证

### 9.3 性能优化

1. **缓存策略**: 多级缓存（Redis、本地缓存）
2. **异步处理**: @Async 异步执行
3. **连接池**: HikariCP 高性能连接池
4. **数据降采样**: 长时间范围查询优化
5. **请求限流**: Redis 实现限流

### 9.4 监控运维

1. **Prometheus 集成**: 指标采集
2. **日志记录**: 完整的请求日志
3. **异常日志**: 异常信息详细记录
4. **审计日志**: 操作审计追踪
5. **健康检查**: Actuator 端点

## 十、未来规划

### 10.1 待完善文档

以下文档可根据实际需要补充：

1. **数据库迁移**: Flyway/Liquibase 使用
2. **服务元数据配置**: 各组件配置文件分析
3. **API 测试**: 单元测试、集成测试
4. **性能测试**: 压力测试、性能调优

### 10.2 文档维护

- **定期更新**: 随代码变更及时更新文档
- **版本管理**: 文档版本与代码版本对应
- **反馈收集**: 收集使用反馈，持续改进
- **示例补充**: 增加更多实际使用示例

---

**文档版本**: v1.0  
**最后更新**: 2025-11-16  
**维护团队**: DataSophon 源码分析团队
