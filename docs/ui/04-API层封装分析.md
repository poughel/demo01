# DataSophon UI 模块 - API 层封装分析

## 一、API 层概述

DataSophon UI 的 API 层采用**分层封装 + 拦截器 + 统一处理**的设计模式，基于 Axios 实现 HTTP 请求管理，提供了清晰的 API 调用接口和完善的错误处理机制。

### 核心特性

- ✅ **分层架构**: request → httpApi → services 三层封装
- ✅ **请求拦截**: 统一添加认证信息、处理跨域
- ✅ **响应拦截**: 统一错误处理、数据格式化
- ✅ **认证管理**: Cookie-based 会话管理
- ✅ **类型封装**: 支持 GET、POST、JSON POST、文件上传
- ✅ **Promise 化**: 所有请求返回 Promise
- ✅ **环境适配**: 开发/生产环境自动切换

## 二、API 层文件结构

```
src/api/
├── baseUrl.js              # API 基础地址配置
├── request.js              # 底层请求封装
├── interceptors.js         # 请求/响应拦截器
├── index.js                # API 统一导出
├── httpApi/                # HTTP API 定义
│   ├── cluster.js          # 集群管理 API
│   ├── host.js             # 主机管理 API
│   ├── services.js         # 服务管理 API
│   ├── system.js           # 系统管理 API
│   ├── user.js             # 用户管理 API
│   └── index.js            # API 统一导出
└── services/               # 业务服务封装
    ├── api.js              # 通用 API 服务
    ├── user.js             # 用户服务
    ├── dataSource.js       # 数据源服务
    └── index.js            # 服务统一导出
```

## 三、核心文件详细分析

### 3.1 baseUrl.js - API 基础地址

#### 文件路径
`src/api/baseUrl.js`

#### 源码分析

```javascript
export default {
  path() {
    let path = ''
    // 根据环境返回不同的 API 基础路径
    // if(process.env.NODE_ENV === 'production') {
    //   path = process.env.VUE_APP_API_BASE_URL
    // }
    return path
  }
}
```

**设计说明**:
- 提供统一的 API 基础路径
- 支持环境变量配置
- 开发环境通过代理访问，返回空字符串
- 生产环境可配置完整 URL

**使用方式**:
```javascript
import paths from '@/api/baseUrl'

let path = paths.path() + '/ddh'
// 开发环境: '/ddh'
// 生产环境: 'http://api.example.com/ddh'
```

### 3.2 request.js - 底层请求封装

#### 文件路径
`src/api/request.js`

#### 核心功能

##### 1. FormData 处理

```javascript
/**
 * 处理 POST 数据为 FormData 格式
 * @param data 请求数据对象
 * @returns {FormData}
 */
const handleParams = function(data) {
  const params = new FormData()
  for(let key in data) {
    params.append(key, data[key])
  }
  return params
}
```

**作用**:
- 将对象转换为 FormData 格式
- 支持文件上传
- 兼容传统表单提交

##### 2. GET 请求封装

```javascript
/**
 * GET 请求
 * @param url 请求地址
 * @param params 查询参数
 * @returns {Promise}
 */
const axiosGet = function(url, params = {}) {
  return new Promise((resolve, reject) => {
    axios({
      method: 'get',
      url: url,
      params: params,
    }).then(res => {
      resolve(res.data)
    }).catch(error => {
      reject(error)
    })
  })
}
```

**特点**:
- Promise 封装
- 自动提取 `res.data`
- 统一错误处理

**使用示例**:
```javascript
this.$axiosGet('/api/cluster/list', {
  page: 1,
  size: 10
}).then(data => {
  console.log(data)
}).catch(error => {
  console.error(error)
})
```

##### 3. POST 请求封装（FormData）

```javascript
/**
 * POST 请求（FormData 格式）
 * @param url 请求地址
 * @param params 请求参数
 * @returns {Promise}
 */
const axiosPost = function(url, params = {}) {
  return new Promise((resolve, reject) => {
    axios({
      method: 'post',
      url: url,
      data: handleParams(params),
      ContentType: "application/json;charset=UTF-8"
    }).then(res => {
      resolve(res.data)
    }).catch(error => {
      reject(error)
    })
  })
}
```

**使用场景**:
- 表单数据提交
- 简单的 POST 请求
- 需要 FormData 格式的场景

##### 4. POST 请求封装（JSON）

```javascript
/**
 * POST 请求（JSON 格式）
 * @param url 请求地址
 * @param params 请求参数
 * @returns {Promise}
 */
const axiosJsonPost = function(url, params = {}) {
  return new Promise((resolve, reject) => {
    axios({
      method: 'post',
      url: url,
      data: params,
    }).then(res => {
      resolve(res.data)
    }).catch(error => {
      reject(error)
    })
  })
}
```

**使用场景**:
- RESTful API 调用
- 复杂对象提交
- 需要 JSON 格式的场景

**使用示例**:
```javascript
this.$axiosJsonPost('/api/cluster/save', {
  clusterName: 'test-cluster',
  clusterCode: 'TEST',
  frameCode: 'HADOOP'
}).then(data => {
  this.$message.success('创建成功')
})
```

##### 5. 文件上传封装

```javascript
/**
 * 文件上传（POST 请求）
 * @param url 请求地址
 * @param params FormData 对象
 * @returns {Promise}
 */
const axiosPostUpload = function(url, params = {}) {
  return new Promise((resolve, reject) => {
    axios({
      method: 'post',
      url: url,
      data: params
    }).then(res => {
      resolve(res.data)
    }).catch(error => {
      reject(error)
    })
  })
}
```

**使用示例**:
```javascript
const formData = new FormData()
formData.append('file', file)
formData.append('clusterId', clusterId)

this.$axiosPostUpload('/api/upload', formData)
  .then(data => {
    this.$message.success('上传成功')
  })
```

##### 6. Vue 原型挂载

```javascript
Vue.prototype.$axiosGet = axiosGet            // GET 请求
Vue.prototype.$axiosPost = axiosPost          // POST 请求
Vue.prototype.$axiosPostUpload = axiosPostUpload  // 文件上传
Vue.prototype.$axiosJsonPost = axiosJsonPost  // JSON POST 请求
```

**作用**: 在所有 Vue 组件中可以通过 `this.$axiosGet` 等方法直接调用

### 3.3 interceptors.js - 拦截器配置

#### 文件路径
`src/api/interceptors.js`

#### 请求拦截器

```javascript
axios.interceptors.request.use(
  config => {
    // IE 浏览器缓存处理
    if(config.method === 'get') {
      if(window.ActiveXObject || 'ActiveXObject' in window) {
        config.url = `${config.url}?${new Date().getTime()}`
      }
    }
    
    // 设置 Content-Type
    config.headers['Content-Type'] = config.ContentType 
      ? config.ContentType 
      : 'application/json;charset=UTF-8'
    
    return config
  },
  error => {
    return Promise.reject(error)
  }
)
```

**功能说明**:
1. **IE 缓存问题**: GET 请求添加时间戳，避免 IE 缓存
2. **Content-Type 设置**: 默认 JSON 格式，支持自定义
3. **可扩展**: 可以在这里添加 token、签名等

#### 响应拦截器

```javascript
axios.interceptors.response.use(
  response => {
    // 对响应数据操作
    return response
  },
  error => {
    return Promise.reject(error)
  }
)
```

**扩展建议**:
```javascript
axios.interceptors.response.use(
  response => {
    const {code, msg, data} = response.data
    
    // 统一处理响应码
    if (code === 200) {
      return data
    } else if (code === 401) {
      // 未登录，跳转登录页
      router.push('/login')
    } else {
      // 其他错误，显示提示
      Message.error(msg || '请求失败')
      return Promise.reject(msg)
    }
  },
  error => {
    // 网络错误处理
    if (error.response) {
      const {status} = error.response
      switch (status) {
        case 401:
          Message.error('未授权，请登录')
          break
        case 403:
          Message.error('拒绝访问')
          break
        case 404:
          Message.error('请求地址不存在')
          break
        case 500:
          Message.error('服务器错误')
          break
        default:
          Message.error('网络错误')
      }
    }
    return Promise.reject(error)
  }
)
```

### 3.4 utils/request.js - 高级请求工具

#### 文件路径
`src/utils/request.js`

#### 核心功能

##### 1. 认证类型定义

```javascript
const AUTH_TYPE = {
  BEARER: 'Bearer',    // Bearer Token
  BASIC: 'basic',      // Basic Auth
  AUTH1: 'auth1',      // 自定义认证1
  AUTH2: 'auth2',      // 自定义认证2
}
```

##### 2. HTTP 方法定义

```javascript
const METHOD = {
  GET: 'get',
  POST: 'post'
}
```

##### 3. 统一请求方法

```javascript
/**
 * axios 请求
 * @param url 请求地址
 * @param method {METHOD} HTTP 方法
 * @param params 请求参数
 * @param config 额外配置
 * @returns {Promise<AxiosResponse>}
 */
async function request(url, method, params, config) {
  switch (method) {
    case METHOD.GET:
      return axios.get(url, {params, ...config})
    case METHOD.POST:
      return axios.post(url, params, config)
    default:
      return axios.get(url, {params, ...config})
  }
}
```

##### 4. 认证信息管理

**设置认证**:
```javascript
/**
 * 设置认证信息
 * @param auth {Object} 认证对象
 * @param authType {AUTH_TYPE} 认证类型
 */
function setAuthorization(auth, authType = AUTH_TYPE.BEARER) {
  var today = new Date()
  var expires = new Date()
  expires.setTime(today.getTime() + 1000 * 60 * 60 * 24)  // 24小时

  switch (authType) {
    case AUTH_TYPE.BEARER:
      Cookie.set(xsrfHeaderName, auth.sessionId, {expires: expires})
      break
    case AUTH_TYPE.BASIC:
    case AUTH_TYPE.AUTH1:
    case AUTH_TYPE.AUTH2:
    default:
      break
  }
}
```

**移除认证**:
```javascript
/**
 * 移除认证信息
 * @param authType {AUTH_TYPE} 认证类型
 */
function removeAuthorization(authType = AUTH_TYPE.BEARER) {
  switch (authType) {
    case AUTH_TYPE.BEARER:
      Cookie.remove(xsrfHeaderName)
      break
    default:
      break
  }
}
```

**检查认证**:
```javascript
/**
 * 检查认证信息
 * @param authType {AUTH_TYPE} 认证类型
 * @returns {boolean}
 */
function checkAuthorization(authType = AUTH_TYPE.BEARER) {
  switch (authType) {
    case AUTH_TYPE.BEARER:
      if (Cookie.get(xsrfHeaderName)) {
        return true
      }
      break
    default:
      break
  }
  return false
}
```

**使用示例**:
```javascript
// 登录成功后设置认证
setAuthorization({sessionId: 'xxx-xxx-xxx'})

// 检查是否已登录
if (checkAuthorization()) {
  // 已登录
} else {
  // 未登录，跳转登录页
  router.push('/login')
}

// 退出登录
removeAuthorization()
```

##### 5. 加载拦截器

```javascript
/**
 * 加载 axios 拦截器
 * @param interceptors 拦截器配置
 * @param options 选项
 */
function loadInterceptors(interceptors, options) {
  const {request, response} = interceptors
  
  // 加载请求拦截器
  request.forEach(item => {
    let {onFulfilled, onRejected} = item
    if (!onFulfilled || typeof onFulfilled !== 'function') {
      onFulfilled = config => config
    }
    if (!onRejected || typeof onRejected !== 'function') {
      onRejected = error => Promise.reject(error)
    }
    axios.interceptors.request.use(
      config => onFulfilled(config, options),
      error => onRejected(error, options)
    )
  })
  
  // 加载响应拦截器
  response.forEach(item => {
    let {onFulfilled, onRejected} = item
    if (!onFulfilled || typeof onFulfilled !== 'function') {
      onFulfilled = response => response
    }
    if (!onRejected || typeof onRejected !== 'function') {
      onRejected = error => Promise.reject(error)
    }
    axios.interceptors.response.use(
      response => onFulfilled(response, options),
      error => onRejected(error, options)
    )
  })
}
```

### 3.5 httpApi/* - API 接口定义

#### 文件结构

```
httpApi/
├── cluster.js      # 集群管理 API（50+ 接口）
├── host.js         # 主机管理 API（10+ 接口）
├── services.js     # 服务管理 API（30+ 接口）
├── system.js       # 系统管理 API（20+ 接口）
├── user.js         # 用户管理 API（10+ 接口）
└── index.js        # 统一导出
```

#### cluster.js - 集群管理 API

##### 文件路径
`src/api/httpApi/cluster.js`

##### API 定义示例

```javascript
import paths from '@/api/baseUrl'

let path = paths.path() + '/ddh'

export default {
  // 集群相关
  getColonyList: path + '/api/cluster/list',           // 获取集群列表
  saveColony: path + '/api/cluster/save',              // 保存集群
  updateColony: path + '/api/cluster/update',          // 更新集群
  deleteColony: path + '/api/cluster/delete',          // 删除集群
  runningClusterList: path + '/api/cluster/runningClusterList',  // 运行中集群列表
  
  // 集群授权
  authCluster: path + '/api/cluster/user/saveClusterManager',  // 集群授权
  
  // 框架相关
  getFrameList: path + '/api/frame/list',              // 获取框架列表
  
  // 总览
  getDashboardUrl: path + '/cluster/service/dashboard/getDashboardUrl',  // 获取总览地址
  
  // 角色组相关
  reNameGroup: path + '/cluster/service/instance/role/group/rename',  // 重命名角色组
  delGroup: path + '/cluster/service/instance/role/group/delete',     // 删除角色组
  
  // 标签相关
  saveLabel: path + '/cluster/node/label/save',        // 保存标签
  assginLabel: path + '/cluster/node/label/assign',    // 分配标签
  deleteLabel: path + '/cluster/node/label/delete',    // 删除标签
  getLabelList: path + '/cluster/node/label/list',     // 获取标签列表
  
  // 机架相关
  saveRack: path + '/cluster/rack/save',               // 保存机架
  assginRack: path + '/api/cluster/host/assignRack',   // 分配机架
  deleteRack: path + '/cluster/rack/delete',           // 删除机架
  deleteClusterRack: path + '/cluster/rack/delete',    // 删除集群机架
  getRackList: path + '/cluster/rack/list',            // 获取机架列表
  
  // Parcel 相关
  getParcelList: path + '/cluster/parcel/list',        // 获取 Parcel 列表
  getParcelParse: path + '/cluster/parcel/parse',      // 解析 Parcel
  getParcelProcess: path + '/cluster/parcel/process',  // Parcel 进度
  downloadComponent: path + '/cluster/parcel/download', // 下载组件
  installComponent: path + '/cluster/parcel/install',  // 安装组件
}
```

**API 分类**:

| 分类 | 数量 | 说明 |
|------|------|------|
| 集群管理 | 5 | 增删改查、运行列表 |
| 集群授权 | 1 | 用户权限管理 |
| 框架管理 | 1 | 大数据框架列表 |
| 总览 | 1 | Dashboard URL |
| 角色组 | 2 | 重命名、删除 |
| 标签 | 4 | 增删改查 |
| 机架 | 5 | 增删改查、分配 |
| Parcel | 5 | 列表、解析、下载、安装 |

**使用示例**:
```javascript
import clusterApi from '@/api/httpApi/cluster'

// 获取集群列表
this.$axiosGet(clusterApi.getColonyList, {
  page: 1,
  size: 10
}).then(data => {
  this.clusterList = data.list
})

// 创建集群
this.$axiosJsonPost(clusterApi.saveColony, {
  clusterName: 'test-cluster',
  clusterCode: 'TEST'
}).then(() => {
  this.$message.success('创建成功')
})
```

## 四、API 调用流程

### 4.1 完整调用链

```
Component (组件)
    ↓
Vue Prototype Method ($axiosGet/$axiosPost/...)
    ↓
Request Interceptor (请求拦截器)
    ├─ 添加时间戳 (IE 缓存)
    ├─ 设置 Content-Type
    └─ 添加 Token (可选)
    ↓
Axios Request
    ↓
Backend Server (后端服务器)
    ↓
Axios Response
    ↓
Response Interceptor (响应拦截器)
    ├─ 提取数据
    ├─ 错误处理
    └─ 格式化响应
    ↓
Promise (resolve/reject)
    ↓
Component (组件处理响应)
```

### 4.2 请求示例

#### GET 请求
```javascript
// 1. 组件中调用
methods: {
  async loadClusterList() {
    try {
      const data = await this.$axiosGet(clusterApi.getColonyList, {
        page: 1,
        size: 10
      })
      this.clusterList = data.list
      this.total = data.total
    } catch (error) {
      this.$message.error('加载失败')
    }
  }
}

// 2. 实际请求
// GET /ddh/api/cluster/list?page=1&size=10
```

#### POST 请求（JSON）
```javascript
// 1. 组件中调用
methods: {
  async saveCluster() {
    try {
      await this.$axiosJsonPost(clusterApi.saveColony, {
        clusterName: this.form.name,
        clusterCode: this.form.code,
        frameCode: this.form.frame
      })
      this.$message.success('保存成功')
      this.getClusterList()
    } catch (error) {
      this.$message.error('保存失败')
    }
  }
}

// 2. 实际请求
// POST /ddh/api/cluster/save
// Content-Type: application/json
// Body: {"clusterName":"test","clusterCode":"TEST","frameCode":"HADOOP"}
```

#### POST 请求（FormData）
```javascript
// 1. 组件中调用
methods: {
  async updateCluster() {
    await this.$axiosPost(clusterApi.updateColony, {
      id: this.form.id,
      clusterName: this.form.name
    })
  }
}

// 2. 实际请求
// POST /ddh/api/cluster/update
// Content-Type: multipart/form-data
// Body: FormData {id: 1, clusterName: "test"}
```

#### 文件上传
```javascript
// 1. 组件中调用
methods: {
  async uploadFile(file) {
    const formData = new FormData()
    formData.append('file', file)
    formData.append('clusterId', this.clusterId)
    
    await this.$axiosPostUpload('/api/upload', formData)
    this.$message.success('上传成功')
  }
}

// 2. 实际请求
// POST /api/upload
// Content-Type: multipart/form-data
// Body: FormData {file: File, clusterId: 1}
```

## 五、API 模块化管理

### 5.1 按业务模块划分

```javascript
// httpApi/index.js
import cluster from './cluster'
import host from './host'
import services from './services'
import system from './system'
import user from './user'

export default {
  cluster,
  host,
  services,
  system,
  user
}
```

### 5.2 使用方式

```javascript
// 方式 1: 导入全部
import api from '@/api/httpApi'
this.$axiosGet(api.cluster.getColonyList)

// 方式 2: 按需导入
import clusterApi from '@/api/httpApi/cluster'
this.$axiosGet(clusterApi.getColonyList)

// 方式 3: 解构导入
import {cluster, host} from '@/api/httpApi'
this.$axiosGet(cluster.getColonyList)
this.$axiosGet(host.getHostList)
```

## 六、错误处理

### 6.1 统一错误处理

```javascript
// 在组件中
methods: {
  async loadData() {
    try {
      const data = await this.$axiosGet(api.getList)
      // 处理成功响应
      this.list = data
    } catch (error) {
      // 处理错误
      this.$message.error('加载失败')
      console.error(error)
    }
  }
}
```

### 6.2 全局错误处理

```javascript
// main.js
Vue.config.errorHandler = (err, vm, info) => {
  console.error('全局错误:', err)
  console.error('错误组件:', vm)
  console.error('错误信息:', info)
  
  // 上报错误到监控系统
  // reportError(err, vm, info)
}
```

### 6.3 响应拦截器错误处理

```javascript
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response) {
      // 服务器响应错误
      const {status, data} = error.response
      switch (status) {
        case 401:
          Message.error('未登录')
          router.push('/login')
          break
        case 403:
          Message.error('无权限')
          break
        case 404:
          Message.error('接口不存在')
          break
        case 500:
          Message.error('服务器错误')
          break
        default:
          Message.error(data.msg || '请求失败')
      }
    } else if (error.request) {
      // 请求已发出，但没有收到响应
      Message.error('网络错误，请检查网络连接')
    } else {
      // 请求配置错误
      Message.error('请求错误')
    }
    return Promise.reject(error)
  }
)
```

## 七、环境配置

### 7.1 开发环境

```javascript
// vue.config.js
module.exports = {
  devServer: {
    port: 8080,
    proxy: {
      '/ddh': {
        target: 'http://localhost:8081',  // 后端服务地址
        changeOrigin: true,
        pathRewrite: {
          '^/api': '/ddh'
        }
      }
    }
  }
}
```

**请求流程**:
```
前端: GET /ddh/api/cluster/list
  ↓
代理: GET http://localhost:8081/ddh/api/cluster/list
  ↓
后端: 处理请求
```

### 7.2 生产环境

```javascript
// .env.production
VUE_APP_API_BASE_URL=/api

// baseUrl.js
export default {
  path() {
    return process.env.VUE_APP_API_BASE_URL || ''
  }
}
```

**Nginx 配置**:
```nginx
server {
  listen 80;
  server_name example.com;
  
  # 前端静态资源
  location / {
    root /usr/share/nginx/html;
    try_files $uri $uri/ /index.html;
  }
  
  # 后端 API 代理
  location /ddh/ {
    proxy_pass http://backend:8081/ddh/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

## 八、最佳实践

### 8.1 API 定义规范

```javascript
// ✅ 推荐
export default {
  getClusterList: path + '/api/cluster/list',     // 动词 + 名词
  saveCluster: path + '/api/cluster/save',        // 动词 + 名词
  updateCluster: path + '/api/cluster/update',    // 动词 + 名词
  deleteCluster: path + '/api/cluster/delete',    // 动词 + 名词
}

// ❌ 避免
export default {
  list: path + '/api/cluster/list',               // 不明确
  save: path + '/api/cluster/save',               // 不明确
  update: path + '/api/cluster/update',           // 不明确
  del: path + '/api/cluster/delete',              // 缩写
}
```

### 8.2 请求方法选择

```javascript
// ✅ GET 请求用于查询
this.$axiosGet(api.getClusterList, {page: 1})

// ✅ POST JSON 用于创建/更新复杂对象
this.$axiosJsonPost(api.saveCluster, {
  clusterName: 'test',
  config: {...}
})

// ✅ POST FormData 用于简单表单
this.$axiosPost(api.updateCluster, {
  id: 1,
  name: 'test'
})

// ✅ POST Upload 用于文件上传
this.$axiosPostUpload(api.uploadFile, formData)
```

### 8.3 Loading 状态处理

```javascript
data() {
  return {
    loading: false,
    list: []
  }
},
methods: {
  async loadData() {
    this.loading = true
    try {
      const data = await this.$axiosGet(api.getList)
      this.list = data
    } catch (error) {
      this.$message.error('加载失败')
    } finally {
      this.loading = false
    }
  }
}
```

### 8.4 请求取消

```javascript
import axios from 'axios'

data() {
  return {
    cancelToken: null
  }
},
methods: {
  async loadData() {
    // 取消之前的请求
    if (this.cancelToken) {
      this.cancelToken.cancel('取消请求')
    }
    
    // 创建新的取消令牌
    this.cancelToken = axios.CancelToken.source()
    
    try {
      const data = await axios.get(api.getList, {
        cancelToken: this.cancelToken.token
      })
      this.list = data
    } catch (error) {
      if (axios.isCancel(error)) {
        console.log('请求已取消')
      } else {
        this.$message.error('加载失败')
      }
    }
  }
},
beforeDestroy() {
  // 组件销毁时取消请求
  if (this.cancelToken) {
    this.cancelToken.cancel()
  }
}
```

### 8.5 请求重试

```javascript
// utils/request.js
function requestWithRetry(url, method, params, config = {}, retryCount = 3) {
  return new Promise((resolve, reject) => {
    const doRequest = (count) => {
      request(url, method, params, config)
        .then(resolve)
        .catch(error => {
          if (count > 0) {
            console.log(`请求失败，剩余重试次数: ${count}`)
            setTimeout(() => {
              doRequest(count - 1)
            }, 1000)  // 1秒后重试
          } else {
            reject(error)
          }
        })
    }
    doRequest(retryCount)
  })
}
```

## 九、安全性考虑

### 9.1 CSRF 防护

```javascript
// 请求拦截器中添加 CSRF Token
axios.interceptors.request.use(config => {
  const csrfToken = Cookie.get('csrf-token')
  if (csrfToken) {
    config.headers['X-CSRF-Token'] = csrfToken
  }
  return config
})
```

### 9.2 请求签名

```javascript
import md5 from 'md5'

axios.interceptors.request.use(config => {
  const timestamp = Date.now()
  const nonce = Math.random().toString(36).substr(2)
  const signature = md5(`${timestamp}${nonce}${config.url}`)
  
  config.headers['X-Timestamp'] = timestamp
  config.headers['X-Nonce'] = nonce
  config.headers['X-Signature'] = signature
  
  return config
})
```

### 9.3 敏感数据加密

```javascript
import CryptoJS from 'crypto-js'

function encryptData(data, key) {
  return CryptoJS.AES.encrypt(JSON.stringify(data), key).toString()
}

function decryptData(encrypted, key) {
  const bytes = CryptoJS.AES.decrypt(encrypted, key)
  return JSON.parse(bytes.toString(CryptoJS.enc.Utf8))
}

// 使用
const encrypted = encryptData({password: '123456'}, 'secret-key')
this.$axiosPost(api.login, {data: encrypted})
```

## 十、性能优化

### 10.1 请求合并

```javascript
// 使用 Promise.all 并发请求
async loadData() {
  this.loading = true
  try {
    const [clusters, hosts, services] = await Promise.all([
      this.$axiosGet(api.getClusterList),
      this.$axiosGet(api.getHostList),
      this.$axiosGet(api.getServiceList)
    ])
    this.clusters = clusters
    this.hosts = hosts
    this.services = services
  } catch (error) {
    this.$message.error('加载失败')
  } finally {
    this.loading = false
  }
}
```

### 10.2 请求缓存

```javascript
const cache = new Map()

function getCached(url, params) {
  const key = `${url}?${JSON.stringify(params)}`
  return cache.get(key)
}

function setCache(url, params, data, ttl = 5000) {
  const key = `${url}?${JSON.stringify(params)}`
  cache.set(key, data)
  setTimeout(() => {
    cache.delete(key)
  }, ttl)
}

// 使用
async function request(url, params) {
  const cached = getCached(url, params)
  if (cached) {
    return cached
  }
  
  const data = await axios.get(url, {params})
  setCache(url, params, data)
  return data
}
```

### 10.3 请求节流

```javascript
import {debounce, throttle} from 'lodash'

methods: {
  // 防抖：用户停止输入后才发送请求
  search: debounce(function(keyword) {
    this.loadData(keyword)
  }, 500),
  
  // 节流：限制请求频率
  scroll: throttle(function() {
    this.loadMore()
  }, 1000)
}
```

## 十一、总结

### 核心特性

| 特性 | 说明 |
|------|------|
| 分层封装 | request → httpApi → services |
| 类型支持 | GET、POST、JSON POST、文件上传 |
| 拦截器 | 请求/响应统一处理 |
| 认证管理 | Cookie-based 会话管理 |
| 环境适配 | 开发/生产环境自动切换 |
| 模块化 | 按业务模块划分 API |

### 技术亮点

1. **Promise 化**: 所有请求返回 Promise，支持 async/await
2. **Vue 集成**: 挂载到 Vue 原型，全局可用
3. **灵活配置**: 支持多种请求类型和配置
4. **统一处理**: 拦截器统一处理认证、错误、格式化
5. **类型安全**: 明确的 API 定义和参数类型

### API 统计

| 模块 | 接口数量 | 说明 |
|------|---------|------|
| cluster.js | 50+ | 集群管理、标签、机架、Parcel |
| host.js | 10+ | 主机管理、监控 |
| services.js | 30+ | 服务管理、配置、操作 |
| system.js | 20+ | 系统配置、框架管理 |
| user.js | 10+ | 用户管理、权限 |
| **总计** | **120+** | 覆盖所有业务功能 |

---

**文档状态**: ✅ 已完成  
**API 文件覆盖**: baseUrl.js, request.js, interceptors.js, utils/request.js, httpApi/*  
**最后更新**: 2025-11-16
