# menily · schema

> A specification for **task-level demonstration data** for vision-language-action (VLA) models.
>
> *A project of [Menily Intelligence](https://www.menily.ai).*

## Motivation

VLA models (π0, OpenVLA, NVIDIA GR00T, Gemini Robotics, Ψ-Zero, …) consume something very specific: a *task-level* trajectory where a natural-language goal is paired with a visual context and a sequence of actions that, together, constitute one semantic unit of robot behavior.

This is different from:

- raw video (no semantic boundaries)
- motion capture files (no task annotation, no language)
- RLHF datasets (reward signals, not demonstrations)
- teleoperation traces alone (no language grounding)

There is **no standard format** for this today. Every lab invents their own. Transfer between labs is broken. Open datasets can't be pooled. Tooling can't be reused.

`menily/schema` is one attempt at a common ground.

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
    "viewpoint": "ego"  // "ego" | "third-person" | "overhead"
  },
  "action": {
    "space": "ee_6dof",  // end-effector 6DoF, or "joint_7dof", "whole_body_19dof", ...
    "trajectory": [ /* N × action_dim */ ],
    "timestamps": [ /* N */ ],
    "gripper": [ /* N × 1 binary or continuous */ ]
  },
  "body": {
    "morphology": "bimanual_humanoid",
    "dof_map": { "right_arm": [0,1,2,3,4,5,6], "left_arm": [7,8,9,10,11,12,13] }
  },
  "meta": {
    "source": "pov_video" | "vr_demo" | "mocap" | "teleop",
    "collection_region": "SEA" | "EU" | "NA" | "EA",
    "collection_time": "2026-01-14T08:20:00Z",
    "quality_flags": ["no_slip", "no_contact_gap"]
  }
}
```

## Why these fields

- **Language + variants** — same task in multiple natural languages enables multi-lingual VLA
- **Viewpoint** — ego vs third-person requires different training handling
- **Action space** — explicit space definition (end-effector vs joint vs whole-body) enables cross-embodiment transfer
- **Morphology + dof_map** — lets the same task data train different robot bodies if the semantic action is decomposable
- **Source** — retains provenance (POV video has different noise than MoCap)
- **Collection_region** — for regional distribution tracking and bias analysis

## Status

Draft v1. Comments welcome via GitHub issues — or email <Masashi@Menily.AI> for direct feedback.

Things explicitly out of scope for v1:

- [ ] Reward / return-to-go fields (this is not RL data)
- [ ] Full scene graph (visual tokens come from the frames; scene parsing is downstream)
- [ ] Human biometric metadata (we do not collect this)

## Related

- [menily/toolkit](https://github.com/MenilyIntelligence/toolkit) — reference implementation of encoders/decoders for this schema
- [menily/research](https://github.com/MenilyIntelligence/research) — design notes behind the schema decisions

---

中文：`menily/schema` 是一份针对 VLA（视觉-语言-动作）模型训练的**任务级示教数据规范**。定义 task_id / language / visual / action / body / meta 六大顶层字段，统一异构数据源的格式，便于跨机构数据池化与跨具身迁移。欢迎 issue 或邮件讨论。
