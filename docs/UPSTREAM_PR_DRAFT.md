# 上游 PR 草稿

## Title

`[犀牛鸟-E3]：五族混合系统的路由透视镜`

## Summary

E3 需要为诊断工具、后续研究题和 WebUI 提供稳定的跨族 routing snapshot。现有解释器能够采集多类路由，但单图 CLI 只输出聚合报告，Latent 未进入统一文档与回归覆盖。

本 PR 在现有 `RoutingInterpreter` 上增加版本化的 `yolo_master.routing_snapshot.v1` 记录，将每个已观测路由层的家族、专家负载、平均路由概率、概率形状、空间能力、熵、Gini、主导占比、死亡专家、坍塌状态和辅助损失可用性写为 JSON-safe 字段。单图 CLI 在保留原有静态图与 `routing_report.json` 的同时写出 `routing_snapshot.jsonl`。

- **P0：** 统一 schema；显式支持 MOE、MOT、MOA、LATENT、MOLoRA，并覆盖 Latent 的真实 forward。
- **P1：** JSONL 事件可直接供已完成的本地实时面板消费；三族、3 seeds、18 组 paired on/off 测量的中位减速均低于 10%。
- **P2：** 依据实际概率张量形状区分空间路由 `[B,E,H,W]` 与全局分布 `[B,E]`，无法恢复二维 token 网格时显式降级，避免制造伪热图。
- **安全边界：** 只读取既有 hooks 和 `last_routing_snapshot`；不修改模型核心 `forward`、路由决策、参数、checkpoint 或训练行为。

## Testing

环境：Windows；Python 3.11.16；PyTorch 2.11.0+cu128；Ultralytics 8.4.101。

```bash
python scripts/check_changed_quality.py
python -m pytest -q tests/test_routing_interpreter.py tests/test_routing_diagnostics.py tests/test_latent_mixture.py
python tools/routing_interpreter.py ultralytics/cfg/models/26/yolo26-master-latent-n.yaml path/to/image.jpg --imgsz 64 --device cpu --output runs/e3-review
```

- Repository quality gate: **PASS**。
- 路由、diagnostics 与 Latent 回归：**58 passed**。
- 真实 forward CLI smoke：**PASS**；输出 9 条版本化记录，其中 6 个 MOE 叶子和 3 个 LATENT 模块，没有 `family=unknown`。
- 2026-09-11 最新腾讯 `main` 为 `af961b99b8ef80491e58cb5fd16e25ebaf3741eb`；分叉后的 10 个上游提交未修改本 PR 的 4 个目标文件。

Reviewer 可检查 `runs/e3-review/routing_snapshot.jsonl`。每行都声明 schema 版本、家族、层、路由统计、空间可用性、坍塌指标与辅助损失状态。

## Ablation study

额外消融比较随机初始化、迁移腾讯兼容权重、迁移后训练 10 epochs 三种条件。各组使用同一 COCO8、输入尺寸、外观扰动、统计口径和 3 个 seeds；共归档 1,728 份路由捕获及 1,440 组对照。

| Family | Condition | mAP50–95 | Mean active experts | All-3-active rate | Spatial variation | Appearance agreement |
|---|---|---:|---:|---:|---:|---:|
| MOT | Random | 0.0000 | 1.00 | 0.0% | 0.0000 | 100.0% |
| MOT | Transfer | 0.0059 | 1.00 | 0.0% | 0.0000 | 100.0% |
| MOT | Transfer + 10 epochs | 0.0225 | 2.46 | 54.2% | 0.0882 | 96.8% |
| MOA | Random | 0.0000 | 2.25 | 50.0% | 0.0000 | 98.4% |
| MOA | Transfer | 0.0368 | 2.75 | 77.1% | 0.0047 | 98.1% |
| MOA | Transfer + 10 epochs | 0.0359 | 2.92 | 91.7% | 0.0050 | 96.6% |

MOT 满足预先定义的路由分工判据：三专家覆盖率提高、空间变化为正，且外观一致率下降少于 5 个百分点。MOA 的专家覆盖率提高，但未同时满足 margin 与空间变化阈值，因此保留为负结果。COCO8 检测值只验证流水线，不作为泛化结论。

## P1 overhead

| Family | Median slowdown | 3-seed bootstrap 95% CI | Verdict |
|---|---:|---:|---|
| MOE | -0.95% | [-1.84%, 3.31%] | PASS |
| MOT | 2.15% | [-0.06%, 9.62%] | PASS |
| LATENT | 1.66% | [-1.53%, 3.02%] | PASS |

负数按调度噪声解释，不声称 observer 能加速训练。on/off 对照的数据、预算、增广、warm-up 与计时边界一致。

## Evidence and reproduction

- Smoke: <https://github.com/XavierYChen/e3-routing-smoke>
- P0: <https://github.com/XavierYChen/e3-routing-p0>
- P1: <https://github.com/XavierYChen/e3-routing-p1>
- P2: <https://github.com/XavierYChen/e3-routing-p2>
- Ablation and final report: <https://github.com/XavierYChen/e3-routing-final-report>
- Contribution branch: <https://github.com/XavierYChen/YOLO-Master/tree/feat/e3-routing-snapshot-ready>

P2 仓库包含可复现的 120 秒浏览器演示、录屏脚本和中文女声硬字幕版。最新版移除了模拟光标，使用 Noto Sans SC 渲染 51 条中文字幕；开头、中段和结尾均有抽帧证据。

## Known limitations

- `v1` 只序列化当前解释器能够观测到的路由层；无法从统一协议或父模块识别的第三方路由器标为 `unknown`。
- MOE、LATENT 与 MOLoRA 在本次审计的 hook 边界不提供可逆二维 token 网格，只报告全局分布。
- 主导专家颜色是 categorical argmax ID，不是聚类或语义标签；需结合概率、熵和 margin 解读。
- 正式实验使用 COCO8 和 3 seeds，支持机制与工具链结论，不支持检测器泛化结论。
- 开销数据来自 Windows/RTX 3060，不外推到其他设备。
- PR 创建后仍需腾讯 GitHub CI 与 CLA 完成最终检查。
