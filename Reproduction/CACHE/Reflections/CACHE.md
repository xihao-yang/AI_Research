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
