# CACHE 复现实验报告（Day6，固定随机种子与结果可信度验证阶段）

## 1. 本阶段目标

本阶段是在 Day5 已经跑通 CACHE 主流程的基础上继续推进。  
Day5 的重点是完成从 preprocessing 到 full-dataset training 的工程闭环，而 Day6 的重点不是继续扩大数据，也不是盲目重复训练，而是提高当前结果的可复查性和可信度。

本阶段主要目标包括：

1. 检查 `source/main.py` 和 `DatasetBuilder` 中的 train/test split 逻辑；
2. 确认上一轮训练结果是否受到随机划分影响；
3. 修复日志中 `TRUE/PRED` debug 输出的显示错误；
4. 固定随机种子 `seed=42` 后重新运行 40 epoch full-dataset training；
5. 将 seed=42 的训练日志、模型 checkpoint 和指标 CSV 归档；
6. 与 Day5 未固定 seed 的结果进行对比，判断当前阶段结果是否更适合作为复现记录。

---

## 2. Day5 阶段结果回顾

Day5 已经完成了 CACHE 主流程的第一次完整闭环：

```text
correct / overfitting preprocessing
→ method-level buggy/fixed extraction
→ materials_full construction
→ ASTMiner full extraction
→ subtoken vocabulary generation
→ full-dataset 40 epoch training
```

Day5 的关键训练数据规模为：

```text
corpus = 1122
train item size = 898
test item size = 224
```

未固定随机种子的 40 epoch 训练中，最佳结果为：

```text
best epoch = 21
accuracy = 0.6875
precision = 0.6818181818181818
recall = 0.7627118644067796
f1 = 0.72
```

保存的模型 checkpoint 包括：

```text
output/cache_full_cpu.model_0
output/cache_full_cpu.model_8
output/cache_full_cpu.model_21
```

其中：

```text
output/cache_full_cpu.model_21
```

对应 Day5 中的 best F1 checkpoint。

### Day5 遗留问题

Day5 虽然证明主流程已经跑通，但仍有两个需要确认的问题：

1. train/test split 是否固定；
2. debug 输出中的 `TRUE` 计数是否可信。

因此 Day6 的任务是从“能跑通”推进到“更可复查”。

---

## 3. 检查 train/test split 逻辑

本阶段首先搜索代码中的随机划分相关逻辑：

```bash
grep -R -n -E "random|shuffle|seed|train_test|split|test_size|train item|test item" source \
  --include="*.py"
```

结果显示，关键逻辑位于：

```text
source/model/dataset_builder.py
source/main.py
```

在 `source/main.py` 中，训练入口使用：

```python
builder = DatasetBuilder(reader, option, split_ratio=0.2)
```

而 `DatasetBuilder` 的构造函数为：

```python
def __init__(self, reader, option, split_ratio=0.1, shuffle=True, seed=None, fragment=0):
```

其中核心逻辑为：

```python
if seed is not None:
    random.seed(seed)
if shuffle:
    random.shuffle(reader.items)
```

这说明 Day5 的训练实际上是：

```text
split_ratio = 0.2
shuffle = True
seed = None
```

也就是说：

> Day5 的 train/test split 是随机的，但没有固定随机种子。

因此，Day5 的结果虽然可以证明训练流程跑通，但不够稳定，也不够适合作为最终阶段性结果。每次重新运行时，train/test split 可能不同，指标也可能变化。

---

## 4. 修复 TRUE/PRED debug 输出问题

本阶段继续检查 `source/main.py` 中的 debug 输出：

```bash
grep -R -n -E "PRED|TRUE|pred_counts|true_counts|unique" source --include="*.py"
```

定位到如下代码：

```python
pred_vals, pred_counts = np.unique(actual_labels, return_counts=True)
true_vals, true_counts = np.unique(expected_labels, return_counts=True)

print("PRED:", dict(zip(pred_vals, pred_counts)))
print("TRUE:", dict(zip(true_vals, pred_counts)))
```

这里的问题是：

```python
print("TRUE:", dict(zip(true_vals, pred_counts)))
```

错误地使用了 `pred_counts`，而不是 `true_counts`。

因此日志中的：

```text
TRUE: {...}
```

在 Day5 中是不可信的。  
不过正式指标 `accuracy / precision / recall / f1` 是通过：

```python
exact_match(expected_labels, actual_labels)
```

计算的，因此这些 metric 本身仍然可以参考。

本阶段将该行修改为：

```python
print("TRUE:", dict(zip(true_vals, true_counts)))
```

修改后，seed=42 训练日志中可以看到测试集真实标签分布稳定显示为：

```text
TRUE: {np.int64(0): np.int64(97), np.int64(1): np.int64(127)}
```

这说明 debug 输出已修复。

---

## 5. 固定随机种子

为了让 train/test split 可复查，本阶段将 `source/main.py` 中的 DatasetBuilder 调用从：

```python
builder = DatasetBuilder(reader, option, split_ratio=0.2)
```

修改为：

```python
builder = DatasetBuilder(reader, option, split_ratio=0.2, seed=42)
```

修改后的含义是：

```text
split_ratio = 0.2
shuffle = True
seed = 42
```

也就是说，仍然保持 80% train / 20% test 的随机划分方式，但划分过程固定，使得后续重复运行更容易复查。

确认修改后，`grep` 输出中可以看到：

```text
builder = DatasetBuilder(reader, option, split_ratio=0.2, seed=42)  # fixed split seed for reproducibility
```

同时确认 debug 输出已经改为：

```text
print("TRUE:", dict(zip(true_vals, true_counts)))
```

---

## 6. seed=42 后台训练

本阶段使用 `nohup` 在后台运行 seed=42 的 40 epoch training：

```bash
cd ~/workspace/paper-reproduction/CACHE && nohup .venv/bin/python source/main.py \
  --dataset_path ./dataset_ast_full/java/ \
  --batch_size 16 \
  --max_epoch 40 \
  --no_cuda \
  --model_path ./output/cache_full_cpu_seed42.model \
  > run_train_full_40epoch_cpu_seed42.log 2>&1 &
```

这样即使 SSH 断开，只要服务器本身没有关机，训练仍然可以继续运行。

启动后确认进程存在：

```text
.venv/bin/python source/main.py --dataset_path ./dataset_ast_full/java/ ...
```

日志开始正常写入：

```text
device: cpu
path vocab size: 9722
corpus: 1122
train item size: 898
test item size: 224
```

训练完成后再次检查：

```bash
ps -ef | grep "source/main.py" | grep -v grep
```

没有输出，说明训练进程已经结束。

日志中已经跑到：

```text
epoch 39
```

因此，seed=42 的 40 epoch training 已经完整完成。

---

## 7. seed=42 训练结果

本阶段训练数据规模仍然为：

```text
corpus = 1122
train item size = 898
test item size = 224
```

固定 seed 后，测试集真实标签分布为：

```text
TRUE: {np.int64(0): np.int64(97), np.int64(1): np.int64(127)}
```

这说明测试集中包含：

```text
correct label 0 = 97
overfitting label 1 = 127
```

### 7.1 最后一轮 epoch 39

epoch 39 的结果为：

```text
epoch = 39
train_loss = 14.451321542263031
test_loss = 15.316052824258804
accuracy = 0.7098214285714286
precision = 0.7719298245614035
recall = 0.6929133858267716
f1 = 0.7302904564315352
```

这个结果已经高于 Day5 未固定 seed 的最后一轮结果。

---

## 8. 生成 seed42 CSV 指标文件

训练完成后，本阶段将完整日志解析为 CSV：

```text
reproduction_records/train_40epoch_metrics_seed42.csv
```

CSV 文件包含 40 个 epoch 的以下字段：

```text
epoch
train_loss
test_loss
accuracy
precision
recall
f1
```

解析结果显示：

```text
num_epochs = 40
```

说明完整 40 epoch 指标都已被正确提取。

---

## 9. seed=42 最佳结果

根据 `train_40epoch_metrics_seed42.csv`，按 F1 选择的最佳结果出现在 epoch 8：

```text
epoch = 8
train_loss = 21.990964874625206
test_loss = 10.1554264575243
accuracy = 0.71875
precision = 0.75
recall = 0.7559055118110236
f1 = 0.7529411764705882
```

保存的 seed42 模型 checkpoint 为：

```text
output/cache_full_cpu_seed42.model_0
output/cache_full_cpu_seed42.model_6
output/cache_full_cpu_seed42.model_8
```

因为模型只在 F1 刷新 best 时保存，所以：

```text
output/cache_full_cpu_seed42.model_8
```

对应当前 seed=42 实验的 best checkpoint。

---

## 10. 与 Day5 未固定 seed 结果对比

Day5 未固定 seed 的最佳结果为：

```text
best epoch = 21
accuracy = 0.6875
precision = 0.6818181818181818
recall = 0.7627118644067796
f1 = 0.72
```

Day6 固定 seed=42 后的最佳结果为：

```text
best epoch = 8
accuracy = 0.71875
precision = 0.75
recall = 0.7559055118110236
f1 = 0.7529411764705882
```

对比来看：

| 设置 | best epoch | accuracy | precision | recall | f1 |
|---|---:|---:|---:|---:|---:|
| 未固定 seed | 21 | 0.6875 | 0.6818 | 0.7627 | 0.7200 |
| seed=42 | 8 | 0.71875 | 0.7500 | 0.7559 | 0.7529 |

### 结果解释

1. 固定 seed 后，best F1 从 `0.72` 提升到 `0.7529`；
2. accuracy 和 precision 均有所提高；
3. recall 基本保持在相近水平；
4. seed=42 结果更容易复查，因此更适合作为当前阶段的主要记录结果。

---

## 11. 当前阶段结论

本阶段可以明确判断：

> CACHE 主流程不仅已经跑通，而且已经完成了一次固定随机种子的 full-dataset 训练验证。

与 Day5 相比，Day6 的进展主要体现在：

1. 确认原始训练代码的 split 是随机划分；
2. 修复 `TRUE` debug 输出错误；
3. 固定 `seed=42` 重新训练；
4. 生成并归档 seed42 的完整 40 epoch CSV；
5. 找到更可复查的 best checkpoint；
6. 将阶段性 best F1 提升到 `0.7529`。

因此，当前更适合作为阶段性结果的是：

```text
seed = 42
best checkpoint = output/cache_full_cpu_seed42.model_8
best epoch = 8
accuracy = 0.71875
precision = 0.75
recall = 0.7559055118110236
f1 = 0.7529411764705882
```

---

## 12. 已归档文件

当前与 seed42 相关的归档文件包括：

```text
reproduction_records/run_train_full_40epoch_cpu_seed42.log
reproduction_records/train_40epoch_metrics_seed42.csv
reproduction_records/saved_models_seed42.txt
```

其中：

```text
reproduction_records/train_40epoch_metrics_seed42.csv
```

记录了 40 个 epoch 的完整指标。

```text
reproduction_records/saved_models_seed42.txt
```

记录了 seed42 训练过程中保存的 checkpoint：

```text
cache_full_cpu_seed42.model_0
cache_full_cpu_seed42.model_6
cache_full_cpu_seed42.model_8
```

---

## 13. 当前仍然不能直接声称完全复现论文结果的原因

虽然 Day6 已经完成固定 seed 的 full-dataset 训练，但目前仍然不能直接声称完全复现论文结果，原因包括：

1. 当前只确认了 Small dataset 下 Defects4J 部分数据；
2. Bugs.jar、Bears、QuixBugs 等其他数据源尚未完整纳入训练；
3. 当前 split 是代码中的随机 80/20 split，尚未确认是否与论文实验设置完全一致；
4. 当前只固定了 Python `random.seed(42)`，没有完全固定 NumPy 和 PyTorch 的所有随机源；
5. 当前没有做多次重复实验或交叉验证；
6. 当前参数虽然来自仓库默认值，但仍需进一步对照论文和 README。

因此，当前最严谨的表述是：

> CACHE 的工程主流程已经跑通，并完成了固定 seed=42 的 full-dataset 训练验证；当前结果可作为阶段性复现结果，但还不是严格意义上的论文结果完全复现。

---

## 14. 下一步计划

下一步建议不要继续只跑同一个训练命令，而是转向论文设置对齐和稳定性验证。

### 14.1 检查论文和 README 的实验设置

需要确认：

1. 论文使用的是 Small dataset 还是 full dataset；
2. 是否只使用 Defects4J，还是混合 Bugs.jar、Bears、QuixBugs；
3. 是否采用 80/20 random split；
4. 是否使用 cross-validation；
5. 是否报告单次结果，还是多次运行平均值；
6. 是否有固定 random seed。

### 14.2 进一步固定随机源

当前只固定了 `DatasetBuilder` 的 split seed。  
更严格的 reproducibility 还应考虑：

```python
random.seed(42)
np.random.seed(42)
torch.manual_seed(42)
```

如果使用 GPU，还需要考虑：

```python
torch.cuda.manual_seed_all(42)
torch.backends.cudnn.deterministic = True
torch.backends.cudnn.benchmark = False
```

### 14.3 做重复实验

可以再设置不同 seed，例如：

```text
seed = 1
seed = 7
seed = 42
seed = 2026
```

然后比较 best F1 的均值和方差。

### 14.4 对比论文指标

最终需要将当前结果和论文 reported results 对照，判断差距来自：

1. 数据范围不同；
2. split 方式不同；
3. 参数不同；
4. 预处理策略不同；
5. 当前只复现了部分 benchmark。

---

## 15. 阶段总结

Day6 的核心进展不是继续修 preprocessing，而是提升实验结果的可信度。

目前已经完成：

- train/test split 逻辑检查；
- debug 输出修复；
- 固定 seed=42；
- 40 epoch full-dataset training；
- seed42 指标 CSV 生成；
- seed42 模型 checkpoint 归档；
- best F1 从 Day5 的 `0.72` 提升到 `0.7529`。

因此，当前阶段可以总结为：

> CACHE 复现已经从“工程主流程跑通”推进到“固定随机种子后的可复查训练结果阶段”。下一步应重点对齐论文实验设置，并考虑多 seed 稳定性验证。
