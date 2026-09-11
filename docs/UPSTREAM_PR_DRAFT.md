# 上游 PR 草稿

## 标题

`[犀牛鸟-E3]：增加版本化跨族路由快照与 Latent 支持`

## 改动摘要

E3 需要为诊断工具和 WebUI 提供稳定的跨族 routing snapshot。现有解释器能够采集多类路由，但 CLI 只输出聚合报告，Latent 也没有出现在文档和回归覆盖中。

本改动在现有 `RoutingInterpreter` 上增加 `yolo_master.routing_snapshot.v1` 记录，将每个已观测路由层的家族、专家负载、概率形状、空间能力、熵、Gini、坍塌指标和辅助损失状态写成 JSON-safe 字段。单图 CLI 同时输出 `routing_snapshot.jsonl`。家族识别支持叶子路由器从最近的 routed parent 继承 MOE/MOA/MOT/LATENT/MOLoRA 类型，不修改任何模型核心 `forward`。

## 测试证据

环境：Windows；Python 3.11.16；PyTorch 2.11.0+cu128；Ultralytics 8.4.101。

```bash
python scripts/check_changed_quality.py
python -m pytest -q tests/test_routing_interpreter.py tests/test_routing_diagnostics.py tests/test_latent_mixture.py
```

结果：质量门禁通过，58 项测试通过。另用 `yolo26-master-latent-n.yaml`、一张 COCO8 图片、CPU、`imgsz=64` 完成 CLI smoke，输出 9 条版本化记录（6 MOE、3 LATENT）及对应静态图。

## 消融数据

该 PR 只增加旁路诊断与序列化，不改变路由概率、模型参数或训练路径，因此模型精度消融不适用。on/off 验证口径是：关闭导出时沿用原有推理；开启 CLI 快照导出时读取同一次 capture 的结果。现有测试验证概率归一化、top-k 重建、空间/全局区分、Latent 记录和 CLI 文件输出。

E3 的训练后专家分工、三 seed 和开销数据保存在独立研究包中，不把实验资产并入该上游代码 PR，以保持 review 范围集中。

## 已知局限

- `v1` 只序列化当前解释器能够观测到的路由层；未实现统一协议且无法从父模块识别的第三方路由器会标为 `unknown`。
- 图像级路由只报告全局分布，不能生成空间热图。
- `aux_loss.available=false` 表示本次采集没有可读状态，不代表模型从未定义辅助损失。
- 正式 PR 仍需 GitHub CI 与 CLA；若腾讯 `main` 在提交前更新，需要重新检查 merge base 和 4 个目标文件。


