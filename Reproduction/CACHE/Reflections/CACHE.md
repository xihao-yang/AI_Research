# 📘 CACHE 复现实验报告（Day 1）

---

## 🧠 一、实验目标

本次实验旨在复现 CACHE 论文中的核心流程，重点在于打通以下 pipeline：patch → 数据预处理（AST path）→ dataset → 模型训练

具体目标包括：

- 理解 CACHE 的数据构造方式
- 构建最小可运行实验环境
- 成功生成训练数据（dataset）
- 为后续 APR 研究打基础

---

## 🖥️ 二、实验环境

| 项目 | 配置 |
|------|------|
| 操作系统 | Ubuntu 22.04 |
| Python | 3.10（venv 虚拟环境） |
| Java | OpenJDK 11 |
| 工具 | Defects4J、PyTorch、sklearn |
| 运行方式 | SSH 远程服务器 |

---

## ⚙️ 三、实验过程

### 3.1 初始尝试：手动构造数据

尝试手动创建 dataset 所需文件：
tokens.csv
paths.csv
path_contexts.csv
subtoken_dict_stem.csv
node_types.csv

并直接运行：

```bash

python main.py
```


#### 结果
corpus: 0
train dataset size: 0


### 3.2 问题分析

虽然文件格式看似正确，但所有数据被模型过滤，原因如下：

1. path_contexts 并非真实 AST 路径
2. 数据未从源码中提取，缺乏语义信息
3. 模型内部存在严格过滤机制：
   无效 context → 自动丢弃（silent drop）
   手写数据 ≠ 模型期望的数据分布

## 四、转向标准复现流程

决定采用官方 preprocessing 脚本：

python Preprocessing/genOverfittingPatches.py

### 4.1 遇到问题：权限错误
Permission denied

✔ 解决方法：

```bash
python genOverfittingPatches.py
```

### 4.2 遇到问题：路径错误
FileNotFoundError: patches/Small/correct
🧠 原因分析

当前执行路径为：

CACHE/Preprocessing/

而代码使用相对路径：

patches/Small/correct

实际访问路径变为：

CACHE/Preprocessing/patches/Small/correct ❌
✅ 正确执行方式

必须在项目根目录执行：

cd ~/workspace/paper-reproduction/CACHE
python Preprocessing/genOverfittingPatches.py
📁 五、数据状态检查

确认 patches 数据存在：

ls patches/Small

输出：

correct/
overfitting/

说明：

patch 数据已准备完成
可以进入 dataset 构建阶段
📊 六、当前进度总结
阶段	状态
环境配置	✅ 完成
Defects4J	✅ 可用
patches 数据	✅ 已存在
dataset 构建	🔄 进行中
模型训练	⏳ 未完成
🧠 七、关键收获
1. CACHE 并非端到端系统
不负责生成 patch
依赖 APR 工具（如 GenProg / TBar）
2. 数据预处理是核心难点
Java → AST → path context

这一过程对数据合法性要求极高

3. 模型具有严格过滤机制
非法输入 → 直接丢弃

这也是导致：

corpus: 0

的根本原因

4. 触及重要研究问题

本次实验暴露出一个关键问题：

Feature Representation Bias（特征表示偏差）

即：

输入表示方式直接影响模型行为与实验结果

🚀 八、后续计划
完成 dataset 自动生成
成功运行模型训练流程
导出训练结果（CSV / JSON）
分析以下异常现象：
accuracy = 1.0
precision = 0
recall = 0
