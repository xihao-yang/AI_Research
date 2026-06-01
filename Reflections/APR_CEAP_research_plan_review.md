# APR Execution-Aware APCA 研究计划与可行性评审整理

整理日期：2026-05-30  
原始资料文件：`D:\Paper&rResearch\APR_execution_aware_assessment_report.md`  
整理目标：将本次对话中关于 APR、APCA、CEAP、ML/DL/LLM 使用方式、相关工作、竞争风险与 IEEE/TOSEM 审稿视角下的可行性判断整合成一份 Markdown 文档。

---

## 1. 总体结论

建议将研究方向从宽泛的“APR + LLM”收窄为：

> **CEAP: A Calibrated Execution-Aware Framework for Patch Correctness Assessment in Automated Program Repair**

中文解释：

> 提出一个面向 APR 的执行感知补丁正确性评估框架，把静态代码变化、上下文表示、测试执行行为、覆盖率、路径、trace 等证据融合起来，对“已经通过测试的 plausible patch”输出一个可校准的正确性或过拟合风险判断。

相比题目：

> **An Empirical Study of Execution-Aware Features for Patch Correctness Assessment in APR**

更推荐使用 CEAP 版本。原因是后者听起来更像“只做特征实验”，容易和已有 APCA 工作撞题；CEAP 强调 **new model/framework**，更符合教授希望看到的研究贡献形式。

---

## 2. 推荐题目与研究边界

### 2.1 推荐题目

首选：

> **CEAP: A Calibrated Execution-Aware Framework for Patch Correctness Assessment in Automated Program Repair**

备选：

> **Toward Execution-Aware Patch Correctness Assessment for Overfitting Patch Detection in Automated Program Repair**

更强调 oracle 的版本：

> **Execution-Aware Oracle Calibration for Patch Correctness Assessment in Automated Program Repair**

不建议现在使用：

> **An Empirical Study of Execution-Aware Features for Patch Correctness Assessment in APR**

原因是这个题目太像已有 APCA 论文的命名方式，也会把贡献压低成“经验研究”。教授既然说最好是 proposing a new model/framework，就应该把题目写成 framework，而不是 empirical study。

### 2.2 研究边界

你的研究不应该定位为：

- 不是新的 patch generation 工具。
- 不是 Breaking Change 专用修复工具。
- 不是单纯 prompt LLM 判断 patch 正误。
- 不是把所有向量、trace、coverage、代码上下文都塞进 LLM。
- 不是围绕部署优化或 token 消耗展开。

你的研究应该定位为：

- APR 流程中的 **patch correctness assessment / overfitting patch detection**。
- 输入是已经由 APR 工具生成、并且通过已有测试的 plausible patch。
- 输出是 correctness probability、overfitting risk score，或 accept/reject/rerank/need-more-validation 决策。
- 贡献是“如何把静态补丁表示和执行证据组织成一个可验证的评估框架”。

APR 流程中的位置：

```text
Buggy Program
   |
Fault Localization
   |
Patch Generation
   |
Available Test Suite
   |
Plausible Patches
   |
CEAP: Execution-Aware Patch Correctness Assessment
   |
Accept / Reject / Rerank / Trigger Extra Validation
```

---

## 3. 和関口智也论文的关系

関口智也的硕论题目是：

> **Template-based Automated Program Repair for Breaking Changes**

日文方向是：

> **Breaking Change に由来するバグの自動修正**

### 3.1 他做了什么

根据本地 thesis 的摘要和正文，他的研究核心是：

- 研究对象：依赖库升级导致的 API Breaking Change。
- 目标：自动修复受 Breaking Change 影响的 client code。
- APR 阶段重点：fault localization + patch generation + patch validation。
- Fault Localization：使用 APIMiner 检测 Breaking Change，并做影响分析。
- 依赖分析：使用 DependencyFinder。
- 生成方式：template-based repair。
- 修复对象：APIMiner 可检测的 10 类 Breaking Change。
- 实现：设计 8 种修复模板。
- 结果：方法 1 在 error level 减少 81%，client level 减少 64%；方法 2 在 error level 减少 15%，client level 减少 3%。
- RQ3 中，对不会直接引起错误的 Breaking Change，15 个 client 中 8 个没有引入新错误，约 53%。

### 3.2 他和你的方向有什么不同

| 维度 | 関口智也 | 你的方向 |
|---|---|---|
| 主问题 | Breaking Change 导致的 client code 修复 | APR 中 plausible patch 的正确性评估 |
| APR 阶段 | patch generation 为主 | patch validation/APCA 为主 |
| 输入 | API 变化、依赖图、client code | buggy program、candidate patch、tests、execution evidence |
| 输出 | 修复后的 client code patch | correctness/overfitting risk score |
| 方法 | APIMiner + DependencyFinder + template repair + code2vec 辅助 | static representation + execution evidence + calibrated ML/DL model |
| 评估重点 | 错误减少率、新错误、复杂度、技术债 | false acceptance reduction、correct patch preservation、calibration、generalization |
| 主要风险 | template 覆盖不足、类型解析、Breaking Change 检测误差 | 数据标签、执行证据设计、overfitting patch 判定边界 |

可以这样衔接：

> 高田研已经有 APR/Breaking Change 自动修复方向的积累，但我的方向不是继续做 Breaking Change patch generation，而是做 APR 生成 patch 之后的 correctness assessment。関口的结果也说明，自动修复后仍然需要判断 patch 是否真的可靠，尤其是是否引入新错误或只是表面通过验证。

这能避免和他撞题，也能把你的题目接在实验室已有 APR 脉络之后。

---

## 4. 为什么用 ML / DL / LLM，而不是盲目使用

教授问的核心不是“能不能用 LLM”，而是：

> 这个子问题为什么需要这种模型？为什么不用更简单、更可解释的方法？为什么不是把所有信息塞进一个黑箱？

### 4.1 如果只能选一种：建议以 DL 为核心

如果整个研究只能选一个主模型家族，建议选 **DL**，不是纯 LLM，也不是只用传统 ML。

理由：

- APCA 的输入不是单纯表格数据，而是代码 token、AST、patch diff、上下文、coverage、trace、test outcome 等混合结构。
- 传统 ML 适合做强 baseline，但很难学习代码结构和执行序列中的隐含模式。
- 纯 LLM 容易被质疑为 prompt-based judgment，难以校准，也不适合直接吃 numeric execution features。
- DL 可以用已有 code encoder、trace encoder、fusion layer，不需要从零发明复杂神经网络。

推荐表述：

> The core model should be a controlled DL-based assessment model with explicit static and execution-aware representations. Traditional ML will be used as interpretable baselines, while LLMs may be used only for auxiliary semantic tasks such as assertion generation, code-intent summarization, or explanation. The correctness decision itself should not rely on an uncalibrated LLM judgment.

### 4.2 什么时候用传统 ML

传统 ML 适合：

- patch size
- edit type
- test pass/fail pattern
- coverage delta
- suspiciousness score
- changed branch count
- exception/runtime error category
- trace distance

可用模型：

- Logistic Regression
- Random Forest
- XGBoost / LightGBM
- SVM with calibration

作用：

- 做 baseline。
- 证明 execution-aware features 本身有没有信号。
- 给教授展示不是一开始就用大模型。

### 4.3 什么时候用 DL

DL 适合：

- code token sequence
- AST path
- control/data-flow graph
- patch diff representation
- coverage matrix
- execution trace sequence
- static + dynamic evidence fusion

可用模型：

- CodeBERT / GraphCodeBERT encoder
- Transformer encoder
- GNN
- MLP fusion network
- attention-based fusion

作用：

- 学习手工特征不容易表达的代码语义和执行行为模式。
- 把静态补丁表示和执行证据融合起来。
- 作为 CEAP 的核心评估模型。

### 4.4 什么时候用 LLM

LLM 适合：

- 生成 test oracle / assertion。
- 生成额外测试。
- 总结 bug report 或 code intent。
- 为模型输出生成 explanation。
- 在没有足够训练数据时做 zero-shot / few-shot baseline。

LLM 不适合：

- 直接作为最终 correctness oracle。
- 直接吃一堆 numeric vectors。
- 代替明确的 execution evidence design。
- 没有 calibration 的 accept/reject 决策。

### 4.5 给教授的回答

中文：

> 我不会把所有向量直接扔进 LLM。我的假设是：patch 是否过拟合，不仅取决于 patch 的静态外观，也取决于它执行时是否改变了关键路径、覆盖了哪些语句、原 failing test 和新生成 test 的行为是否一致。因此我会先用传统 ML 验证结构化 execution-aware features 是否有效，再用 DL 学习代码上下文和执行 trace 这种高维结构化输入。LLM 如果使用，只会用于生成 assertion、总结代码意图或解释模型输出，而不是直接替代 correctness decision。

英文：

> I am not proposing to feed all vectors into an LLM. My hypothesis is that execution behavior contains correctness signals that are missing from static patch representations. I would first validate these signals with interpretable ML baselines. Then I would use DL only where representation learning is necessary, such as code context, AST paths, and execution traces. LLMs, if used, should support semantic or generative subtasks such as assertion generation, code-intent summarization, or explanation. The core contribution is an evidence-guided execution-aware assessment framework, not blind LLM usage.

---

## 5. Proposed Framework: CEAP

### 5.1 Framework 名称

> **CEAP: Calibrated Execution-Aware Patch Correctness Assessment**

### 5.2 输入与输出

输入：

- buggy program
- candidate patch
- original test suite
- test outcomes
- coverage information
- execution traces or path summaries
- optional generated assertions/tests
- optional developer patch, if benchmark provides it

输出：

- correctness probability
- overfitting risk score
- accept/reject/rerank decision
- optional explanation

### 5.3 架构

```text
Candidate Patch
   |
   +--> Static Evidence Extractor
   |       - patch diff
   |       - changed method context
   |       - AST/control/data-flow features
   |       - edit type and patch size
   |
   +--> Execution Evidence Extractor
   |       - failing/passing test outcomes
   |       - coverage delta
   |       - trace/path behavior
   |       - runtime exception behavior
   |       - generated oracle/test response
   |
   +--> Representation Fusion Model
   |       - ML baseline
   |       - DL fusion model
   |       - optional LLM semantic auxiliary signal
   |
   +--> Calibration Layer
   |       - reliability curve
   |       - confidence calibration
   |
   +--> APCA Decision
           - correct / overfitting
           - risk score
           - reranking decision
```

### 5.4 新意应该放在哪里

不要把新意写成“我也用 LLM 判断 patch correctness”。更好的新意是：

1. **Execution-aware evidence design**  
   明确定义哪些执行信息可以帮助判断 plausible patch 是否过拟合。

2. **Static-dynamic fusion framework**  
   把 ODS/CACHE/APPT/LLM4PatchCorrect 一类静态或语义表示，和执行行为证据合并。

3. **Calibrated APCA decision**  
   输出不仅是 binary label，还包括 confidence/risk score，避免盲目接受模型判断。

4. **Ablation-driven validation**  
   比较 static only、execution only、static + execution、DL fusion、LLM-only、DL + LLM auxiliary。

5. **Correct patch preservation**  
   不只看抓出多少 overfitting patch，也要避免误杀稀缺的 correct patch。

---

## 6. 与已有 APR/APCA 工作的关系

### 6.1 2020-2026 时间线

| 年份 | 代表工作/趋势 | 对你题目的意义 |
|---:|---|---|
| 2020 | CoCoNuT, APCA empirical comparison, CodeBERT | neural APR 和 code representation 成为基础；APCA 开始系统化比较。 |
| 2021 | CURE, ODS, DeepRL4FL | code-aware NMT、静态特征 APCA、coverage representation learning 出现。 |
| 2022 | CACHE, RewardRepair, DEAR | APCA 从手工特征走向 context/AST embedding；execution feedback 开始进入 repair training。 |
| 2023 | APR in the era of large PLMs, PatchZero/LLM4PatchCorrect early versions | LLM/PLM 进入 APR 和 APCA，但也带来 prompt sensitivity、leakage、memorization 风险。 |
| 2024 | APPT, LLM4PatchCorrect, Oracle-Guided Program Selection, RePair | PLM/LLM APCA 和 oracle-guided selection 活跃；“通过测试不等于正确”成为更明确问题。 |
| 2025 | RepairAgent, TOGLL, ChatAssert, PRISM, APR@ICSE 2025 | agentic repair、test oracle generation、semantic overfitting detection、benchmark/data quality 成为热点。 |
| 2026 | ICSE/ICST 2026 overfitting/APCA/oracle 相关工作 | overfitting 被更明确地视为理论和实践边界；APCA 更需要 execution/oracle-aware evidence。 |

### 6.2 关键论文如何支撑你的方向

| 工作 | 已有贡献 | 你的延伸点 |
|---|---|---|
| ODS | 使用静态代码特征分类 overfitting patch | 加入执行行为，验证静态特征之外的信号。 |
| CACHE | 用上下文和 AST 做 code change embedding | 把 context-aware 扩展为 execution-aware。 |
| APPT | fine-tune pretrained model 做 patch correctness prediction | 不只依赖预训练语义表示，引入 test/trace/coverage evidence。 |
| LLM4PatchCorrect | 使用 LLM 做 patch correctness assessment | 避免 LLM-only，强调可校准和可消融的 evidence fusion。 |
| PRISM | 使用 semantic features 检测 overfitting patch | 可作为最接近的竞争工作；你的差异要放在 execution-aware calibration framework。 |
| ChatAssert / TOGLL | 用 LLM 生成 test oracle/assertion | 可作为 optional oracle enhancement，不是核心 APCA 决策器。 |
| RewardRepair | 使用 execution-based backpropagation | 支撑“执行信息对 APR 有价值”这一动机。 |

### 6.3 最近 1-2 年 IEEE/ACM 趋势

最近 1-2 年的趋势不是简单的“大家都在用 LLM 修 bug”，而是：

1. **LLM/agentic APR 增强了 patch generation 能力**  
   这会产生更多 plausible patch，也让 correctness assessment 更重要。

2. **oracle/test quality 成为瓶颈**  
   ChatAssert、TOGLL、oracle-guided selection 都说明社区在关注“测试和 oracle 是否足够强”。

3. **APCA 正从静态表示走向语义和行为证据**  
   ODS/CACHE/APPT/LLM4PatchCorrect/PRISM 是一条清楚的演进线。

4. **evaluation threat 更受重视**  
   LLM memorization、benchmark leakage、data quality、overfitting undecidability 都会影响 APR 研究可信度。

---

## 7. ChatAssert 对你的启发

ChatAssert 不是 APCA 论文，但它很适合回答“为什么用 LLM”。

### 7.1 ChatAssert 为什么用 LLM

ChatAssert 的任务是 test oracle/assertion generation。它要输出具体 assertion code，而不是简单分类标签。这个任务需要：

- 理解 test prefix。
- 理解 focal method。
- 选择合适 API。
- 生成可编译的 assertion。
- 根据编译错误和测试失败反馈修改。

所以它使用 LLM 是合理的，因为这是 open-ended code generation。

### 7.2 为什么不是盲目用 LLM

ChatAssert 不是直接相信 LLM。它把 LLM 放在工具反馈循环里：

- LLM 生成候选 assertion。
- 静态检查发现编译问题。
- test execution 发现运行失败。
- 再把错误反馈给 LLM 修正。

这给你的启发是：

> LLM 可以用于生成候选 oracle 或解释，但最终判断必须被 execution evidence 和 evaluation protocol 约束。

### 7.3 和你的题目的连接

可以这样说：

> ChatAssert tries to strengthen test oracles using LLMs and execution feedback. My research direction is complementary: given a test-passing patch, I want to assess whether the patch should be trusted by combining code representation with execution-aware behavioral evidence.

---

## 8. 竞争风险与避免撞题策略

APR/APCA 竞争确实激烈，而且未来 1-2 年很可能有人做相近方向。降低撞题风险的办法不是回避 APR，而是把题目边界写得更清楚。

### 8.1 高风险表述

这些题目容易撞：

- LLM-based Patch Correctness Assessment
- An Empirical Study of Patch Correctness Assessment
- Execution-Aware Features for APCA
- Using LLMs to Detect Overfitting Patches

问题是它们太宽、太像已有工作标题，也容易被别人用不同数据集快速做掉。

### 8.2 更稳的差异化

建议坚持下面的组合：

- **framework**：CEAP，而不是单一模型。
- **execution-aware**：明确定义 coverage、trace、path、test outcome、oracle strength。
- **calibration**：输出 risk/confidence，而不是只输出 correct/incorrect。
- **ablation**：证明 execution evidence 的独立贡献。
- **correct patch preservation**：强调不能误杀正确 patch。
- **LLM as auxiliary**：LLM 用于 oracle/assertion/explanation，不作为唯一裁判。

### 8.3 防撞题核心句

> Unlike prior APCA methods that mainly rely on static patch representation or direct LLM judgment, this work proposes a calibrated execution-aware assessment framework that explicitly models behavioral evidence from test execution and studies how such evidence changes overfitting-risk estimation for plausible APR patches.

---

## 9. 可行性判断

你现在数学和 Python 工具基础不强，但这个方向不是完全不能做。关键是不要把题目设计成“发明新的深度学习架构”。

更可行的路线：

1. 先复现/阅读 ODS、CACHE、APPT、LLM4PatchCorrect、PRISM 的输入输出。
2. 先做传统 ML baseline，确认 execution-aware features 是否有信号。
3. 再用现成 encoder 做 DL fusion，不从零写复杂模型。
4. LLM 只做辅助实验或 baseline。
5. 数据集先选 Defects4J/APCA 既有 patch correctness datasets，不自己造大数据集。

建议和教授沟通时不要说：

> 我想做一个很强的 LLM 系统。

而说：

> I want to start from an interpretable baseline, then gradually introduce representation learning only where the input structure requires it. This makes the project manageable and also lets me justify every model choice.

---

## 10. 本地资料整理

### 10.1 `Paper&rResearch.zip`

已检查内容：56 个文件。

数量：

- PDF：38
- Markdown：9
- PPTX：3
- TXT：3
- JPEG：1
- WEBP：1
- LNK：1

按研究价值分组：

| 类别 | 代表文件 | 用途 |
|---|---|---|
| Core APCA | ODS, CACHE, APPT, LLM4PatchCorrect, PRISM, Patch Correctness Assessment Survey | 你的主要 related work。 |
| LLM/APR | PLM APR, RePair, RepairAgent, fine-tuning code LLMs for APR | 说明 APR 进入 LLM/agentic 阶段。 |
| Oracle/Test | ChatAssert, TOGLL, Oracle-Guided Program Selection | 支撑 oracle quality 和 execution feedback 动机。 |
| Classic APR | GenProg, TBar, Defects4J | 背景和 benchmark。 |
| Execution/Testing background | RewardRepair, DeepRL4FL, DeepXplore | 支撑 execution evidence 的价值。 |
| 本地组会材料 | Harrison slides, Q&A markdown, motivation txt | 可直接转成组会回答和 proposal。 |
| 弱相关/无关 | malware detection, sports poster, generic software reuse, shortcut | 不建议放进 proposal。 |

### 10.2 上传图片

上传的 APR mind map 可以作为 classical APR 背景图，但不适合作为 2020-2026 SOTA 时间线。

正确用法：

- 标为 “classical APR background map”。
- 只用于介绍 fault localization、patch generation、patch validation、overfitting 等基本概念。
- 后面必须单独加现代时间线：ODS -> CACHE -> APPT/LLM4PatchCorrect -> PRISM/ChatAssert/TOGLL。

### 10.3 関口智也 thesis/PPT

可用价值：

- 证明高田研已有 APR/Breaking Change 自动修复基础。
- 说明 template-based repair 在特定 Breaking Change 场景有效。
- 用他的 limitation 支撑“修复后仍需要 validation/assessment”。
- 帮你和已有研究划清边界：他做 patch generation，你做 patch correctness assessment。

不要把他的研究写成你的直接 baseline，因为任务不同。

---

## 11. 组会可用版本

### 11.1 一分钟版本

> Automated Program Repair has moved from search-based and template-based repair to learning-based and LLM-based repair. However, passing the available tests does not guarantee that a generated patch is correct, because the test suite is only a partial oracle. Existing APCA methods such as ODS, CACHE, APPT, LLM4PatchCorrect, and PRISM show that static features, code context, pretrained models, and semantic features are useful, but static or direct LLM-based assessment may still miss behavior that only appears during execution. Therefore, I want to propose a calibrated execution-aware APCA framework that combines static patch representation with execution evidence such as test outcomes, coverage, traces, and oracle strength to estimate the overfitting risk of plausible patches.

### 11.2 和関口论文衔接

> Sekiguchi's work applies APR to bugs caused by breaking changes and focuses mainly on template-based patch generation for client code. My topic is different: I focus on the validation stage after APR tools have generated plausible patches. In other words, his work asks how to repair breaking-change-induced bugs, while my work asks how to judge whether a generated APR patch should be trusted.

### 11.3 为什么不是盲目 LLM

> I do not plan to use an LLM as a black-box correctness oracle. The core hypothesis is that execution behavior provides evidence that static patch representations miss. Therefore, the main framework should explicitly extract static and execution-aware features, test them with interpretable ML baselines, and then use DL for representation fusion when the input structure becomes too complex. LLMs can be used only for auxiliary semantic tasks, such as assertion generation or explanation.

---

## 12. 下一步建议

最实际的下一步不是继续扩大文献范围，而是把 proposal 收窄成 1 页：

1. **Problem**：plausible patch may be overfitting.
2. **Gap**：static/LLM-only APCA misses execution behavior and lacks calibrated decision.
3. **Proposal**：CEAP framework.
4. **Evidence**：static + execution + calibration.
5. **Evaluation**：compare static only, execution only, static + execution, DL fusion, LLM-only, and DL + LLM auxiliary.
6. **Difference**：not Breaking Change patch generation, not blind LLM, but calibrated execution-aware APCA.

---

## 13. IEEE / TOSEM 审稿人视角下的可行性判断

作为 IEEE/TSE 或 TOSEM 审稿人的判断是：

> **方向可行，但现在的计划还不够稳；作为硕博课题可以推进，作为 TOSEM/TSE 级论文还需要进一步收窄和强化差异点。**

核心问题在于：你把方向定为 **execution-aware APCA** 是合理的，但 “execution-aware” 本身已经不是完全空白。比如 LLM4PatchCorrect 已经使用 bug descriptions、execution traces、failing tests、coverage 等信息；APPT、CACHE、ODS、PRISM 也都在不同角度推进 patch correctness assessment / overfitting detection。

所以如果你的贡献只是“把 coverage/trace 加进模型”，审稿人很可能会认为 **incremental**。

更可行的写法是把贡献压到这句话上：

> 不是简单提出 execution-aware APCA，而是提出一个 **calibrated, differential, execution-evidence-based APCA framework**，研究执行证据如何在跨项目、跨 APR 工具、跨数据集设置下提升过拟合风险估计，并降低误杀 correct patch 的风险。

### 13.1 审稿人评分

| 维度 | 判断 |
|---|---|
| 问题重要性 | 强。plausible patch 过拟合仍然是 APR 的核心问题。 |
| 可行性 | 中高。用 Defects4J/APCA 既有数据集 + coverage/trace 特征 + ML/DL baseline 可以做出来。 |
| 新颖性 | 中等偏风险。必须和 LLM4PatchCorrect、PRISM、APPT、动态 patch assessment 拉开距离。 |
| TOSEM/TSE 潜力 | 有，但前提是实验设计非常扎实，且贡献不只是模型组合。 |
| 最大风险 | 数据泄漏、标签可靠性、execution evidence 采集成本、只在一个 benchmark 上有效。 |

### 13.2 建议的核心 RQ

建议把计划改成三个核心 RQ：

1. **RQ1:** Execution evidence 相比 static/code-context representation 是否真的提供独立信号？
2. **RQ2:** 哪类 execution evidence 最有用：coverage delta、branch/path delta、trace summary、test outcome pattern、generated oracle response？
3. **RQ3:** Calibration 和 abstention 是否能减少 false acceptance，同时保留 correct patches？

### 13.3 实验必须包含

- Baselines：ODS、CACHE、APPT、LLM4PatchCorrect、PRISM，至少要覆盖代表性方法。
- Ablation：static only、execution only、static + execution、calibration/no calibration。
- Split：不能只 random split；要做 by-project、by-bug、by-APR-tool split。
- Metrics：AUROC/AUPRC、F1、false acceptance rate、correct patch preservation、ECE/Brier score、reranking top-k。
- Threats：label noise、benchmark leakage、LLM memorization、execution overhead。

### 13.4 最终判断

> **可行，但要降维。**

不要把它包装成“大而全的新框架 + ML/DL/LLM 全部都用”。更稳的是：

1. 先证明 execution evidence 的独立价值。
2. 再证明 calibration 让 APCA 决策更可信。

这样更像严肃软件工程论文，也更符合 TOSEM/IEEE 审稿人会认可的证据链。

---

## 14. Key External Sources

- CoCoNuT, ISSTA 2020: https://conf.researchr.org/details/issta-2020/issta-2020-papers/23/CoCoNuT-Combining-Context-Aware-Neural-Translation-Models-using-Ensemble-for-Program
- CURE, ICSE 2021: https://conf.researchr.org/details/icse-2021/icse-2021-papers/50/CURE-Code-Aware-Neural-Machine-Translation-for-Automatic-Program-Repair
- DeepRL4FL, ICSE 2021: https://2021.icse-conferences.org/details/icse-2021-papers/93/Fault-Localization-with-Code-Coverage-Representation-Learning
- RewardRepair, ICSE 2022: https://conf.researchr.org/details/icse-2022/icse-2022-papers/154/Neural-Program-Repair-using-Execution-based-Backpropagation
- CACHE, FSE 2022 Journal First: https://2022.esec-fse.org/details/fse-2022-journal-first/9/Context-Aware-Code-Change-Embedding-for-Better-Patch-Correctness-Assessment
- PLM APR, ICSE 2023: https://conf.researchr.org/details/icse-2023/icse-2023-technical-track/68/Automated-Program-Repair-in-the-Era-of-Large-Pre-trained-Language-Models
- CodeBERT, EMNLP Findings 2020: https://aclanthology.org/2020.findings-emnlp.139/
- APPT, TSE 2024: https://researchr.org/publication/ZhangFSLHHC24
- LLM4PatchCorrect, TSE 2024: https://colab.ws/articles/10.1109%2Ftse.2024.3452252
- Oracle-Guided Program Selection, ISSTA 2024: https://2024.issta.org/details/issta-2024-papers/51/Oracle-Guided-Program-Selection-from-Large-Language-Models
- RepairAgent, ICSE 2025: https://conf.researchr.org/details/icse-2025/icse-2025-research-track/160/RepairAgent-An-Autonomous-LLM-Based-Agent-for-Program-Repair
- TOGLL, ICSE 2025: https://colab.ws/articles/10.1109%2FICSE55347.2025.00098
- APR@ICSE 2025 proceedings: https://researchr.org/publication/icse-apr-2025
- PRISM, PACMPL/OOPSLA 2025: https://pure.korea.ac.kr/en/publications/enhancing-apr-with-prism-a-semantic-based-approach-to-overfitting/
- ICSE 2026 overfitting undecidability: https://conf.researchr.org/details/icse-2026/icse-2026-nier/4/The-Undecidability-of-Overfitting-in-Automated-Program-Repair
- ICST 2026 LLM-based APCA oracles: https://conf.researchr.org/details/icst-2026/icst-2026-research/6/Improving-Automated-Patch-Correctness-Assessment-by-Designing-LLM-Based-Oracles

