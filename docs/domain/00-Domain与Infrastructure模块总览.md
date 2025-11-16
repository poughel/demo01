# DataSophon Domain 与 Infrastructure 模块总览

## 一、模块简介

DataSophon 的 Domain 和 Infrastructure 模块负责数据模型定义和持久化操作，包含 **39 个实体类**、**38 个 Mapper 接口**和完善的数据库访问层。

### 核心实体分类

**1. 集群管理**: ClusterInfoEntity、ClusterHostDO、ClusterRack
**2. 服务管理**: ClusterServiceInstanceEntity、ClusterServiceRoleInstanceEntity
**3. 配置管理**: ClusterServiceRoleGroupConfig、ClusterVariable
**4. 命令执行**: ClusterServiceCommandEntity、ClusterServiceCommandHostEntity
**5. 告警管理**: AlertGroupEntity、ClusterAlertRule、ClusterAlertHistory、ClusterAlertQuota
**6. 用户权限**: UserInfoEntity、ClusterUser、SessionEntity
**7. 框架元数据**: FrameInfoEntity、FrameServiceEntity、FrameServiceRoleEntity

### 设计特点

- ✅ 使用 MyBatis-Plus 简化 CRUD 操作
- ✅ 完善的枚举类型管理状态和类型
- ✅ 清晰的实体关联关系
- ✅ 统一的命名规范和字段定义
- ✅ 支持分页查询和批量操作
- ✅ 完整的审计字段（创建时间、更新时间）

## 二、详细文档索引

### Domain 模块文档

- [01-告警实体类详解](./01-告警实体类详解.md) - **NEW** 完整的告警相关实体类分析
  - AlertGroupEntity - 告警组实体
  - ClusterAlertQuota - 告警指标实体
  - ClusterAlertRule - 告警规则实体
  - ClusterAlertHistory - 告警历史实体
  - ClusterAlertGroupMap - 集群告警组映射实体

### Infrastructure 模块文档

- [01-告警数据访问层详解](../infrastructure/01-告警数据访问层详解.md) - **NEW** Mapper 层完整分析
  - AlertGroupMapper - 告警组数据访问
  - ClusterAlertQuotaMapper - 告警指标数据访问
  - ClusterAlertRuleMapper - 告警规则数据访问
  - ClusterAlertHistoryMapper - 告警历史数据访问
  - ClusterAlertGroupMapMapper - 映射关系数据访问

---

**文档版本**: v1.1  
**创建日期**: 2025-11-15  
**最后更新**: 2025-11-16  
**维护者**: DataSophon 源码分析团队
