# CACHE 复现实验报告（Day3，统计与失败归因阶段）

## 1. 本阶段目标

本阶段的主要目标不是继续“让脚本跑起来”，而是：

1. 在 preprocessing 主流程已跑通的基础上，进一步统计成功与失败的分布；
2. 分析为什么最终只有部分 patch 成功生成 method-level 输出；
3. 对 `apply_patch` 失败样本进行人工抽样验证，判断失败是脚本实现问题，还是 patch 与源码版本不匹配。

---

## 2. 当前整体运行结果

在当前版本的 `genOverfittingPatches.py` 上，脚本已经可以稳定跑完整轮 preprocessing，最终统计结果如下：

- `total_patches = 535`
- `successful = 181`
- `skipped_checkout = 4`
- `apply_patch_failed = 350`
- `method_extract_failed = 0`

### 结果解释

1. **总 patch 数为 535**
   - 表示主循环已经完整处理了整个 patch 集。

2. **成功生成 method-level 输出的 patch 为 181**
   - 这些 patch 已经顺利通过：
     - metadata parsing
     - checkout
     - patch application
     - method extraction
     - method saving

3. **checkout 跳过数为 4**
   - 这 4 个样本对应 Defects4J 中已经 deprecated 的 bug。
   - 属于已知不可用样本，而不是脚本异常。

4. **apply_patch 失败数为 350**
   - 这是当前 preprocessing 阶段的核心瓶颈。

5. **method_extract_failed = 0**
   - 说明只要 patch 能成功应用，后续 method 抽取和保存基本可以正常完成。

---

## 3. 当前关键结论

### 3.1 主流程已经稳定

到这一阶段，可以明确判断：

**CACHE 的 preprocessing 主流程已经基本跑通。**

理由包括：

- 脚本能够完整处理 535 个 patch；
- 没有新的未捕获异常把主流程打断；
- checkout 逻辑已经稳定；
- method extraction 逻辑没有成为主要故障点。

### 3.2 当前主要瓶颈集中在 patch application

从统计结果看：

- checkout 失败仅 4 个；
- method extraction 失败为 0；
- 绝大多数未成功样本都集中在 `apply_patch_failed = 350`。

因此，当前真正限制成功率的核心问题是：

> patch application compatibility

也就是说，大部分失败并不是因为脚本找不到文件、checkout 失败或 method 提取失败，而是 patch 无法成功应用到当前 checkout 下来的源码文件上。

---

## 4. apply_patch 失败分布

对 `[APPLY-FAILED]` 日志进行项目级统计后，得到如下结果：

- Closure: 131
- Math: 106
- Lang: 61
- Time: 26
- Chart: 26

### 分析

这说明 `apply_patch` 失败不是只集中在单一项目，而是跨多个项目广泛存在。  
其中失败最多的是：

1. Closure
2. Math
3. Lang

因此，问题更像是 **developer patch 集和当前 Defects4J checkout 版本之间的整体不对齐**，而不是某个单独项目的特殊异常。

---

## 5. 人工抽样验证：patch-version mismatch

为了验证 `apply_patch` 失败的真实原因，本阶段人工抽查了 3 个失败样本：

- Time-20
- Closure-111
- Math-25

### 5.1 Time-20

目标 patch：

- `patches/Small/correct/defects4j-developer/Time/patch1-Time-20-dev.patch`

patch 想修改的旧代码包含：

- `String best = null`
- `best.length()`
- `bucket.setZone(DateTimeZone.forID(best))`

但在当前 checkout 出来的 `defects4j_Time_20` 源码中，这些旧代码已经不存在，源码已经是 patch 目标中的“新实现”。

**结论：**
Time-20 的 patch application 失败，是因为 patch 想应用到的旧上下文在当前源码中已经不存在。

---

### 5.2 Closure-111

目标 patch：

- `patches/Small/correct/defects4j-developer/Closure/patch1-Closure-111-dev.patch`

patch 想修改的是：

```java
return topType.isAllType() ?
    getNativeType(ARRAY_TYPE) : topType;
```
但在当前 checkout 出来的 defects4j_Closure_111 源码中，已经是：

```return topType;```

结论：
Closure-111 的 patch 同样与当前源码不匹配，旧上下文已经不存在。

### 5.3 Math-25

目标 patch：
```
patches/Small/correct/defects4j-developer/Math/patch1-Math-25-dev.patch
```
patch 想删除：
```
if (c2 == 0) {
    throw new MathIllegalStateException(LocalizedFormats.ZERO_DENOMINATOR);
}
```
但在当前 checkout 出来的 defects4j_Math_25 源码中，这段旧代码已经不存在。

结论：
Math-25 的 patch failure 也属于同一种问题：patch 目标上下文和当前源码版本不一致。

## 6. 本阶段结论

基于统计结果和人工抽样验证，可以得出当前阶段的核心结论：

CACHE 的 preprocessing 流程已经能够稳定运行，但 developer patch 集中的大量 patch 与当前 Defects4J checkout 下来的 buggy version 源码并不完全对齐。
因此，大量 apply_patch 失败更可能来源于 patch-version mismatch，而不是 preprocessing 主流程本身的实现错误。

换句话说：

主流程：基本打通
checkout：基本稳定
method extraction：基本稳定
主要瓶颈：patch 和源码版本不匹配

## 7. 当前阶段定位

如果从“主流程是否跑通”来看，本阶段已经基本完成。
如果从“整个 CACHE 实验结果是否彻底闭环”来看，还没有完全结束。

当前更准确的定位是：
preprocessing pipeline: stable and runnable
output generation: partial success (181 / 535)
dominant failure mode: patch application mismatch
next focus: decide whether to continue with 181 successful samples or align patch/version sources

## 8. 下一步计划

接下来有两条可行路线：

## 路线 A：基于 181 个成功样本继续后续实验

直接使用当前成功生成的 method-level outputs，继续推进：

AST mining
vocabulary construction
dataset building
small-scale training / validation

## 路线 B：进一步追查 patch-version mismatch

如果目标是尽可能提高 preprocessing 成功率，则需要继续分析：

developer patch 集对应的真实源码版本
当前 Defects4J checkout 版本与 patch 集之间的偏差
是否需要切换到更匹配的数据源或 checkout 版本

## 9. 阶段总结

本阶段最大的进展，不是简单地“又跑了一轮脚本”，而是明确了 preprocessing 失败的主要来源。

从结果看，当前已经可以较为确定地说：

preprocessing 主流程本身已经基本跑通；
181 个 patch 已成功生成 method-level 输出；
4 个 deprecated bug 被正确跳过；
剩余 350 个主要失败集中在 patch application；
抽样验证表明，这些失败的主要原因很可能是 patch 与当前源码版本不匹配。

因此，这一阶段已经完成了从“流程修通”到“失败归因”的关键过渡，为后续继续做 CACHE 复现或转入后续训练实验提供了明确依据。
