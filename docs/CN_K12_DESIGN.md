# 中国 K12 课程接入设计方案（CN_GENERIC）

> 状态：设计草案，待维护者审核
> 日期：2026-10-30
> 关联文档：docs/SPEC.md §10（KST 概念图）、docs/CONTENT_LICENSES.md §2b（国家课程版权策略）、docs/ARCHITECTURE.md（模块布局）
> 遵循纪律：spec-first、测试驱动、文档与代码不静默漂移（SESSION_FSM.md 先例）、AI 编写 + 人类维护者决策与审核（README『How this is built』）

---

## 1. 背景与目标

为中国 K12 学生提供本地优先的数学辅导课程包（CN_GENERIC），覆盖小学一年级至高中三年级（Grade 1-12）。第一期只做数学；科学（物理/化学/生物）与语文分列后续阶段。

## 2. 关键决策（维护者已确认/待确认）

| 编号 | 决策点 | 结论 |
|---|---|---|
| D1 | 对齐策略 | **不宣称对齐**中国《课程标准》：年级名称用中国学制（Grade 1-12），概念进度按中国真实教学节奏自研编排，不引用课标条文编号。与 SG/US/IN 通用包的版权纪律一致（CONTENT_LICENSES.md §2b） |
| D2 | 生成器组织 | **不走** PACK_LEVELS 程序生成；新建 engine/cn_items.py 独立维护 CN_Gn_GENERATORS，在 item_sources.py 的build_registry() 中逐条手写注册（参照 au_items.py 先例：有独立进度主张的包不复用通用表） |
| D3 | 题目语言 | 生成器直接产出**中文题干**（不做英文占位路线），与中式题型改造合并进行 |
| D4 | 界面语言联动 | **独立语言设置**：Settings 新增『界面语言』选项，默认跟随当前课程国家（CN → 中文），可手动覆盖为英文。**不**把语言绑死在课程上 |
| D5 | 实施分期 | P1a 单年级样板先行（建议三年级），走通全链路后再批量铺开 |

## 3. 中国题目与通用包的差异分析

### 3.1 题型风格差异
- 中国数学以**应用题**为骨架：鸡兔同笼、行程问题、工程问题、盈亏问题、浓度问题等经典母题；题干叙事、单位体系本土化（元/角/分、千米/小时、亩）。
- 中式独有题型载体：竖式计算、脱式计算、简便运算（凑整/运算律刻意训练）、估算、单位换算大题。
- 通用包生成器产出的是直算题 + 简短西式情境题（pizza、marbles），不可直接搬用。

### 3.2 进度节奏差异
- 中国进度整体超前 1-2 年：多位数乘除、分数四则、负数、方程、平面几何证明均早于美国同年级；通用包中性 stage（按 US 节奏对齐）直接映射会『太简单』。
- 中国独有或高权重内容：速算训练、初中几何证明（全等/相似）、高中导数/数列/立体几何的深度。

### 3.3 能力导向差异
- 中式题目强调计算熟练度与技巧；通用包偏向概念理解。影响的不只是题干文字，而是 Item 的结构（参数随机方式、答案形态、method_steps 内容）。

## 4. 技术方案

### 4.1 生成器层：新建 src/mentar/engine/cn_items.py

三层复用策略：

| 层 | 内容 | 复用方式 |
|---|---|---|
| 直接复用 | 纯计算类：四则运算、分数/小数运算、负数、方程求解 | import 现有生成器函数，重新挂到 cn_gN_ 前缀 node_id 下（in_generic_items.py 模式） |
| 改造 | 情境题：换中式情境与单位（元角分、千米/小时），数值随机逻辑不变 | 复制生成器改题干模板与单位；量大时抽取题干模板表 |
| 新写 | 中式独有题型：应用题母题（鸡兔同笼、行程、工程）、简便运算、竖式、几何证明、高中导数/数列 | 全新 GenFn，遵守 Item 契约（problem/answer_type/checker/answer/format_hint/method_steps/visual） |

约束：
- node_id 统一 cn_g{N}_ 前缀，全局唯一性由 tests/engine/test_template_catalog.py::test_no_skill_id_collides_across_any_shipped_template 强制；
- GenFn 契约不变，参考 itemgen.py / au_items.py / science_items.py；
- 概念进度按中国真实教学节奏编排（知识点结构事实不受版权保护；不引用课标条文、不宣称官方对齐）。

### 4.2 注册表：engine/item_sources.py

在 build_registry() 中逐条手写注册：cn_g1_maths ~ cn_g12_maths，共 12 条，形如 cn_g1_maths -> {generators: CN_G1_GENERATORS, itembank: None}。

模板 front matter 写 item_source: cn_gN_maths，缺失时启动报错（web/app.py 既有校验）。

### 4.3 课程模板：curriculum/templates/CN_GENERIC/

新建 g1_maths.md ~ g12_maths.md，按 W3.1 schema：
- country: CN、year_level: Grade N、subject: mathematics、item_source: cn_gN_maths；
- concepts[].id 与 cn_items.py 的 node_id 一致；配 prereqs 构成 DAG、verifier、bkt_priors、transfer_seeds（中文种子题）；
- language_register 写明中文表达规范（字数、句式、禁用代数符号的年龄下限等）；
- 参考样板：curriculum/templates/US_GENERIC/g10_maths.md、编写指南 curriculum/_template.md。

### 4.4 Web 层改动

**A. Settings 课程选项（自动 + 2 处小改）**
- web/app.py 的 COUNTRY_NAMES 新增 CN -> China；
- web/static/settings.js 的 FLAGS 内联 SVG 字典新增中国国旗（手绘简化 SVG，遵守 U-80 禁止 CDN、Windows 无国旗 emoji 字形的既有决策）；
- tab 分组、国家总开关、年级开关、pack_state.json 持久化、_reload_curricula() 热生效均由现有机制自动覆盖，**UI 逻辑零改动**。

**B. 界面 i18n（新增机制，D4）**

现状：全项目无 i18n 框架，15 个 Jinja 模板页（_base/learner/settings/parent/progress/subjects/topics/done/setup 等）与 settings.js/nudge.js 文案全部英文硬编码。

方案：
1. 新建 i18n 机制：模板文案抽成键值 + 语言包（en/zh）；语言偏好存 cookie/session（参照 _learner_subject 模式）；
2. Settings 新增『界面语言』选项：默认跟随当前课程国家（CN → zh），可手动覆盖；
3. JS 文案（settings.js 的 Turn the X curriculum on or off 等）抽为由接口下发的字典；
4. 覆盖范围：孩子端（learner/done/topics/subjects）+ 家长端（settings/parent/progress/setup）15 页 + 2 个 JS 文件。

### 4.5 对话/提示层

1. **答案格式提示**：dialogue/controller.py 的 _answer_format_hint()（约 1485 行）提示语硬编码英文（answer like _/_、answer with a letter: A, B, C or D），需按当前科目语言返回中文版本；
2. **Prompts**：prompts/system_prompt.md 及 12 个 help/pattern/transfer 模板全英文。方案：在既有插槽机制（{{subject}}/{{learner_register}}/{{answer_type}}）上新增 {{language}} 插槽注入语言指令；如效果不足再为 prompts 建中文变体文件；
3. **LLM 中文能力**：确认 config/model_roster.yaml 中本地模型的中文输出能力（qwen 系通常可用）。

### 4.6 TTS

web/static/tts.js 用浏览器 speechSynthesis，声音按 localStorage mentar-tts-voice 选取。中文题干需默认选中文语音（zh-CN），否则英文嗓音读中文不可用。改动小但必须做。

### 4.7 答案解析兼容性（验证为主）

eval/verify_numeric.py 与 checker（int_exact/fraction_equiv/mc_choice）解析数字/分数/ABCD，中文作答数字一般兼容；需测试中文全角字符（，。／）混入答案时的解析健壮性，必要时在解析前做全角→半角归一化。

## 5. 改动清单汇总

| 层 | 改动 | 工作量 |
|---|---|---|
| Settings 选项 | COUNTRY_NAMES +1 行、FLAGS SVG +1 段 | 极小 |
| 界面 i18n | 新建语言机制 + 15 页模板 + 2 个 JS 文案抽取 + 语言设置项 | 大（新增机制） |
| 生成器 | engine/cn_items.py（三层策略，中文题干） | 中-大 |
| 注册表 | item_sources.py 手写 12 条 | 小 |
| 课程模板 | CN_GENERIC/ 12 个 W3.1 模板 | 中 |
| 格式提示 | _answer_format_hint 按语言分支 | 小 |
| Prompts | {{language}} 插槽 × 13 个文件（或中文变体） | 中 |
| TTS | 中文语音默认选择 | 小 |
| 答案解析 | 全角字符兼容验证/归一化 | 验证为主 |
| 文档 | ARCHITECTURE.md、README（pack 表/nodes 数）、CONTENT_LICENSES.md §2b、PHASE0_STATUS.md 决策记录 | 小 |

## 6. 分期

1. **P1a（样板先行）**：只做三年级数学（cn_g3），完整走通『cn_items.py → 注册表 → 模板 → 中文题干 → i18n → TTS → 测试 → 选课页』全链路，暴露所有机制风险；
2. **P1b**：按 P1a 样板批量铺 12 个年级数学；
3. **P2**：科学/物理/化学/生物（可复用 science_items / senior_science_items 机制，参照 SENIOR_LEVELS 先例）；
4. **P3**：语文（全新题型体系，单独立项）。

## 7. 风险

1. **版权红线**：模板不出现课标原文/条目编号/对齐声称；
2. **进度合理性**：中国专属进度编排需以国内主流教材目录的知识点结构为依据，由维护者审核；
3. **i18n 是本方案最大新增机制**，建议独立 workstream 管理（类比 W7 grounding），避免与课程数据改动耦合；
4. **AI 编写纪律**：D1-D5 属于人类维护者决策；实施产物（模板/生成器/测试/文档）按项目惯例由智能体生成、测试门控、维护者审核。

## 8. 测试与实施计划

见配套文档：docs/CN_K12_PLAN.md（含里程碑、任务分解与测试方案）。
