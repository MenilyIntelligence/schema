# menily/schema

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Status: Draft v1](https://img.shields.io/badge/status-draft%20v1-orange.svg)](#status)
[![Schema Version](https://img.shields.io/badge/schema-menily.task--demo%2F1-green.svg)](#core-fields)
[![Menily Intelligence](https://img.shields.io/badge/by-Menily%20Intelligence-black.svg)](https://www.menily.ai)

> An open specification for **task-level demonstration data** for vision–language–action (VLA) models.
> Six fields. Controlled vocabularies. Apache-2.0. Draft v1.

## Table of contents

- [What this is](#what-this-is)
- [Why it exists](#why-it-exists)
- [Core fields (v1 draft)](#core-fields-v1-draft)
- [Field-level rationale](#field-level-rationale)
- [Out of scope for v1](#out-of-scope-for-v1)
- [How to use](#how-to-use)
- [Interoperability](#interoperability)
- [Extended design notes](#extended-design-notes)
- [Status and roadmap](#status-and-roadmap)
- [Related projects](#related-projects)
- [Citation](#citation)
- [License](#license)
- [Contributing](#contributing)
- [中文简介](#中文简介)

---

## What this is

`menily/schema` is a JSON specification that defines **one unit of task-level demonstration data** for training vision–language–action (VLA) models. A single file = a single task. Every file conforms to the same six top-level fields: `task_id`, `language`, `visual`, `action`, `body`, `meta`.

The specification is language-agnostic (JSON). A Python reference implementation is available in [menily/toolkit](https://github.com/MenilyIntelligence/toolkit).

## Why it exists

VLA models (π0, OpenVLA, NVIDIA GR00T N1, Gemini Robotics, Ψ₀, ...) consume something very specific: a *task-level* trajectory where a natural-language goal is paired with a visual context and an action sequence that, together, form one semantically closed unit of robot behavior.

This is different from:

- raw video (no semantic boundaries)
- motion capture files (no task annotation, no language)
- RLHF datasets (reward signals, not demonstrations)
- teleoperation traces alone (no language grounding)

As of April 2026 there is **no publicly released, interoperable specification** for this layer. Every laboratory and company invents its own format. Cross-institutional data pooling is broken. Open datasets can't be merged. Tooling can't be reused.

`menily/schema` is one attempt at a common ground. Its twin goals:

1. **Interoperate** with Open X-Embodiment / RLDS (trajectory layer) downstream and BONES-SEED / NVIDIA SOMA (motion layer) upstream.
2. **Be adoptable** by labs that don't want to be locked into a single vendor — the schema is Apache-2.0 and requires no runtime dependencies on Menily Intelligence.

## Core fields (v1 draft)

```json
{
  "schema_version": "menily.task-demo/1",
  "task_id": "uuid",
  "language": {
    "instruction": "Pour water from the blue cup into the kettle.",
    "language_code": "en",
    "variants": ["给水壶加水", "…"]
  },
  "visual": {
    "frames": "path/to/frames/",
    "fps": 30,
    "camera_intrinsics": { "fx": 1128.5, "fy": 1128.5, "cx": 960, "cy": 540 },
    "viewpoint": "ego"
  },
  "action": {
    "space": "ee_6dof",
    "trajectory": [ /* N × action_dim */ ],
    "timestamps": [ /* N */ ],
    "gripper": [ /* N × 1 */ ]
  },
  "body": {
    "morphology": "bimanual_humanoid",
    "dof_map": {
      "right_arm": [0,1,2,3,4,5,6],
      "left_arm":  [7,8,9,10,11,12,13]
    }
  },
  "meta": {
    "source": "pov_video",
    "collection_region": "SEA",
    "collection_time": "2026-01-14T08:20:00Z",
    "quality_flags": ["no_slip", "no_contact_gap"]
  }
}
```

### Controlled vocabularies

| Field | Allowed values |
|---|---|
| `visual.viewpoint` | `"ego"` \| `"third-person"` \| `"overhead"` |
| `action.space` | `"ee_6dof"` \| `"joint_Ndof"` \| `"whole_body_Mdof"` |
| `body.morphology` | `"single_arm"` \| `"bimanual"` \| `"bimanual_humanoid"` \| `"mobile_manipulator"` \| `"quadruped"` \| `"humanoid"` |
| `meta.source` | `"pov_video"` \| `"vr_demo"` \| `"mocap"` \| `"teleop"` \| `"sim_generated"` |
| `meta.collection_region` | `"NA"` \| `"EU"` \| `"SEA"` \| `"EA"` \| `"SA"` \| `"AF"` \| `"OC"` |

## Field-level rationale

| Field | Decision | Why |
|---|---|---|
| `language.variants` | Recommended-required | Multilingual paraphrase is ~zero marginal cost (LLM-generated) and critical for deployment robustness. |
| `visual.viewpoint` | Controlled vocabulary | Ego vs third-person are qualitatively different training signals; mixing without labels degrades visual encoders. |
| `action.space` | Controlled vocabulary; single space per file | Implicit action spaces are the most common cause of silent training corruption. |
| `body.morphology` | Required | Cross-embodiment transfer is unrecoverable without explicit morphology. |
| `body.dof_map` | Required | DoF index → joint mapping is non-discoverable from raw trajectories. |
| `meta.source` | Controlled vocabulary | Different sources have qualitatively different noise profiles; downstream cleaning depends on this. |
| `meta.collection_region` | Top-level field | Geographic distribution is a commonly-ignored bias source; making it first-class forces awareness. |

## Out of scope for v1

The following are deliberately **excluded** from v1 scope:

- ❌ **Reward / return-to-go fields** — `menily/schema` is not an RL data format. Use D4RL or RLDS for reinforcement learning datasets.
- ❌ **Scene graphs** — scene parsing is a downstream task; visual tokens come from frames.
- ❌ **Human biometric metadata** — Menily does not collect, and no schema field is reserved.
- ❌ **Embedded URDF / MJCF** — body morphology is a compact index; full physics models are referenced externally.

## How to use

### Reading a file

Any JSON parser will do. Below is Python using the `menily/toolkit` helper:

```python
from menily.toolkit import schema

task = schema.TaskLevelDemoV1.load("./task_001.json")

print(task.language.instruction)      # "Pour water from the blue cup..."
print(task.action.space)              # "ee_6dof"
print(task.body.morphology)           # "bimanual_humanoid"
print(len(task.action.trajectory))    # N
```

### Validating a file

```python
report = task.validate()

assert report.passed
for w in report.warnings:
    print("warning:", w)
```

Or as a standalone CLI (PyPI release pending):

```bash
menily-schema validate ./task_001.json
```

### Writing a file

```python
from menily.toolkit import pov, schema

tasks = pov.segment(
    video_path="./demo.mp4",
    language="Pour water from the blue cup into the kettle.",
    language_variants=["把蓝色杯子里的水倒进水壶里"],
    fps=30, viewpoint="ego",
    body_morphology="bimanual_humanoid",
    collection_region="SEA",
)

for task in tasks:
    task.save_as(schema="menily.task-demo/1", out_dir="./out/")
```

## Interoperability

Designed to interoperate with existing standards:

| Direction | Target | Method |
|---|---|---|
| Downstream | [Open X-Embodiment / RLDS](https://robotics-transformer-x.github.io/) | `Task.to_rlds()` |
| Downstream | [HuggingFace Datasets](https://huggingface.co/docs/datasets) | `Task.to_hf_dataset()` |
| Upstream | [NVIDIA SOMA / SOMA-X](https://arxiv.org/abs/2603.16858) | `body.morphology` + `body.dof_map` namespace-aligned |
| Upstream | [BONES-SEED](https://huggingface.co/datasets/bones-studio/seed) | Consumed as motion source; task-level semantics overlaid |
| Bidirectional | RLDS | `from_rlds()` converts Open X-Embodiment data in |

## Extended design notes

A long-form walkthrough of every field decision — why `language.variants` is recommended-required, why `action.space` is a controlled vocabulary, why `body.morphology` + `body.dof_map` are the real keys to cross-embodiment transfer, and the 15→6 field consolidation process:

- 📝 [VLA 任务级示教数据 schema 设计笔记：Menily/schema v1 规范与六字段解析](https://blog.csdn.net/2611_95864581/article/details/160290391) — Masashi, CSDN, April 2026
- 📝 [给 VLA 训练数据设计一份 schema：六字段是怎么砍下来的](https://www.cnblogs.com/) — Masashi, cnblogs.com, April 2026 *(retrospective on field cuts)*
- 📄 [Task-Level Demonstration Data for VLA Models: A Survey](https://www.menily.ai/research/01-task-level-vla-data-survey.pdf) — 12-page preprint, April 2026

## Status and roadmap

**v1 is a draft**, not a finalized standard. Field-level critique via GitHub Issues is welcome and actively incorporated.

- [x] v1 field set frozen (6 top-level fields)
- [x] Controlled vocabularies defined
- [x] Interoperability tested against RLDS + HF Datasets
- [ ] Reference validator CLI (pending [menily/toolkit](https://github.com/MenilyIntelligence/toolkit) PyPI release)
- [ ] Worked examples for Unitree G1, Fourier GR-1, Apptronik Apollo
- [ ] Schema v2 — long-horizon task decomposition, multi-agent scenarios, `invariant_landmarks` waypoint schema

## Related projects

| Repo | Description |
|---|---|
| [menily/toolkit](https://github.com/MenilyIntelligence/toolkit) | Python reference implementation — three adapters (POV / VR / MoCap) + schema validator |
| [menily/research](https://github.com/MenilyIntelligence/research) | Research notes on design decisions behind this schema |
| [menily.ai](https://www.menily.ai) | Organization site — team, publications, contact |

## Citation

If you use `menily/schema` in research, please cite:

```bibtex
@misc{menily2026schema,
  author       = {Masashi},
  title        = {menily/schema: A Task-Level Demonstration Data
                  Specification for Vision-Language-Action Models},
  year         = {2026},
  howpublished = {Menily Intelligence, Apache-2.0 open specification},
  url          = {https://github.com/MenilyIntelligence/schema},
  note         = {Version menily.task-demo/1, draft v1}
}
```

The companion survey paper:

```bibtex
@misc{masashi2026tasklevel,
  author       = {Masashi},
  title        = {Task-Level Demonstration Data for Vision-Language-Action
                  Models: A Survey of Schemas, Adapters, and
                  Cross-Embodiment Transfer},
  year         = {2026},
  month        = {April},
  howpublished = {Menily Intelligence Research, self-hosted preprint},
  url          = {https://www.menily.ai/research/01-task-level-vla-data-survey.pdf},
  note         = {Draft v0.1}
}
```

## License

Apache License 2.0 — see [`LICENSE`](./LICENSE) (to be added with first tagged release).

## Contributing

- 🐛 **Bug reports & clarifications** → [open an Issue](https://github.com/MenilyIntelligence/schema/issues/new)
- 💡 **Field-level design proposals** → PRs welcome for spec text; discuss in an Issue first
- 📧 **Direct technical discussion** → <Masashi@Menily.AI>
- 🌐 **Organization** → [github.com/MenilyIntelligence](https://github.com/MenilyIntelligence) · [menily.ai](https://www.menily.ai)

v1 is a draft — we expect two kinds of feedback:

1. **Field-level critique** — naming, semantics, granularity, split/merge decisions.
2. **Format mapping requests** — if your team has existing data pipelines and wants to see how to map to `menily/schema`, email to open a discussion.

---

## 中文简介

`menily/schema` 是一份针对 VLA（视觉-语言-动作）模型训练的**任务级示教数据规范**。定义 `task_id` / `language` / `visual` / `action` / `body` / `meta` 六大顶层字段，统一异构数据源的格式，便于跨机构数据池化与跨具身迁移。

设计目标：**与 Open X-Embodiment / RLDS（轨迹层）向下兼容**，**与 BONES-SEED / NVIDIA SOMA（动作层）向上兼容**，填补中间的任务级语义层。

v1 是草案，欢迎通过 [GitHub Issues](https://github.com/MenilyIntelligence/schema/issues) 提字段设计建议，或邮件 <Masashi@Menily.AI> 讨论现有数据格式与 `menily/schema` 的互转方案。

更长的设计笔记：[VLA 任务级示教数据 schema 设计笔记（CSDN）](https://blog.csdn.net/2611_95864581/article/details/160290391)
