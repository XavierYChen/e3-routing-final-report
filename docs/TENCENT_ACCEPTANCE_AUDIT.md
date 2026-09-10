# 腾讯 E3 最终验收审计

审计时间：2026-09-10。口径来自 E3 原始任务书、P0/P1/P2 分级目标、8.24 准入检查、统一实验红线和交付物要求。

## 总体判断

**Smoke、P0、P1 的核心验收条件已经满足；P2 的代码、真实空间叠加、五族能力边界、训练后交互页和两分钟视频已经满足；独立且公开的 Final 仓库与阶段导航也已建立。P2 已把最新版合并至 main，并清理为单分支。正式提交前只剩一个外部决定：确认“工具 PR”最终应投向课程收集仓库还是 Tencent/YOLO-Master。**

消融属于额外研究证据，不是 P0/P1/P2 的替代物。它补齐了多 seed 专家分工问题：MOT 得到正结论，MOA 得到有证据链的负结论。

## 验收矩阵

| 要求 | 证据 | 状态 | 审计意见 |
|---|---|---|---|
| 8.24：COCO8 routing snapshot | Smoke JSON、静态图、字段字典 | ✅ | 单图、命令、配置与环境均保留 |
| 8.24：字段字典 | Smoke/P0 schema 文档 | ✅ | usage 语义和 aux loss 缺失状态明确 |
| 8.24：开销测量方案 | Smoke overhead plan | ✅ | 与正式 P1 训练开销区分 |
| P0：统一 schema | `e3.routing_record.v1` | ✅ | MOE/MOT/LATENT 三族，共 13 records |
| P0：结构化日志 | JSON、JSONL sinks | ✅ | 非有限值、stale snapshot 会失败 |
| P0：静态图 | contract coverage 与 snapshot 图 | ✅ | 图中带数值，避免仅凭颜色 |
| P1：TensorBoard/W&B 或实时面板 | 本地实时 HTML/JS 面板 | ✅ | 三选一要求；无需再接 W&B |
| P1：覆盖至少 3 族 | MOE/MOT/LATENT | ✅ | dashboard `latest.json` 与历史 JSONL 均有 |
| P1：训练减速 `<10%` | GPU 3 seeds / 18 paired runs | ✅ | 中位数 -0.95%、+2.15%、+1.66% |
| P1：on/off、warm-up、重复与区间 | 配置、raw runs、bootstrap CI | ✅ | MOT GPU 非确定性已显式说明 |
| P2：token 路由叠加原图 | MOT/MOA 真 `[E,H,W]` overlay | ✅ | letterbox 反变换与 padding 排除有验证 |
| P2：更多路由族 | 五族 capability manifest | ✅/降级 | 只有 MOT/MOA 有空间轴；其余三族禁止伪造热图 |
| P2：外观/路由分析图 | sensitivity、attribution、scatter、usage | ✅ | 概率、argmax、margin 联合解释 |
| P2：2 分钟 demo | 训练后交互页 + 120.0 秒 MP4 + SHA-256 | ✅ | 1600×900、10 fps；抽帧覆盖路由与四类分析图 |
| 至少 3 seed 或声明局限 | P1 与 Final 消融 3 seeds | ✅ | 单 checkpoint P2 图仍标明单 seed |
| 负结果有预定义判读线 | Final `configs/ablation.yaml` | ✅ | MOA 未过线，如实报告 |
| PR 四节齐全 | 各阶段 PR description、Final 模板 | ✅ | 改动、测试、消融、局限齐全 |
| 版本/环境/配置路径 | env、config、commit、SHA-256 | ✅ | 已区分上游基线和本地集成 commit |
| 路径白名单、日志脱敏、无任意 shell | runner 固定参数与公开边界 | ✅ | 权重和原始 logits 不上传 |
| 工具 PR | 阶段仓库与 Draft PR | ⚠️ | 还未向 Tencent/YOLO-Master 上游提交最终 PR |

## P1 面板判定

P1 选择任务允许的第三条路线“实时面板”，没有接 TensorBoard 或 W&B。浏览器页面每秒轮询原子替换的 `latest.json`，训练端同时追加 `routing_records.jsonl`，所以既有实时状态也有断电后历史。`run_dashboard.cmd` 启动只监听 `127.0.0.1:8765` 的本地服务。该实现覆盖 MOE、MOT、LATENT，并且写盘和面板更新时间已计入 on 条件，因此满足 P1。

## P2 图像判定

旧截图容易造成“前两层仍是单色”的误解。最新 seed-0 capture 表明 MOT `model.13/16/19/22` 在四张 identity 图上的平均活跃专家数为 `2.25/2.50/3.00/2.00`；对展示样本则为 `2/2/3/2`。所以只有 `model.19` 稳定三色，其他层两色属于真实逐层差异。P2 新增 `trained-mot-layer-focus.png`，直接在图下注明 active experts 和 dominant token counts。

三色本身不是验收目标。更可靠的证据是：训练后 MOT 在三种子消融中平均活跃专家为 2.46、全专家率为 54.2%、空间变化为 0.0882、外观一致率为 96.8%。

## 正式提交前建议顺序

1. 在答辩材料中统一使用总 manifest 锁定的这组数字。
2. 与导师确认“工具 PR”提交到课程收集仓库还是 Tencent/YOLO-Master；若要求上游 PR，再从独立工具仓库准备最小、无权重的集成分支。

## 不建议继续做的工作

- 不为了让每层都出现三色而改阈值或调色。
- 不用 600 epochs 的检测训练代替路由工具验收。
- 不把 COCO8 mAP 写成泛化性能提升。
- 不为 MOE/LATENT/MOLoRA 伪造像素级空间轴。
- 不额外接 W&B；现有实时面板已经满足 P1，新增云依赖只会扩大风险。
