# 中国 K12 课程接入实施计划与测试方案

> 状态：计划草案，待维护者审核
> 日期：2026-10-30
> 配套设计文档：docs/CN_K12_DESIGN.md（决策 D1-D5、技术方案、差异分析）
> 任务编号约定：CN-{阶段}.{序号}；测试编号 CN-T{序号}

---

## 1. 里程碑总览

| 里程碑 | 内容 | 完成判据 |
|---|---|---|
| M0 | 决策与基线确认 | D1-D5 经维护者确认；当前测试套件（约 790+）全绿作为基线 |
| M1（P1a） | 三年级数学样板全链路 | cn_g3 从生成器到选课页完整可用；新增测试全绿；界面中文可选 |
| M2（P1b） | 12 个年级数学批量铺开 | cn_g1-g12 全部模板通过校验；skill_id 无冲突；全量测试绿 |
| M3（P2） | 科学/物理/化学/生物 | 参照 senior_science_items 先例接入 |
| M4（P3） | 语文（单独立项） | 全新题型体系设计文档先行 |

## 2. P1a 任务分解（三年级样板）

### 阶段 A：生成器与注册表


| 任务 | 内容 | 产出 |
|---|---|---|
| CN-A.1 | 盘点三年级概念清单（约 10-14 个节点：万以内加减、乘法口诀、除法、简单分数、周长、单位换算等），按中国进度编排 prereqs DAG | 概念清单表（维护者审核） |
| CN-A.2 | 新建 src/mentar/engine/cn_items.py：直接复用层 import 现有计算类生成器并挂 cn_g3_ 前缀 | CN_G3_GENERATORS 骨架 |
| CN-A.3 | 改造层：中式情境题（元角分、千米/小时），中文题干 | 情境题生成器若干 |
| CN-A.4 | 新写层：中式题型（如乘法口诀速算、简单应用题），中文题干 + method_steps | 新 GenFn 若干 |
| CN-A.5 | engine/item_sources.py 的 build_registry() 注册 cn_g3_maths | 注册表条目 1 条 |

### 阶段 B：课程模板

| 任务 | 内容 | 产出 |
|---|---|---|
| CN-B.1 | 新建 curriculum/templates/CN_GENERIC/g3_maths.md：W3.1 schema front matter（country: CN、year_level: Grade 3、item_source: cn_g3_maths）+ concepts（id 对齐生成器 node_id、prereqs、verifier、bkt_priors、中文 transfer_seeds、language_register 中文表达规范） | 模板文件 1 个 |
| CN-B.2 | 通过 validate_or_raise() 校验（A16），DAG 无环、prereq 引用有效 | 校验通过 |

### 阶段 C：Web 层

| 任务 | 内容 | 产出 |
|---|---|---|
| CN-C.1 | web/app.py 的 COUNTRY_NAMES 新增 CN -> China | 1 行 |
| CN-C.2 | web/static/settings.js 的 FLAGS 新增中国国旗内联 SVG（手绘简化，U-80 禁 CDN） | SVG 1 段 |
| CN-C.3 | i18n 机制立项：文案键值抽取规范 + en/zh 语言包结构 + 语言偏好存取（cookie/session，参照 _learner_subject） | i18n 基础设施 |
| CN-C.4 | Settings 新增『界面语言』选项：默认跟随课程国家（CN → zh），可手动覆盖 | 设置项 |
| CN-C.5 | P1a 范围内页面中文化：learner、settings、subjects、topics、done 五页 + settings.js 文案 | 语言包 zh 条目 |

### 阶段 D：对话与语音

| 任务 | 内容 | 产出 |
|---|---|---|
| CN-D.1 | dialogue/controller.py 的 _answer_format_hint() 按当前科目语言返回中文提示 | 语言分支逻辑 |
| CN-D.2 | prompts/system_prompt.md 等 13 个 prompt 模板新增 {{language}} 插槽 | prompt 更新 |
| CN-D.3 | web/static/tts.js 中文语音默认选择（zh-CN） | TTS 适配 |
| CN-D.4 | 确认 config/model_roster.yaml 本地模型中文能力 | 验证结论记录 |

## 3. P1b 任务分解（批量铺开）

| 任务 | 内容 |
|---|---|
| CN-E.1 | 按 P1a 模式逐年级编写概念清单（维护者逐个审核进度合理性） |
| CN-E.2 | 扩展 cn_items.py：CN_G1_GENERATORS ~ CN_G12_GENERATORS；初中几何证明、高中导数/数列等新题型在此阶段新写 |
| CN-E.3 | item_sources.py 补齐 12 条注册 |
| CN-E.4 | CN_GENERIC/ 补齐 g1-g12 模板 |
| CN-E.5 | i18n 语言包补齐剩余 10 页（parent、progress、setup、gallery 等）与 nudge.js |

## 4. 测试方案

### 4.1 测试原则

- 遵循项目测试驱动纪律：新增代码必须带测试，全量套件门控变更；
- 基线：实施前记录当前测试总数与全绿状态（近期记录 792 个），完成后不得减少、不得变红；
- ruff 检查保持干净。

### 4.2 单元测试

| 编号 | 测试内容 | 位置 |
|---|---|---|
| CN-T1 | cn_items.py 每个 GenFn：随机种子确定性（固定 seed 产出可复现）、Item 契约完整（problem/answer_type/checker/answer 非空且类型正确） | tests/engine/test_cn_items.py |
| CN-T2 | 中文题干基本质量：非空、含中文字符、参数插值正确（无残留占位符） | 同上 |
| CN-T3 | 新写应用题母题：答案数学正确性（多组随机参数断言 checker 通过自身 answer） | 同上 |
| CN-T4 | _answer_format_hint 中文分支：cn 上下文返回中文提示，其余返回英文（回归） | tests/dialogue/ |
| CN-T5 | 全角字符归一化：全角数字/斜杠混入答案时 checker 仍正确判分 | tests/eval/ 或 tests/engine/ |

### 4.3 集成测试

| 编号 | 测试内容 | 位置 |
|---|---|---|
| CN-T6 | 注册表集成：build_registry() 含 cn_g3_maths，键 → generators 映射可解析 | tests/engine/ |
| CN-T7 | 模板校验：g3_maths.md 通过 validate()；故意构造环/坏 prereq 的负例被拒（A16） | tests/tools/ 或 tests/engine/ |
| CN-T8 | 端到端出题链路：加载 cn_g3 → fringe 选节点 → 生成器出题 → _answer_spec 对齐 → checker 判分 → BKT 用模板 bkt_priors 更新 | tests/dialogue/ 或 tests/engine/ |
| CN-T9 | 现有强制检查自动覆盖：skill_id 全局唯一（test_template_catalog.py）、answer_type/checker 语法合法 | 已有测试，确认通过即可 |

### 4.4 Web 测试

| 编号 | 测试内容 | 位置 |
|---|---|---|
| CN-T10 | /settings/curricula 返回含 CN 国家组与 cn_g3 条目；country_names 含 CN | tests/web/ |
| CN-T11 | 开关 cn_g3 pack：/settings/curricula/<key>/enable 后 _reload_curricula() 热生效，/choose 出现该科目；禁用后消失（参照 test_curriculum_toggle.py 模式） | tests/web/ |
| CN-T12 | 语言设置：设置 zh 后页面渲染中文文案；手动覆盖 en 生效；默认跟随课程国家 | tests/web/ |
| CN-T13 | 坏模板回滚：cn 模板损坏时 toggle 返回错误且不改状态（_apply_pack_change 既有行为回归） | tests/web/ |

### 4.5 人工验收（维护者）

| 编号 | 内容 |
|---|---|
| CN-A1 | 三年级概念清单与进度的教学合理性审核 |
| CN-A2 | 中文题干语言质量抽检（适合 8-9 岁、无歧义、单位正确） |
| CN-A3 | 浏览器实测：Settings 中国 tab、国旗渲染（Windows 环境重点验证）、中文界面切换、TTS 中文朗读 |
| CN-A4 | 版权合规检查：模板无课标原文/编号/对齐声称 |

### 4.6 文档同步检查（防漂移）

| 编号 | 内容 |
|---|---|
| CN-T14 | 完成时同步：ARCHITECTURE.md 模块表、README pack 表（nodes/templates 计数重算）、CONTENT_LICENSES.md §2b、PHASE0_STATUS.md 决策记录 |

## 5. 依赖与顺序

M0 决策确认 → CN-A.1 概念清单（维护者审核，阻塞后续）→ CN-A.2~A.5 生成器 + 注册表；CN-B.1~B.2 模板；CN-C.1~C.5 Web（C.3 i18n 机制可与 A/B 并行）；CN-D.1~D.4 对话与语音 → 测试 CN-T1~T13 + 人工验收 CN-A1~A4 → M1 完成 → P1b 批量铺开。

## 6. 风险与缓解

| 风险 | 缓解 |
|---|---|
| i18n 机制改动面大、易引发回归 | 独立阶段（CN-C.3）先行；P1a 只做 5 页；每页切换后跑既有 web 测试 |
| 中文 LLM 输出质量不达预期 | CN-D.4 提前验证；prompt {{language}} 插槽不足时降级为中文 prompt 变体文件 |
| 中国进度编排争议 | 概念清单维护者审核（CN-A.1）为硬阻塞点 |
| 新题型 Item 契约不适配（如几何证明无法自动判分） | 证明题用 mc4（判断证明步骤正误）等可判分形态；不可判分节点走 LLM 路径 + free_text（不参与 BKT，符合现有 bkt.py 行为） |
