# docs — 项目文档索引

| 文档 | 内容 |
| :--- | :--- |
| [01-调研-现成方案盘点.md](01-调研-现成方案盘点.md) | ESP32 + 显示屏展示 AI token plan 用量的现成开源项目调研（2026-11），含 Stack-chan 复用路线分析 |
| [02-方案-Token用量桌面显示屏.md](02-方案-Token用量桌面显示屏.md) | 选型结论、Stack-chan 集成详案、主机桥设计、一周实施计划 |

## 一句话结论

**有大量现成方案可直接套用**：社区最成熟的是 [Clawdmeter](https://github.com/HermannBjorgvin/Clawdmeter)（⭐2352，只支持 Claude Code）；多平台可参考 [token-genie](https://github.com/Jensen-JZ/token-genie)（MIT，含中文界面）。

**若已有 Stack-chan（M5Stack CoreS3 桌面机器人），优先复用它**：零硬件成本，把用量显示做成机器人的新技能（脸屏底部 overlay + 超限变脸/语音提醒），主机桥代码在所有显示路线间通用。详见 02 文档 §2-3。
