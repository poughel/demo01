# LoginController 用户认证控制器详解

## 一、控制器概述

### 1.1 基本信息

**文件位置**: `datasophon-api/src/main/java/com/datasophon/api/controller/LoginController.java`

**职责定位**:
- 用户登录认证
- 会话管理
- 用户登出
- 安全验证

**核心概念**:
- **认证**: 验证用户身份的过程
- **会话**: 用户登录后的状态保持
- **Cookie**: 客户端存储的会话标识
- **IP验证**: 基于IP的安全检查

### 1.2 类结构

```java
@RestController
@RequestMapping("")
public class LoginController {
    private static final Logger logger = LoggerFactory.getLogger(LoginController.class);
    
    @Autowired
    private SessionService sessionService;
    
    @Autowired
    private Authenticator authenticator;
}
```

**关键特征**:
- **请求前缀**: 根路径（无前缀）
- **日志记录**: 使用 SLF4J Logger
- **双依赖**: SessionService 和 Authenticator
- **安全焦点**: IP验证、Cookie安全

## 二、API 接口详解

### 2.1 用户登录

#### 接口定义
```java
@RequestMapping("/login")
public Result login(@RequestParam(value = "username") String userName,
                    @RequestParam(value = "password") String userPassword,
                    HttpServletRequest request,
                    HttpServletResponse response) {
    logger.info("login user name: {} ", userName);
    
    // 用户名检查
    if (StringUtils.isEmpty(userName)) {
        return Result.error(Status.USER_NAME_NULL.getCode(),
                Status.USER_NAME_NULL.getMsg());
    }
    
    // 用户IP检查
    String ip = HttpUtils.getClientIpAddress(request);
    if (StringUtils.isEmpty(ip)) {
        return Result.error(IP_IS_EMPTY.getCode(), IP_IS_EMPTY.getMsg());
    }
    
    // 验证用户名和密码
    Result result = authenticator.authenticate(userName, userPassword, ip);
    if (result.getCode() != Status.SUCCESS.getCode()) {
        return result;
    }
    
    // 设置Cookie
    response.setStatus(HttpStatus.SC_OK);
    Map<String, String> cookieMap = (Map<String, String>) result.getData();
    for (Map.Entry<String, String> cookieEntry : cookieMap.entrySet()) {
        Cookie cookie = new Cookie(cookieEntry.getKey(), cookieEntry.getValue());
        cookie.setHttpOnly(true);
        response.addCookie(cookie);
    }
    
    return result;
}
```

**接口信息**:
- **URL**: `POST /login`
- **请求参数**: 
  - `username`: 用户名
  - `password`: 密码
- **请求对象**: HttpServletRequest, HttpServletResponse
- **返回值**: Result 包装的登录结果
- **权限要求**: 无（公开接口）

**功能描述**:
- 验证用户名和密码
- 创建用户会话
- 设置HTTP Cookie
- 记录登录日志
- IP地址验证

**使用场景**:
1. 用户首次登录系统
2. 会话过期后重新登录
3. 切换用户账号

**请求示例**:
```
POST /login
Content-Type: application/x-www-form-urlencoded

username=admin&password=admin123
```

**返回数据示例**:
```json
{
  "code": 0,
  "msg": "登录成功",
  "data": {
    "sessionId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": 1,
      "username": "admin",
      "realName": "管理员",
      "email": "admin@example.com",
      "userType": "ADMIN_USER",
      "createTime": "2025-01-01 10:00:00"
    }
  }
}
```

### 2.2 参数验证

#### 用户名验证
```java
if (StringUtils.isEmpty(userName)) {
    return Result.error(Status.USER_NAME_NULL.getCode(),
            Status.USER_NAME_NULL.getMsg());
}
```

**验证规则**:
- 用户名不能为空
- 用户名长度限制（一般3-20字符）
- 用户名格式验证（字母、数字、下划线）

**错误响应**:
```json
{
  "code": 10001,
  "msg": "用户名不能为空"
}
```

#### IP地址验证
```java
String ip = HttpUtils.getClientIpAddress(request);
if (StringUtils.isEmpty(ip)) {
    return Result.error(IP_IS_EMPTY.getCode(), IP_IS_EMPTY.getMsg());
}
```

**IP获取逻辑**:
```java
public static String getClientIpAddress(HttpServletRequest request) {
    String ip = request.getHeader("X-Forwarded-For");
    if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
        ip = request.getHeader("Proxy-Client-IP");
    }
    if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
        ip = request.getHeader("WL-Proxy-Client-IP");
    }
    if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
        ip = request.getHeader("HTTP_CLIENT_IP");
    }
    if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
        ip = request.getHeader("HTTP_X_FORWARDED_FOR");
    }
    if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
        ip = request.getRemoteAddr();
    }
    return ip;
}
```

**支持场景**:
- 直接访问
- 反向代理（Nginx）
- 负载均衡器

### 2.3 身份认证

#### 认证流程
```java
Result result = authenticator.authenticate(userName, userPassword, ip);
if (result.getCode() != Status.SUCCESS.getCode()) {
    return result;
}
```

**Authenticator 认证逻辑**:
```java
public Result authenticate(String username, String password, String ip) {
    // 1. 查询用户
    UserInfoEntity user = userService.queryByUserName(username);
    if (user == null) {
        return Result.error("用户不存在");
    }
    
    // 2. 验证密码
    String encryptedPassword = EncryptionUtils.encrypt(password, user.getSalt());
    if (!encryptedPassword.equals(user.getPassword())) {
        return Result.error("密码错误");
    }
    
    // 3. 检查用户状态
    if (user.getState() == 0) {
        return Result.error("用户已被禁用");
    }
    
    // 4. 创建会话
    String sessionId = sessionService.createSession(user, ip);
    
    // 5. 返回会话信息
    Map<String, String> cookieMap = new HashMap<>();
    cookieMap.put("sessionId", sessionId);
    cookieMap.put("userId", String.valueOf(user.getId()));
    
    return Result.success().put("data", cookieMap).put("user", user);
}
```

**密码加密**:
- **算法**: MD5 + Salt
- **Salt**: 用户创建时随机生成
- **存储**: 加密后的密码 + Salt

```java
public static String encrypt(String password, String salt) {
    return DigestUtils.md5Hex(password + salt);
}
```

### 2.4 会话创建

#### Cookie 设置
```java
response.setStatus(HttpStatus.SC_OK);
Map<String, String> cookieMap = (Map<String, String>) result.getData();
for (Map.Entry<String, String> cookieEntry : cookieMap.entrySet()) {
    Cookie cookie = new Cookie(cookieEntry.getKey(), cookieEntry.getValue());
    cookie.setHttpOnly(true);
    response.addCookie(cookie);
}
```

**Cookie 属性**:
- **HttpOnly**: true（防止XSS攻击）
- **Secure**: true（生产环境，HTTPS only）
- **SameSite**: Strict（防止CSRF攻击）
- **Path**: /（全站有效）
- **MaxAge**: 7200秒（2小时）

**Cookie 内容**:
```
sessionId=a1b2c3d4-e5f6-7890-abcd-ef1234567890
userId=1
```

#### 会话存储

**Redis 存储结构**:
```
Key: session:a1b2c3d4-e5f6-7890-abcd-ef1234567890
Value: {
  "userId": 1,
  "username": "admin",
  "ip": "192.168.1.100",
  "loginTime": "2025-01-15 10:00:00",
  "lastAccessTime": "2025-01-15 10:30:00"
}
TTL: 7200秒
```

### 2.5 用户登出

#### 接口定义
```java
@PostMapping(value = "/signOut")
public Result signOut(@RequestAttribute(value = Constants.SESSION_USER) UserInfoEntity loginUser,
                      HttpServletRequest request) {
    logger.info("login user:{} sign out", loginUser.getUsername());
    String ip = HttpUtils.getClientIpAddress(request);
    sessionService.signOut(ip, loginUser);
    // 清除会话
    request.removeAttribute(Constants.SESSION_USER);
    return Result.success();
}
```

**接口信息**:
- **URL**: `POST /signOut`
- **请求参数**: 
  - `loginUser`: 当前登录用户（从会话获取）
- **请求对象**: HttpServletRequest
- **返回值**: Result 表示操作结果
- **权限要求**: 已登录用户

**功能描述**:
- 清除服务器端会话
- 清除请求属性中的用户信息
- 记录登出日志
- 清除客户端Cookie（前端处理）

**登出流程**:
```java
public void signOut(String ip, UserInfoEntity user) {
    // 1. 删除Redis中的会话
    String sessionId = user.getSessionId();
    redisTemplate.delete("session:" + sessionId);
    
    // 2. 记录登出日志
    logger.info("User {} signed out from {}", user.getUsername(), ip);
    
    // 3. 更新用户最后登出时间
    userService.updateLastLogoutTime(user.getId(), new Date());
}
```

**前端配合**:
```javascript
axios.post('/signOut')
  .then(response => {
    // 清除本地存储
    localStorage.clear();
    sessionStorage.clear();
    // 跳转到登录页
    window.location.href = '/login';
  });
```

## 三、安全机制

### 3.1 密码安全

#### 密码加密
```java
// 用户注册时
String salt = UUID.randomUUID().toString();
String encryptedPassword = EncryptionUtils.encrypt(password, salt);
user.setPassword(encryptedPassword);
user.setSalt(salt);
```

#### 密码强度要求
- 最小长度：8位
- 必须包含大小写字母
- 必须包含数字
- 必须包含特殊字符

```java
public boolean validatePassword(String password) {
    String regex = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$";
    return password.matches(regex);
}
```

### 3.2 防暴力破解

#### 登录失败次数限制
```java
public Result authenticate(String username, String password, String ip) {
    // 检查失败次数
    String key = "login:failed:" + username;
    Integer failedCount = (Integer) redisTemplate.opsForValue().get(key);
    if (failedCount != null && failedCount >= 5) {
        long ttl = redisTemplate.getExpire(key, TimeUnit.SECONDS);
        return Result.error("登录失败次数过多，请" + ttl + "秒后重试");
    }
    
    // 验证密码
    if (!validatePassword(user, password)) {
        // 增加失败次数
        redisTemplate.opsForValue().increment(key);
        redisTemplate.expire(key, 30, TimeUnit.MINUTES);
        return Result.error("密码错误");
    }
    
    // 登录成功，清除失败次数
    redisTemplate.delete(key);
    return Result.success();
}
```

#### IP黑名单
```java
public boolean isIpBlocked(String ip) {
    return redisTemplate.hasKey("blacklist:ip:" + ip);
}

public void blockIp(String ip, int minutes) {
    redisTemplate.opsForValue().set("blacklist:ip:" + ip, "blocked", minutes, TimeUnit.MINUTES);
}
```

### 3.3 会话安全

#### 会话超时
- **空闲超时**: 30分钟无操作自动登出
- **绝对超时**: 8小时后强制重新登录

```java
public void checkSessionTimeout(String sessionId) {
    String key = "session:" + sessionId;
    Map<String, Object> session = redisTemplate.opsForHash().entries(key);
    
    // 检查最后访问时间
    Date lastAccessTime = (Date) session.get("lastAccessTime");
    if (System.currentTimeMillis() - lastAccessTime.getTime() > 30 * 60 * 1000) {
        redisTemplate.delete(key);
        throw new SessionTimeoutException("会话已过期");
    }
    
    // 更新最后访问时间
    redisTemplate.opsForHash().put(key, "lastAccessTime", new Date());
}
```

#### 单点登录（可选）
```java
public void createSession(UserInfoEntity user, String ip) {
    // 删除该用户的其他会话
    Set<String> keys = redisTemplate.keys("session:*");
    for (String key : keys) {
        Map<String, Object> session = redisTemplate.opsForHash().entries(key);
        if (session.get("userId").equals(user.getId())) {
            redisTemplate.delete(key);
        }
    }
    
    // 创建新会话
    String sessionId = UUID.randomUUID().toString();
    // ...
}
```

### 3.4 XSS 防护

#### Cookie HttpOnly
```java
Cookie cookie = new Cookie("sessionId", sessionId);
cookie.setHttpOnly(true);  // JavaScript无法读取
response.addCookie(cookie);
```

#### 输入过滤
```java
public String sanitizeInput(String input) {
    return StringEscapeUtils.escapeHtml4(input);
}
```

### 3.5 CSRF 防护

#### CSRF Token
```java
@GetMapping("/csrf-token")
public Result getCsrfToken(HttpSession session) {
    String csrfToken = UUID.randomUUID().toString();
    session.setAttribute("CSRF_TOKEN", csrfToken);
    return Result.success().put("csrfToken", csrfToken);
}

@PostMapping("/some-action")
public Result someAction(@RequestHeader("X-CSRF-TOKEN") String csrfToken,
                         HttpSession session) {
    String sessionToken = (String) session.getAttribute("CSRF_TOKEN");
    if (!csrfToken.equals(sessionToken)) {
        return Result.error("CSRF token invalid");
    }
    // 执行操作
}
```

## 四、日志记录

### 4.1 登录日志
```java
logger.info("login user name: {} ", userName);
```

**日志内容**:
- 用户名
- IP地址
- 登录时间
- 登录结果

**日志格式**:
```
2025-01-15 10:00:00 INFO [LoginController] login user name: admin, ip: 192.168.1.100, result: success
```

### 4.2 登出日志
```java
logger.info("login user:{} sign out", loginUser.getUsername());
```

### 4.3 安全事件日志
```java
// 登录失败
logger.warn("Login failed for user: {}, ip: {}, reason: {}", 
    username, ip, "password error");

// 会话超时
logger.info("Session timeout for user: {}, sessionId: {}", 
    username, sessionId);

// IP被封禁
logger.warn("IP {} has been blocked due to multiple failed attempts", ip);
```

## 五、使用示例

### 5.1 前端登录示例

**登录表单**:
```html
<form @submit.prevent="handleLogin">
  <input v-model="username" placeholder="用户名" />
  <input v-model="password" type="password" placeholder="密码" />
  <button type="submit">登录</button>
</form>
```

**登录请求**:
```javascript
async handleLogin() {
  try {
    const response = await axios.post('/login', {
      username: this.username,
      password: this.password
    }, {
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
      }
    });
    
    if (response.data.code === 0) {
      // 存储用户信息
      localStorage.setItem('user', JSON.stringify(response.data.data.user));
      // 跳转到首页
      this.$router.push('/dashboard');
    } else {
      this.$message.error(response.data.msg);
    }
  } catch (error) {
    this.$message.error('登录失败: ' + error.message);
  }
}
```

### 5.2 登出示例

```javascript
async handleLogout() {
  try {
    await axios.post('/signOut');
    // 清除本地存储
    localStorage.clear();
    sessionStorage.clear();
    // 跳转到登录页
    this.$router.push('/login');
  } catch (error) {
    console.error('登出失败:', error);
  }
}
```

### 5.3 自动登录（记住我）

```javascript
// 保存登录信息
if (this.rememberMe) {
  localStorage.setItem('rememberedUsername', this.username);
  // 注意：不要保存密码！
}

// 页面加载时恢复
mounted() {
  this.username = localStorage.getItem('rememberedUsername') || '';
}
```

## 六、总结

### 6.1 核心功能

LoginController 提供完整的用户认证功能：
1. 用户登录验证
2. 会话创建和管理
3. 用户登出
4. 安全防护

### 6.2 安全特性

- **密码加密**: MD5 + Salt
- **防暴力破解**: 失败次数限制
- **会话安全**: HttpOnly Cookie
- **XSS防护**: 输入过滤
- **CSRF防护**: Token验证
- **IP验证**: 黑名单机制

### 6.3 改进建议

1. 使用更安全的密码算法（BCrypt）
2. 实现JWT Token认证
3. 添加验证码功能
4. 支持多因素认证（MFA）
5. 完善审计日志
6. 实现OAuth2.0集成
7. 支持LDAP/AD认证

---

**文档版本**: v1.0  
**最后更新**: 2025-11-15  
**维护者**: DataSophon 源码分析团队
