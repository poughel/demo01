# DataSophon Domain 与 Infrastructure 模块总览

## 一、模块简介

DataSophon 的 Domain 和 Infrastructure 模块负责数据模型定义和持久化操作，包含 **39 个实体类**、**38 个 Mapper 接口**和完善的数据库访问层。

### 核心实体分类

**1. 集群管理**: ClusterInfoEntity、ClusterHostDO、ClusterRack
**2. 服务管理**: ClusterServiceInstanceEntity、ClusterServiceRoleInstanceEntity
**3. 配置管理**: ClusterServiceRoleGroupConfig、ClusterVariable
**4. 命令执行**: ClusterServiceCommandEntity、ClusterServiceCommandHostEntity
**5. 告警管理**: AlertGroupEntity、ClusterAlertRule、ClusterAlertHistory
**6. 用户权限**: UserInfoEntity、ClusterUser、SessionEntity
**7. 框架元数据**: FrameInfoEntity、FrameServiceEntity、FrameServiceRoleEntity

### 设计特点

- ✅ 使用 MyBatis-Plus 简化 CRUD 操作
- ✅ 完善的枚举类型管理状态和类型
- ✅ 清晰的实体关联关系
- ✅ 统一的命名规范和字段定义
- ✅ 支持分页查询和批量操作
- ✅ 完整的审计字段（创建时间、更新时间）

---

**文档版本**: v1.0  
**创建日期**: 2025-11-15  
**维护者**: DataSophon 源码分析团队
