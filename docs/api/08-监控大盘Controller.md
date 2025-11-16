# ClusterServiceDashboardController 监控大盘控制器详解

## 一、模块概述

### 1.1 基本信息

**文件位置**: `datasophon-api/src/main/java/com/datasophon/api/controller/ClusterServiceDashboardController.java`

**功能定位**: 监控大盘数据控制器，负责提供集群整体运行状况、服务监控指标、节点健康状态等监控数据的 REST API 接口。

**URL 前缀**: `/api/dashboard`

### 1.2 核心职责

1. **集群概览数据**: 提供集群整体运行状况概览
2. **服务监控指标**: 查询各服务实例的运行指标
3. **主机状态统计**: 统计主机节点的健康状态
4. **性能数据查询**: 提供历史性能数据查询接口
5. **告警数据展示**: 展示当前活跃告警信息
6. **资源使用统计**: 统计集群资源使用情况

### 1.3 依赖关系

```java
@RestController
@RequestMapping("api/dashboard")
public class ClusterServiceDashboardController {
    
    @Autowired
    private ClusterInfoService clusterInfoService;
    
    @Autowired
    private ClusterHostService clusterHostService;
    
    @Autowired
    private ClusterServiceInstanceService serviceInstanceService;
    
    @Autowired
    private ClusterAlertHistoryService alertHistoryService;
    
    @Autowired
    private PrometheusService prometheusService;
}
```

**依赖说明**:
- `ClusterInfoService`: 获取集群基本信息
- `ClusterHostService`: 获取主机节点信息
- `ClusterServiceInstanceService`: 获取服务实例信息
- `ClusterAlertHistoryService`: 获取告警历史信息
- `PrometheusService`: 从 Prometheus 获取监控指标数据

## 二、核心功能详解

### 2.1 集群概览数据

#### 2.1.1 获取集群概览

```java
@RequestMapping("/clusterOverview")
public Result getClusterOverview(Integer clusterId) {
    // 获取集群基本信息
    ClusterInfoEntity cluster = clusterInfoService.getById(clusterId);
    
    // 获取服务统计信息
    Map<String, Object> serviceStats = serviceInstanceService.getServiceStats(clusterId);
    
    // 获取主机统计信息
    Map<String, Object> hostStats = clusterHostService.getHostStats(clusterId);
    
    // 获取告警统计信息
    Map<String, Object> alertStats = alertHistoryService.getAlertStats(clusterId);
    
    // 组装概览数据
    Map<String, Object> overview = new HashMap<>();
    overview.put("clusterInfo", cluster);
    overview.put("serviceStats", serviceStats);
    overview.put("hostStats", hostStats);
    overview.put("alertStats", alertStats);
    
    return Result.success(overview);
}
```

**功能说明**:
- **HTTP 方法**: GET/POST
- **URL**: `/api/dashboard/clusterOverview`
- **请求参数**: `clusterId` - 集群 ID

**返回数据结构**:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "clusterInfo": {
      "id": 1,
      "clusterName": "生产集群",
      "clusterCode": "prod-cluster-01",
      "clusterState": 1
    },
    "serviceStats": {
      "totalServices": 20,
      "runningServices": 18,
      "stoppedServices": 2,
      "abnormalServices": 0
    },
    "hostStats": {
      "totalHosts": 100,
      "normalHosts": 98,
      "abnormalHosts": 2,
      "offlineHosts": 0
    },
    "alertStats": {
      "activeAlerts": 5,
      "criticalAlerts": 1,
      "warningAlerts": 4
    }
  }
}
```

**使用场景**:
- 监控大盘首页展示
- 快速了解集群整体健康状况
- 管理员日常巡检

#### 2.1.2 获取服务健康度统计

```java
@RequestMapping("/serviceHealth")
public Result getServiceHealth(Integer clusterId) {
    List<ServiceHealthInfo> healthList = serviceInstanceService.getServiceHealthList(clusterId);
    
    // 统计各健康状态的服务数量
    Map<String, Long> healthStats = healthList.stream()
        .collect(Collectors.groupingBy(
            ServiceHealthInfo::getHealthStatus,
            Collectors.counting()
        ));
    
    Map<String, Object> result = new HashMap<>();
    result.put("healthList", healthList);
    result.put("healthStats", healthStats);
    
    return Result.success(result);
}
```

**功能说明**:
- 统计各服务的健康状态
- 按健康度分类汇总
- 提供服务级别的健康详情

**返回数据**:
```json
{
  "healthList": [
    {
      "serviceName": "HDFS",
      "healthStatus": "GOOD",
      "healthScore": 95.5,
      "checkTime": "2025-11-16 10:30:00"
    }
  ],
  "healthStats": {
    "GOOD": 15,
    "CONCERNING": 3,
    "BAD": 2
  }
}
```

### 2.2 监控指标查询

#### 2.2.1 获取服务监控指标

```java
@RequestMapping("/serviceMetrics")
public Result getServiceMetrics(
    Integer clusterId,
    String serviceName,
    String metricName,
    Long startTime,
    Long endTime
) {
    // 从 Prometheus 查询监控指标
    PrometheusQuery query = PrometheusQuery.builder()
        .clusterId(clusterId)
        .serviceName(serviceName)
        .metricName(metricName)
        .startTime(startTime)
        .endTime(endTime)
        .build();
    
    List<MetricData> metrics = prometheusService.queryMetrics(query);
    
    return Result.success(metrics);
}
```

**功能说明**:
- **HTTP 方法**: GET/POST
- **URL**: `/api/dashboard/serviceMetrics`
- **请求参数**:
  - `clusterId`: 集群 ID
  - `serviceName`: 服务名称（HDFS、YARN、HBase 等）
  - `metricName`: 指标名称（CPU、内存、磁盘 IO 等）
  - `startTime`: 开始时间（时间戳）
  - `endTime`: 结束时间（时间戳）

**返回数据**:
```json
{
  "code": 0,
  "msg": "success",
  "data": [
    {
      "timestamp": 1700123400000,
      "value": 45.6,
      "metric": "cpu_usage_percent"
    },
    {
      "timestamp": 1700123460000,
      "value": 46.2,
      "metric": "cpu_usage_percent"
    }
  ]
}
```

**支持的指标类型**:

| 指标分类 | 指标名称 | 单位 | 说明 |
|---------|---------|------|------|
| CPU | cpu_usage_percent | % | CPU 使用率 |
| 内存 | memory_used_bytes | Bytes | 内存使用量 |
| 内存 | memory_usage_percent | % | 内存使用率 |
| 磁盘 | disk_read_bytes | Bytes/s | 磁盘读速率 |
| 磁盘 | disk_write_bytes | Bytes/s | 磁盘写速率 |
| 网络 | network_receive_bytes | Bytes/s | 网络接收速率 |
| 网络 | network_transmit_bytes | Bytes/s | 网络发送速率 |
| JVM | jvm_heap_used_bytes | Bytes | JVM 堆内存使用 |
| JVM | jvm_gc_time_ms | ms | GC 耗时 |

#### 2.2.2 获取主机监控指标

```java
@RequestMapping("/hostMetrics")
public Result getHostMetrics(
    Integer clusterId,
    String hostname,
    String[] metricNames,
    Long startTime,
    Long endTime
) {
    Map<String, List<MetricData>> metricsMap = new HashMap<>();
    
    // 批量查询多个指标
    for (String metricName : metricNames) {
        PrometheusQuery query = PrometheusQuery.builder()
            .clusterId(clusterId)
            .hostname(hostname)
            .metricName(metricName)
            .startTime(startTime)
            .endTime(endTime)
            .build();
        
        List<MetricData> metrics = prometheusService.queryMetrics(query);
        metricsMap.put(metricName, metrics);
    }
    
    return Result.success(metricsMap);
}
```

**功能说明**:
- 支持批量查询多个指标
- 按主机维度聚合数据
- 适用于主机详情页展示

### 2.3 性能趋势分析

#### 2.3.1 获取集群资源趋势

```java
@RequestMapping("/resourceTrend")
public Result getResourceTrend(
    Integer clusterId,
    String resourceType,
    String timeRange
) {
    // 解析时间范围
    TimeRange range = TimeRangeParser.parse(timeRange);
    
    // 查询资源趋势数据
    List<ResourceTrendData> trendData = prometheusService.queryResourceTrend(
        clusterId,
        resourceType,
        range.getStartTime(),
        range.getEndTime()
    );
    
    // 计算趋势统计信息
    ResourceTrendStats stats = calculateTrendStats(trendData);
    
    Map<String, Object> result = new HashMap<>();
    result.put("trendData", trendData);
    result.put("stats", stats);
    
    return Result.success(result);
}
```

**功能说明**:
- **时间范围参数**: `1h`, `6h`, `24h`, `7d`, `30d`
- **资源类型**: `cpu`, `memory`, `disk`, `network`

**返回数据**:
```json
{
  "trendData": [
    {
      "timestamp": 1700123400000,
      "avgValue": 45.6,
      "maxValue": 78.3,
      "minValue": 23.5
    }
  ],
  "stats": {
    "avgUsage": 45.6,
    "maxUsage": 78.3,
    "minUsage": 23.5,
    "p95Usage": 65.2,
    "p99Usage": 72.8
  }
}
```

#### 2.3.2 获取服务性能对比

```java
@RequestMapping("/serviceComparison")
public Result getServiceComparison(
    Integer clusterId,
    String[] serviceNames,
    String metricName,
    Long startTime,
    Long endTime
) {
    Map<String, List<MetricData>> comparisonData = new HashMap<>();
    
    for (String serviceName : serviceNames) {
        PrometheusQuery query = PrometheusQuery.builder()
            .clusterId(clusterId)
            .serviceName(serviceName)
            .metricName(metricName)
            .startTime(startTime)
            .endTime(endTime)
            .build();
        
        List<MetricData> metrics = prometheusService.queryMetrics(query);
        comparisonData.put(serviceName, metrics);
    }
    
    return Result.success(comparisonData);
}
```

**使用场景**:
- 对比多个服务的资源使用情况
- 识别资源消耗异常的服务
- 容量规划和优化决策

### 2.4 告警数据展示

#### 2.4.1 获取活跃告警列表

```java
@RequestMapping("/activeAlerts")
public Result getActiveAlerts(
    Integer clusterId,
    String severity,
    Integer page,
    Integer pageSize
) {
    // 构建查询条件
    QueryWrapper<ClusterAlertHistory> queryWrapper = new QueryWrapper<>();
    queryWrapper.eq("cluster_id", clusterId);
    queryWrapper.eq("alert_state", AlertState.FIRING);
    
    if (StringUtils.isNotEmpty(severity)) {
        queryWrapper.eq("severity", severity);
    }
    
    queryWrapper.orderByDesc("create_time");
    
    // 分页查询
    Page<ClusterAlertHistory> pageResult = alertHistoryService.page(
        new Page<>(page, pageSize),
        queryWrapper
    );
    
    return Result.success(pageResult);
}
```

**功能说明**:
- **HTTP 方法**: GET/POST
- **URL**: `/api/dashboard/activeAlerts`
- **请求参数**:
  - `clusterId`: 集群 ID
  - `severity`: 告警级别（可选）- `critical`, `warning`, `info`
  - `page`: 页码
  - `pageSize`: 每页大小

**返回数据**:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "total": 25,
    "pages": 3,
    "records": [
      {
        "id": 1001,
        "alertName": "HDFS NameNode 内存告警",
        "severity": "critical",
        "alertState": "FIRING",
        "alertInfo": "NameNode 内存使用率超过 85%",
        "createTime": "2025-11-16 10:25:00",
        "hostname": "node01"
      }
    ]
  }
}
```

#### 2.4.2 获取告警趋势统计

```java
@RequestMapping("/alertTrend")
public Result getAlertTrend(Integer clusterId, String timeRange) {
    TimeRange range = TimeRangeParser.parse(timeRange);
    
    // 查询时间范围内的告警数据
    List<ClusterAlertHistory> alerts = alertHistoryService.getAlertsByTimeRange(
        clusterId,
        range.getStartTime(),
        range.getEndTime()
    );
    
    // 按时间分组统计
    Map<String, Long> trendMap = alerts.stream()
        .collect(Collectors.groupingBy(
            alert -> formatTime(alert.getCreateTime(), "yyyy-MM-dd HH:00"),
            Collectors.counting()
        ));
    
    // 按严重级别统计
    Map<String, Long> severityMap = alerts.stream()
        .collect(Collectors.groupingBy(
            ClusterAlertHistory::getSeverity,
            Collectors.counting()
        ));
    
    Map<String, Object> result = new HashMap<>();
    result.put("trendData", trendMap);
    result.put("severityStats", severityMap);
    result.put("totalAlerts", alerts.size());
    
    return Result.success(result);
}
```

**使用场景**:
- 告警趋势分析
- 识别告警高发时段
- 评估告警规则合理性

### 2.5 实时数据推送

#### 2.5.1 获取实时监控数据

```java
@RequestMapping("/realtimeMetrics")
public Result getRealtimeMetrics(Integer clusterId) {
    // 获取最近 5 分钟的实时数据
    Long endTime = System.currentTimeMillis();
    Long startTime = endTime - 5 * 60 * 1000;
    
    // 查询关键指标的实时数据
    Map<String, Object> realtimeData = new HashMap<>();
    
    // CPU 使用率
    realtimeData.put("cpuUsage", prometheusService.queryLatestMetric(
        clusterId, "cpu_usage_percent"
    ));
    
    // 内存使用率
    realtimeData.put("memoryUsage", prometheusService.queryLatestMetric(
        clusterId, "memory_usage_percent"
    ));
    
    // 磁盘 IO
    realtimeData.put("diskIO", prometheusService.queryLatestMetric(
        clusterId, "disk_io_util_percent"
    ));
    
    // 网络流量
    realtimeData.put("networkTraffic", prometheusService.queryLatestMetric(
        clusterId, "network_total_bytes_per_sec"
    ));
    
    return Result.success(realtimeData);
}
```

**功能说明**:
- 提供最新的监控数据快照
- 适用于实时监控大盘
- 支持自动刷新机制

## 三、设计模式与架构

### 3.1 数据聚合模式

监控大盘 Controller 采用 **数据聚合模式**，从多个数据源聚合数据：

```
┌─────────────────────────────────────┐
│  ClusterServiceDashboardController  │
└─────────────┬───────────────────────┘
              │
              ├─→ ClusterInfoService (集群信息)
              ├─→ ClusterHostService (主机信息)
              ├─→ ServiceInstanceService (服务信息)
              ├─→ AlertHistoryService (告警信息)
              └─→ PrometheusService (监控指标)
                          │
                          ↓
                  ┌───────────────┐
                  │  Prometheus   │
                  │  Time Series  │
                  │   Database    │
                  └───────────────┘
```

**优势**:
1. **单一入口**: 前端只需调用一个接口获取完整数据
2. **减少请求**: 减少前后端交互次数
3. **统一处理**: 统一的数据格式转换和异常处理
4. **易于维护**: 业务逻辑集中管理

### 3.2 缓存策略

监控数据查询频繁，采用多级缓存策略：

```java
@Service
public class DashboardCacheService {
    
    @Cacheable(value = "dashboard:overview", key = "#clusterId", 
               unless = "#result == null")
    public Map<String, Object> getCachedOverview(Integer clusterId) {
        // 缓存 5 分钟
        return buildOverviewData(clusterId);
    }
    
    @Cacheable(value = "dashboard:metrics", 
               key = "#clusterId + ':' + #metricName",
               unless = "#result == null")
    public List<MetricData> getCachedMetrics(Integer clusterId, String metricName) {
        // 缓存 1 分钟
        return queryMetricsFromPrometheus(clusterId, metricName);
    }
}
```

**缓存配置**:

| 数据类型 | 缓存时间 | 更新策略 | 说明 |
|---------|---------|---------|------|
| 集群概览 | 5 分钟 | 定时刷新 | 变化较慢，可以较长缓存 |
| 实时指标 | 30 秒 | 定时刷新 | 实时性要求高 |
| 历史数据 | 30 分钟 | 按需加载 | 历史数据不变 |
| 告警列表 | 1 分钟 | 主动失效 | 新告警触发时清除缓存 |

### 3.3 异步查询优化

对于耗时的查询操作，采用异步并行处理：

```java
@Service
public class AsyncDashboardService {
    
    @Async("dashboardExecutor")
    public CompletableFuture<Map<String, Object>> getServiceStatsAsync(Integer clusterId) {
        Map<String, Object> stats = serviceInstanceService.getServiceStats(clusterId);
        return CompletableFuture.completedFuture(stats);
    }
    
    @Async("dashboardExecutor")
    public CompletableFuture<Map<String, Object>> getHostStatsAsync(Integer clusterId) {
        Map<String, Object> stats = clusterHostService.getHostStats(clusterId);
        return CompletableFuture.completedFuture(stats);
    }
    
    public Map<String, Object> getClusterOverviewParallel(Integer clusterId) {
        // 并行执行多个查询
        CompletableFuture<Map<String, Object>> serviceFuture = getServiceStatsAsync(clusterId);
        CompletableFuture<Map<String, Object>> hostFuture = getHostStatsAsync(clusterId);
        CompletableFuture<Map<String, Object>> alertFuture = getAlertStatsAsync(clusterId);
        
        // 等待所有查询完成
        CompletableFuture.allOf(serviceFuture, hostFuture, alertFuture).join();
        
        // 组装结果
        Map<String, Object> result = new HashMap<>();
        result.put("serviceStats", serviceFuture.join());
        result.put("hostStats", hostFuture.join());
        result.put("alertStats", alertFuture.join());
        
        return result;
    }
}
```

**线程池配置**:
```java
@Configuration
public class AsyncConfig implements AsyncConfigurer {
    
    @Bean(name = "dashboardExecutor")
    public Executor dashboardExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("dashboard-");
        executor.initialize();
        return executor;
    }
}
```

## 四、性能优化

### 4.1 查询优化

#### 4.1.1 时间范围限制

```java
private static final long MAX_QUERY_RANGE = 30 * 24 * 60 * 60 * 1000L; // 30 天

private void validateTimeRange(Long startTime, Long endTime) {
    if (endTime - startTime > MAX_QUERY_RANGE) {
        throw new ApiException("查询时间范围不能超过 30 天");
    }
}
```

#### 4.1.2 数据降采样

对于长时间范围的查询，自动进行数据降采样：

```java
private List<MetricData> downsampleIfNeeded(
    List<MetricData> rawData,
    Long timeRange
) {
    // 超过 24 小时的数据，按 5 分钟聚合
    if (timeRange > 24 * 60 * 60 * 1000L) {
        return downsample(rawData, 5 * 60 * 1000L);
    }
    
    // 超过 7 天的数据，按 1 小时聚合
    if (timeRange > 7 * 24 * 60 * 60 * 1000L) {
        return downsample(rawData, 60 * 60 * 1000L);
    }
    
    return rawData;
}
```

### 4.2 批量查询优化

```java
@RequestMapping("/batchMetrics")
public Result getBatchMetrics(@RequestBody BatchMetricsRequest request) {
    // 批量查询多个服务的多个指标
    List<MetricQuery> queries = request.getQueries();
    
    // 按 Prometheus 数据源分组
    Map<String, List<MetricQuery>> groupedQueries = groupByPrometheusInstance(queries);
    
    // 并行查询每个 Prometheus 实例
    Map<String, List<MetricData>> results = groupedQueries.entrySet()
        .parallelStream()
        .collect(Collectors.toMap(
            Map.Entry::getKey,
            entry -> prometheusService.batchQuery(entry.getValue())
        ));
    
    return Result.success(results);
}
```

### 4.3 响应数据压缩

对于大数据量响应，启用 GZIP 压缩：

```java
@Configuration
public class CompressionConfig {
    
    @Bean
    public FilterRegistrationBean<GzipFilter> gzipFilter() {
        FilterRegistrationBean<GzipFilter> registration = new FilterRegistrationBean<>();
        registration.setFilter(new GzipFilter());
        registration.addUrlPatterns("/api/dashboard/*");
        return registration;
    }
}
```

## 五、安全性考虑

### 5.1 权限控制

```java
@RequestMapping("/clusterOverview")
@RequiresPermissions("dashboard:view")
public Result getClusterOverview(Integer clusterId) {
    // 验证用户是否有该集群的访问权限
    if (!permissionService.hasClusterPermission(clusterId)) {
        throw new ForbiddenException("无权访问该集群");
    }
    
    // ... 业务逻辑
}
```

### 5.2 参数验证

```java
private void validateMetricsRequest(
    Integer clusterId,
    String metricName,
    Long startTime,
    Long endTime
) {
    // 验证集群 ID
    if (clusterId == null || clusterId <= 0) {
        throw new ApiException("无效的集群 ID");
    }
    
    // 验证指标名称
    if (!isValidMetricName(metricName)) {
        throw new ApiException("不支持的指标名称");
    }
    
    // 验证时间范围
    if (startTime >= endTime) {
        throw new ApiException("开始时间必须小于结束时间");
    }
    
    validateTimeRange(startTime, endTime);
}
```

### 5.3 敏感数据脱敏

```java
private Map<String, Object> maskSensitiveData(Map<String, Object> data) {
    // 脱敏敏感配置信息
    if (data.containsKey("configDetails")) {
        Map<String, String> configs = (Map<String, String>) data.get("configDetails");
        configs.replaceAll((k, v) -> {
            if (isSensitiveKey(k)) {
                return "******";
            }
            return v;
        });
    }
    
    return data;
}
```

## 六、监控指标设计

### 6.1 HDFS 监控指标

```java
public class HDFSMetrics {
    // 容量指标
    public static final String HDFS_CAPACITY_TOTAL = "hdfs_capacity_total_bytes";
    public static final String HDFS_CAPACITY_USED = "hdfs_capacity_used_bytes";
    public static final String HDFS_CAPACITY_REMAINING = "hdfs_capacity_remaining_bytes";
    
    // 文件系统指标
    public static final String HDFS_FILES_TOTAL = "hdfs_files_total";
    public static final String HDFS_BLOCKS_TOTAL = "hdfs_blocks_total";
    public static final String HDFS_CORRUPT_BLOCKS = "hdfs_blocks_corrupt_total";
    public static final String HDFS_MISSING_BLOCKS = "hdfs_blocks_missing_total";
    
    // 性能指标
    public static final String HDFS_READ_BYTES = "hdfs_bytes_read";
    public static final String HDFS_WRITE_BYTES = "hdfs_bytes_written";
    
    // DataNode 指标
    public static final String HDFS_DATANODES_LIVE = "hdfs_datanodes_live";
    public static final String HDFS_DATANODES_DEAD = "hdfs_datanodes_dead";
}
```

### 6.2 YARN 监控指标

```java
public class YARNMetrics {
    // 集群资源
    public static final String YARN_MEMORY_TOTAL = "yarn_memory_total_mb";
    public static final String YARN_MEMORY_USED = "yarn_memory_used_mb";
    public static final String YARN_VCORES_TOTAL = "yarn_vcores_total";
    public static final String YARN_VCORES_USED = "yarn_vcores_used";
    
    // 应用程序
    public static final String YARN_APPS_RUNNING = "yarn_apps_running";
    public static final String YARN_APPS_PENDING = "yarn_apps_pending";
    public static final String YARN_APPS_FAILED = "yarn_apps_failed";
    
    // 容器
    public static final String YARN_CONTAINERS_RUNNING = "yarn_containers_running";
    public static final String YARN_CONTAINERS_PENDING = "yarn_containers_pending";
    
    // 队列
    public static final String YARN_QUEUE_MEMORY_USED = "yarn_queue_memory_used_mb";
    public static final String YARN_QUEUE_APPS_RUNNING = "yarn_queue_apps_running";
}
```

### 6.3 HBase 监控指标

```java
public class HBaseMetrics {
    // RegionServer
    public static final String HBASE_REGIONS_COUNT = "hbase_regions_count";
    public static final String HBASE_REQUESTS_COUNT = "hbase_requests_total";
    public static final String HBASE_READ_REQUESTS = "hbase_read_requests_total";
    public static final String HBASE_WRITE_REQUESTS = "hbase_write_requests_total";
    
    // 存储
    public static final String HBASE_STORE_FILES = "hbase_store_files_count";
    public static final String HBASE_STORE_FILE_SIZE = "hbase_store_file_size_bytes";
    public static final String HBASE_MEMSTORE_SIZE = "hbase_memstore_size_bytes";
    
    // 压缩
    public static final String HBASE_COMPACTION_QUEUE = "hbase_compaction_queue_length";
    public static final String HBASE_FLUSH_QUEUE = "hbase_flush_queue_length";
}
```

## 七、使用示例

### 7.1 前端集成示例

#### 7.1.1 Vue.js 组件示例

```javascript
<template>
  <div class="dashboard-container">
    <el-row :gutter="20">
      <!-- 集群概览卡片 -->
      <el-col :span="24">
        <el-card class="overview-card">
          <div slot="header">
            <span>集群概览</span>
            <el-button style="float: right" type="text" @click="refreshOverview">
              刷新
            </el-button>
          </div>
          <div class="overview-content">
            <div class="stat-item">
              <div class="stat-label">总服务数</div>
              <div class="stat-value">{{ overview.serviceStats.totalServices }}</div>
            </div>
            <div class="stat-item">
              <div class="stat-label">运行服务</div>
              <div class="stat-value success">{{ overview.serviceStats.runningServices }}</div>
            </div>
            <div class="stat-item">
              <div class="stat-label">总主机数</div>
              <div class="stat-value">{{ overview.hostStats.totalHosts }}</div>
            </div>
            <div class="stat-item">
              <div class="stat-label">活跃告警</div>
              <div class="stat-value warning">{{ overview.alertStats.activeAlerts }}</div>
            </div>
          </div>
        </el-card>
      </el-col>
      
      <!-- 监控图表 -->
      <el-col :span="12">
        <el-card>
          <div slot="header">CPU 使用率趋势</div>
          <v-chart :options="cpuChartOptions" autoresize />
        </el-card>
      </el-col>
      
      <el-col :span="12">
        <el-card>
          <div slot="header">内存使用率趋势</div>
          <v-chart :options="memoryChartOptions" autoresize />
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
export default {
  name: 'ClusterDashboard',
  data() {
    return {
      clusterId: this.$route.params.clusterId,
      overview: {
        serviceStats: {},
        hostStats: {},
        alertStats: {}
      },
      cpuChartOptions: {},
      memoryChartOptions: {}
    }
  },
  mounted() {
    this.loadDashboardData()
    this.startAutoRefresh()
  },
  methods: {
    async loadDashboardData() {
      // 加载集群概览
      const overviewRes = await this.$api.dashboard.getClusterOverview({
        clusterId: this.clusterId
      })
      this.overview = overviewRes.data
      
      // 加载 CPU 趋势
      const cpuRes = await this.$api.dashboard.getResourceTrend({
        clusterId: this.clusterId,
        resourceType: 'cpu',
        timeRange: '24h'
      })
      this.updateCpuChart(cpuRes.data)
      
      // 加载内存趋势
      const memRes = await this.$api.dashboard.getResourceTrend({
        clusterId: this.clusterId,
        resourceType: 'memory',
        timeRange: '24h'
      })
      this.updateMemoryChart(memRes.data)
    },
    
    updateCpuChart(data) {
      this.cpuChartOptions = {
        xAxis: {
          type: 'time',
          data: data.trendData.map(d => d.timestamp)
        },
        yAxis: {
          type: 'value',
          max: 100,
          axisLabel: {
            formatter: '{value} %'
          }
        },
        series: [{
          name: 'CPU 使用率',
          type: 'line',
          data: data.trendData.map(d => d.avgValue),
          smooth: true
        }]
      }
    },
    
    startAutoRefresh() {
      // 每 30 秒自动刷新
      this.refreshTimer = setInterval(() => {
        this.loadDashboardData()
      }, 30000)
    }
  },
  beforeDestroy() {
    if (this.refreshTimer) {
      clearInterval(this.refreshTimer)
    }
  }
}
</script>
```

#### 7.1.2 API 封装

```javascript
// api/dashboard.js
export default {
  // 获取集群概览
  getClusterOverview(params) {
    return request({
      url: '/api/dashboard/clusterOverview',
      method: 'get',
      params
    })
  },
  
  // 获取服务监控指标
  getServiceMetrics(params) {
    return request({
      url: '/api/dashboard/serviceMetrics',
      method: 'get',
      params
    })
  },
  
  // 获取资源趋势
  getResourceTrend(params) {
    return request({
      url: '/api/dashboard/resourceTrend',
      method: 'get',
      params
    })
  },
  
  // 获取活跃告警
  getActiveAlerts(params) {
    return request({
      url: '/api/dashboard/activeAlerts',
      method: 'get',
      params
    })
  }
}
```

### 7.2 命令行工具示例

```bash
# 使用 curl 查询集群概览
curl -X GET "http://localhost:8080/api/dashboard/clusterOverview?clusterId=1" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json"

# 查询 HDFS 监控指标
curl -X GET "http://localhost:8080/api/dashboard/serviceMetrics" \
  -H "Authorization: Bearer ${TOKEN}" \
  -d "clusterId=1" \
  -d "serviceName=HDFS" \
  -d "metricName=hdfs_capacity_used_bytes" \
  -d "startTime=1700100000000" \
  -d "endTime=1700200000000"
```

## 八、最佳实践

### 8.1 监控大盘设计原则

1. **关键指标优先**: 首屏展示最关键的监控指标
2. **层次化展示**: 从集群概览到服务详情，层层深入
3. **实时更新**: 关键指标实时刷新，历史数据按需加载
4. **告警突出**: 异常状态和告警信息醒目显示
5. **交互友好**: 支持钻取、筛选、时间范围选择等交互

### 8.2 性能优化建议

1. **合理使用缓存**: 根据数据特点设置不同的缓存策略
2. **异步并行查询**: 多个独立查询并行执行
3. **数据降采样**: 长时间范围查询自动降采样
4. **分页加载**: 大数据量列表使用分页
5. **前端缓存**: 利用浏览器缓存减少请求

### 8.3 告警配置建议

```yaml
# 监控大盘相关告警规则
alerts:
  - name: dashboard_query_slow
    expr: histogram_quantile(0.95, dashboard_query_duration_seconds) > 2
    severity: warning
    description: "监控大盘查询响应时间超过 2 秒"
    
  - name: prometheus_query_error
    expr: rate(prometheus_query_errors_total[5m]) > 0.1
    severity: critical
    description: "Prometheus 查询错误率超过 10%"
```

## 九、总结

### 9.1 核心功能

ClusterServiceDashboardController 提供了完整的监控大盘数据接口：

1. ✅ **集群概览**: 快速了解集群整体状况
2. ✅ **监控指标**: 详细的性能指标查询
3. ✅ **趋势分析**: 资源使用趋势和对比
4. ✅ **告警展示**: 实时告警信息展示
5. ✅ **性能优化**: 缓存、异步、降采样等优化手段

### 9.2 技术亮点

1. **数据聚合**: 多数据源统一聚合
2. **异步优化**: 并行查询提升性能
3. **缓存策略**: 多级缓存减少查询压力
4. **降采样**: 智能数据降采样
5. **实时刷新**: 支持实时数据更新

### 9.3 扩展方向

1. **自定义大盘**: 支持用户自定义监控大盘
2. **智能分析**: 基于历史数据的智能异常检测
3. **对比分析**: 多集群、多时段对比分析
4. **导出功能**: 支持监控数据和图表导出
5. **移动端适配**: 提供移动端监控大盘

---

**文档版本**: v1.0  
**最后更新**: 2025-11-16  
**维护团队**: DataSophon 源码分析团队
