# 上游 PR 草稿

## Title

`[犀牛鸟-E3]：五族混合系统的路由透视镜`

## Summary

This PR adds a versioned routing-snapshot contract to the existing YOLO-Master routing interpreter. It reuses removable hooks and current `last_routing_snapshot` state, and does not change model forward signatures, routing decisions, checkpoints, or training behavior.

- **P0:** exports one JSON-safe schema for routed-layer identity, expert load, mean router probabilities, entropy, Gini, dominant share, dead experts, spatial capability, and auxiliary-loss availability.
- **P1:** keeps the event format directly consumable by the completed localhost dashboard and paired overhead benchmark.
- **P2:** distinguishes genuine spatial `[B,E,H,W]` routes from global `[B,E]` distributions so downstream overlays fail closed instead of inventing pixel alignment.
- **Family coverage:** explicitly documents and tests Latent, and resolves nested router leaves through their nearest MOE/MOA/MOT/LATENT/MOLoRA parent.

Single-image CLI runs now write `routing_snapshot.jsonl` beside the existing static figures and `routing_report.json`. The schema is versioned as `yolo_master.routing_snapshot.v1`, allowing WebUI and downstream studies to reject incompatible records rather than guessing field meanings.

## Acceptance results

The code diff stays focused on four existing upstream files. Full experimental evidence remains in separate, manifest-bound repositories and is linked below.

| Level | Result |
|---|---|
| Smoke | COCO8 admission path completed for MOE/MOT/LATENT; field dictionary and overhead plan archived |
| P0 | 13 routed modules emitted one validated contract with JSONL and static summaries |
| P1 | MOE/MOT/LATENT localhost dashboard; 3 seeds and 18 paired runs; all median slowdowns below 10% |
| P2 | Five-family capability audit; true original-image overlays for MOT/MOA; non-spatial families explicitly degraded |
| Extra ablation | Random, transferred, and transferred + 10-epoch conditions; 3 seeds; 1,728 captures and 1,440 aligned comparisons |

### P1 overhead

| Family | Median slowdown | 3-seed bootstrap 95% CI | Verdict |
|---|---:|---:|---|
| MOE | -0.95% | [-1.84%, 3.31%] | PASS |
| MOT | 2.15% | [-0.06%, 9.62%] | PASS |
| LATENT | 1.66% | [-1.53%, 3.02%] | PASS |

Negative values are treated as scheduler noise, not observer speedups. Data, warm-up, augmentation, training budget, and timing boundaries are matched between on/off pairs.

## Ablation study

The additional study asks whether compatible Tencent weight transfer and ten COCO8 epochs change spatial expert participation. It compares random initialization, transferred weights without training, and transferred weights followed by 10 epochs under identical settings and three seeds.

| Family | Condition | mAP50–95 | Mean active experts | All-3-active rate | Spatial variation | Appearance agreement |
|---|---|---:|---:|---:|---:|---:|
| MOT | Random | 0.0000 | 1.00 | 0.0% | 0.0000 | 100.0% |
| MOT | Transfer | 0.0059 | 1.00 | 0.0% | 0.0000 | 100.0% |
| MOT | Transfer + 10 epochs | 0.0225 | 2.46 | 54.2% | 0.0882 | 96.8% |
| MOA | Random | 0.0000 | 2.25 | 50.0% | 0.0000 | 98.4% |
| MOA | Transfer | 0.0368 | 2.75 | 77.1% | 0.0047 | 98.1% |
| MOA | Transfer + 10 epochs | 0.0359 | 2.92 | 91.7% | 0.0050 | 96.6% |

MOT passes the preregistered routing-specialization rule through increased three-expert coverage and positive spatial variation while keeping the appearance-agreement drop below five percentage points. MOA increases expert coverage but misses the margin and spatial-variation thresholds, so the result is retained as a negative finding rather than described as clearer specialization. COCO8 detection values verify the pipeline only and are not presented as generalization claims.

## Implementation notes

- `RoutingInterpreter.routing_snapshot_records()` creates one versioned record per observed routed layer.
- The CLI writes newline-delimited records for streaming consumers while preserving its existing JSON report and figures.
- Spatial capability is derived from the captured probability shape; global routers remain distributions.
- A leaf such as `EfficientSpatialRouter` inherits family identity from its nearest routed parent.
- Tests are added to the existing `tests/test_routing_interpreter.py`, following the repository contribution rules.

## Verification

Environment: Windows, Python 3.11.16, PyTorch 2.11.0+cu128, Ultralytics 8.4.101.

```bash
python scripts/check_changed_quality.py
python -m pytest -q tests/test_routing_interpreter.py tests/test_routing_diagnostics.py tests/test_latent_mixture.py
python tools/routing_interpreter.py ultralytics/cfg/models/26/yolo26-master-latent-n.yaml <COCO8-image> --imgsz 64 --device cpu --output <output>
```

- Repository quality gate: **PASS**.
- Routing, diagnostics, and Latent regression suite: **58 passed**.
- Real-forward CLI smoke: **PASS**; 9 versioned records, comprising 6 MOE leaves and 3 LATENT modules; no `family=unknown` records.
- Upstream comparison: one contribution commit and four changed files; no experiment artifacts or model weights.

## Reviewer quick start

From the YOLO-Master repository root:

```bash
python scripts/check_changed_quality.py
python -m pytest -q tests/test_routing_interpreter.py tests/test_routing_diagnostics.py tests/test_latent_mixture.py
python tools/routing_interpreter.py ultralytics/cfg/models/26/yolo26-master-latent-n.yaml path/to/image.jpg --imgsz 64 --device cpu --output runs/e3-review
```

Inspect `runs/e3-review/routing_snapshot.jsonl`. Each line declares its schema version, family, layer, routing statistics, spatial availability, collapse metrics, and auxiliary-loss status.

## Evidence

- Smoke: <https://github.com/XavierYChen/e3-routing-smoke>
- P0: <https://github.com/XavierYChen/e3-routing-p0>
- P1: <https://github.com/XavierYChen/e3-routing-p1>
- P2: <https://github.com/XavierYChen/e3-routing-p2>
- Ablation and final report: <https://github.com/XavierYChen/e3-routing-final-report>
- Contribution branch: <https://github.com/XavierYChen/YOLO-Master/tree/feat/e3-routing-snapshot-ready>

## Demo video

The P2 repository contains the reproducible 120-second browser demo and its recording script. A narrated edition with real UI state changes and burned Chinese captions is prepared as a separate release asset so the clean evidence video remains reproducible.

## Limitations

- The upstream code change serializes observations; it does not train experts or alter routing behavior.
- MOE, LATENT, and MOLoRA do not expose a reversible two-dimensional token grid at the audited hook boundary and are reported as non-spatial.
- Dominant-expert colors are categorical argmax IDs, not clusters or semantic labels; probability, entropy, and margin must be read with the map.
- Formal experiments use COCO8 and three seeds. They support mechanism and tooling claims, not detector generalization.
- The overhead result is Windows/RTX 3060 evidence; no cross-device latency claim is made.
- GitHub CI and CLA remain required after the PR is opened.


