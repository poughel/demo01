# DataSophon Common 模块 - Constants 常量定义详解

## 一、文件概述

**文件路径**: `datasophon-common/src/main/java/com/datasophon/common/Constants.java`

**功能定位**: 定义系统全局使用的常量，为整个 DataSophon 系统提供统一的常量值引用

**设计特点**:
- 使用 `final` 关键字，确保类不可被继承
- 私有构造函数，防止实例化
- 所有常量均为 `public static final`，全局可访问
- 从配置文件读取部分动态配置值

## 二、常量分类详解

### 2.1 路径相关常量

#### 安装路径常量
```java
// 从配置文件读取安装路径，支持自定义部署路径
public static final String INSTALL_PATH = PropertyUtils.getString("install.path");

// Master 节点包管理路径
public static final String MASTER_MANAGE_PACKAGE_PATH = INSTALL_PATH + "/DDP/packages";

// Worker 节点路径
public static final String WORKER_PATH = INSTALL_PATH + "/datasophon-worker";
public static final String WORKER_SCRIPT_PATH = INSTALL_PATH + "/datasophon-worker/script/";
```

**使用场景**:
- 服务包安装位置确定
- 配置文件路径拼接
- 脚本执行路径定位
- 软件包上传下载路径

**设计优势**:
- 支持自定义安装路径，提高灵活性
- 统一管理路径，便于维护
- 避免硬编码，提升可配置性

#### 配置和元数据路径
```java
public static final String META_PATH = "meta";              // 元数据路径
public static final String SHELL_SCRIPT_PATH = "/scripts";  // Shell脚本路径
public static final String GRAFANA_PATH = "/grafana";       // Grafana路径
```

### 2.2 命令和脚本常量

#### Worker 部署命令
```java
// Worker包名称
public static final String WORKER_PACKAGE_NAME = "datasophon-worker.tar.gz";

// 解压Worker包命令
public static final String UNZIP_DDH_WORKER_CMD = 
    "tar -zxvf " + INSTALL_PATH + "/datasophon-worker.tar.gz -C " + INSTALL_PATH;

// 启动Worker服务命令
public static final String START_DDH_WORKER_CMD = "service datasophon-worker restart";

// 检查Worker包MD5值命令
public static final String CHECK_WORKER_MD5_CMD = 
    "md5sum " + INSTALL_PATH + "/datasophon-worker.tar.gz | awk '{print $1}'";

// 更新通用配置命令
public static final String UPDATE_COMMON_CMD = 
    "sh " + INSTALL_PATH + "/datasophon-worker/script/sed_common.sh ";

// Worker分发脚本
public static final String DISPATCHER_WORK = "dispatcher-worker.sh";
```

**命令设计思路**:
1. **解压命令**: 使用 tar 命令解压 Worker 包到指定目录
2. **启动命令**: 通过 service 命令管理 Worker 服务生命周期
3. **MD5校验**: 确保传输过程中文件完整性
4. **配置更新**: 通过 sed 脚本批量更新配置

**实际应用场景**:
```java
// 在部署新节点时使用
public void deployWorker(String hostname) {
    // 1. 上传Worker包
    uploadFile(hostname, WORKER_PACKAGE_NAME);
    
    // 2. 校验MD5
    String md5 = execCommand(hostname, CHECK_WORKER_MD5_CMD);
    
    // 3. 解压安装
    execCommand(hostname, UNZIP_DDH_WORKER_CMD);
    
    // 4. 启动服务
    execCommand(hostname, START_DDH_WORKER_CMD);
}
```

### 2.3 JDK 版本常量

```java
// x86_64架构JDK包
public static final String X86JDK = "jdk-8u333-linux-x64.tar.gz";

// aarch64架构JDK包 (ARM架构支持)
public static final String ARMJDK = "jdk-8u333-linux-aarch64.tar.gz";

// CPU架构标识
public static final String X86_64 = "x86_64";
public static final String CPU_ARCHITECTURE = "cpu_architecture";
```

**多架构支持设计**:
```java
// 根据主机架构选择合适的JDK
public String selectJDK(String architecture) {
    if (X86_64.equals(architecture)) {
        return X86JDK;
    } else {
        return ARMJDK;  // ARM架构
    }
}
```

**国产化适配**: 支持信创环境，兼容ARM服务器（如华为鲲鹏、飞腾等）

### 2.4 集群和服务管理常量

#### 标识符常量
```java
public static final String CLUSTER_ID = "cluster_id";                   // 集群ID
public static final String CLUSTER_CODE = "cluster_code";               // 集群编码
public static final String CLUSTER_STATE = "cluster_state";             // 集群状态
public static final String CLUSTER_FRAME = "cluster_frame";             // 集群框架
public static final String SERVICE_NAME = "service_name";               // 服务名称
public static final String SERVICE_INSTANCE_ID = "service_instance_id"; // 服务实例ID
public static final String SERVICE_STATE = "service_state";             // 服务状态
public static final String SERVICE_CATEGORY = "service_category";       // 服务分类
```

#### 服务角色常量
```java
public static final String SERVICE_ROLE_NAME = "service_role_name";               // 角色名称
public static final String SERVICE_ROLE_STATE = "service_role_state";             // 角色状态
public static final String SERVICE_ROLE_INSTANCE_ID = "service_role_instance_id"; // 角色实例ID
public static final String SERVICE_ROLE_HOST_MAPPING = "service_role_host_mapping"; // 角色主机映射
public static final String HOST_SERVICE_ROLE_MAPPING = "host_service_role_mapping"; // 主机角色映射
public static final String SERVICE_ROLE_JMX_MAP = "service_role_jmx_port";         // JMX端口映射
```

**映射关系说明**:
- `SERVICE_ROLE_HOST_MAPPING`: 服务角色 -> 主机列表 (一对多)
- `HOST_SERVICE_ROLE_MAPPING`: 主机 -> 服务角色列表 (一对多)

**数据结构示例**:
```json
{
  "service_role_host_mapping": {
    "NameNode": ["host1", "host2"],
    "DataNode": ["host1", "host2", "host3"]
  },
  "host_service_role_mapping": {
    "host1": ["NameNode", "DataNode"],
    "host2": ["NameNode", "DataNode"],
    "host3": ["DataNode"]
  }
}
```

### 2.5 主机管理常量

```java
public static final String HOSTNAME = "hostname";           // 主机名
public static final String HOST_MAP = "_host_map";          // 主机映射
public static final String HOST_MD5 = "_host_md5";          // 主机MD5值
public static final String HOST_STATE = "host_state";       // 主机状态
public static final String HOST_COMMAND_ID = "host_command_id"; // 主机命令ID
public static final String COMMAND_HOST_ID = "command_host_id"; // 命令主机ID
public static final String RACK = "rack";                   // 机架信息
public static final String NODE_LABEL = "node_label";       // 节点标签
```

**机架感知支持**: 
- `RACK` 常量用于实现 Hadoop 的机架感知功能
- 优化数据存储和任务调度
- 提高集群容错能力

### 2.6 命令执行常量

```java
public static final String COMMAND_TYPE = "command_type";       // 命令类型
public static final String COMMAND_STATE = "command_state";     // 命令状态
public static final String START_DISTRIBUTE_AGENT = "start_distribute_agent"; // 启动分发代理
```

**命令状态流转**:
```
等待执行 -> 执行中 -> 成功/失败
```

### 2.7 配置管理常量

```java
public static final String CONFIG = "_config";                  // 配置
public static final String CONFIG_FILE = "_config_file";        // 配置文件
public static final String CONFIG_VERSION = "config_version";   // 配置版本
public static final String NEET_RESTART = "need_restart";       // 需要重启标识
```

**配置版本控制**:
- 每次配置修改增加版本号
- 支持配置回滚
- 追踪配置变更历史

### 2.8 角色组管理常量

```java
public static final String ROLE_GROUP_ID = "role_group_id";         // 角色组ID
public static final String ROLE_GROUP_NAME = "role_group_name";     // 角色组名称
public static final String ROLE_GROUP_TYPE = "role_group_type";     // 角色组类型
public static final String ROLE_TYPE = "role_type";                 // 角色类型
```

**角色组功能**: 
- 将相同角色的多个实例分组管理
- 支持分组配置差异化
- 便于批量操作

### 2.9 用户和权限常量

```java
public static final String USERNAME = "username";               // 用户名
public static final String PASSWORD = "password";               // 密码
public static final String SESSION_USER = "session.user";       // 会话用户
public static final String SESSION_ID = "sessionId";            // 会话ID
public static final int SESSION_TIME_OUT = 7200;                // 会话超时(2小时)
public static final String USER_INFO = "userInfo";              // 用户信息
public static final String DETAILS_USER_ID = "user_id";         // 用户ID
public static final String ROOT = "root";                       // root用户
```

**会话管理**:
- 默认2小时超时
- 超时后需要重新登录
- 支持记住登录状态

#### Unix用户组管理
```java
public static final String USER_ID = "user_id";                 // 用户ID
public static final String GROUP_ID = "group_id";               // 组ID
public static final String GROUP_NAME = "group_name";           // 组名称
```

### 2.10 SSH连接常量

```java
// SSH密钥文件路径，支持自定义配置
public static final String ID_RSA = PropertyUtils.getString("id_rsa", "/.ssh/id_rsa");
```

**SSH免密登录**:
- 使用公钥认证
- 避免明文传输密码
- 提高安全性

### 2.11 HTTP相关常量

```java
public static final String HTTP_HEADER_UNKNOWN = "unKnown";         // 未知头
public static final String HTTP_X_FORWARDED_FOR = "X-Forwarded-For"; // 转发头
public static final String HTTP_X_REAL_IP = "X-Real-IP";            // 真实IP头
```

**IP获取策略**:
1. 先检查 `X-Forwarded-For` 头
2. 再检查 `X-Real-IP` 头
3. 最后使用 `RemoteAddr`

**应用场景**: 
- 反向代理环境获取真实客户端IP
- 审计日志记录
- 安全防护

### 2.12 正则表达式常量

```java
// 用户名正则：3-39位字母、数字、下划线、点、横线
public static final Pattern REGEX_USER_NAME = Pattern.compile("^[a-zA-Z0-9._-]{3,39}$");

// 邮箱正则
public static final Pattern REGEX_MAIL_NAME = 
    Pattern.compile("^([a-z0-9A-Z]+[_|\\-|\\.]?)+[a-z0-9A-Z]@([a-z0-9A-Z]+(-[a-z0-9A-Z]+)?\\.)+[a-zA-Z]{2,}$");

// 变量占位符正则：匹配 ${变量名}
public static final String REGEX_VARIABLE = "\\$\\{(.*?)\\}";

// 英文检测正则
public static final String HAS_EN = ".*[a-zA-z].*";
```

**正则应用**:
```java
// 用户名验证
if (!Constants.REGEX_USER_NAME.matcher(username).matches()) {
    return Result.error("用户名格式不正确");
}

// 邮箱验证
if (!Constants.REGEX_MAIL_NAME.matcher(email).matches()) {
    return Result.error("邮箱格式不正确");
}

// 配置变量替换
String config = "hdfs.namenode=${NAMENODE_HOST}:9000";
config = config.replaceAll(Constants.REGEX_VARIABLE, actualValue);
```

### 2.13 字符和分隔符常量

```java
public static final String UTF_8 = "UTF-8";                     // 字符编码
public static final String COMMA = ",";                         // 逗号
public static final String SLASH = "/";                         // 斜杠
public static final String SINGLE_SLASH = "/";                  // 单斜杠
public static final String SPACE = " ";                         // 空格
public static final String UNDERLINE = "_";                     // 下划线
public static final String HYPHEN = "-";                        // 连接号
public static final String EQUAL_SIGN = "=";                    // 等号
public static final String CENTER_BRACKET_LEFT = "[";           // 左中括号
public static final String CENTER_BRACKET_RIGHT = "]";          // 右中括号
```

**字符串拼接优化**:
```java
// 不推荐：硬编码
String path = basePath + "/" + fileName;

// 推荐：使用常量
String path = basePath + Constants.SLASH + fileName;
```

### 2.14 配置文件类型常量

```java
public static final String XML = "xml";                         // XML配置
public static final String PROPERTIES = "properties";           // Properties配置
public static final String PROPERTIES2 = "properties2";         // Properties配置(变体2)
public static final String PROPERTIES3 = "properties3";         // Properties配置(变体3)
public static final String JSON = "json";                       // JSON配置
```

**多配置格式支持**:
- **XML**: Hadoop、Spark 等配置文件
- **Properties**: 标准 Java 配置文件
- **JSON**: 现代化配置格式

**变体说明**:
- `PROPERTIES2`: 可能是键值对格式的变体（如冒号分隔）
- `PROPERTIES3`: 可能是另一种特殊格式

### 2.15 大数据组件特定常量

#### Flink组件
```java
public static final String TASK_MANAGER = "taskmanager";        // TaskManager
public static final String JOB_MANAGER = "jobmanager";          // JobManager
```

#### ZooKeeper组件
```java
public static final String ZKSERVER = "zkserver";               // ZooKeeper Server
```

#### Prometheus监控
```java
public static final String PROMETHEUS = "prometheus";           // Prometheus
```

### 2.16 操作系统相关常量

```java
public static final String OSNAME_PROPERTIES = "os.name";           // OS名称属性
public static final String OSNAME_WINDOWS = "Windows";              // Windows系统
public static final String WINDOWS_HOST_DIR = "C:/Windows/System32/drivers"; // Windows hosts文件目录
```

**跨平台支持**:
```java
// 检测操作系统
public boolean isWindows() {
    String os = System.getProperty(Constants.OSNAME_PROPERTIES);
    return os.startsWith(Constants.OSNAME_WINDOWS);
}
```

### 2.17 数值常量

```java
public static final int ONE_HUNDRRD = 100;                      // 数值100
public static final int TWO_HUNDRRD = 200;                      // 数值200
public static final int TEN = 10;                               // 数值10
```

**使用场景**:
- 百分比计算
- 分页大小
- 默认限制值

### 2.18 通用状态常量

```java
public static final String STATUS = "status";                   // 状态
public static final String MSG = "msg";                         // 消息
public static final String CODE = "code";                       // 状态码
public static final String SUCCESS = "success";                 // 成功
public static final String FAILED = "failed";                   // 失败
```

### 2.19 其他业务常量

```java
public static final String DATASOPHON = "datasophon";           // 产品名称
public static final String MASTER = "master";                   // Master角色
public static final String MASTER_HOST = "masterHost";          // Master主机
public static final String MASTER_WEB_PORT = "masterWebPort";   // Master Web端口
public static final String DATA = "data";                       // 数据
public static final String TOTAL = "total";                     // 总数
public static final String QUERY = "query";                     // 查询
public static final String MANAGED = "managed";                 // 受管
public static final String CUSTOM = "custom";                   // 自定义
public static final String INPUT = "input";                     // 输入
public static final String MULTIPLE = "multiple";               // 多选
public static final String PATH = "path";                       // 路径
public static final String MV_PATH = "mv_path";                 // 移动路径
public static final String NAME = "name";                       // 名称
public static final String IS_ENABLED = "is_enabled";           // 是否启用
public static final String SORT_NUM = "sort_num";               // 排序号
public static final String CREATE_TIME = "create_time";         // 创建时间
public static final String LOCALE_LANGUAGE = "language";        // 语言
public static final String CN = "chinese";                      // 中文
public static final String INSTALL_TYPE = "install_type";       // 安装类型
public static final String FRAME_CODE_1 = "frame_code";         // 框架编码
public static final String VARIABLE_NAME = "variable_name";     // 变量名
public static final String QUEUE_NAME = "queue_name";           // 队列名称
public static final String ALERT_TARGET_NAME = "alert_target_name"; // 告警目标名称
```

## 三、设计模式分析

### 3.1 工具类模式

```java
public final class Constants {
    // 私有构造函数，防止实例化
    private Constants() {
        throw new IllegalStateException("Constants Exception");
    }
}
```

**设计要点**:
1. **final类**: 不可被继承
2. **私有构造**: 不可被实例化
3. **静态常量**: 所有字段都是 `public static final`
4. **抛出异常**: 防止反射创建实例

### 3.2 配置外部化

```java
// 从配置文件读取，支持运行时配置
public static final String INSTALL_PATH = PropertyUtils.getString("install.path");
public static final String ID_RSA = PropertyUtils.getString("id_rsa", "/.ssh/id_rsa");
```

**优势**:
- 支持不同环境配置
- 无需重新编译
- 提供默认值兜底

### 3.3 命令模板模式

```java
// 使用常量拼接命令模板
public static final String UNZIP_DDH_WORKER_CMD = 
    "tar -zxvf " + INSTALL_PATH + "/datasophon-worker.tar.gz -C " + INSTALL_PATH;
```

**特点**:
- 预定义命令模板
- 自动替换路径变量
- 减少错误

## 四、最佳实践

### 4.1 常量命名规范

✅ **推荐**:
```java
public static final String SERVICE_ROLE_NAME = "service_role_name";  // 清晰明了
public static final int SESSION_TIME_OUT = 7200;                      // 含义明确
```

❌ **不推荐**:
```java
public static final String SRN = "srn";                               // 缩写不明
public static final int TIMEOUT = 7200;                               // 缺少上下文
```

### 4.2 常量使用

✅ **推荐**:
```java
// 使用常量
if (status.equals(Constants.SUCCESS)) {
    // 处理成功
}

String path = basePath + Constants.SLASH + fileName;
```

❌ **不推荐**:
```java
// 魔法值
if (status.equals("success")) {
    // 处理成功
}

String path = basePath + "/" + fileName;
```

### 4.3 正则预编译

```java
// 预编译正则，提高性能
public static final Pattern REGEX_USER_NAME = Pattern.compile("^[a-zA-Z0-9._-]{3,39}$");

// 使用时直接调用
if (Constants.REGEX_USER_NAME.matcher(username).matches()) {
    // 验证通过
}
```

### 4.4 配置默认值

```java
// 提供默认值，增强健壮性
public static final String ID_RSA = PropertyUtils.getString("id_rsa", "/.ssh/id_rsa");
```

## 五、使用场景示例

### 5.1 服务安装流程

```java
public class ServiceInstaller {
    public void install(String hostname, String serviceName) {
        // 1. 构建安装路径
        String installPath = Constants.INSTALL_PATH + 
                           Constants.SLASH + serviceName;
        
        // 2. 创建配置目录
        String confPath = installPath + Constants.SLASH + 
                         Constants.CONFIG;
        
        // 3. 执行安装命令
        // ...
    }
}
```

### 5.2 命令执行封装

```java
public class CommandExecutor {
    public Result deployWorker(String hostname) {
        try {
            // 1. 检查Worker包
            String md5 = exec(hostname, Constants.CHECK_WORKER_MD5_CMD);
            
            // 2. 解压Worker
            exec(hostname, Constants.UNZIP_DDH_WORKER_CMD);
            
            // 3. 启动Worker
            exec(hostname, Constants.START_DDH_WORKER_CMD);
            
            return Result.success();
        } catch (Exception e) {
            return Result.error(Constants.FAILED);
        }
    }
}
```

### 5.3 配置文件生成

```java
public class ConfigGenerator {
    public void generateConfig(Map<String, String> params) {
        // 使用正则替换配置变量
        String template = loadTemplate();
        
        for (Map.Entry<String, String> entry : params.entrySet()) {
            String placeholder = "${" + entry.getKey() + "}";
            template = template.replace(placeholder, entry.getValue());
        }
        
        // 保存配置文件
        String configPath = Constants.INSTALL_PATH + 
                          Constants.SLASH + 
                          Constants.CONFIG_FILE;
        saveConfig(configPath, template);
    }
}
```

### 5.4 用户验证

```java
public class UserValidator {
    public Result validateUser(String username, String email) {
        // 验证用户名格式
        if (!Constants.REGEX_USER_NAME.matcher(username).matches()) {
            return Result.error("用户名格式不正确，支持3-39位字母、数字、下划线、点、横线");
        }
        
        // 验证邮箱格式
        if (!Constants.REGEX_MAIL_NAME.matcher(email).matches()) {
            return Result.error("邮箱格式不正确");
        }
        
        return Result.success();
    }
}
```

## 六、扩展建议

### 6.1 常量分类管理

当前所有常量都在一个类中，随着系统扩展可能导致类过大。建议按功能域拆分：

```java
// 路径相关常量
public class PathConstants {
    public static final String INSTALL_PATH = PropertyUtils.getString("install.path");
    public static final String WORKER_PATH = INSTALL_PATH + "/datasophon-worker";
    // ...
}

// 命令相关常量
public class CommandConstants {
    public static final String START_DDH_WORKER_CMD = "service datasophon-worker restart";
    // ...
}

// 服务相关常量
public class ServiceConstants {
    public static final String SERVICE_NAME = "service_name";
    public static final String SERVICE_STATE = "service_state";
    // ...
}
```

### 6.2 枚举替代部分常量

对于有限的、有语义关联的常量，建议使用枚举：

```java
// 替代字符串常量
public enum ConfigFileType {
    XML("xml"),
    PROPERTIES("properties"),
    JSON("json");
    
    private String type;
    
    ConfigFileType(String type) {
        this.type = type;
    }
    
    public String getType() {
        return type;
    }
}
```

### 6.3 配置中心集成

对于需要动态调整的配置，建议集成配置中心：

```java
// 支持动态刷新
public class DynamicConstants {
    @Value("${session.timeout}")
    private int sessionTimeout;
    
    @Value("${install.path}")
    private String installPath;
    
    // 使用 @RefreshScope 支持动态刷新
}
```

### 6.4 国际化支持

对于面向用户的文本常量，建议使用国际化：

```java
// 使用 i18n 资源文件
public class MessageConstants {
    public static String getMessage(String key, String language) {
        if (Constants.CN.equals(language)) {
            return messageSource.getMessage(key, null, Locale.CHINESE);
        } else {
            return messageSource.getMessage(key, null, Locale.ENGLISH);
        }
    }
}
```

## 七、注意事项

### 7.1 常量不可变性

```java
// ✅ 正确：基本类型和不可变对象
public static final String NAME = "datasophon";
public static final int TIMEOUT = 7200;

// ⚠️ 注意：集合类型虽然引用不变，但内容可变
public static final List<String> LIST = new ArrayList<>();
// LIST 引用不能改变，但可以 LIST.add()

// ✅ 推荐：使用不可变集合
public static final List<String> LIST = Collections.unmodifiableList(
    Arrays.asList("item1", "item2")
);
```

### 7.2 静态初始化顺序

```java
// 确保依赖的常量先初始化
public static final String INSTALL_PATH = PropertyUtils.getString("install.path");
public static final String WORKER_PATH = INSTALL_PATH + "/datasophon-worker";  // 依赖 INSTALL_PATH
```

### 7.3 配置加载时机

```java
// PropertyUtils.getString() 需要在配置加载后调用
// 确保 Spring 容器启动完成后再使用这些常量
```

## 八、总结

### 核心价值

1. **统一管理**: 集中定义系统常量，便于维护
2. **避免硬编码**: 提高代码可读性和可维护性
3. **类型安全**: 编译期检查，减少运行时错误
4. **配置灵活**: 支持外部化配置，适应不同环境

### 设计亮点

1. **工具类模式**: 防止实例化，确保正确使用
2. **配置外部化**: 关键路径支持自定义配置
3. **正则预编译**: 提高正则表达式性能
4. **多架构支持**: 兼容 x86 和 ARM 架构
5. **国际化准备**: 支持中英文切换

### 改进方向

1. 常量分类拆分，降低单文件复杂度
2. 引入枚举类型，增强类型安全
3. 集成配置中心，支持动态配置
4. 完善国际化支持

---

**文档版本**: v1.0  
**最后更新**: 2025-11-15  
**分析人员**: DataSophon 源码分析团队
