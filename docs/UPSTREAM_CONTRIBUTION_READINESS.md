# E3 上游贡献就绪审计

审计日期：2026-09-11（UTC+8）

## 结论

贡献代码已经整理在 fork 分支 [`feat/e3-routing-snapshot`](https://github.com/XavierYChen/YOLO-Master/tree/feat/e3-routing-snapshot)。该分支直接基于腾讯 `main` 的 `9eb6312fa3e4a17abd79bf47ab1e49d567d1a910`，领先 1 个提交、落后 0 个提交。跨仓库比较仅包含 4 个目标文件，没有夹带 Smoke、P0、P1、P2 的实验产物，也没有修改模型核心 `forward`。

比较入口：<https://github.com/Tencent/YOLO-Master/compare/main...XavierYChen:YOLO-Master:feat/e3-routing-snapshot>

## 贡献范围

| 文件 | 上游改动 |
|---|---|
| `ultralytics/utils/routing_interpreter.py` | 增加版本化、JSON-safe 的逐层 routing snapshot；补充 Latent 和嵌套路由家族识别 |
| `tools/routing_interpreter.py` | 单图 CLI 额外输出 `routing_snapshot.jsonl`，并给报告增加 schema 版本 |
| `tests/test_routing_interpreter.py` | 在既有测试文件中覆盖 Latent、JSONL、空间能力字段和父层家族识别 |
| `docs/governance/routing-interpretability.md` | 补充 Latent 支持、调用示例、字段范围和版本处理原则 |

Schema 版本为 `yolo_master.routing_snapshot.v1`。每条记录包含家族、层名、模块类型、运行上下文、专家数、top-k、专家负载、平均路由概率、张量形状、空间能力、熵、Gini、主导专家、死亡专家、坍塌状态和辅助损失可用性。

## 与腾讯 main 的关系

| 检查项 | 结果 |
|---|---|
| 上游基线 | `Tencent/YOLO-Master@9eb6312` |
| merge base | `9eb6312`，与审计时腾讯 `main` 一致 |
| ahead / behind | `1 / 0` |
| 变更文件 | 4 |
| 内容冲突 | 未发现 |
| 原始腾讯工作区 | 未修改、未 rebase |
| Discussion / PR | 均未创建 |

如果腾讯 `main` 在正式提交前继续变化，应重新运行跨仓库比较，并确认这 4 个路径没有新冲突。GitHub 在创建 PR 时会再次计算可合并状态。

## 本地验证

环境：Windows，Python 3.11.16，PyTorch 2.11.0+cu128，Ultralytics 8.4.101；测试使用仓库开发环境，CPU smoke 使用 `imgsz=64`。

```text
python scripts/check_changed_quality.py
结果：通过（Ruff lint、Ruff format、codespell）

python -m pytest -q tests/test_routing_interpreter.py tests/test_routing_diagnostics.py tests/test_latent_mixture.py
结果：58 passed

python tools/routing_interpreter.py ultralytics/cfg/models/26/yolo26-master-latent-n.yaml <COCO8 image> --imgsz 64 --device cpu --output <output>
结果：成功；9 条 JSONL，6 条 MOE、3 条 LATENT；生成 routing_report.json、routing_snapshot.jsonl 和静态图
```

真实 smoke 曾发现内部 `EfficientSpatialRouter` 只能从父模块判断所属家族；代码已修正，并增加回归测试。最终结果中不再出现 `family=unknown`。

## 贡献准则对照

- 使用 fork 和描述性分支；改动聚焦于 E3 路由快照基础设施。
- 新功能测试添加到既有 `tests/test_routing_interpreter.py`。
- 新公开方法采用 Google 风格 docstring，并同步更新既有治理文档。
- 官方本地质量门禁已通过。
- 未上传数据集、权重、个人路径、运行日志或大体积图片。
- 代码不执行任意 shell，不写入模型状态，不改变推理和训练结果。

腾讯准则建议首次贡献保持较小范围，并要求 feature PR 先有获批的 feature request。E3 是犀牛鸟认领任务，但这是否等同于仓库维护者的 GitHub feature approval，应在正式提交时用任务通知或导师说明作为上下文，不在此文档中擅自宣称已获维护者批准。

## 正式提交前动作

1. 确认腾讯 `main` 仍以当前 merge base 为基础，或重新同步一个新贡献分支。
2. PR 标题使用：`[犀牛鸟-E3]：增加版本化跨族路由快照与 Latent 支持`。
3. 使用 `UPSTREAM_PR_DRAFT.md` 的四节正文，并附 E3 任务来源或认领说明。
4. 创建 PR 后等待 CI；按机器人提示签署 CLA。


