# CACHE 复现实验报告（Day5，主流程闭环与全量训练阶段）

## 1. 本阶段目标

本阶段是在 Day4 correct patch preprocessing 基本闭环的基础上继续推进。Day4 已经解决了 `apply_patch_failed = 350` 的核心阻塞，并将 correct patch 的成功处理数量提升到 531 / 535。

Day5 的目标不再是继续修复 correct patch application，而是把 CACHE 的后续主流程跑通，重点包括：

1. 确认 correct preprocessing 的输出是否真正落盘；
2. 生成 overfitting patch 的 method-level 数据；
3. 构造 full labeled materials，即同时包含 correct 和 overfitting 的训练输入；
4. 使用 `astminer_revised.jar` 对全量 Java 文件抽取 AST path contexts；
5. 生成 subtoken vocabulary；
6. 启动 `source/main.py`，完成 binary classifier 的全量训练；
7. 固化训练日志、模型 checkpoint 和每个 epoch 的指标。

---

## 2. Day4 结果回顾

Day4 修复 `patch_file()` 后，correct patch preprocessing 的最终结果为：

```text
total_patches = 535
successful = 531
skipped_checkout = 4
apply_patch_failed = 0
method_extract_failed = 0
```

Day4 的核心修复是：

```text
normal patch
→ 如果失败且检测到 Reversed patch
→ 自动使用 patch -R 重试
```

因此，Day4 之后的状态可以概括为：

> correct patch 的 preprocessing 阶段已经基本完成，下一步应从 patch application 修复转入主流程实验复现。

---

## 3. Correct method-level 输出确认

首先检查 Day4 生成的 correct 输出目录。脚本末尾显示：

```python
patches_path = 'patches/Small/correct'
output_path = 'correct_D4j'
```

实际输出目录存在：

```text
./correct_D4j/methods
./correct_D4j/files
```

统计 method-level 输出：

```text
correct_D4j/methods/buggy.java = 529
correct_D4j/methods/fixed.java = 529
```

统计 file-level 输出：

```text
correct_D4j/files/buggy-*.java = 565
correct_D4j/files/tool-patch-*.java = 565
```

### 关于 531 successful 与 529 method outputs 的差异

脚本统计为 `successful = 531`，但最终磁盘上只有 529 个 method-level 输出目录。进一步检查发现，这是因为有两个输出路径发生了覆盖。

重复输出路径 1：

```text
correct_D4j/methods/defects4j/defects4j_Math_5/ICSE18_1
```

对应两个原始 patch：

```text
patches/Small/correct/ICSE18/patch1-Math-5-ACS.patch
patches/Small/correct/ICSE18/patch1-Math-5-jGenProg.patch
```

重复输出路径 2：

```text
correct_D4j/methods/defects4j/defects4j_Math_50/ICSE18_1
```

对应两个原始 patch：

```text
patches/Small/correct/ICSE18/patch1-Math-50-Nopol2015.patch
patches/Small/correct/ICSE18/patch1-Math-50-jGenProg.patch
```

### 结论

correct 阶段的输出本身是成对完整的，不存在只生成 `buggy.java` 或只生成 `fixed.java` 的问题。`531` 与 `529` 的差异来自路径命名覆盖，不是 method extraction 缺失。

---

## 4. Overfitting preprocessing 结果

为了继续主流程，需要生成另一类样本：overfitting / plausible patch。

输入目录：

```text
patches/Small/overfitting
```

输出目录：

```text
overfitting_D4j
```

使用 wrapper 调用当前已经修复过的 `genOverfittingPatches.py`，最终运行结果为：

```text
total_patches = 648
successful = 636
skipped_checkout = 11
apply_patch_failed = 0
method_extract_failed = 1
```

method-level 输出：

```text
overfitting_D4j/methods/buggy.java = 593
overfitting_D4j/methods/fixed.java = 593
```

file-level 输出：

```text
overfitting_D4j/files/buggy-*.java = 624
overfitting_D4j/files/tool-patch-*.java = 624
```

### 结果解释

1. `apply_patch_failed = 0`，说明 Day4 的 patch application 修复对 overfitting 阶段同样有效。
2. `skipped_checkout = 11`，主要来自 Defects4J deprecated bugs，例如 `Lang-18`、`Closure-93`、`Closure-63`。
3. `method_extract_failed = 1`，说明 overfitting 阶段只有一个样本在 method extraction 或文件定位阶段失败。
4. `successful = 636` 与 method-level 输出 `593` 之间存在差异，主要原因推测仍然是输出路径覆盖。

---

## 5. 小样本 pipeline 验证

在跑全量数据之前，先进行了两次小样本验证。

### 5.1 Correct-only 小样本

先从 `correct_D4j/methods` 抽取 20 对真实样本，构造：

```text
materials_small_try/correct/0/buggy.java
materials_small_try/correct/0/fixed.java
...
```

随后使用 `astminer_revised.jar` 生成：

```text
path_contexts.csv
paths.csv
tokens.csv
node_types.csv
```

并使用 `genSubtokenVocab.py` 成功生成：

```text
subtoken_dict_stem.csv
```

此后运行 `source/main.py` 可以完成 1 epoch。该步骤说明：

> ASTMiner → subtoken vocab → training entry 的链路可以跑通。

但 correct-only 数据只有一个 label，因此不能作为正式实验结果。

### 5.2 Correct + Overfitting mixed 小样本

随后构造 20 个 correct + 20 个 overfitting 的 mixed 小样本：

```text
materials_mixed_try/correct/...
materials_mixed_try/overfitting/...
```

统计结果：

```text
Java files = 80
corpus = 40
train item size = 32
test item size = 8
```

`source/main.py` 成功完成 1 epoch，并输出 accuracy / precision / recall / f1。

### 小样本验证结论

mixed 小样本验证说明：

> 训练代码可以识别 correct / overfitting 两类 label，并且 binary classifier 的主入口可以完整运行。

但由于样本规模很小，指标不具有正式实验意义。

---

## 6. Full materials 构建

在小样本验证通过后，构造全量训练输入目录：

```text
materials_full/correct
materials_full/overfitting
```

从 method-level 输出复制所有成对的 `buggy.java` / `fixed.java`。

最终统计：

```text
correct buggy.java = 529
correct fixed.java = 529
overfitting buggy.java = 593
overfitting fixed.java = 593
```

因此 patch-level 样本总数为：

```text
529 + 593 = 1122
```

Java 文件总数为：

```text
1122 × 2 = 2244
```

### 结论

`materials_full` 已经包含完整的 correct / overfitting 两类训练样本，可以作为 ASTMiner 的全量输入。

---

## 7. 全量 ASTMiner 抽取

检查服务器内存：

```text
total = 11Gi
available = 11Gi
swap = 0B
```

由于服务器内存不足 32GB，因此没有使用 README 中较大的 `-Xmx32g` 或 `-Xmx128g` 设置，而是采用保守配置：

```bash
java -Xms1g -Xmx6g -jar ./lib/astminer_revised.jar pathContexts \
  --lang java \
  --project ./materials_full \
  --output ./dataset_ast_full \
  --maxH 9 \
  --maxW 2 \
  --maxContexts 200 \
  --maxTokens 500 \
  --maxPaths 500
```

ASTMiner 成功退出：

```text
Exit status: 0
Maximum resident set size: 1530772 KB
```

即实际最大内存占用约 1.5GB。

输出目录：

```text
dataset_ast_full/java
```

输出文件统计：

```text
path_contexts.csv = 2244 lines
tokens.csv = 5053 lines
paths.csv = 9722 lines
node_types.csv = 135 lines
```

### 结果解释

`path_contexts.csv = 2244 lines` 正好对应 `materials_full` 中的 2244 个 Java 文件。因此可以认为：

> 全量 AST path context extraction 已经完成。

---

## 8. Subtoken vocabulary 生成

使用脚本：

```text
Preprocessing/genSubtokenVocab.py
```

输入文件：

```text
dataset_ast_full/java/tokens.csv
```

输出文件：

```text
dataset_ast_full/java/subtoken_dict_stem.csv
```

统计结果：

```text
subtoken_dict_stem.csv = 1629 lines
```

说明 subtoken vocabulary 已成功生成，可以进入训练阶段。

---

## 9. Full dataset 1 epoch 验证训练

在正式训练 40 epoch 前，先进行 1 epoch 验证：

```bash
.venv/bin/python source/main.py \
  --dataset_path ./dataset_ast_full/java/ \
  --batch_size 16 \
  --max_epoch 1 \
  --no_cuda \
  --model_path ./output/cache_full_epoch1.model
```

数据读取结果：

```text
path vocab size = 9722
corpus = 1122
train item size = 898
test item size = 224
train dataset size = 898
test dataset size = 224
```

epoch 0 结果：

```text
accuracy = 0.5758928571428571
precision = 0.5663265306122449
recall = 0.9173553719008265
f1 = 0.7003154574132492
```

### 结论

1 epoch 验证训练说明：

> 全量数据可以被 `source/main.py` 正常读取并训练，主流程已经从 preprocessing 推进到 full dataset training。

---

## 10. Full dataset 40 epoch 训练

随后进行 40 epoch CPU 训练：

```bash
.venv/bin/python source/main.py \
  --dataset_path ./dataset_ast_full/java/ \
  --batch_size 16 \
  --max_epoch 40 \
  --no_cuda \
  --model_path ./output/cache_full_cpu.model
```

训练设备：

```text
CPU
```

训练数据规模：

```text
corpus = 1122
train item size = 898
test item size = 224
```

训练完整跑到：

```text
epoch 39
```

说明 40 epoch 训练完整完成，没有中途崩溃。

---

## 11. 40 epoch 训练结果

### 11.1 最后一轮结果

epoch 39 的结果为：

```text
accuracy = 0.6651785714285714
precision = 0.6747967479674797
recall = 0.7033898305084746
f1 = 0.6887966804979253
```

### 11.2 Best checkpoint

从 40 epoch 的 metrics CSV 中统计，按 F1 选择的最佳 epoch 为：

```text
epoch = 21
train_loss = 16.188313826918602
test_loss = 12.911522597074509
accuracy = 0.6875
precision = 0.6818181818181818
recall = 0.7627118644067796
f1 = 0.72
```

保存的模型 checkpoint 为：

```text
output/cache_full_cpu.model_0
output/cache_full_cpu.model_8
output/cache_full_cpu.model_21
```

其中：

```text
output/cache_full_cpu.model_21
```

对应当前 best F1 checkpoint。

### 11.3 结果解释

从当前日志看，模型在 40 epoch 内的 F1 大致稳定在 0.65 到 0.72 之间。最佳 F1 为 0.72，出现在 epoch 21。

这说明当前实现已经可以完成完整的 binary classification training，但还需要进一步确认与论文实验设置是否一致。

---

## 12. 结果归档

本阶段将关键日志和指标整理到：

```text
reproduction_records
```

该目录包含：

```text
run_astminer_full_xmx6g.log
run_overfitting_D4j_full.log
run_reverse_patch_fallback_full.log
run_train_full_40epoch_cpu.log
run_train_full_epoch1_cpu.log
saved_models.txt
train_40epoch_metrics.csv
train_40epoch_metrics.txt
```

其中：

```text
train_40epoch_metrics.csv
```

包含 40 个 epoch 的：

```text
train_loss
test_loss
accuracy
precision
recall
f1
```

CSV 解析结果显示：

```text
num_epochs = 40
best_by_f1 = epoch 21, f1 = 0.72
last_epoch = epoch 39, f1 = 0.6887966804979253
```

---

## 13. 当前阶段定位

如果从 CACHE 复现流程来看，本阶段已经从 Day4 的：

```text
preprocessing mostly completed
```

推进到了：

```text
main pipeline completed once
```

当前已经跑通的链路包括：

```text
correct / overfitting patch preprocessing
→ method-level buggy/fixed extraction
→ full labeled materials construction
→ AST path extraction
→ subtoken vocabulary generation
→ binary classifier training
→ checkpoint saving
→ metrics extraction
```

这说明 CACHE 的工程主流程已经可以完整执行。

---

## 14. 与论文复现的关系

当前结果可以说明：

> CACHE 代码仓库的主实验流程已经被工程性跑通，并得到了一组完整的 full dataset training metrics。

但还不能直接声称：

> 已经完全复现论文 reported results。

原因是仍然有若干实验设置尚未严格核对：

1. 当前数据集是否与论文使用的数据完全一致；
2. 当前 train/test split 是否与论文一致；
3. 当前随机种子是否固定；
4. 当前模型参数是否与论文完全一致；
5. 当前 `maxH / maxW / maxContexts / maxTokens / maxPaths` 是否与论文或作者脚本一致；
6. 是否需要多次重复实验并取平均值；
7. `source/main.py` 中 debug 输出的 TRUE/PRED 计数存在显示问题，不能直接用于混淆矩阵解释。

因此更准确的表述是：

> 当前已经完成 CACHE 的一次端到端工程复现，并获得可记录的训练结果；下一步应进行实验设置对齐和结果可靠性验证。

---

## 15. 当前关键结论

本阶段最大的进展是：

**CACHE 主流程第一次完整闭环。**

具体体现在：

1. correct preprocessing 输出确认完成；
2. overfitting preprocessing 成功生成；
3. full materials 共 1122 个 patch-level samples；
4. ASTMiner 成功处理 2244 个 Java 文件；
5. subtoken vocabulary 成功生成；
6. `source/main.py` 成功读取 full dataset；
7. 40 epoch CPU training 完整跑完；
8. 最佳 F1 为 0.72；
9. best checkpoint 已保存为 `cache_full_cpu.model_21`；
10. 训练日志和指标已经归档。

因此，当前可以认为：

> CACHE 复现已经从 preprocessing debugging 阶段进入了 full pipeline training and result validation 阶段。

---

## 16. 下一步计划

下一阶段不建议继续盲目重复训练，而应优先做实验设置确认和结果可靠性验证。

### 16.1 检查 train/test split 逻辑

需要查看：

```text
source/model/dataset_builder.py
source/model/dataset_reader_binary.py
source/main.py
```

重点确认：

- train/test 是否随机划分；
- 划分比例是否固定；
- 是否有随机种子；
- 是否存在数据泄漏风险；
- correct / overfitting 的 label 映射是否符合预期。

### 16.2 固定随机种子后复跑

当前 40 epoch 结果可能受随机初始化和随机 split 影响。下一步应固定 seed 后复跑一次，比较：

```text
best f1
best epoch
last epoch f1
accuracy / precision / recall
```

### 16.3 对照论文指标

需要回到 CACHE 论文和 README，确认论文报告的具体指标、数据设置和训练参数。然后判断当前 `F1 = 0.72` 与论文结果的差距来自：

- 数据规模差异；
- split 差异；
- 参数差异；
- 预处理路径覆盖；
- checkout skipped / deprecated bugs；
- 或代码实现本身的问题。

### 16.4 整理可汇报版本

可以将当前结果整理为组会汇报中的一页：

```text
What has been reproduced?
What has been fixed?
What is the current metric?
What remains uncertain?
```

重点突出：

- patch application 问题已解决；
- full pipeline 已跑通；
- 当前指标是 first runnable result；
- 下一步是对齐论文实验设置，而不是继续修环境。

---

## 17. 阶段总结

Day5 的核心进展是将 CACHE 从 preprocessing 阶段推进到 full training 阶段。

相比 Day4，本阶段新增完成了：

- overfitting patch preprocessing；
- full labeled materials 构建；
- full ASTMiner extraction；
- full subtoken vocabulary generation；
- full dataset 1 epoch sanity check；
- full dataset 40 epoch CPU training；
- best checkpoint 和 metrics CSV 归档。

当前最重要的结果是：

```text
best epoch = 21
accuracy = 0.6875
precision = 0.6818
recall = 0.7627
f1 = 0.72
```

因此，目前可以将复现状态更新为：

> CACHE 工程主流程已经成功跑通一次，并获得完整训练指标。下一步需要对齐论文实验设置、固定随机性并验证结果稳定性。
