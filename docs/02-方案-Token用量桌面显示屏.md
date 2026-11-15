# 方案：Token 用量桌面显示（Stack-chan / ESP32 + 屏幕）

> 状态：v2 草案（2026-11-15，新增 Stack-chan 路线） · 调研依据见 [01-调研-现成方案盘点.md](01-调研-现成方案盘点.md)

## 1. 目标

做一台放在桌面的实体小屏，**一眼看到各个 token plan 的余量**，避免干活干到一半撞限流：

- **Must**：Claude Code 订阅 5h / 7d 利用率 + 重置倒计时
- **Should**：智谱 GLM Coding Plan 配额（`open.bigmodel.cn/api/monitor/usage/quota/limit`）
- **Nice**：Codex、OpenRouter 余额、Claude API 用量；接近满时变色/提醒

原则：**有现成的直接套用**，自研部分只做「数据聚合」这一层。

## 2. 路线选型

| 路线 | 做法 | 优点 | 缺点 | 结论 |
| :--- | :--- | :--- | :--- | :--- |
| **0. 复用已有 Stack-chan** | 给桌面的 Stack-chan（M5Stack CoreS3 机器人）加"用量显示"技能 | **¥0 硬件成本**；表现力上限高（变脸/语音/转头提醒）；本来就常驻桌面 | 屏幕与"脸"共用（320×240）；刷自定义固件会临时顶掉原 AI 功能；无先例可抄 | **✅ 已有设备则主推** |
| A. 直接刷 Clawdmeter | 买 AMOLED 2.16 板，跑其 daemon | 最成熟、效果炫 | 只支持 Claude Code；无 license 不能再分发；绑死主机蓝牙；要买板 | 备选 |
| B. token-genie 底子 + 扩展 GLM 源 | 买圆形 AMOLED 板刷其固件，host 桥里加 GLM provider | 架构天然多平台、MIT、自带中文 UI、WiFi 独立于主机 | 项目小众；要买板（¥230+）；固件构建繁琐 | 备选（想要常驻专用仪表盘时） |
| C. Ulanzi TC001 + AWTRIX 3 | 现成像素钟，REST 推内容 | 零固件开发，当天完成 | 32×8 点阵只能滚字；要买钟 | 快速验证 |

**决策**：**已有一台 Stack-chan → 走路线 0**。理由：

1. 硬件能力完全够用：CoreS3（ESP32-S3 + 8MB PSRAM + WiFi）轮询一个 JSON 接口毫无压力，320×240 画几条进度条绰绰有余；
2. 它本来就是"桌面上看着我干活的伙伴"——用量数据显示给它比给一块冷冰冰的仪表屏更自然，且告警表现力（见 §3.2）是所有专用屏都做不到的；
3. 零新增成本，一周内可完成。

之后若想要"信息密度更高的常驻仪表盘"，再按原方案 B 补一块 Waveshare 屏——**主机桥完全复用**（这是本方案最重要的架构不变量：桥与显示端解耦，Stack-chan / Waveshare / Ulanzi 都只是不同的 sink）。

## 3. Stack-chan 集成设计（路线 0 详案）

### 3.1 系统结构

```
┌──────────────────────────────┐        ┌──────────────────────┐
│ 主机 (Linux, 常开)            │        │ 各官方用量端点         │
│  usage_bridge.py (stdlib)    │        │  Anthropic (OAuth)   │
│   ├ provider: claude ────────┼───────►│  bigmodel (API key)  │
│   ├ provider: glm ───────────┤        └──────────────────────┘
│   └ GET /usage?key=xxx       │
└──────────────┬───────────────┘
               │ 局域网 WiFi（60s 轮询）
               ▼
┌──────────────────────────────────────────┐
│ Stack-chan (CoreS3, 320×240 触摸屏)       │
│  默认：脸 + 屏幕底部 32px 用量条 overlay    │
│  触摸/长按切换：全屏 stats 页（多 plan 列表）│
└──────────────────────────────────────────┘
```

### 3.2 显示与告警设计

| 场景 | 表现 |
| :--- | :--- |
| 默认待机 | 正常显示脸；屏幕底部固定一条 24–32px 高的 overlay：`CC ▓▓▓░ 42%` `GLM ▓░░ 8%`（细进度条 + 百分比，不遮挡眼睛） |
| 触摸屏幕 / 触摸条 | 切换全屏 stats 页：每个 plan 一行（名称 + 进度条 + 百分比 + 重置倒计时），再触摸返回脸 |
| 任一 plan > 80% | 眼睛变累（半闭眼）+ 对应进度条变橙；>95% 变红 + LED 变红 |
| 越过阈值瞬间 | 语音播报（如"5 小时的量快用完啦"）+ 转头朝向主人（反馈舵机） |
| Claude Code 等输入 | 借鉴 clawdometer-eink 思路：桥解析会话状态，NEEDS-YOU 时 Stack-chan 转头 + 呼叫（远期） |

### 3.3 固件路径（二选一，可先后都做）

| 路径 | 做法 | 适用 |
| :--- | :--- | :--- |
| **MVP：自定义 Arduino/PlatformIO sketch** | 用官方 [StackChan-BSP](https://github.com/m5stack/StackChan-BSP)（PlatformIO 库，带 examples）写独立程序：M5Unified 画 UI + HTTPClient 轮询 + Avatar 库画脸 | 一周内出活；代价是原厂 AI 语音功能被临时顶掉（M5Burner/OTA 随时可刷回原厂） |
| **进阶：fork 官方固件加 app 模式** | 在 [m5stack/StackChan](https://github.com/m5stack/StackChan) 固件源码上加一个"用量"应用模式，与 AI 功能共存、按键切换 | 保留全部原厂能力；需先读官方固件源码结构，工期 +2~3 天 |

先做 MVP 验证数据流和 UI，再决定是否投入 fork 官方固件。

### 3.4 聚合 JSON

与原方案一致（桥不变，见 §4），Stack-chan 端只消费：

```json
{ "updated": 1781063381,
  "claude": { "five_hour": {"util": 42, "reset_at": 1781070000},
              "seven_day": {"util": 17, "reset_at": 1781600000} },
  "glm":    { "util": 8, "reset_at": 1781069000, "plan": "GLM Coding Pro" },
  "ok": true }
```

## 4. 主机桥：provider 插件化

token-genie 的 `usage_server.py`（纯 stdlib）改造点：

```
host/
  usage_bridge.py        # HTTP server + 缓存最后一帧成功快照（401 时续用旧值）
  providers/
    base.py              # Provider 协议: fetch() -> dict, 每 provider 独立 try/except
    claude.py            # 读 ~/.claude/.credentials.json（复用 token-genie 逻辑）
    glm.py               # 读 BIGMODEL_API_KEY 环境变量 → quota/limit 端点
    codex.py             # 可选，沿用原实现
```

要点（直接照抄各现成项目的成熟做法）：
- token **每次刷新时重读磁盘**，自动跟随 CLI 的轮换（token-genie）
- 单个 provider 挂掉不影响整帧（ESP32-Token-Display 的多 key 思路）
- 轮询 60s 一次，带 `?key=` 简单鉴权，只监听局域网
- Claude 走**响应头方案**（Clawdmeter 路线）而非 `/api/oauth/usage`（有 429 前科）

## 5. 硬件与成本

**路线 0（Stack-chan，主推）：¥0** — 复用已有设备，无需采购。

备选路线硬件（仅当想要常驻专用仪表盘时再买）：

| 物品 | 型号 | 参考价* | 备注 |
| :--- | :--- | :--- | :--- |
| 主控+屏 | Waveshare ESP32-S3-Touch-AMOLED-1.75（圆形 466×466） | ¥230–260 | token-genie 官方适配板 |
| 或 | Waveshare ESP32-S3-Touch-AMOLED-2.16（方形 480×480） | ¥200–230 | Clawdmeter 回退路线用 |
| 电池（可选） | 3.7V Li-Po (MX1.25 2-pin) | ¥20 | 板载 AXP2101 PMU |

\* 微雪官方/淘宝大致价位，以实际为准。

## 6. 一周实施计划（路线 0：Stack-chan MVP）

| 天 | 任务 | 产出 |
| :--- | :--- | :--- |
| D1 | **数据层先行**：`usage_bridge.py` + `providers/claude.py` + `providers/glm.py`；主机 curl 验证 | `GET /usage` 返回聚合 JSON |
| D2 | StackChan-BSP 环境：PlatformIO + BSP examples 跑通；最小 sketch：连 WiFi、轮询 `/usage`、串口打印 | 设备能拿到 JSON |
| D3 | UI v1：Avatar 脸 + 屏幕底部 overlay 进度条（CC 5h/7d + GLM） | 待机态可见用量条 |
| D4 | 全屏 stats 页：触摸切换、每个 plan 一行（条形 + 百分比 + 重置倒计时） | 完整信息页 |
| D5 | 告警：>80% 累眼变橙 / >95% 变红；跨阈值语音播报 | 主动提醒生效 |
| D6 | 稳定性：WiFi 重连、桥断连降级（灰条 + "offline"）、主机 systemd 自启桥 | 7×24 常驻 |
| D7 | 录视频/拍照、回写本仓库 README 项目表、（可选）给 StackChan 社区发帖 | 文档归档 |

进阶（第二周起）：fork [m5stack/StackChan](https://github.com/m5stack/StackChan) 官方固件，把用量页做成与 AI Agent 共存的 app 模式。

## 7. 验收标准

- [ ] 桌面抬眼可见各 plan 实时利用率（overlay 条），触摸可看详情页，与官方 App 数字一致
- [ ] Claude OAuth token 轮换后桥接服务无需重启
- [ ] 任一 provider 失效时其余照常显示，失效项标灰
- [ ] >80% / >95% 阈值告警准确触发且不重复轰炸（同一窗口只提醒一次）
- [ ] 原厂固件可随时刷回（M5Burner/OTA 验证一次），机器人本体无变砖风险
- [ ] 无人值守运行 7×24h 不掉线（WiFi 重连 + 桥异常恢复）

## 8. 后续 Idea（做完再说）

- 让 Stack-chan 的情绪跟着用量走：用量高 → 疲惫脸/叹气，窗口重置 → 开心转圈（表现力是这条路线独有的乐趣）
- 学 clawdometer-eink：解析 Claude Code 会话状态，轮到人工确认时转头呼叫主人
- 把 ccusage 的历史统计（周/月 token 消耗与成本）画成 sparkline 页
- 用 AWTRIX 3 / Ulanzi 做一个便宜的副屏同步显示（桥已就绪，只加一个 sink）
- OpenRouter / DeepSeek / Kimi provider 插件化沉淀为独立开源项目（用 MIT 干净资产，规避 Clawdmeter 的版权字体/吉祥物）
