# 汽车救援定位自动化工具 — 需求文档

> 版本：v1.0  
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

### 2.2 定位上报接口

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

### 2.3 高德路径规划 API

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

---

## 四、非功能需求

| 需求 | 说明 |
|------|------|
| 长时间运行 | 支持连续运行数小时不中断 |
| 稳定性 | 上报失败自动重试（最多 3 次） |
| 可恢复 | 暂停后可继续，不丢失进度 |
| 易用性 | 界面简洁，操作步骤少 |
| 可维护性 | 代码结构清晰，有日志输出 |

---

## 五、部署方案

### 5.1 前端部署

- **平台：** Vercel
- **配置：**
  - Root Directory: `frontend`
  - Framework: Vue.js
  - Build: `npm run build`

### 5.2 后端部署

- **平台：** Railway
  - 自动检测 Dockerfile
  - 配置环境变量：`AMAP_KEY`、`REPORT_URL`、`MONGODB_URI`
  - ⚠️ 注意：Railway 免费版有休眠，长时间任务需升级或用 keep-alive

### 5.3 环境变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `AMAP_KEY` | 高德 API Key | 用户申请 |
| `REPORT_URL` | 定位上报接口 URL | `https://dragon.deploy-test.xiaopeng.com/...` |
| `PORT` | 后端服务端口 | `8080` |
| `MONGODB_URI` | MongoDB 连接字符串 | `mongodb+srv://...` |

---

## 六、开发计划

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

## 七、风险与依赖

| 风险 | 影响 | 应对措施 |
|------|------|---------|
| 高德 API 配额限制 | 路径规划失败 | 申请企业 Key 或降级方案 |
| Railway 休眠 | 上报中断 | 使用 VPS 或付费计划 |
| 网络不稳定 | 上报失败 | 自动重试 + 本地缓存 |
| 接口变更 | 上报失败 | 监控日志 + 及时更新 |

---

## 八、待确认事项

- [ ] 是否需要多任务并发（同时跑多个订单）？
- [ ] 订单信息（caseId、workId）是否可以从现有系统自动获取？
- [ ] 上报日志是否需要持久化存储？
- [ ] 是否需要任务历史记录功能？

---

## 九、参考资料

- 高德路径规划 API：https://lbs.amap.com/api/webservice/guide/api/driving
- 高德地图 JS API：https://lbs.amap.com/api/javascript-api/summary
- 小鹏救援接口：见 2.2 节
