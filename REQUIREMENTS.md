# 汽车救援定位自动化工具 — 需求文档

> 版本：v1.1  
> 日期：2026-04-26  
> 状态：需求确认中

---

## 一、项目背景

### 1.1 业务场景

汽车救援业务流程：

```
客服建单 → 派单给救援公司 → 救援公司调度给技师 → 技师执行任务
                                                    ↓
                                    出车 → 到达 → 开始 → 完成
                                            ↑
                                    全程每 30 秒上报定位
```

### 1.2 当前痛点

- 技师执行任务时需要**手动上报定位**（每 30 秒一次）
- 目前使用**爱思助手虚拟定位** + **手机手动绘制地图路径**
- 整个流程操作繁琐，占用大量时间

### 1.3 目标

开发一个 Web 工具，**自动沿路径上报定位**，减少人工操作：

- 手动输入起点和终点
- 自动调用高德路径规划 API 生成路线
- 沿路线自动模拟行驶，每 30 秒上报一次定位
- 支持速度切换、暂停/继续/停止

---

## 二、功能需求

### 2.1 核心功能

| 功能 | 说明 | 优先级 |
|------|------|--------|
| 起点终点输入 | 手动输入地址或经纬度，支持地图选点 | P0 |
| 路径规划 | 调用高德驾车路径规划 API，生成行驶路线 | P0 |
| 速度选择 | 40/60/80 km/h 三档可调，支持自定义 | P0 |
| 自动上报 | 每 30 秒调用接口上报一次定位 | P0 |
| 任务控制 | 开始、暂停、继续、停止 | P0 |
| 状态显示 | 运行状态、已上报次数、剩余时间、当前位置 | P1 |
| 日志输出 | 实时显示每次上报结果 | P1 |

### [新增] 2.2 地理编码功能

**输入方式（双模）：**
- 模式 A：输入文字地址 → 调用高德地理编码 API → 自动解析为经纬度
- 模式 B：直接输入经纬度（格式：`116.481028,39.989643`）

**地理编码接口：**
```
GET https://restapi.amap.com/v3/geocode/geo
```

| 参数 | 说明 |
|------|------|
| `key` | 高德 API Key |
| `address` | 用户输入的地址文本 |
| `city` | 城市（可选，提高精度） |

**解析规则：**
- 返回 `geocodes[0].location` 作为坐标
- 若 `count == 0` 则提示用户"地址未找到，请修改后重试"
- 解析成功后在地图上显示 Pin，供用户二次确认

### [新增] 2.3 上报频率可选

| 选项 | 间隔 | 适用场景 |
|------|------|---------|
| 10 秒 | 10s | 短途任务（< 5km） |
| 20 秒 | 20s | 中途任务 |
| 30 秒（默认） | 30s | 标准任务 |

- 频率在任务启动前配置，启动后不可修改
- 坐标采样密度随频率自动调整（速度 × 间隔 = 采样步长）

### [新增] 2.4 路径保存与复用

**保存：**
- 任务完成后弹出「保存路线」对话框
- 用户可输入路线名称（如"南京-常州救援路线"）
- 存入 MongoDB `saved_routes` 集合

**复用：**
- 新建任务时，顶部显示「从历史路线选择」下拉框
- 选中后自动填充起点、终点文字地址和对应经纬度
- 不影响当前的 caseId / workId 输入

### 2.5 定位上报接口（原 2.2）

**接口地址：**
```
POST https://dragon.deploy-test.xiaopeng.com/open/dragon/yeKeDaRescue/technicianPointCallback
```

**请求体：**
```json
{
  "actionTime": "2026-04-10 15:09:14",
  "caseId": "2604102700015",
  "lon": "118.77678",
  "lat": "31.97488",
  "workId": "DRTC_RO_20260410_QV0010",
  "locateTime": "2026-04-10 15:09:14"
}
```

**返回：**
```json
{"code":200,"msg":null,"data":"成功"}
```

**字段说明：**
| 字段 | 说明 | 来源 |
|------|------|------|
| actionTime | 动作时间 | 当前时间 |
| caseId | 订单号 | 用户输入 |
| lon | 经度 | 路径规划生成 |
| lat | 纬度 | 路径规划生成 |
| workId | 工单号 | 用户输入 |
| locateTime | 定位时间 | 当前时间 |

### 2.6 高德路径规划 API（原 2.3）

**接口地址：**
```
GET https://restapi.amap.com/v3/direction/driving
```

**请求参数：**
| 参数 | 说明 | 示例 |
|------|------|------|
| key | 高德 Key | 用户申请 |
| origin | 起点经纬度 | 116.481028,39.989643 |
| destination | 终点经纬度 | 116.465302,39.970676 |
| policy | 路径策略 | 0（最快捷） |

**返回处理：**
- 解析 `route.paths[0].steps` 获取路径坐标
- 从 polyline 中等间距采样坐标点
- 根据速度计算采样间隔（示例：60km/h，30 秒上报 ≈ 每 500 米一个点）

---

## 三、技术架构

### 3.1 技术栈

| 组件 | 技术选型 | 说明 |
|------|---------|------|
| 前端 | Vue 3 + TypeScript + Vite | 地图选点 + 任务控制界面 |
| UI 库 | Element Plus | 表单、按钮、日志输出 |
| 地图 | 高德地图 JS API | 地图展示 + 选点 + 路径绘制 |
| 后端 | Node.js + Express | 路径规划 + 定时上报 |
| 数据库 | MongoDB (Atlas) | 任务记录 + 上报日志 |
| 部署 | 前端 Vercel + 后端 Railway | 长时间运行 |

### 3.2 项目结构

```
rescue-location-automation/
├── frontend/                    # Vue 3 前端
│   ├── src/
│   │   ├── views/
│   │   │   └── Task.vue         # 任务配置页面
│   │   ├── components/
│   │   │   └── MapPicker.vue    # 地图选点组件
│   │   ├── api/
│   │   │   └── task.ts          # 任务控制 API
│   │   ├── main.ts
│   │   └── App.vue
│   ├── package.json
│   └── vite.config.ts
│
├── backend/                     # Node.js 后端
│   ├── src/
│   │   ├── routes/
│   │   │   └── task.ts          # 任务控制接口
│   │   ├── services/
│   │   │   ├── amap.ts          # 高德 API 封装
│   │   │   └── reporter.ts      # 定位上报服务
│   │   ├── index.ts             # Express 入口
│   │   └── types.ts             # 类型定义
│   ├── package.json
│   └── Dockerfile
│
├── docker-compose.yml           # Docker 编排
└── README.md                    # 使用说明
```

### 3.3 API 设计

| 接口 | 方法 | 请求体 | 说明 |
|------|------|--------|------|
| `/api/task/start` | POST | `{caseId, workId, origin, destination, speed}` | 开始任务 |
| `/api/task/pause` | POST | - | 暂停任务 |
| `/api/task/resume` | POST | - | 继续任务 |
| `/api/task/stop` | POST | - | 停止任务 |
| `/api/task/status` | GET | - | 获取任务状态 |
| `/api/task/logs` | GET | - | 获取上报日志 |

### [新增] 3.3.1 API 完整 Request / Response 规范

**POST /api/task/start**

Request:
```json
{
  "caseId": "2604102700015",
  "workId": "DRTC_RO_20260410_QV0010",
  "origin": {
    "address": "南京南站",
    "lon": "118.77678",
    "lat": "31.97488"
  },
  "destination": {
    "address": "常州北站",
    "lon": "119.97386",
    "lat": "31.78234"
  },
  "speed": 60,
  "interval": 30
}
```

Response (成功):
```json
{
  "code": 200,
  "data": {
    "taskId": "task_20260426_001",
    "totalPoints": 48,
    "estimatedDuration": 1440,
    "routeDistance": 87500
  }
}
```

Response (失败):
```json
{
  "code": 400,
  "msg": "路径规划失败：起终点距离超出范围",
  "data": null
}
```

**GET /api/task/status**

Response:
```json
{
  "code": 200,
  "data": {
    "taskId": "task_20260426_001",
    "state": "running",
    "reportedCount": 12,
    "totalPoints": 48,
    "progress": 25,
    "remainingTime": 1080,
    "currentPosition": { "lon": "118.92345", "lat": "31.88765" },
    "lastReportTime": "2026-04-26 10:15:30",
    "lastReportSuccess": true
  }
}
```

state 枚举：`idle` / `running` / `paused` / `stopped` / `completed`

**GET /api/task/logs?page=1&limit=20**

Response:
```json
{
  "code": 200,
  "data": {
    "total": 12,
    "logs": [
      {
        "seq": 1,
        "timestamp": "2026-04-26 10:00:00",
        "lon": "118.77678",
        "lat": "31.97488",
        "success": true,
        "responseCode": 200,
        "responseMsg": "成功"
      }
    ]
  }
}
```

### [新增] 3.3.2 路径保存 / 复用接口

**POST /api/routes/save**

Request:
```json
{
  "name": "南京南站-常州北站",
  "origin": { "address": "南京南站", "lon": "118.77678", "lat": "31.97488" },
  "destination": { "address": "常州北站", "lon": "119.97386", "lat": "31.78234" }
}
```

Response:
```json
{ "code": 200, "data": { "routeId": "route_abc123" } }
```

**GET /api/routes/list**

Response:
```json
{
  "code": 200,
  "data": [
    {
      "routeId": "route_abc123",
      "name": "南京南站-常州北站",
      "origin": { "address": "南京南站", "lon": "118.77678", "lat": "31.97488" },
      "destination": { "address": "常州北站", "lon": "119.97386", "lat": "31.78234" },
      "usedCount": 3,
      "lastUsedAt": "2026-04-25 09:00:00"
    }
  ]
}
```

**DELETE /api/routes/:routeId**

Response:
```json
{ "code": 200, "data": null }
```

### 3.4 数据模型

```typescript
// 任务配置
interface TaskConfig {
  caseId: string;      // 订单号
  workId: string;      // 工单号
  origin: Coordinate;  // 起点
  destination: Coordinate; // 终点
  speed: number;       // 速度 (km/h)
}

// 坐标
interface Coordinate {
  lat: number;
  lon: number;
}

// 任务状态
interface TaskStatus {
  running: boolean;    // 是否运行中
  progress: number;    // 进度 (0-100)
  reportedCount: number; // 已上报次数
  remainingTime: number; // 剩余时间 (秒)
  currentPosition: Coordinate; // 当前位置
}

// 上报日志
interface ReportLog {
  timestamp: string;
  success: boolean;
  caseId: string;
  position: Coordinate;
  message?: string;
}
```

### [新增] 3.4.1 MongoDB 集合定义

**集合 `tasks`**

```js
{
  _id: ObjectId,
  taskId: String,           // 唯一任务 ID，格式 task_YYYYMMDD_seq
  caseId: String,
  workId: String,
  origin: {
    address: String,
    lon: String,
    lat: String
  },
  destination: {
    address: String,
    lon: String,
    lat: String
  },
  speed: Number,            // km/h
  interval: Number,         // 上报间隔秒数
  state: String,            // idle / running / paused / stopped / completed
  totalPoints: Number,
  reportedCount: Number,
  routePoints: [            // 完整路径坐标序列（由高德返回后存储）
    { lon: String, lat: String }
  ],
  createdAt: Date,
  updatedAt: Date
}
```

索引：`taskId`（唯一），`createdAt`（降序）

**集合 `report_logs`**

```js
{
  _id: ObjectId,
  taskId: String,
  seq: Number,              // 本次任务第几次上报
  lon: String,
  lat: String,
  actionTime: String,
  locateTime: String,
  responseCode: Number,
  responseMsg: String,
  success: Boolean,
  retryCount: Number,       // 本次上报重试次数（0-3）
  createdAt: Date
}
```

索引：`taskId + seq`（复合唯一），`createdAt`

**集合 `saved_routes`**

```js
{
  _id: ObjectId,
  routeId: String,
  name: String,
  origin: { address: String, lon: String, lat: String },
  destination: { address: String, lon: String, lat: String },
  usedCount: Number,
  lastUsedAt: Date,
  createdAt: Date
}
```

索引：`routeId`（唯一），`lastUsedAt`（降序）

---

## 四、非功能需求

| 需求 | 说明 |
|------|------|
| 长时间运行 | 支持连续运行数小时不中断 |
| 稳定性 | 上报失败自动重试（最多 3 次） |
| 可恢复 | 暂停后可继续，不丢失进度 |
| 易用性 | 界面简洁，操作步骤少 |
| 可维护性 | 代码结构清晰，有日志输出 |

### [新增] 4.1 性能需求

| 指标 | 目标 |
|------|------|
| 地理编码响应 | < 1s（高德 API SLA） |
| 路径规划响应 | < 2s（高德 API SLA） |
| 任务状态刷新 | 前端轮询间隔 5s，状态延迟 ≤ 5s |
| 上报成功率 | ≥ 99%（3 次重试后） |
| MongoDB 写入 | 单次 < 100ms |
| 前端首屏加载 | < 3s（Vercel CDN） |

### [新增] 4.2 安全需求

| 项目 | 要求 |
|------|------|
| 高德 Key 保护 | 仅在后端使用，不暴露给前端；前端通过后端代理调用 |
| 接口防滥用 | 后端对 `/api/task/start` 加速率限制（10次/分钟/IP） |
| 环境变量 | 所有密钥通过环境变量注入，不写入代码 |
| HTTPS | 前后端全程 HTTPS，禁止 HTTP 降级 |
| 错误信息 | 生产环境不返回堆栈信息，只返回业务错误码 |

### [新增] 4.3 兼容性需求

| 项目 | 要求 |
|------|------|
| 浏览器 | Chrome 90+、Edge 90+（主要使用场景） |
| 屏幕分辨率 | 1280×720 及以上 |
| 网络环境 | 支持 4G/WiFi，弱网下上报失败自动重试 |
| 移动端 | 不作要求（PC 工具） |

---

## [新增] 五、用户操作流程与交互细节

### 5.1 完整操作流程（12 步）

```
1. 打开工具页面
2. 在「历史路线」下拉框选择已保存路线（可跳过）
3. 输入起点地址（或经纬度）
4. 输入终点地址（或经纬度）
5. 系统自动地理编码，地图 Pin 显示解析结果
6. 用户确认 Pin 位置正确（如不正确，修改地址重新解析）
7. 选择行驶速度（默认 60 km/h）
8. 选择上报频率（默认 30 秒）
9. 输入 caseId（订单号）和 workId（工单号）
10. 点击「预览路线」→ 地图显示完整路径，展示里程和预计时长
11. 确认无误后点击「开始任务」
12. 任务运行中：查看进度条、上报日志；可随时暂停/继续/停止
```

### 5.2 表单校验规则

| 字段 | 校验规则 | 错误提示 |
|------|---------|---------|
| 起点地址 | 非空 | "请输入起点地址" |
| 终点地址 | 非空 | "请输入终点地址" |
| 起点经纬度 | 格式 `lon,lat`，数值范围有效 | "经纬度格式错误，示例：116.481,39.989" |
| caseId | 非空，长度 ≤ 50 | "请输入订单号" |
| workId | 非空，长度 ≤ 50 | "请输入工单号" |
| 速度自定义 | 正整数，10–200 | "速度需在 10-200 km/h 之间" |
| 频率 | 枚举值（10/20/30） | — |

### 5.3 任务状态机

```
         ┌─────────────────────────────┐
         │                             │
        idle ──[start]──> running ──[pause]──> paused
                            │                    │
                         [stop]               [resume]
                            │                    │
                         stopped           running ◄──┘
                            
        running ──[全部点上报完毕]──> completed
```

状态说明：
- `idle`：初始状态，表单可编辑
- `running`：定时器运行中，表单锁定，显示进度
- `paused`：定时器暂停，保留当前进度，可继续
- `stopped`：用户主动停止，进度清零，表单解锁
- `completed`：所有坐标点上报完毕，弹出「保存路线」对话框

---

## [新增] 六、异常场景处理

### 6.1 地理编码失败

| 情况 | 处理方式 |
|------|---------|
| 地址解析结果为空 | 提示"地址未找到，请修改后重试"，不进入路径规划 |
| 高德 API 超时（> 5s） | 提示"地理编码超时，请检查网络后重试" |
| 高德 API 返回错误码 | 显示具体错误（如 "INVALID_USER_KEY"） |

### 6.2 路径规划失败

| 情况 | 处理方式 |
|------|---------|
| 起终点相同 | 前端校验拦截，提示"起终点不能相同" |
| 起终点超出距离限制（> 500km） | 提示"路程过长，请检查起终点" |
| 高德 API 无路线结果 | 提示"无法规划路线，请尝试其他地址" |
| 高德 API 限额超出 | 提示"API 调用超限，请联系管理员更换 Key" |

### 6.3 定位上报失败

| 情况 | 处理方式 |
|------|---------|
| HTTP 超时（> 10s） | 标记为失败，自动重试，最多 3 次 |
| 返回非 200 状态码 | 记录响应码，标记失败，加入重试队列 |
| 3 次重试均失败 | 在日志中标红显示，任务继续执行（不中断） |
| 连续 5 次上报失败 | 暂停任务，弹出警告"连续上报失败，请检查网络或接口" |

### 6.4 浏览器关闭 / 刷新

- 任务状态存储在**后端内存 + MongoDB**，前端关闭后任务继续运行
- 重新打开页面时，前端轮询 `/api/task/status`，恢复显示运行状态
- 若后端进程重启（Railway 休眠后唤醒），任务状态从 MongoDB 恢复，但定时器需重新激活（用户手动点「恢复任务」）

### 6.5 网络断开

- 前端检测到 `/api/task/status` 请求连续失败 3 次，显示"后端连接异常"横幅
- 后端若无法触达上报接口，按 6.3 策略处理，不影响计时器运行

---

## 七（原五）、部署方案

### 7.1 前端部署

- **平台：** Vercel
- **配置：**
  - Root Directory: `frontend`
  - Framework: Vue.js
  - Build: `npm run build`

### 7.2 后端部署

- **平台：** Railway
  - 自动检测 Dockerfile
  - 配置环境变量：`AMAP_KEY`、`REPORT_URL`、`MONGODB_URI`
  - ⚠️ 注意：Railway 免费版有休眠，长时间任务需升级或用 keep-alive

### 7.3 环境变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `AMAP_KEY` | 高德 API Key | 用户申请 |
| `REPORT_URL` | 定位上报接口 URL | `https://dragon.deploy-test.xiaopeng.com/...` |
| `PORT` | 后端服务端口 | `8080` |
| `MONGODB_URI` | MongoDB 连接字符串 | `mongodb+srv://...` |

---

## 八（原六）、开发计划

### 阶段 1：核心功能（预计 2-3 天）

- [ ] 后端：高德路径规划 API 封装
- [ ] 后端：定位上报服务（定时任务）
- [ ] 后端：任务控制接口（开始/暂停/继续/停止）
- [ ] 前端：任务配置页面（起点终点输入、速度选择）
- [ ] 前端：地图选点组件（高德地图集成）

### 阶段 2：完善功能（预计 1-2 天）

- [ ] 前端：任务状态显示（进度、剩余时间）
- [ ] 前端：日志输出组件
- [ ] 后端：上报失败重试机制
- [ ] 后端：任务状态持久化（可选）

### 阶段 3：部署上线（预计 1 天）

- [ ] Docker 镜像构建
- [ ] Railway/VPS 部署
- [ ] 环境变量配置
- [ ] 联调测试

---

## 九（原七）、风险与依赖

| 风险 | 影响 | 应对措施 |
|------|------|---------|
| 高德 API 配额限制 | 路径规划失败 | 申请企业 Key 或降级方案 |
| Railway 休眠 | 上报中断 | 使用 VPS 或付费计划 |
| 网络不稳定 | 上报失败 | 自动重试 + 本地缓存 |
| 接口变更 | 上报失败 | 监控日志 + 及时更新 |

---

## 十（原八）、已确认事项

| 项 | 结论 |
|------|------|
| 任务并发 | 一次一个订单 |
| caseId/workId | 手动输入（从原系统复制） |
| 接口环境 | 仅测试环境 |
| 接口认证 | 无鉴权，直接 POST |
| 起点终点输入 | 输入地址自动转经纬度（高德地理编码 API） |
| 上报频率 | 可选（10秒/20秒/30秒） |
| 路径预览 | 先预览再开始，地图上显示完整路线 |
| 速度选择 | 40/60/80 km/h + 自定义 |
| 路径保存 | 保存常用路线到 MongoDB，下次直接选用 |
| 任务历史 | 暂不做，后续加 |
| 数据库 | MongoDB（Atlas） |

---

## 十一（原九）、参考资料

- 高德路径规划 API：https://lbs.amap.com/api/webservice/guide/api/driving
- 高德地理编码 API：https://lbs.amap.com/api/webservice/guide/api/georegeo
- 高德地图 JS API：https://lbs.amap.com/api/javascript-api/summary
- 小鹏救援接口：见 2.5 节
