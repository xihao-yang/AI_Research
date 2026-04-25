# CACHE 复现实验报告（Day 2）

---

## 与 Day 1 的衔接

在 Day 1 的实验中，我主要完成了 CACHE 复现环境的初步搭建，并确认了 ```patches/Small/correct``` 与 ```patches/Small/overfitting``` 等 patch 数据已经存在。同时，我也尝试过手动构造 CACHE 模型所需的输入文件，例如：
```
- `tokens.csv`
- `paths.csv`
- `path_contexts.csv`
- `subtoken_dict_stem.csv`
- `node_types.csv`
```
但是实际运行后发现，虽然这些文件在格式上看似完整，但并不能被模型有效使用。模型在读取数据时会对 path context 进行严格过滤，手写数据由于并非从真实 Java 源码 AST 中提取，缺乏合法的语义路径信息，最终导致：


```corpus: 0```
```train dataset size: 0```

因此，Day 1 的结论是：CACHE 不能简单依赖人工构造输入数据，而必须回到官方 preprocessing 流程，从真实源码和 patch 中提取 AST path context。

基于这一判断，Day 2 的工作重点转向 CACHE 项目中的官方预处理脚本：
```
Preprocessing/genOverfittingPatches.py
```
本阶段的核心任务不再是手动构造 dataset，而是修复 preprocessing 主流程中遇到的实际问题，包括 patch 文件名解析、源码路径定位、Defects4J checkout、系统依赖缺失以及异常样本跳过等问题。

## 1. 实验目标

本阶段实验是在 Day 1 已完成基础环境检查和 patch 数据确认的基础上继续推进。

本阶段的主要目标包括：

1. 在服务器环境中复现 CACHE 项目的 preprocessing 主流程；
2. 修复原始脚本在 patch 元信息解析、源码路径定位、Defects4J checkout、异常跳过等环节中的关键问题；
3. 验证脚本是否能够稳定批量处理 patch；
4. 输出可用于后续 method-level 分析和 dataset 构建的数据结果；
5. 为后续 AST mining、dataset 构建和模型训练阶段打基础。

## 2. 实验环境
### 2.1 系统环境

|  项目      | 配置  |
|  ----     | ----  |
| 操作系统	  | Ubuntu 22.04 |
| Python 环境  | Python 3.10.12（.venv） |
| Java 环境  | OpenJDK 11 |
| 主要依赖	    | Defects4J|
| 额外修复依赖 |  svn  |
| 运行方式	SSH |  远程服务器 |
	
	

	
 

### 2.2 相关目录

本次实验主要在以下目录下进行：
```
~/workspace/paper-reproduction/CACHE
```
关键文件和目录包括：
```
Preprocessing/genOverfittingPatches.py
temp_projects/
correct_D4j/
patches/Small/correct/
patches/Small/overfitting/
```
其中：

```Preprocessing/genOverfittingPatches.py``` 是本次主要调试和修复的脚本；
```temp_projects/``` 用于存放 Defects4J checkout 出来的临时项目；
```correct_D4j/``` 用于保存最终提取出的 method-level 输出；
```patches/Small/``` 中存放 CACHE 提供的 patch 数据。

3. 从 Day 1 遗留问题出发

Day 1 的实验已经说明，直接手动构造 CACHE 输入数据并不可行。主要原因是：

```path_contexts.csv``` 不是普通文本数据，而是从 Java AST 中提取出的结构化路径信息；
模型会对非法 context 进行过滤；
手写数据缺少真实源码语义；
最终会导致所有样本被 silent drop。

因此，Day 2 的重点转为运行官方 preprocessing 脚本。

但是，在最初运行 ```genOverfittingPatches.py``` 时，脚本虽然能够启动，却在实际处理 patch 时暴露出多个严重问题，主要包括：

```patch``` 文件名解析不稳健；
```patch``` 中的源码路径无法正确映射到 checkout 后的真实源码路径；
Chart 项目大量 patch 处理失败；
Defects4J checkout 存在隐性失败；
服务器缺少 svn，导致部分项目无法 checkout；
部分 Defects4J bug 已经 deprecated，需要显式跳过。

这些问题导致 preprocessing 主流程无法稳定推进。

4. 初始问题分析
4.1 patch 文件名解析错误

原始 parse_info() 对 patch 文件名的解析方式较为简单，对不同 patch 命名格式的适应性不足。

例如，patch 文件名可能类似：

```patch1-Math-25-dev.patch```
```patch1-Chart-11-SketchFix.patch```
```patch1-Math-50-ACS.patch```

如果直接用固定位置拆分，很容易导致：

APR 名称解析错误；
project 名称解析错误；
bug id 提取失败；
patch id 提取不稳定。

这会进一步影响后续 checkout 路径和输出目录命名。

4.2 源码路径定位失败

patch 文件中记录的 Java 文件路径并不完全统一，可能出现多种形式，例如：

```source/org/...```
```/source/org/...```
```a/source/org/...```
```src/main/java/org/...```
```a/src/main/java/org/...```
```src/java/org/...```

而 checkout 后的 Defects4J 项目中，真实源码路径也可能因项目不同而不同。

原始 get_original_file() 对这些路径形式处理不统一，导致大量 patch 无法定位到对应的 Java 源文件。

4.3 Chart 项目大面积失败

初始阶段，很多 Chart-* patch 都无法成功处理。

一开始怀疑是路径映射问题，但进一步排查后发现，部分 checkout 目录本身就是空的。这说明问题不完全在 get_original_file()，而是在更早的 Defects4J checkout 阶段。

4.4 Defects4J checkout 隐性失败

原始 checkout() 逻辑存在一个比较隐蔽的问题：

只要目标目录存在，就认为 checkout 成功。

但是实际情况是：

目录可能已经创建；
checkout 过程可能失败；
目录可能为空；
脚本仍然会误判为成功，并继续进入后续 patch 应用流程。

这样会导致后面的路径定位函数报错，但真正的根因其实是 checkout 没有成功。

4.5 系统依赖缺失

进一步查看日志后发现，服务器环境中缺少 svn，导致部分 Defects4J 项目无法正常 checkout。

日志中出现过类似错误：

Can't exec "svn": No such file or directory

这也是 Chart 项目大面积失败的重要原因之一。

4.6 deprecated bug 引发异常

Defects4J 中部分 bug 已经被标记为 deprecated，例如：

```Time-21```
```Lang-18```
```Lang-48```
```Lang-2```

这些样本无法正常 checkout，不应该被当作普通脚本错误处理，而应该被显式识别并跳过。

5. 修改与修复过程
5.1 修复 parse_info()

首先对 patch 文件名解析逻辑进行了增强。

修改后的思路是：

先获取 patch 文件名；
去掉 .patch 扩展名；
按 - 拆分字段；
显式提取：
patch_id
project
bug_id
对字段数量和数字格式进行检查；
对异常格式抛出明确错误，而不是默默解析错误。

修复后的逻辑能够更稳定地处理类似下面的 patch 文件名：

patch1-Math-25-dev.patch
patch1-Chart-11-SketchFix.patch
patch1-Math-50-ACS.patch

这一步解决了 APR 名称、project 名称和 bug id 混淆的问题。

5.2 修复 get_original_file() 的路径标准化

接下来对 get_original_file() 中的 patched_file 进行了路径标准化。

主要处理包括：

将 Windows 风格路径分隔符 \ 统一替换为 /；
去除路径开头多余的 /；
自动修正缺失点号的 .java 后缀；
去掉 patch 中常见的 a/ 或 b/ 前缀；
识别并保留关键源码路径前缀，例如：
src/main/java/
src/java/
source/

例如：

a/source/org/example/Foo.java

会被标准化为：

source/org/example/Foo.java

又如：

a/src/main/java/org/example/Bar.java

会被标准化为：

src/main/java/org/example/Bar.java

这样可以减少 patch 路径和真实项目路径之间的不一致。

5.3 强化源码文件匹配逻辑

在路径标准化的基础上，进一步增强了源码文件匹配逻辑。

原始实现主要依赖：

文件名；
父目录；
简单的 path_mapping。

但这种方式在多个项目中不够稳定，尤其是当不同模块中存在同名 Java 文件时，容易匹配失败或误匹配。

因此，本阶段增加了更稳健的匹配策略：

优先尝试标准化后的相对路径；
尝试通过 project 名称、父目录名和文件名组合生成候选 key；
当直接匹配失败时，使用 os.walk() 在 checkout 后的源码树中进行 fallback 搜索；
对候选文件路径进行父目录和文件名层面的二次判断。

这一步对以下项目的 patch 处理都有帮助：

Chart
Lang
Math
Time
Closure
5.4 修复 checkout() 逻辑

原始 checkout() 的问题是只检查目录是否存在，不检查 checkout 是否真正成功。

本阶段对 checkout 流程进行了增强：

明确构造 target_path；
当目标目录已存在但为空时，输出：
[CHECKOUT-EMPTY-EXISTING]
执行 defects4j checkout 后检查返回状态；
checkout 失败时输出：
[CHECKOUT-FAILED]
checkout 后目录不存在时输出：
[CHECKOUT-MISSING]
checkout 后目录为空时输出：
[CHECKOUT-EMPTY]
如果 checkout 失败，则不再继续进入后续 patch 应用和 method 提取流程。

这样可以把“checkout 失败”和“源码路径定位失败”区分开来，避免后续调试方向错误。

5.5 安装 svn，解决 Chart checkout 失败

在查看日志后确认，Chart 项目失败的重要原因之一是服务器环境缺少 svn。

错误信息类似：

Can't exec "svn": No such file or directory

因此安装 svn：

sudo apt update
sudo apt install subversion

安装完成后，再次运行 preprocessing 脚本，Chart 相关项目能够正常 checkout，之前的大面积失败明显减少。

这说明 Chart 的问题并不完全是路径解析导致的，而是 checkout 阶段就已经失败。

5.6 对 deprecated bug 进行显式跳过

部分 Defects4J bug 已经 deprecated，无法正常 checkout。

目前确认会被跳过的典型样本包括：

Time-21
Lang-18
Lang-48
Lang-2

针对这类样本，当前脚本会输出类似：

[CHECKOUT-FAILED]
[SKIPPED-CHECKOUT]

这样可以将它们和普通失败样本区分开来。

这些 deprecated bug 不再污染主流程，而是作为已知不可用样本单独处理。

6. 运行结果
6.1 主流程运行情况

完成上述修改后，重新运行：

python Preprocessing/genOverfittingPatches.py

脚本已经能够稳定推进，并顺利处理多个 Defects4J 项目，包括：

Chart
Lang
Math
Closure
Time

从日志来看，脚本不再出现大面积未捕获 Python 异常，主循环可以持续运行。

这说明 preprocessing 主流程已经基本跑通。

6.2 最终统计结果

脚本运行结束后，末尾打印出两个关键数字：

535
181

根据代码尾部逻辑，这两个数字分别对应：

指标	数值	含义
cnt	535	总共处理到的 patch 数量
successful	181	成功完成 method-level 输出的 patch 数量

也就是说，本轮实验中：

共处理 patch：535 个；
最终成功生成 method-level 输出：181 个。

这说明当前脚本已经不只是“能启动”，而是能够实际完成一部分有效数据生成。

6.3 错误统计结果

进一步统计日志后得到：

日志标记	数量	含义
[SKIPPED-CHECKOUT]	4	因 checkout 失败被跳过的样本
[CHECKOUT-FAILED]	4	checkout 失败样本
Failed at	0	未捕获 Python 异常数量

这说明：

当前主流程中没有未捕获异常破坏运行；
checkout 层面的失败主要集中在少量 deprecated bug；
preprocessing 主循环已经具备基本稳定性；
剩余未成功样本需要进一步做阶段性失败分类。
7. 当前结论
7.1 已完成的工作

本阶段已经完成以下工作：

打通 CACHE preprocessing 主流程；
修复 parse_info() 的 patch 文件名解析问题；
修复 get_original_file() 的路径标准化问题；
增强源码文件定位逻辑；
修复 Defects4J checkout 的隐性失败判断；
安装 svn，解决 Chart 项目 checkout 失败问题；
对 deprecated bug 进行显式跳过；
初步获得 method-level 输出结果；
将阶段脚本和日志快照进行整理。
7.2 当前判断

根据目前结果，可以认为：

CACHE 的 preprocessing 主流程已经基本复现成功。

更准确地说，本阶段已经从 Day 1 的“确认 patch 数据存在，但无法构造有效 dataset”推进到：

能够批量处理 535 个 patch，并成功为其中 181 个 patch 生成 method-level 输出。

这说明 CACHE 复现中最容易卡住的 preprocessing 阶段已经取得实质性进展。

8. 仍然存在的问题

虽然 preprocessing 主流程已经基本跑通，但仍有一些问题需要继续分析。

8.1 为什么只有 181 个 patch 最终 successful

当前只知道：

cnt = 535
successful = 181

但是对于剩余 patch，目前还没有完整拆分失败原因。

可能的失败类别包括：

checkout skipped；
patch apply 失败；
method 提取失败；
save methods 失败；
patch 修改文件无法定位；
patch 内容和 checkout 版本不匹配；
输出目录清理导致结果缺失。

下一阶段需要增加更细粒度的统计信息。

8.2 需要验证 181 个成功样本的质量

虽然有 181 个 patch 成功生成输出，但还需要进一步抽样检查这些结果是否真的可用。

需要检查的问题包括：

输出文件是否完整；
method-level 内容是否为空；
correct / overfitting 标签是否保留正确；
patch 前后方法是否匹配；
是否存在错误映射到其他同名 Java 文件的情况。
8.3 尚未进入模型训练指标分析阶段

Day 1 中曾经提到过模型结果异常问题，例如：

accuracy = 1.0
precision = 0
recall = 0

但需要注意的是，Day 2 的重点仍然是 preprocessing 主流程修复。

目前还没有进入正式的训练结果分析阶段。只有在 method-level 输出稳定生成、AST mining 完成、dataset 构建完成之后，才适合继续分析这些训练指标异常问题。

## 9. 下一阶段计划

下一阶段计划主要包括以下几个方向。

### 9.1 增加更细粒度的失败统计

准备在脚本中加入阶段性计数器，例如：

skipped_checkout = 0
apply_patch_failed = 0
method_extract_failed = 0
save_methods_failed = 0
file_mapping_failed = 0

这样可以明确知道 535 个 patch 中，每一类失败分别占多少。

### 9.2 抽样验证 successful 输出

对 181 个成功样本进行抽样检查，重点确认：

输出目录是否存在；
输出内容是否为空；
method 提取是否合理；
patch 前后方法是否对应；
correct / overfitting 标签是否正确。
9.3 继续推进 AST mining

如果 method-level 输出质量确认没有明显问题，下一步可以进入：

AST mining

也就是从 Java 方法中进一步提取 AST path context，为后续 dataset 构建做准备。

9.4 构建正式 dataset

在 AST path context 能够成功生成之后，继续构建 CACHE 模型需要的输入文件，例如：

tokens.csv
paths.csv
path_contexts.csv
subtoken_dict_stem.csv
node_types.csv

这一步将直接决定后续模型训练是否能够正常运行。

9.5 进入模型训练与结果分析

当 dataset 构建完成后，再进入模型训练阶段，并分析 Day 1 中提到的异常指标问题，例如：

accuracy = 1.0
precision = 0
recall = 0

重点判断这些异常结果是由以下哪类原因导致：

数据集为空或极小；
标签分布严重不平衡；
数据加载逻辑错误；
模型预测单一类别；
evaluation 逻辑存在问题。

## 10. 阶段性总结

本次实验从 Day 1 的“手写数据不可行，需要回到官方 preprocessing 流程”继续推进。

Day 2 的核心工作是修复 Preprocessing/genOverfittingPatches.py 中阻塞主流程的关键问题，包括：

patch 文件名解析；
Java 源码路径标准化；
checkout 结果检查；
Chart 项目 checkout 失败；
svn 系统依赖缺失；
deprecated bug 跳过逻辑。

完成这些修复后，脚本已经能够稳定处理多个 Defects4J 项目，并最终在 535 个 patch 中成功为 181 个 patch 生成 method-level 输出。

因此，本阶段可以认为：

CACHE 复现的 preprocessing 主流程已经基本跑通。

这为后续 AST mining、dataset 构建、模型训练和 overfitting patch detection 实验打下了基础。

## 11. 当前状态概览
| 阶段	| 状态 |
| ----  | ---- |
| 环境配置	|已完成|

patch 数据检查	已完成
手写 dataset 尝试	已验证不可行
preprocessing 脚本启动	已完成
patch 文件名解析修复	已完成
源码路径定位修复	已完成
Defects4J checkout 修复	已完成
svn 依赖修复	已完成
deprecated bug 跳过	已完成
method-level 输出	初步完成
AST mining	待进行
dataset 构建	待进行
模型训练	待进行
指标异常分析	待进行
