# CACHE 复现实验报告（阶段性）

## 1. 实验目标

本阶段实验的目标是：

1. 在服务器环境中复现 CACHE 项目的 preprocessing 主流程。
2. 修复原始脚本在 patch 元信息解析、源码路径定位、Defects4J checkout、异常跳过等环节中的关键问题。
3. 验证脚本是否能够稳定批量处理 patch，并输出可用于后续方法级分析的数据结果。

---

## 2. 实验环境

### 2.1 系统环境

- 操作系统：Ubuntu
- Python 环境：Python 3.10.12（`.venv`）
- Java 环境：OpenJDK 11
- 依赖基准：Defects4J
- 额外修复的系统依赖：`svn`

### 2.2 相关目录

本次实验主要在以下目录下进行：

- 项目目录：`~/workspace/paper-reproduction/CACHE`
- 预处理脚本：`Preprocessing/genOverfittingPatches.py`
- 临时 checkout 目录：`temp_projects/`
- 输出目录：`correct_D4j/`

---

## 3. 实验背景与初始问题

在最初运行 `genOverfittingPatches.py` 时，脚本虽然能够启动，但在实际处理 patch 时存在多个严重问题，主要包括：

1. **patch 文件名解析错误**
   - `parse_info()` 对 patch 名称的解析不稳健；
   - 某些 patch 的 APR 名称、项目名称、bug id 会被错误拆分。

2. **源码路径定位失败**
   - `get_original_file()` 无法正确把 patch 中的文件路径映射到 checkout 后的真实源码路径；
   - 对 `source/...`、`src/main/java/...`、`a/source/...`、`a/src/main/java/...` 等不同形式处理不统一。

3. **Chart 项目大面积失败**
   - 初始阶段大量 `Chart-*` patch 无法定位源码文件；
   - 后续排查确认并非纯路径问题，而是 checkout 目录本身为空。

4. **Defects4J checkout 隐性失败**
   - 脚本原始实现只检查目标目录是否存在，不检查是否为空；
   - 即使 checkout 没有成功，也会误判为成功并继续执行。

5. **系统依赖缺失**
   - 进一步排查发现服务器缺少 `svn`，导致部分 Chart bug 无法正确 checkout。

6. **deprecated bug 引发的异常**
   - Defects4J 中部分 bug（如 `Time-21`、`Lang-18`、`Lang-48`、`Lang-2`）已经被标记为 deprecated；
   - 原始流程没有很好地区分这类“应跳过”样本和普通失败样本。 :contentReference[oaicite:0]{index=0}

---

## 4. 修改与修复过程

### 4.1 修复 `parse_info()`

对 patch 文件名解析逻辑进行了增强，主要改进如下：

- 先取 patch 文件名主体（去掉扩展名）；
- 按 `-` 进行拆分；
- 显式检查字段数量是否合法；
- 显式提取并校验：
  - `patch_id`
  - `project`
  - `bug_id`
- 对异常格式抛出明确错误信息，而不是默默解析错误。

该修改解决了 APR 名称与项目名混淆的问题，为后续 checkout 和路径定位打下了基础。

---

### 4.2 修复 `get_original_file()` 的路径标准化

为了适应不同 patch 中记录的源码路径形式，对 `patched_file` 增加了标准化逻辑，主要包括：

- 统一路径分隔符为 `/`
- 去掉前导 `/`
- 自动修正缺失点号的 `java` 后缀
- 优先识别并保留以下关键路径前缀：
  - `src/main/java/`
  - `src/java/`
  - `source/`

例如，原始 patch 中可能出现：

- `/source/org/...`
- `a/source/org/...`
- `a/src/main/java/...`

标准化后统一映射为：

- `source/org/...`
- `src/main/java/...`

这一步确认在实际日志中已经生效。 

---

### 4.3 强化源码文件匹配逻辑

在 `get_original_file()` 中，不再单纯依赖：

- `basename`
- 父目录名
- `path_mapping`

而是增加了基于标准化相对路径的更稳健匹配策略，使脚本能够更准确地定位 checkout 后源码树中的真实文件。

这一步对 Chart、Lang、Math、Time 等 Defects4J 项目的 patch 应用都有帮助。

---

### 4.4 修复 `checkout()` 逻辑

原始 `checkout()` 存在两个核心问题：

1. 只要目录存在就直接返回成功；
2. 不检查 `defects4j checkout` 的返回状态。

对此进行了以下修复：

- 增加 `target_path` 判断；
- 当目录存在但为空时，打印：
  - `[CHECKOUT-EMPTY-EXISTING]`
- 对 Defects4J checkout 使用更明确的失败输出；
- 增加：
  - `[CHECKOUT-FAILED]`
  - `[CHECKOUT-MISSING]`
  - `[CHECKOUT-EMPTY]`

这样脚本就能够把“checkout 真失败”和“后续路径定位失败”区分开来。

---

### 4.5 安装 `svn`，解决 Chart checkout 失败

进一步查看日志后确认，Chart 项目失败的根因之一并不是路径本身，而是服务器上缺少 `svn`。  
安装 `svn` 之后，Chart 相关 Defects4J checkout 才能够正常工作。此前日志中明确出现过：

- `Can't exec "svn": No such file or directory`

安装后，该类错误消失，Chart patch 开始正常流过主流程。 

---

### 4.6 对 deprecated bug 进行显式跳过

针对 Defects4J 中已经废弃的 bug，当前脚本在 checkout 失败时会显式输出：

- `[CHECKOUT-FAILED] ...`
- `[SKIPPED-CHECKOUT] ...`

目前确认会被跳过的典型样本包括：

- `Time-21`
- `Lang-18`
- `Lang-48`
- `Lang-2`

这类样本不再污染正常的 patch 应用流程，而是作为“已知不可用样本”单独处理。 

---

## 5. 运行结果

### 5.1 主流程运行情况

在完成上述修复后，`genOverfittingPatches.py` 已能够稳定运行，并顺利穿过多个项目：

- Chart
- Lang
- Math
- Closure
- Time 等

从日志来看，脚本不再出现大面积未捕获异常，主循环可以持续推进，说明 preprocessing 主流程已经基本跑通。 

### 5.2 最终统计

脚本末尾打印出的两个核心数字为：

- `535`
- `181`

根据代码尾部逻辑可知，这两个数分别对应：

- `cnt = 535`：总共处理到的 patch 数
- `successful = 181`：最终成功完成到“提取并保存 methods”这一步的 patch 数

也就是说，本轮运行中：

- **共处理 patch：535**
- **最终成功生成 method-level 输出：181** :contentReference[oaicite:5]{index=5}

### 5.3 错误统计

进一步统计日志可得：

- `[SKIPPED-CHECKOUT]` 数量：**4**
- `[CHECKOUT-FAILED]` 数量：**4**
- `Failed at` 数量：**0**

这说明：

1. 没有未捕获 Python 异常破坏主流程；
2. checkout 层面的失败只剩少量 deprecated bug；
3. 当前主流程已经具备稳定性。 :contentReference[oaicite:6]{index=6}

---

## 6. 当前结论

### 6.1 已完成的工作

本阶段已完成：

- preprocessing 主流程打通
- patch 文件名解析修复
- 路径标准化与源码定位增强
- Chart checkout 问题定位并解决
- `svn` 依赖问题修复
- deprecated bug 的显式跳过
- 日志与阶段脚本快照归档到 GitHub

### 6.2 当前可以得出的判断

根据目前结果，可以认为：

**CACHE 的 preprocessing 主流程已经基本复现成功。**

更准确地说：

> 脚本已经可以完成 535 个 patch 的批处理，并成功为其中 181 个 patch 生成最终 method-level 输出；另有 4 个样本因 Defects4J deprecated bug 被显式跳过。 :contentReference[oaicite:7]{index=7}

---

## 7. 仍然存在的问题

虽然主流程已经跑通，但仍有一些后续工作需要继续：

1. **分析为何仅有 181 个 patch 进入最终 successful**
   - 需要进一步拆分失败类别，例如：
     - checkout skipped
     - apply_patch 失败
     - method 提取失败
     - 输出目录清理导致的结果缺失

2. **对剩余未成功样本做更细粒度统计**
   - 当前只知道总处理数和最终成功数；
   - 下一阶段应增加阶段性计数器。

3. **整理正式实验日志与复现说明**
   - 包括环境、修改点、日志归档位置、成功/失败分布等。

---

## 8. 下一阶段计划

下一阶段准备开展如下工作：

1. 在脚本中增加更细粒度的统计信息：
   - `skipped_checkout`
   - `apply_patch_failed`
   - `method_extract_failed`
   - `save_methods_failed`

2. 对 181 个成功样本的输出目录进行抽样验证，确认生成结果质量；

3. 基于本次运行结果撰写正式复现实验日志与阶段总结；

4. 若后续继续推进 CACHE 全流程，可进入：
   - AST mining
   - vocab 构建
   - dataset 构建
   - 训练与验证阶段

---

## 9. 阶段性总结

本次实验从“脚本能启动但主流程不稳定”的状态，逐步推进到了“preprocessing 主流程基本跑通”的状态。  
其中最关键的突破包括：

- 修复 `parse_info()` 解析逻辑
- 修复 `get_original_file()` 的路径标准化和文件匹配
- 识别并修复 Defects4J checkout 的隐性失败
- 安装 `svn`，彻底解决 Chart 大面积 checkout 失败问题
- 将 deprecated bug 明确划入 skip 逻辑

总体来看，这一阶段已经完成了 CACHE 复现中的最核心、最容易卡住的一段基础工作，为后续数据处理与训练实验奠定了基础。
