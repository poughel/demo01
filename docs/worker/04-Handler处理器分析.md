# DataSophon Worker Handler 处理器完整分析

## 一、Handler 层概述

### 1.1 Handler 层职责

Handler 层是 Worker 中的**业务逻辑处理层**，位于 Actor 和底层工具类之间，负责具体的服务操作逻辑。

**架构层次**:
```
Actor 层 (消息接收)
    │
    ▼
Handler 层 (业务逻辑)
    │
    ▼
工具层 (ShellUtils, FileUtils...)
```

### 1.2 Handler 类列表

| Handler 类 | 文件名 | 行数 | 核心职责 |
|-----------|--------|------|---------|
| `InstallServiceHandler` | InstallServiceHandler.java | 244 行 | 服务安装：下载、解压、配置权限 |
| `ConfigureServiceHandler` | ConfigureServiceHandler.java | 299 行 | 配置生成：FreeMarker 模板渲染 |
| `ServiceHandler` | ServiceHandler.java | 184 行 | 服务操作：启动、停止、重启、状态检查 |

## 二、InstallServiceHandler - 服务安装处理器

### 2.1 类基本信息

**文件路径**: `com/datasophon/worker/handler/InstallServiceHandler.java`  
**行数**: 244 行  
**职责**: 处理服务安装的完整流程

### 2.2 类结构

```java
@Slf4j
@Data
public class InstallServiceHandler {
    private String frameCode;        // 框架代码 (DDP)
    private String serviceName;      // 服务名 (HDFS, YARN...)
    private String serviceRoleName;  // 角色名 (NameNode, DataNode...)
    private Logger logger;           // 动态日志记录器
    
    public InstallServiceHandler(String frameCode, String serviceName, String serviceRoleName) {
        this.frameCode = frameCode;
        this.serviceName = serviceName;
        this.serviceRoleName = serviceRoleName;
        
        // 创建任务专属日志记录器
        String loggerName = String.format("%s-%s-%s-%s", 
            TaskConstants.TASK_LOG_LOGGER_NAME,
            frameCode, serviceName, serviceRoleName
        );
        logger = LoggerFactory.getLogger(loggerName);
    }
}
```

**动态日志记录器设计**:
- **目的**: 每个安装任务有独立的日志文件
- **格式**: `task-logger-{frameCode}-{serviceName}-{serviceRoleName}`
- **示例**: `task-logger-DDP-HDFS-NameNode`

### 2.3 核心方法：install()

```java
public ExecResult install(InstallServiceRoleCommand command) {
    ExecResult execResult = new ExecResult();
    try {
        // 1. 准备安装路径
        String destDir = Constants.INSTALL_PATH + Constants.SLASH + "DDP/packages" + Constants.SLASH;
        String packageName = command.getPackageName();  // hadoop-3.3.3.tar.gz
        String packagePath = destDir + packageName;
        String decompressPackageName = command.getDecompressPackageName(); // hadoop-3.3.3
        
        // 2. 判断是否需要下载
        Boolean needDownLoad = !Objects.equals(
            PropertyUtils.getString(Constants.MASTER_HOST),
            CacheUtils.get(Constants.HOSTNAME)
        ) && isNeedDownloadPkg(packagePath, command.getPackageMd5());
        
        if (Boolean.TRUE.equals(needDownLoad)) {
            downloadPkg(packageName, packagePath);
        }
        
        // 3. 解压安装包
        boolean result = decompressPkg(packageName, decompressPackageName, destDir);
        
        if (result) {
            // 4. 执行资源策略
            if (CollUtil.isNotEmpty(command.getResourceStrategies())) {
                for (Map<String, Object> strategy : command.getResourceStrategies()) {
                    String type = (String) strategy.get(ResourceStrategy.TYPE_KEY);
                    ResourceStrategy rs;
                    
                    switch (type) {
                        case ReplaceStrategy.REPLACE_TYPE:
                            rs = BeanUtil.mapToBean(strategy, ReplaceStrategy.class, true,
                                CopyOptions.create().ignoreError());
                            break;
                        case DownloadStrategy.DOWNLOAD_TYPE:
                            rs = BeanUtil.mapToBean(strategy, DownloadStrategy.class, true,
                                CopyOptions.create().ignoreError());
                            break;
                        case AppendLineStrategy.APPEND_LINE_TYPE:
                            rs = BeanUtil.mapToBean(strategy, AppendLineStrategy.class, true,
                                CopyOptions.create().ignoreError());
                            break;
                        case LinkStrategy.LINK_TYPE:
                            rs = BeanUtil.mapToBean(strategy, LinkStrategy.class, true,
                                CopyOptions.create().ignoreError());
                            break;
                        case ShellStrategy.SHELL_TYPE:
                            rs = BeanUtil.mapToBean(strategy, ShellStrategy.class, true,
                                CopyOptions.create().ignoreError());
                            break;
                        default:
                            rs = new EmptyStrategy();
                    }
                    
                    rs.setFrameCode(frameCode);
                    rs.setService(serviceName);
                    rs.setServiceRole(serviceRoleName);
                    rs.setBasePath(Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName);
                    rs.exec();
                }
            }
            
            // 5. 设置所有者和权限
            if (Objects.nonNull(command.getRunAs())) {
                ExecResult chownResult = ShellUtils.exceShell(
                    " chown -R " + command.getRunAs().getUser() + ":" + 
                    command.getRunAs().getGroup() + " " +
                    Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName
                );
                logger.info("chown {} {}", decompressPackageName, 
                    chownResult.getExecResult() ? "success" : "fail");
            }
            
            ExecResult chmodResult = ShellUtils.exceShell(
                " chmod -R 775 " + Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName
            );
            logger.info("chmod {} {}", decompressPackageName, 
                chmodResult.getExecResult() ? "success" : "fail");
            
            // 6. Prometheus 特殊处理
            if (decompressPackageName.contains(Constants.PROMETHEUS)) {
                String alertPath = Constants.INSTALL_PATH + Constants.SLASH + 
                    decompressPackageName + Constants.SLASH + "alert_rules";
                ShellUtils.exceShell("sed -i \"s/clusterIdValue/" + 
                    PropertyUtils.getString("clusterId") + "/g\" `grep clusterIdValue -rl " + 
                    alertPath + "`");
            }
            
            // 7. Hadoop 特殊处理
            if (decompressPackageName.contains("hadoop")) {
                changeHadoopInstallPathPerm(decompressPackageName);
            }
        }
        
        execResult.setExecResult(result);
    } catch (Exception e) {
        log.error(e.getMessage(), e);
        execResult.setExecOut(e.getMessage());
    }
    return execResult;
}
```

### 2.4 详细流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    服务安装完整流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ① 准备安装路径                                                  │
│     - 安装目录: /opt/datasophon/DDP/packages/                   │
│     - 安装包: hadoop-3.3.3.tar.gz                               │
│     - 解压目录: hadoop-3.3.3                                     │
│                                                                  │
│  ② 判断是否需要下载                                              │
│     ┌─────────────────────────────────────┐                    │
│     │ 是否为 Master 节点？                │                     │
│     └───────┬──────────────────┬──────────┘                    │
│             │ 是               │ 否                             │
│             ▼                  ▼                                 │
│        跳过下载        检查本地 MD5                              │
│                              │                                   │
│                              ▼                                   │
│                      是否匹配远程 MD5？                          │
│                       ┌────┴────┐                               │
│                       │ 是  │ 否                                │
│                       ▼     ▼                                    │
│                   跳过   从 Master 下载                          │
│                           │                                      │
│                           ▼                                      │
│          HTTP GET /ddh/service/install/downloadPackage         │
│                                                                  │
│  ③ 解压安装包                                                    │
│     - tar -zxvf hadoop-3.3.3.tar.gz -C /opt/datasophon         │
│     - 超时时间: 120 秒                                           │
│     - 日志: "Start to use tar -zxvf to decompress..."          │
│                                                                  │
│  ④ 执行资源策略 (ResourceStrategy)                              │
│     ┌────────────────────────────────────────┐                 │
│     │ ReplaceStrategy: 替换文件中的占位符    │                 │
│     │   - 替换 ${JAVA_HOME}、${USER} 等      │                 │
│     ├────────────────────────────────────────┤                 │
│     │ DownloadStrategy: 下载外部依赖        │                 │
│     │   - 下载 MySQL JDBC 驱动               │                 │
│     │   - 下载第三方 JAR 包                  │                 │
│     ├────────────────────────────────────────┤                 │
│     │ LinkStrategy: 创建符号链接             │                 │
│     │   - ln -s /opt/app /opt/link           │                 │
│     ├────────────────────────────────────────┤                 │
│     │ AppendLineStrategy: 追加配置行         │                 │
│     │   - 向 bashrc 追加环境变量             │                 │
│     ├────────────────────────────────────────┤                 │
│     │ ShellStrategy: 执行自定义脚本          │                 │
│     │   - 初始化数据库                       │                 │
│     │   - 创建配置文件                       │                 │
│     └────────────────────────────────────────┘                 │
│                                                                  │
│  ⑤ 设置所有者和权限                                              │
│     - chown -R hdfs:hadoop /opt/datasophon/hadoop-3.3.3        │
│     - chmod -R 775 /opt/datasophon/hadoop-3.3.3                │
│                                                                  │
│  ⑥ 特殊服务处理                                                  │
│     - Prometheus: 替换告警规则中的 clusterId 占位符             │
│     - Hadoop: 设置 container-executor 特殊权限 (6050)           │
│                                                                  │
│  ⑦ 返回安装结果                                                  │
│     - ExecResult.execResult = true/false                        │
│     - ExecResult.execOut = 详细信息                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.5 关键方法解析

#### isNeedDownloadPkg() - 判断是否需要下载

```java
private Boolean isNeedDownloadPkg(String packagePath, String packageMd5) {
    boolean needDownLoad = true;
    logger.info("Remote package md5 is {}", packageMd5);
    
    if (FileUtil.exist(packagePath)) {
        // 计算本地文件 MD5
        String md5 = FileUtils.md5(new File(packagePath));
        logger.info("Local md5 is {}", md5);
        
        // 比对 MD5，相同则跳过下载
        if (StringUtils.isNotBlank(md5) && packageMd5.trim().equals(md5.trim())) {
            needDownLoad = false;
        }
    }
    return needDownLoad;
}
```

**设计思想**:
- **避免重复下载**: 通过 MD5 校验避免重复下载相同的包
- **节省带宽**: 大型安装包（Hadoop 300MB+）下载耗时
- **幂等性**: 多次安装相同版本不会重复下载

#### downloadPkg() - 下载安装包

```java
private void downloadPkg(String packageName, String packagePath) {
    String masterHost = PropertyUtils.getString(Constants.MASTER_HOST);
    String masterPort = PropertyUtils.getString(Constants.MASTER_WEB_PORT);
    String downloadUrl = "http://" + masterHost + ":" + masterPort +
        "/ddh/service/install/downloadPackage?packageName=" + packageName;
    
    logger.info("download url is {}", downloadUrl);
    
    // 使用 Hutool 的 HTTP 下载工具
    HttpUtil.downloadFile(downloadUrl, FileUtil.file(packagePath), new StreamProgress() {
        @Override
        public void start() {
            Console.log("start to install。。。。");
        }
        
        @Override
        public void progress(long progressSize, long totalSize) {
            Console.log("installed：{} / {} ", 
                FileUtil.readableFileSize(progressSize),
                FileUtil.readableFileSize(totalSize));
        }
        
        @Override
        public void finish() {
            Console.log("install success！");
        }
    });
    
    logger.info("download package {} success", packageName);
}
```

**下载进度示例**:
```
start to install。。。。
installed：10.5 MB / 320.0 MB
installed：52.3 MB / 320.0 MB
installed：156.8 MB / 320.0 MB
installed：320.0 MB / 320.0 MB
install success！
```

#### changeHadoopInstallPathPerm() - Hadoop 权限特殊处理

```java
private void changeHadoopInstallPathPerm(String decompressPackageName) {
    // 1. 设置根目录所有者
    ShellUtils.exceShell(
        " chown -R  root:hadoop " + Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName
    );
    
    // 2. 设置根目录权限
    ShellUtils.exceShell(
        " chmod 775 " + Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName
    );
    
    // 3. 设置 etc 目录权限
    ShellUtils.exceShell(
        " chmod -R 775 " + Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName + "/etc"
    );
    
    // 4. 设置 container-executor 特殊权限 (setuid)
    ShellUtils.exceShell(
        " chmod 6050 " + Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName +
        "/bin/container-executor"
    );
    
    // 5. 设置 container-executor.cfg 只读权限
    ShellUtils.exceShell(
        " chmod 400 " + Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName +
        "/etc/hadoop/container-executor.cfg"
    );
    
    // 6. 设置 userlogs 目录所有者
    ShellUtils.exceShell(
        " chown -R yarn:hadoop " + Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName +
        "/logs/userlogs"
    );
    
    // 7. 设置 userlogs 目录权限
    ShellUtils.exceShell(
        " chmod 775 " + Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName + "/logs/userlogs"
    );
    
    // 8. 创建 hadoop 软链接
    ShellUtils.exceShell(
        " ln -s " + Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName + " " +
        Constants.INSTALL_PATH + Constants.SLASH + "hadoop"
    );
}
```

**为什么 container-executor 需要 6050 权限？**
- **6050 = setuid(4000) + setgid(2000) + r-x-------(050)**
- **setuid**: 以文件所有者（root）身份运行
- **setgid**: 以文件所属组（hadoop）身份运行
- **用途**: YARN 的 LinuxContainerExecutor 需要 root 权限来创建和管理用户容器

### 2.6 ResourceStrategy 资源策略详解

资源策略是安装过程中的**可插拔操作**，支持以下 5 种策略：

#### ① ReplaceStrategy - 文件内容替换

```java
{
    "type": "replace",
    "srcFile": "conf/hadoop-env.sh",
    "replaceKey": "${JAVA_HOME}",
    "replaceValue": "/usr/local/jdk1.8.0_211"
}
```

**作用**: 替换文件中的占位符

#### ② DownloadStrategy - 下载外部文件

```java
{
    "type": "download",
    "url": "http://master:8081/packages/mysql-connector-java-8.0.28.jar",
    "targetPath": "lib/mysql-connector-java.jar"
}
```

**使用场景**:
- 下载 MySQL JDBC 驱动
- 下载第三方依赖 JAR

#### ③ LinkStrategy - 创建符号链接

```java
{
    "type": "link",
    "srcPath": "/opt/datasophon/jdk1.8.0_211",
    "targetPath": "jdk"
}
```

**作用**: 创建软链接，简化路径

#### ④ AppendLineStrategy - 追加文本行

```java
{
    "type": "append",
    "targetFile": "/etc/profile",
    "content": "export HADOOP_HOME=/opt/datasophon/hadoop"
}
```

**使用场景**: 添加环境变量

#### ⑤ ShellStrategy - 执行Shell脚本

```java
{
    "type": "shell",
    "script": "bin/init-database.sh"
}
```

**使用场景**:
- 初始化数据库
- 创建目录结构
- 修改配置文件

## 三、ConfigureServiceHandler - 配置生成处理器

### 3.1 类基本信息

**文件路径**: `com/datasophon/worker/handler/ConfigureServiceHandler.java`  
**行数**: 299 行  
**职责**: 使用 FreeMarker 模板引擎生成服务配置文件

### 3.2 核心方法：configure()

```java
public ExecResult configure(
    Map<Generators, List<ServiceConfig>> configFileMap,
    String decompressPackageName,
    Integer clusterId,
    Integer myid,
    String serviceRoleName,
    RunAs runAs
) {
    ExecResult execResult = new ExecResult();
    try {
        String hostName = InetAddress.getLocalHost().getHostName();
        String ip = InetAddress.getLocalHost().getHostAddress();
        
        // 准备占位符参数
        HashMap<String, String> paramMap = new HashMap<>();
        paramMap.put("${clusterId}", String.valueOf(clusterId));
        paramMap.put("${host}", hostName);
        paramMap.put("${ip}", ip);
        paramMap.put("${user}", "root");
        paramMap.put("${myid}", String.valueOf(myid));
        
        logger.info("Start to configure service role {}", serviceRoleName);
        
        for (Generators generators : configFileMap.keySet()) {
            List<ServiceConfig> configs = configFileMap.get(generators);
            String dataDir = "";
            Iterator<ServiceConfig> iterator = configs.iterator();
            ArrayList<ServiceConfig> customConfList = new ArrayList<>();
            
            while (iterator.hasNext()) {
                ServiceConfig config = iterator.next();
                
                // 处理不同类型的配置
                if (StringUtils.isNotBlank(config.getType())) {
                    switch (config.getType()) {
                        case Constants.INPUT:
                            // 输入类型：替换占位符
                            String value = PlaceholderUtils.replacePlaceholders(
                                (String) config.getValue(),
                                paramMap,
                                Constants.REGEX_VARIABLE
                            );
                            config.setValue(value);
                            break;
                        case Constants.MULTIPLE:
                            // 多选类型：转换为字符串
                            conventToStr(config);
                            break;
                        default:
                            break;
                    }
                }
                
                // 处理路径创建
                if (Constants.PATH.equals(config.getConfigType())) {
                    createPath(config, runAs);
                }
                
                // 处理路径移动
                if (Constants.MV_PATH.equals(config.getConfigType())) {
                    movePath(config, runAs);
                }
                
                // 处理自定义配置
                if (Constants.CUSTOM.equals(config.getConfigType())) {
                    addToCustomList(iterator, customConfList, config);
                }
                
                // 移除非必需配置
                if (!config.isRequired() && !Constants.CUSTOM.equals(config.getConfigType())) {
                    iterator.remove();
                }
                
                // 类型转换
                if (config.getValue() instanceof Boolean || config.getValue() instanceof Integer) {
                    logger.info("Convert boolean and integer to string");
                    config.setValue(config.getValue().toString());
                }
                
                // 记录 ZooKeeper dataDir
                if ("dataDir".equals(config.getName())) {
                    logger.info("Find dataDir : {}", config.getValue());
                    dataDir = (String) config.getValue();
                }
            }
            
            // 写入 ZooKeeper myid 文件
            if (Objects.nonNull(myid) && StringUtils.isNotBlank(dataDir)) {
                FileUtil.writeUtf8String(myid + "", dataDir + Constants.SLASH + "myid");
            }
            
            // 添加自定义配置
            configs.addAll(customConfList);
            
            // 生成配置文件
            if (!configs.isEmpty()) {
                // 检查是否有扩展模板目录
                File extTemplateDir = new File(
                    Constants.INSTALL_PATH + File.separator + decompressPackageName,
                    "templates"
                );
                
                if (extTemplateDir.exists() && extTemplateDir.isDirectory()) {
                    // 使用扩展模板
                    logger.info("Add ext app template path: {} to loader path.", 
                        extTemplateDir.getAbsolutePath());
                    FreemakerUtils.generateConfigFile(
                        generators, configs, decompressPackageName,
                        extTemplateDir.getAbsolutePath()
                    );
                } else {
                    // 使用默认模板
                    FreemakerUtils.generateConfigFile(
                        generators, configs, decompressPackageName
                    );
                }
            }
            
            execResult.setExecOut("configure success");
            logger.info("configure success");
        }
        
        // RangerAdmin 特殊处理
        if ("RangerAdmin".equals(serviceRoleName) && !setupRangerAdmin(decompressPackageName)) {
            return execResult;
        }
        
        execResult.setExecResult(true);
    } catch (Exception e) {
        execResult.setExecErrOut(e.getMessage());
        logger.error("load app config template error!", e);
    }
    return execResult;
}
```

### 3.3 配置文件生成流程

```
┌─────────────────────────────────────────────────────────────────┐
│                配置文件生成流程 (FreeMarker)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ① 接收配置数据                                                  │
│     - Generators: 模板文件信息                                   │
│       * filename: hdfs-site.xml                                 │
│       * outputDirectory: etc/hadoop                             │
│       * templateName: hdfs-site.xml.ftl                         │
│     - ServiceConfig: 配置项列表                                  │
│       * name: dfs.replication                                   │
│       * value: 3                                                 │
│                                                                  │
│  ② 准备模板参数                                                  │
│     - 主机信息: ${host}, ${ip}                                   │
│     - 集群信息: ${clusterId}, ${myid}                            │
│     - 用户信息: ${user}                                          │
│     - 配置项: dfs.replication=3, dfs.namenode.name.dir=/data/nn│
│                                                                  │
│  ③ 处理配置项                                                    │
│     - INPUT 类型: 替换占位符                                     │
│       * /data/${clusterId}/nn → /data/1/nn                      │
│     - MULTIPLE 类型: 数组转字符串                                │
│       * ["host1", "host2"] → "host1,host2"                      │
│     - PATH 类型: 创建目录                                        │
│       * mkdir -p /data/nn                                       │
│       * chown hdfs:hadoop /data/nn                              │
│     - CUSTOM 类型: 自定义配置                                    │
│                                                                  │
│  ④ 加载 FreeMarker 模板                                          │
│     - 默认路径: classpath:/templates/                            │
│     - 扩展路径: /opt/datasophon/hadoop-3.3.3/templates/         │
│     - 模板文件: hdfs-site.xml.ftl                                │
│                                                                  │
│  ⑤ 渲染模板                                                      │
│     ┌────────────────────────────────────────┐                 │
│     │ FreeMarker Template (hdfs-site.xml.ftl)│                 │
│     ├────────────────────────────────────────┤                 │
│     │ <configuration>                        │                 │
│     │   <#list configs as config>            │                 │
│     │   <property>                           │                 │
│     │     <name>${config.name}</name>        │                 │
│     │     <value>${config.value}</value>     │                 │
│     │   </property>                          │                 │
│     │   </#list>                             │                 │
│     │ </configuration>                       │                 │
│     └────────────────────────────────────────┘                 │
│                     │                                            │
│                     ▼                                            │
│     ┌────────────────────────────────────────┐                 │
│     │ 生成的配置文件 (hdfs-site.xml)         │                 │
│     ├────────────────────────────────────────┤                 │
│     │ <configuration>                        │                 │
│     │   <property>                           │                 │
│     │     <name>dfs.replication</name>       │                 │
│     │     <value>3</value>                   │                 │
│     │   </property>                          │                 │
│     │   <property>                           │                 │
│     │     <name>dfs.namenode.name.dir</name> │                 │
│     │     <value>/data/nn</value>            │                 │
│     │   </property>                          │                 │
│     │ </configuration>                       │                 │
│     └────────────────────────────────────────┘                 │
│                                                                  │
│  ⑥ 写入目标文件                                                  │
│     - 输出路径: /opt/datasophon/hadoop-3.3.3/etc/hadoop/       │
│     - 文件名: hdfs-site.xml                                     │
│     - 权限: 775                                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 特殊场景处理

#### ZooKeeper myid 文件生成

```java
if (Objects.nonNull(myid) && StringUtils.isNotBlank(dataDir)) {
    FileUtil.writeUtf8String(myid + "", dataDir + Constants.SLASH + "myid");
}
```

**ZooKeeper 集群配置**:
- **Node1**: myid = 1
- **Node2**: myid = 2
- **Node3**: myid = 3

#### RangerAdmin 初始化

```java
private boolean setupRangerAdmin(String decompressPackageName) {
    logger.info("start to execute ranger admin setup.sh");
    
    // 执行 setup.sh 脚本
    ArrayList<String> commands = new ArrayList<>();
    commands.add(Constants.INSTALL_PATH + Constants.SLASH + 
        decompressPackageName + Constants.SLASH + "setup.sh");
    ExecResult execResult = ShellUtils.execWithStatus(
        Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName,
        commands, 300L
    );
    
    // 执行 set_globals.sh 脚本
    ArrayList<String> globalCommand = new ArrayList<>();
    globalCommand.add(Constants.INSTALL_PATH + Constants.SLASH + 
        decompressPackageName + Constants.SLASH + "set_globals.sh");
    ShellUtils.execWithStatus(
        Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName,
        globalCommand, 300L, logger
    );
    
    if (execResult.getExecResult()) {
        logger.info("ranger admin setup success");
        return true;
    }
    logger.info("ranger admin setup failed");
    return false;
}
```

**Ranger 初始化脚本作用**:
- **setup.sh**: 创建数据库表、初始化默认策略
- **set_globals.sh**: 设置全局环境变量

## 四、ServiceHandler - 服务操作处理器

### 4.1 类基本信息

**文件路径**: `com/datasophon/worker/handler/ServiceHandler.java`  
**行数**: 184 行  
**职责**: 处理服务的启动、停止、重启、状态检查

### 4.2 核心方法详解

#### start() - 启动服务

```java
public ExecResult start(
    ServiceRoleRunner startRunner,
    ServiceRoleRunner statusRunner,
    String decompressPackageName,
    RunAs runAs
) {
    // 1. 检查服务是否已启动
    ExecResult statusResult = execRunner(statusRunner, decompressPackageName, null);
    if (statusResult.getExecResult()) {
        logger.info("{} already started", decompressPackageName);
        ExecResult execResult = new ExecResult();
        execResult.setExecResult(true);
        return execResult;
    }
    
    // 2. 执行启动命令
    ExecResult startResult = execRunner(startRunner, decompressPackageName, runAs);
    
    // 3. 验证启动是否成功
    if (startResult.getExecResult()) {
        int times = PropertyUtils.getInt("times");  // 默认 10 次
        int count = 0;
        
        while (count < times) {
            logger.info("check start result at times {}", count + 1);
            ExecResult result = execRunner(statusRunner, decompressPackageName, runAs);
            
            if (result.getExecResult()) {
                logger.info("start success in {}", decompressPackageName);
                break;
            } else {
                try {
                    Thread.sleep(5 * 1000);  // 等待 5 秒
                } catch (InterruptedException e) {
                    logger.error(e.getMessage(), e);
                }
            }
            count++;
        }
        
        if (count == times) {
            logger.info(" start {} timeout", decompressPackageName);
            startResult.setExecResult(false);
        }
    }
    
    return startResult;
}
```

**启动验证机制**:
- 每 5 秒检查一次服务状态
- 最多检查 10 次（50 秒）
- 如果超时，标记为启动失败

#### stop() - 停止服务

```java
public ExecResult stop(
    ServiceRoleRunner stopRunner,
    ServiceRoleRunner statusRunner,
    String decompressPackageName,
    RunAs runAs
) {
    // 1. 检查服务状态
    ExecResult statusResult = execRunner(statusRunner, decompressPackageName, runAs);
    ExecResult execResult = new ExecResult();
    
    if (statusResult.getExecResult()) {
        // 服务正在运行，执行停止命令
        execResult = execRunner(stopRunner, decompressPackageName, runAs);
        
        // 2. 验证停止是否成功
        if (execResult.getExecResult()) {
            int times = PropertyUtils.getInt("times");
            int count = 0;
            
            while (count < times) {
                logger.info("check stop result at times {}", count + 1);
                ExecResult result = execRunner(statusRunner, decompressPackageName, runAs);
                
                if (!result.getExecResult()) {
                    logger.info("stop success in {}", decompressPackageName);
                    break;
                }
                
                try {
                    Thread.sleep(5 * 1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                count++;
            }
            
            if (count == times) {
                // 超时，停止失败
                execResult.setExecResult(false);
            }
        }
    } else {
        // 服务已经停止，直接返回成功
        logger.info("{} already stopped", decompressPackageName);
        execResult.setExecResult(true);
    }
    
    return execResult;
}
```

#### execRunner() - 执行命令

```java
public ExecResult execRunner(
    ServiceRoleRunner runner,
    String decompressPackageName,
    RunAs runAs
) {
    String shell = runner.getProgram();
    List<String> args = runner.getArgs();
    long timeout = Long.parseLong(runner.getTimeout());
    ArrayList<String> command = new ArrayList<>();
    
    // 1. 添加 sudo 切换用户
    if (Objects.nonNull(runAs) && StringUtils.isNotBlank(runAs.getUser())) {
        command.add("sudo");
        command.add("-u");
        command.add(runAs.getUser());
    }
    
    // 2. 检测 Shell 类型
    if (runner.getProgram().contains(Constants.TASK_MANAGER) ||
        runner.getProgram().contains(Constants.JOB_MANAGER)) {
        // Flink 不使用 sh
        logger.info("do not use sh");
    } else {
        File shellFile = new File(
            Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName + Constants.SLASH + shell
        );
        
        if (shellFile.exists()) {
            try {
                // 读取 shebang 行，确定使用 bash 还是 sh
                final String firstLine = StringUtils.trimToEmpty(FileUtils.readFirstLine(shellFile));
                if (firstLine.contains("bash")) {
                    command.add("bash");
                } else if (firstLine.contains("sh")) {
                    command.add("sh");
                } else {
                    command.add("sh");
                }
            } catch (Exception e) {
                logger.warn("read shell script file: " + shell + " error, reason: " + e.getMessage());
                command.add("sh");
            }
        } else {
            command.add("sh");
        }
    }
    
    // 3. 添加脚本路径和参数
    command.add(shell);
    command.addAll(args);
    
    logger.info("execute shell command : {}", command);
    
    // 4. 执行命令
    return ShellUtils.execWithStatus(
        Constants.INSTALL_PATH + Constants.SLASH + decompressPackageName,
        command, timeout, logger
    );
}
```

**Shell 检测逻辑**:
```bash
# 读取脚本第一行
#!/bin/bash  → 使用 bash 执行
#!/bin/sh    → 使用 sh 执行
```

**命令构造示例**:
```bash
# 以 hdfs 用户启动 NameNode
sudo -u hdfs sh sbin/hadoop-daemon.sh start namenode
```

## 五、总结

### 5.1 Handler 层设计模式

- **策略模式**: ResourceStrategy 支持多种安装策略
- **模板方法模式**: ServiceHandler 定义统一的服务操作流程
- **工厂模式**: 通过 type 字段动态创建 ResourceStrategy 实例

### 5.2 核心设计思想

1. **可扩展性**: ResourceStrategy 支持自定义扩展
2. **幂等性**: 重复安装、启动不会出错
3. **容错性**: MD5 校验、超时重试机制
4. **日志隔离**: 每个任务独立的日志记录器

### 5.3 最佳实践

- **安装前校验**: 检查 MD5 避免重复下载
- **权限管理**: 不同服务使用不同的系统用户
- **模板化配置**: 使用 FreeMarker 统一管理配置
- **状态验证**: 启动/停止后轮询检查状态

---

**相关文档**:
- [03-Actor层完整分析.md](./03-Actor层完整分析.md)
- [05-Strategy策略模式详解.md](./05-Strategy策略模式详解.md)
- [06-Utils工具类详解.md](./06-Utils工具类详解.md)
