# 会捣乱的积木 · 后台管理系统 技术方案（PLAN）

> 目标：为现有纯前端游戏补齐 ①可后台配置的过关时长（读不到后端时回退默认值）、②前端埋点、③带难度分析与图表的管理后台。

---

## 0. 已确认的选型

| 项 | 决定 |
|---|---|
| 后端技术栈 | **Node.js + Express + SQLite** |
| 部署场景 | **云服务器对外开放**（公网，需鉴权 / 限流 / IP 隐私处理） |
| 管理员账号 | **单一口令登录** |
| 前端 | 直接在当前游戏工程（`trick-brick/`）分支上改 |
| 后端 | 在游戏工程**同级**新建独立工程 `trick-brick-server/` |
| 图表 | Chart.js（漏斗如需更强可换 ECharts），零构建、CDN 引入 |

---

## 1. 目录结构

```
gamejam/
├── trick-brick/                 # 现有游戏前端（在此分支改）
│   ├── index.html               #   - 启动时拉时长配置；插入埋点调用
│   ├── tracker.js               #   - 新增：埋点模块（sendBeacon 上报）
│   ├── PLAN.md                  #   - 本文档
│   └── ...（图片/视频资源）
│
└── trick-brick-server/          # 新建后端工程（与 trick-brick 同级）
    ├── src/
    │   ├── app.js               # Express 入口，挂载静态/API/后台
    │   ├── db.js                # SQLite 连接 + 建表 + 默认时长灌入
    │   ├── routes/
    │   │   ├── public.js        # GET /api/levels/config, POST /api/track
    │   │   └── admin.js         # 登录 / 改时长 / 统计接口（需鉴权）
    │   ├── middleware/
    │   │   ├── auth.js          # 单口令登录 + token/cookie 校验
    │   │   └── rateLimit.js     # /api/track 限流防刷
    │   ├── stats.js             # 难度分析聚合逻辑
    │   └── seed.js              # 把游戏现有 7 关 time 灌入 level_config
    ├── public/admin/            # 管理后台（零构建单页 HTML + Chart.js）
    │   ├── index.html
    │   └── admin.js
    ├── data/app.db              # SQLite 数据文件（gitignore）
    ├── .env                     # ADMIN_PASSWORD_HASH, IP_SALT, JWT_SECRET...
    ├── package.json
    └── README.md
```

---

## 2. 需求一：可配置过关时长（含 fallback）

### 2.1 默认值即 fallback
现游戏每关时长写死在 `LEVELS[i].time`（第1关 `8` … 第7关 `55`）。
后端 `level_config` 表的**初始值就用这 7 个值灌入**（见 `seed.js`）。

游戏启动时拉配置：
- **成功** → 用返回值覆盖 `LEVELS[i].time`；
- **失败 / 超时 / 无后端** → 跳过覆盖，代码里写死的值原样生效。

> 因为默认值本就在 `LEVELS` 数组里，「覆盖步骤失败时被跳过」即自动回退，无需单独维护默认表。

### 2.2 前端改造点
- 在 `splash-start` 的点击处理里、`loadLevel(0)` **之前**先 `await fetchConfig()`。
- `fetchConfig()` 用 `AbortController` 设 **1.5s 超时**，避免拖慢开场；超时/失败即静默用默认值。
- 拉到的 `{level_index, time_seconds}[]` 逐条写回 `LEVELS[i].time`。

### 2.3 数据表
```sql
CREATE TABLE level_config (
  level_index   INTEGER PRIMARY KEY,   -- 0..6
  time_seconds  INTEGER NOT NULL,
  updated_at    DATETIME,
  updated_by    TEXT DEFAULT 'admin'
);
```

---

## 3. 需求二：前端埋点

### 3.1 数据模型（以「一次从头开始玩」为主线）

```
run（一次完整游玩 = 一个 run_id）
 └── level_attempt（每关每次尝试，重试 +1 条）
```

**run（游玩会话）**
```sql
CREATE TABLE run (
  run_id            TEXT PRIMARY KEY,   -- 前端 crypto.randomUUID()
  ip_hash           TEXT,               -- 后端对请求 IP 加盐哈希（不存明文）
  geo               TEXT,               -- 可选：后端实时解析的国家/城市
  user_agent        TEXT,
  started_at        DATETIME,
  last_seen_at      DATETIME,
  max_level_reached INTEGER,            -- 这次玩到的最高关（"在哪关没过去"）
  ended_reason      TEXT                -- cleared / quit / null(进行中)
);
```

**level_attempt（核心分析表）**
```sql
CREATE TABLE level_attempt (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id         TEXT,
  level_index    INTEGER,
  attempt_no     INTEGER,    -- 本 run 内该关第几次尝试（重试次数）
  allotted_time  INTEGER,    -- 当时配置时长（关联"时长 vs 结果"）
  outcome        TEXT,       -- win / lose / abandon
  duration_ms    INTEGER,    -- 实际花费
  time_left_ms   INTEGER,    -- 通关时剩余时间（判断时长松紧）
  completion_pct INTEGER,    -- 结束时完成度（失败时尤其重要）
  started_at     DATETIME,
  ended_at       DATETIME
);
```

### 3.2 埋点事件与触发位置（现有代码）

| 事件 | 触发位置 | 携带数据 |
|---|---|---|
| `game_start` | `splash-start` 点击 / "从头再玩" | 新 run_id、ua（ip 后端补） |
| `level_start` | `beginLevel()` / 倒计时结束真正开始 | level_index、attempt_no、allotted_time |
| `level_win`   | `showWin()` | duration、time_left、completion_pct(=100) |
| `level_lose`  | `showLose()` | duration、completion_pct |
| `level_retry` | `showLose()` → "重试本关" → `loadLevel` | level_index、attempt_no+1 |
| `level_jump`  | 底部关卡条跳关 | from/to |
| `run_quit`    | 页面关闭/切走未通关 | `sendBeacon` 兜底，记 max_level_reached |

### 3.3 上报实现（tracker.js）
- 主用 `navigator.sendBeacon('/api/track', payload)`（不阻塞、关页也能发），`fetch` 兜底。
- `run_id` = `crypto.randomUUID()`，存内存；"从头开始"即新 run。
- `attempt_no` 用全局计数器按关维护。
- **IP 一律后端从请求头取并哈希**，前端不采集、不传，防篡改。
- 离开页面：`visibilitychange`(hidden) / `pagehide` 触发 `run_quit`。

### 3.4 这些指标如何回答需求
- **哪关容易卡住** → 各关 `lose`/`retry` 占比、平均 `attempt_no` 最高者。
- **在哪关没过去** → run 的 `max_level_reached` 分布 + 各关流失数。
- **重试次数** → `attempt_no` 聚合。
- **时长是否合理** → 通关 `time_left_ms` 占 `allotted_time` 比例；失败 `completion_pct`（差一点=加时有效，差很多=机制偏难）。

---

## 4. 需求三：管理后台

### 4.1 页面
1. **登录页**：单口令 → 后端校验 → 签发 HttpOnly Cookie/JWT。
2. **时长配置页**：表格列出 7 关（当前时长 / 默认时长 / 建议值），可编辑保存。
3. **数据看板**（重点，下节）。

### 4.2 难度分析指标（每关）
| 指标 | 含义 | 信号 |
|---|---|---|
| 到达数 / 通过数 | 漏斗 | 流失在哪关 |
| 首次通关率 | 第1次就过的比例 | 越低越难 |
| 平均尝试次数 | attempt_no 均值 | 越高越卡 |
| 平均剩余时间占比 | time_left/allotted | ≈0 紧；偏高可缩时 |
| 失败平均完成度 | lose 时 pct | 高=差一点；低=机制难 |
| 流失率 | 到达却没继续 | 劝退点 |
| 综合难度分 | 加权 | 最难的关 |

### 4.3 图表（Chart.js）
- 关卡**漏斗图**：第1→7关到达/通过人数。
- 各关**通关率 / 平均尝试次数** 柱状图。
- 各关**耗时分布 / 剩余时间分布** 直方图。
- **难度分排行**；支持时间范围筛选（改时长前后对比）。

### 4.4 自动调参建议（后端按规则生成文案）
- 剩余时间占比 > 50% 且通关率高 → "时长偏松，可下调 N 秒"。
- 通关率低 + 剩余时间≈0 + 失败完成度高 → "玩家差一点，建议加时"。
- 通关率低 + 失败完成度低 → "机制偏难，加时收效有限"。

---

## 5. API 设计

**公开（游戏用）**
- `GET  /api/levels/config` → `[{ level_index, time_seconds }]`
- `POST /api/track` → 接收埋点（sendBeacon）

**管理（需鉴权）**
- `POST /api/admin/login`（单口令）
- `GET  /api/admin/levels` / `PUT /api/admin/levels/:idx`（改时长）
- `GET  /api/admin/stats?from=&to=`（聚合统计）
- `GET  /api/admin/funnel`（漏斗）

---

## 6. 安全 / 合规（因公网开放）

| 关注点 | 做法 |
|---|---|
| 管理鉴权 | 单口令 → HttpOnly Cookie/JWT；口令哈希存 `.env`，不写死代码 |
| 埋点防刷 | `/api/track` 按 IP 限流 + 请求体大小限制 + 字段白名单校验 |
| IP 隐私 | 仅存 `ip_hash`（加盐）；如需地域，后端实时解析成国家/城市再存，原始 IP 不落库 |
| 传输 | HTTPS（Nginx/Caddy 反代或证书） |
| 配置读写 | config GET 公开只读；写时长走管理鉴权 |

---

## 7. 实施阶段

1. **P1 — 可配置时长 + fallback 打通**
   后端骨架 + SQLite 建表 + `seed` 灌默认时长 + `GET /api/levels/config`；
   前端启动拉配置覆盖 `LEVELS`，失败回退默认。
2. **P2 — 埋点**
   `run` / `level_attempt` 表 + `POST /api/track` + 限流；
   前端 `tracker.js` + 各触发点插桩 + `run_quit` 兜底。
3. **P3 — 后台改时长**
   单口令登录 + 鉴权中间件 + 时长配置页。
4. **P4 — 统计图表**
   聚合 API（`stats.js`）+ 看板（漏斗 / 通关率 / 分布）+ 调参建议。

---

## 8. 对游戏现有代码的改动清单（小、低风险）

- 顶部引入 `tracker.js`；启动流程 `splash-start` 里先拉配置再 `loadLevel`。
- 插桩点：`beginLevel`(level_start)、`showWin`、`showLose`、重试入口、关卡条跳关、`splash-start`、"从头再玩"按钮。
- 新增全局变量：`run_id`、按关的 `attempt_no` 计数。
- 不改动游戏核心玩法逻辑（烧裂 / 蔓延 / 渲染等均不动）。
