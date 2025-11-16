# DataSophon Common 模块 - Lifecycle 与 Exception 详解

## 一、模块概述

本文档详细分析 DataSophon Common 模块中的两个子模块：
- **Lifecycle (生命周期管理)**: 管理服务器（Master/Worker）的生命周期状态
- **Exception (异常处理)**: 定义系统中的自定义异常

## 二、Lifecycle 生命周期管理模块

### 2.1 模块定位

**模块路径**: `datasophon-common/src/main/java/com/datasophon/common/lifecycle/`

**核心功能**:
- 管理服务器的运行状态
- 提供优雅停机机制
- 支持状态转换和恢复
- 线程安全的状态管理

**文件列表**:
1. `ServerStatus.java` - 服务器状态枚举
2. `ServerLifeCycleManager.java` - 生命周期管理器
3. `ServerLifeCycleException.java` - 生命周期异常

### 2.2 ServerStatus - 服务器状态枚举

**功能**: 定义服务器的三种运行状态

**完整源码分析**:
```java
/**
 * 服务器状态枚举，适用于 Master 和 Worker
 */
public enum ServerStatus {
    /**
     * 运行中：服务器正常工作，可以接收和处理请求
     */
    RUNNING(0, "The current server is running"),
    
    /**
     * 等待中：服务器暂停工作，不能处理请求
     * 用于优雅停机或维护场景
     */
    WAITING(1, "The current server is waiting, this means it cannot work"),
    
    /**
     * 已停止：服务器已完全停止
     */
    STOPPED(2, "The current server is stopped");
    
    private final int code;
    private final String desc;
    
    ServerStatus(int code, String desc) {
        this.code = code;
        this.desc = desc;
    }
    
    public int getCode() {
        return code;
    }
    
    public String getDesc() {
        return desc;
    }
}
```

**状态说明**:

| 状态 | 代码 | 含义 | 可接受请求 |
|------|------|------|-----------|
| RUNNING | 0 | 正常运行 | ✅ 是 |
| WAITING | 1 | 等待/暂停 | ❌ 否 |
| STOPPED | 2 | 已停止 | ❌ 否 |

**状态转换图**:
```
    启动
     ↓
[RUNNING] ←────┐
     ↓          │
     │ toWaiting│ recoverFromWaiting
     ↓          │
[WAITING]  ────┘
     ↓
     │ toStopped
     ↓
[STOPPED]
     ↓
   (终态)
```

**设计特点**:
1. **有限状态机**: 清晰的状态定义和转换规则
2. **状态码**: 便于日志记录和监控
3. **描述信息**: 提供人类可读的状态说明

### 2.3 ServerLifeCycleManager - 生命周期管理器

**功能**: 管理服务器生命周期的核心类，提供线程安全的状态管理

**完整源码分析**:
```java
@UtilityClass
public class ServerLifeCycleManager {
    
    // 当前服务器状态（使用 volatile 确保可见性）
    private static volatile ServerStatus serverStatus = ServerStatus.RUNNING;
    
    // 服务器启动时间
    private static long serverStartupTime = System.currentTimeMillis();
    
    /**
     * 获取服务器启动时间
     */
    public static long getServerStartupTime() {
        return serverStartupTime;
    }
    
    /**
     * 检查服务器是否运行中
     */
    public static boolean isRunning() {
        return serverStatus == ServerStatus.RUNNING;
    }
    
    /**
     * 检查服务器是否已停止
     */
    public static boolean isStopped() {
        return serverStatus == ServerStatus.STOPPED;
    }
    
    /**
     * 获取当前服务器状态
     */
    public static ServerStatus getServerStatus() {
        return serverStatus;
    }
    
    /**
     * 将服务器状态改为 WAITING（等待）
     * 只有 RUNNING 状态可以转换为 WAITING
     * 
     * @throws ServerLifeCycleException 如果状态转换失败
     */
    public static synchronized void toWaiting() throws ServerLifeCycleException {
        // 已停止的服务器不能转为等待状态
        if (isStopped()) {
            throw new ServerLifeCycleException(
                "The current server is already stopped, cannot change to waiting");
        }
        
        // 只有运行中的服务器才能转为等待状态
        if (serverStatus != ServerStatus.RUNNING) {
            throw new ServerLifeCycleException(
                "The current server is not at running status, cannot change to waiting");
        }
        
        serverStatus = ServerStatus.WAITING;
    }
    
    /**
     * 从 WAITING 状态恢复到 RUNNING 状态
     * 
     * @throws ServerLifeCycleException 如果恢复失败
     */
    public static synchronized void recoverFromWaiting() throws ServerLifeCycleException {
        // 只有等待状态才能恢复
        if (serverStatus != ServerStatus.WAITING) {
            throw new ServerLifeCycleException(
                "The current server status is not waiting, cannot recover from waiting");
        }
        
        // 更新启动时间
        serverStartupTime = System.currentTimeMillis();
        serverStatus = ServerStatus.RUNNING;
    }
    
    /**
     * 停止服务器
     * 
     * @return true 表示状态已改变，false 表示已经是停止状态
     */
    public static synchronized boolean toStopped() {
        if (serverStatus == ServerStatus.STOPPED) {
            return false;
        }
        serverStatus = ServerStatus.STOPPED;
        return true;
    }
}
```

**核心功能详解**:

#### 1. 线程安全设计

```java
// volatile 确保状态变量在多线程间的可见性
private static volatile ServerStatus serverStatus = ServerStatus.RUNNING;

// synchronized 确保状态转换的原子性
public static synchronized void toWaiting() throws ServerLifeCycleException {
    // 状态转换逻辑
}
```

**为什么需要线程安全**:
- Master/Worker 服务器是多线程环境
- 多个线程可能同时检查或修改服务器状态
- 需要防止状态不一致

#### 2. 状态转换规则

| 当前状态 | 允许的转换 | 不允许的转换 |
|---------|-----------|-------------|
| RUNNING | → WAITING, → STOPPED | - |
| WAITING | → RUNNING, → STOPPED | → WAITING |
| STOPPED | - | → RUNNING, → WAITING |

**设计原则**:
- **单向性**: STOPPED 是终态，不能恢复
- **保护性**: 严格检查状态转换的合法性
- **异常抛出**: 非法转换抛出异常，不允许静默失败

#### 3. 启动时间管理

```java
private static long serverStartupTime = System.currentTimeMillis();

// 恢复时更新启动时间
public static synchronized void recoverFromWaiting() {
    serverStartupTime = System.currentTimeMillis();
    serverStatus = ServerStatus.RUNNING;
}
```

**用途**:
- 计算服务器运行时长
- 监控和告警
- 统计信息展示

### 2.4 ServerLifeCycleException - 生命周期异常

**功能**: 生命周期管理过程中抛出的异常

**完整源码分析**:
```java
public class ServerLifeCycleException extends Exception {
    
    /**
     * 构造异常，只包含消息
     */
    public ServerLifeCycleException(String message) {
        super(message);
    }
    
    /**
     * 构造异常，包含消息和原因
     */
    public ServerLifeCycleException(String message, Throwable throwable) {
        super(message, throwable);
    }
}
```

**使用场景**:
1. **非法状态转换**: 尝试从 STOPPED 转到 WAITING
2. **状态不匹配**: 当前状态不满足转换条件
3. **并发冲突**: 状态在检查和修改之间被改变

**异常处理示例**:
```java
try {
    ServerLifeCycleManager.toWaiting();
    logger.info("服务器进入等待状态");
} catch (ServerLifeCycleException e) {
    logger.error("状态转换失败: " + e.getMessage());
    // 进行错误处理
}
```

### 2.5 使用场景详解

#### 场景 1: 优雅停机

**问题**: 如何在不丢失任务的情况下停止服务器？

**解决方案**:
```java
public class GracefulShutdown {
    
    public void shutdown() {
        try {
            // 1. 进入等待状态，停止接收新请求
            ServerLifeCycleManager.toWaiting();
            logger.info("服务器进入等待状态，停止接收新请求");
            
            // 2. 等待正在执行的任务完成
            while (TaskManager.hasRunningTasks()) {
                logger.info("等待 {} 个任务完成...", 
                           TaskManager.getRunningTaskCount());
                Thread.sleep(1000);
            }
            
            // 3. 停止服务器
            ServerLifeCycleManager.toStopped();
            logger.info("服务器已停止");
            
        } catch (ServerLifeCycleException e) {
            logger.error("优雅停机失败", e);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

#### 场景 2: 健康检查

**问题**: 如何让负载均衡器知道服务器是否可用？

**解决方案**:
```java
@RestController
public class HealthCheckController {
    
    @GetMapping("/health")
    public ResponseEntity<String> health() {
        if (ServerLifeCycleManager.isRunning()) {
            return ResponseEntity.ok("OK");
        } else {
            return ResponseEntity
                .status(HttpStatus.SERVICE_UNAVAILABLE)
                .body("Server is " + ServerLifeCycleManager.getServerStatus());
        }
    }
}
```

#### 场景 3: 维护模式

**问题**: 如何暂时停止服务进行维护，之后再恢复？

**解决方案**:
```java
public class MaintenanceService {
    
    public void enterMaintenanceMode() throws ServerLifeCycleException {
        // 进入维护模式
        ServerLifeCycleManager.toWaiting();
        logger.info("进入维护模式");
        
        // 执行维护任务
        performMaintenance();
    }
    
    public void exitMaintenanceMode() throws ServerLifeCycleException {
        // 退出维护模式
        ServerLifeCycleManager.recoverFromWaiting();
        logger.info("退出维护模式，服务恢复");
    }
    
    private void performMaintenance() {
        // 数据库清理、日志归档等
    }
}
```

#### 场景 4: 请求过滤

**问题**: 如何在非运行状态拒绝请求？

**解决方案**:
```java
@Component
public class ServerStatusFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        
        // 检查服务器状态
        if (!ServerLifeCycleManager.isRunning()) {
            HttpServletResponse httpResponse = (HttpServletResponse) response;
            httpResponse.setStatus(HttpStatus.SERVICE_UNAVAILABLE.value());
            httpResponse.getWriter().write(
                "Server is not available, current status: " + 
                ServerLifeCycleManager.getServerStatus());
            return;
        }
        
        // 继续处理请求
        chain.doFilter(request, response);
    }
}
```

### 2.6 设计模式分析

#### 1. 单例模式

```java
@UtilityClass  // Lombok 注解，表示工具类
public class ServerLifeCycleManager {
    // 静态字段和方法，全局唯一实例
}
```

**原因**: 服务器状态是全局唯一的，使用单例模式确保状态一致性

#### 2. 状态模式

通过状态枚举和状态转换方法实现状态模式：
- 每个状态有明确的行为
- 状态转换有明确的规则
- 易于扩展新状态

#### 3. 工具类模式

使用 Lombok 的 `@UtilityClass` 注解：
- 私有构造函数，不能实例化
- 所有方法都是静态的
- 专注于提供工具方法

### 2.7 性能考虑

#### 1. volatile 的代价

```java
private static volatile ServerStatus serverStatus;
```

**性能影响**:
- 读操作：略慢于普通变量（禁止缓存优化）
- 写操作：需要刷新到主内存
- 但相比 synchronized 开销小得多

**适用场景**: 状态检查频繁，状态改变少见

#### 2. synchronized 的使用

```java
public static synchronized void toWaiting() {
    // 状态转换
}
```

**性能影响**:
- 同一时刻只有一个线程可以执行
- 其他线程需要等待

**适用场景**: 状态转换不频繁，可以接受同步开销

### 2.8 改进建议

#### 1. 添加状态监听器

```java
public interface ServerStatusListener {
    void onStatusChange(ServerStatus oldStatus, ServerStatus newStatus);
}

public class ServerLifeCycleManager {
    private static List<ServerStatusListener> listeners = new ArrayList<>();
    
    public static void addListener(ServerStatusListener listener) {
        listeners.add(listener);
    }
    
    private static void notifyListeners(ServerStatus oldStatus, 
                                       ServerStatus newStatus) {
        for (ServerStatusListener listener : listeners) {
            listener.onStatusChange(oldStatus, newStatus);
        }
    }
    
    public static synchronized void toWaiting() 
            throws ServerLifeCycleException {
        ServerStatus oldStatus = serverStatus;
        // ... 状态转换逻辑
        notifyListeners(oldStatus, serverStatus);
    }
}
```

#### 2. 添加状态转换历史

```java
@Data
public class StatusTransition {
    private ServerStatus fromStatus;
    private ServerStatus toStatus;
    private long timestamp;
    private String reason;
}

public class ServerLifeCycleManager {
    private static List<StatusTransition> history = new ArrayList<>();
    
    public static List<StatusTransition> getHistory() {
        return Collections.unmodifiableList(history);
    }
}
```

## 三、Exception 异常处理模块

### 3.1 模块定位

**模块路径**: `datasophon-common/src/main/java/com/datasophon/common/exception/`

**核心功能**:
- 定义系统特定的异常类型
- 提供结构化的错误信息
- 支持异常序列化（跨网络传递）

**文件列表**:
1. `AkkaRemoteException.java` - Akka 远程调用异常

### 3.2 AkkaRemoteException - Akka 远程异常

**功能**: 封装 Akka 远程调用过程中的异常信息

**完整源码分析**:
```java
@Data
public class AkkaRemoteException implements Serializable {
    
    // 主机命令ID，标识哪个命令失败
    private String hostCommandId;
    
    // 错误消息
    private String errMsg;
}
```

**设计特点**:

#### 1. 不继承 Exception

**为什么不继承 Exception？**
```java
// 不是这样设计
public class AkkaRemoteException extends Exception { }

// 而是这样
public class AkkaRemoteException implements Serializable { }
```

**原因**:
- 这不是一个 Java 异常类，而是一个**数据传输对象（DTO）**
- 用于在 Master 和 Worker 之间传递错误信息
- 序列化更简单，无需处理 Exception 的复杂继承结构

#### 2. 实现 Serializable

```java
public class AkkaRemoteException implements Serializable {
```

**原因**:
- Akka 远程调用需要序列化对象
- 需要在网络上传输异常信息
- 支持持久化到数据库

#### 3. 字段设计

```java
// 命令ID：定位是哪个命令失败
private String hostCommandId;

// 错误消息：描述失败原因
private String errMsg;
```

**使用示例**:
```java
// Worker 执行命令失败
try {
    executeCommand(command);
} catch (Exception e) {
    // 封装异常信息
    AkkaRemoteException exception = new AkkaRemoteException();
    exception.setHostCommandId(command.getHostCommandId());
    exception.setErrMsg(e.getMessage());
    
    // 发送给 Master
    masterActor.tell(exception, getSelf());
}
```

### 3.3 异常处理最佳实践

#### 1. 自定义异常的设计原则

**何时创建自定义异常**:
1. 需要特定的错误处理逻辑
2. 需要携带额外的上下文信息
3. 需要在多个层次间传递错误

**示例：设计一个完整的自定义异常**:
```java
public class ServiceInstallException extends Exception {
    
    // 错误码
    private final String errorCode;
    
    // 服务名称
    private final String serviceName;
    
    // 主机名
    private final String hostname;
    
    public ServiceInstallException(String errorCode, 
                                  String serviceName, 
                                  String hostname,
                                  String message) {
        super(message);
        this.errorCode = errorCode;
        this.serviceName = serviceName;
        this.hostname = hostname;
    }
    
    public ServiceInstallException(String errorCode,
                                  String serviceName,
                                  String hostname,
                                  String message,
                                  Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.serviceName = serviceName;
        this.hostname = hostname;
    }
    
    // Getters
    public String getErrorCode() { return errorCode; }
    public String getServiceName() { return serviceName; }
    public String getHostname() { return hostname; }
}
```

#### 2. 异常分层处理

```java
// 1. DAO层：抛出数据访问异常
public class ServiceDAO {
    public Service getById(int id) throws DataAccessException {
        // 数据库操作
    }
}

// 2. Service层：转换为业务异常
public class ServiceManager {
    public Service getService(int id) throws ServiceNotFoundException {
        try {
            return serviceDAO.getById(id);
        } catch (DataAccessException e) {
            throw new ServiceNotFoundException("Service not found: " + id, e);
        }
    }
}

// 3. Controller层：转换为HTTP响应
@RestController
public class ServiceController {
    @GetMapping("/service/{id}")
    public ResponseEntity<Service> getService(@PathVariable int id) {
        try {
            Service service = serviceManager.getService(id);
            return ResponseEntity.ok(service);
        } catch (ServiceNotFoundException e) {
            return ResponseEntity.notFound().build();
        }
    }
}
```

#### 3. 异常日志记录

```java
try {
    executeCommand(command);
} catch (Exception e) {
    // 1. 记录完整堆栈
    logger.error("命令执行失败: commandId={}, hostId={}", 
                 command.getId(), command.getHostId(), e);
    
    // 2. 封装简化的错误信息发送给客户端
    AkkaRemoteException remoteException = new AkkaRemoteException();
    remoteException.setHostCommandId(command.getId());
    remoteException.setErrMsg(e.getMessage());
    
    return remoteException;
}
```

### 3.4 扩展异常体系建议

基于当前的简单异常，建议扩展为完整的异常体系：

```java
// 基础异常
public abstract class DataSophonException extends Exception {
    private final String errorCode;
    
    protected DataSophonException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
    
    public String getErrorCode() {
        return errorCode;
    }
}

// 业务异常
public class ServiceException extends DataSophonException {
    public ServiceException(String errorCode, String message) {
        super(errorCode, message);
    }
}

// 系统异常
public class SystemException extends DataSophonException {
    public SystemException(String errorCode, String message) {
        super(errorCode, message);
    }
}

// 具体异常
public class ServiceInstallException extends ServiceException {
    public ServiceInstallException(String message) {
        super("SERVICE_INSTALL_FAILED", message);
    }
}

public class HostUnreachableException extends SystemException {
    public HostUnreachableException(String hostname) {
        super("HOST_UNREACHABLE", "无法连接到主机: " + hostname);
    }
}
```

## 四、模块协同工作

### 4.1 Lifecycle 与 Exception 的协同

```java
public class ServerManager {
    
    public void startServer() {
        try {
            // 初始化服务器
            initialize();
            
            // 状态转换为 RUNNING（已经是默认状态）
            logger.info("服务器启动成功");
            
        } catch (Exception e) {
            // 启动失败，抛出生命周期异常
            throw new ServerLifeCycleException("服务器启动失败", e);
        }
    }
    
    public void stopServer() {
        try {
            // 先进入等待状态
            ServerLifeCycleManager.toWaiting();
            
            // 清理资源
            cleanup();
            
            // 最后停止
            ServerLifeCycleManager.toStopped();
            
        } catch (ServerLifeCycleException e) {
            logger.error("服务器停止失败", e);
            throw e;
        }
    }
}
```

### 4.2 与 Akka 的集成

```java
public class WorkerActor extends AbstractActor {
    
    @Override
    public Receive createReceive() {
        return receiveBuilder()
            .match(ExecuteCommand.class, this::handleCommand)
            .match(PingCommand.class, this::handlePing)
            .build();
    }
    
    private void handleCommand(ExecuteCommand command) {
        // 检查服务器状态
        if (!ServerLifeCycleManager.isRunning()) {
            AkkaRemoteException exception = new AkkaRemoteException();
            exception.setHostCommandId(command.getHostCommandId());
            exception.setErrMsg("Worker不可用，当前状态: " + 
                              ServerLifeCycleManager.getServerStatus());
            getSender().tell(exception, getSelf());
            return;
        }
        
        // 执行命令
        try {
            CommandResult result = executor.execute(command);
            getSender().tell(result, getSelf());
        } catch (Exception e) {
            AkkaRemoteException exception = new AkkaRemoteException();
            exception.setHostCommandId(command.getHostCommandId());
            exception.setErrMsg(e.getMessage());
            getSender().tell(exception, getSelf());
        }
    }
    
    private void handlePing(PingCommand ping) {
        // 返回服务器状态
        PingResponse response = new PingResponse();
        response.setStatus(ServerLifeCycleManager.getServerStatus());
        response.setUptime(System.currentTimeMillis() - 
                          ServerLifeCycleManager.getServerStartupTime());
        getSender().tell(response, getSelf());
    }
}
```

## 五、总结

### 5.1 Lifecycle 模块总结

**核心价值**:
1. **优雅停机**: 支持不丢失任务的安全停机
2. **状态管理**: 清晰的状态定义和转换
3. **线程安全**: volatile + synchronized 保证并发安全
4. **维护模式**: 支持暂停和恢复服务

**技术亮点**:
1. **有限状态机**: 状态和转换规则明确
2. **异常安全**: 非法转换抛出异常
3. **单例模式**: 全局唯一的状态管理
4. **工具类设计**: 静态方法，易于使用

### 5.2 Exception 模块总结

**核心价值**:
1. **错误传递**: 在分布式环境中传递错误信息
2. **序列化支持**: 支持跨网络传输
3. **结构化信息**: 包含命令ID和错误消息

**设计特点**:
1. **DTO 模式**: 不继承 Exception，作为数据对象
2. **简单高效**: 最小化字段，降低序列化开销
3. **可扩展**: 易于添加新的异常类型

### 5.3 使用建议

1. **生命周期管理**:
   - 在应用启动时初始化 ServerLifeCycleManager
   - 在 API 接口中检查服务器状态
   - 实现优雅停机钩子

2. **异常处理**:
   - 在 Akka Actor 中捕获并封装异常
   - 记录完整的错误日志
   - 向客户端返回简化的错误信息

3. **监控和告警**:
   - 监控服务器状态变化
   - 状态异常时发送告警
   - 记录状态转换历史

这两个模块虽然简单，但在分布式系统中起着重要的基础作用，体现了 DataSophon 对系统稳定性和可维护性的重视。
