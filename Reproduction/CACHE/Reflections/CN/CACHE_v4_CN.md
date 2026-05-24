# CACHE 复现实验报告（Day4，patch 反向应用修复与 preprocessing 基本闭环阶段）

## 1. 本阶段目标

本阶段是在 Day3 失败归因的基础上继续推进，主要目标不是重新搭建环境，而是解决上一阶段暴露出的核心问题：

1. 针对 `apply_patch_failed = 350` 的问题继续定位根因；
2. 验证大量 developer patch 是否需要反向应用；
3. 修改 `genOverfittingPatches.py`，让 `patch_file()` 支持 normal patch 失败后的 `patch -R` fallback；
4. 重新运行 preprocessing，确认 patch application 和 method extraction 是否可以基本闭环。

---

## 2. Day3 遗留问题回顾

Day3 的完整运行结果为：

- `total_patches = 535`
- `successful = 181`
- `skipped_checkout = 4`
- `apply_patch_failed = 350`
- `method_extract_failed = 0`

当时的判断是：

- preprocessing 主流程已经能跑完整个 patch 集；
- checkout 问题很少，只有 4 个样本被跳过；
- method extraction 没有失败；
- 最大瓶颈集中在 `apply_patch_failed = 350`。

因此，Day4 的重点就是进一步确认：

> 这 350 个 patch application failure 到底是不是 patch 方向问题。

---

## 3. 关键日志现象：reversed patch

在分析 `run_clean_checkout_test_full.log` 时，发现大量失败样本中出现类似信息：

```text
Reversed (or previously applied) patch detected! Assume -R? [n]
```

这说明 `patch` 命令并不是找不到文件，也不是 diff 格式完全不可用，而是在提示：

> 当前 patch 的方向和 checkout 出来的源码方向相反，或者源码已经处于 patch 应用后的状态。

也就是说，Day3 中认为的 patch-version mismatch 可以进一步细化为：

> 一大批 developer correct patch 对当前 checkout 源码来说需要反向应用。

---

## 4. 人工验证：Time-20 patch 可以用 `-R`

本阶段选取了 Day3 中已经抽查过的 Time-20 作为验证对象。

目标 patch 为：

```text
patches/Small/correct/defects4j-developer/Time/patch1-Time-20-dev.patch
```

patch 头部内容显示它修改的是：

```text
src/main/java/org/joda/time/format/DateTimeFormatterBuilder.java
```

随后执行 dry-run 反向应用：

```bash
patch --dry-run -R ./temp_projects/defects4j/defects4j_Time_20/src/main/java/org/joda/time/format/DateTimeFormatterBuilder.java \
  patches/Small/correct/defects4j-developer/Time/patch1-Time-20-dev.patch
```

输出为：

```text
checking file ./temp_projects/defects4j/defects4j_Time_20/src/main/java/org/joda/time/format/DateTimeFormatterBuilder.java
```

没有报错。

### 验证结论

这说明至少在 Time-20 样本上：

- 原始 normal patch 方向会失败；
- 使用 `patch -R` 可以通过 dry-run；
- 因此大量失败样本很可能不是路径匹配问题，而是 patch 方向问题。

---

## 5. 代码修改：为 `patch_file()` 添加 reverse fallback

定位脚本中真正执行 patch 的位置：

```python
def patch_file(patch: str, file_path: str) -> bool:
    command_result = subprocess.run(["patch", "-l", "-u", "--fuzz=15", "-i", patch, file_path],
                                    encoding="utf-8", stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
```

该函数原本只尝试 normal direction。  
当 `patch` 检测到 reversed patch 时，脚本不会交互式输入 `-R`，因此直接失败。

本阶段将其修改为两步策略：

1. 先尝试正常 patch；
2. 如果失败，并且输出中包含：

```text
Reversed (or previously applied) patch detected
```

则自动使用：

```bash
patch -R -l -u --fuzz=15 -i <patch> <file_path>
```

进行反向重试。

修改后的核心逻辑如下：

```python
def patch_file(patch: str, file_path: str) -> bool:
    # First try normal direction.
    command_result = subprocess.run(
        ["patch", "-l", "-u", "--fuzz=15", "-i", patch, file_path],
        encoding="utf-8",
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT
    )

    if command_result.returncode == 0:
        return True

    output = command_result.stdout or ""

    # Some developer correct patches are reversed relative to the checked-out source.
    # If patch reports this case, retry with -R.
    if "Reversed (or previously applied) patch detected" in output:
        reverse_result = subprocess.run(
            ["patch", "-R", "-l", "-u", "--fuzz=15", "-i", patch, file_path],
            encoding="utf-8",
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT
        )

        if reverse_result.returncode == 0:
            print(f"[APPLY-REVERSED-SUCCESS] {file_path}")
            return True

        raise RuntimeError(
            f"Patch {file_path} failed with reverse retry\n"
            + output
            + "\n--- reverse retry output ---\n"
            + (reverse_result.stdout or "")
        )

    raise RuntimeError(f"Patch {file_path} failed with error\n{output}")
```

### 修改原则

这里没有直接把所有 patch 都强制改成 `-R`，因为 Day3 中已经有 181 个 patch 可以 normal direction 成功。  
因此更稳妥的策略是：

> normal patch first, reverse fallback only when reversed patch is detected.

---

## 6. 短跑测试结果

修改完成后，执行短跑测试：

```bash
timeout 180s .venv/bin/python Preprocessing/genOverfittingPatches.py 2>&1 | tee run_reverse_patch_fallback_test.log
```

随后统计反向应用成功次数：

```bash
grep -c "APPLY-REVERSED-SUCCESS" run_reverse_patch_fallback_test.log
```

输出为：

```text
380
```

同时日志中出现大量反向应用成功记录，例如：

```text
[APPLY-REVERSED-SUCCESS] ./temp_projects/defects4j/defects4j_Time_20/src/main/java/org/joda/time/format/DateTimeFormatterBuilder.java
[APPLY-REVERSED-SUCCESS] ./temp_projects/defects4j/defects4j_Time_11/src/main/java/org/joda/time/tz/ZoneInfoCompiler.java
[APPLY-REVERSED-SUCCESS] ./temp_projects/defects4j/defects4j_Time_25/src/main/java/org/joda/time/DateTimeZone.java
[APPLY-REVERSED-SUCCESS] ./temp_projects/defects4j/defects4j_Chart_8/source/org/jfree/data/time/Week.java
```

### 短跑结论

这说明 reverse fallback 已经真实生效，而且不是只修复了单个样本，而是覆盖了大量 previously failed patch。

---

## 7. 本阶段最终运行结果

修改后，脚本重新完整处理 535 个 patch，最终结果如下：

- `total_patches = 535`
- `successful = 531`
- `skipped_checkout = 4`
- `apply_patch_failed = 0`
- `method_extract_failed = 0`

与 Day3 对比：

| 指标 | Day3 | Day4 |
|---|---:|---:|
| total_patches | 535 | 535 |
| successful | 181 | 531 |
| skipped_checkout | 4 | 4 |
| apply_patch_failed | 350 | 0 |
| method_extract_failed | 0 | 0 |

### 结果解释

1. **successful 从 181 提升到 531**
   - 说明之前失败的大部分样本已经被成功修复。

2. **apply_patch_failed 从 350 降为 0**
   - 说明 Day3 的核心瓶颈已经解决。

3. **method_extract_failed 仍然为 0**
   - 说明 patch application 修复后，后续 method extraction 阶段依然稳定。

4. **skipped_checkout 仍然为 4**
   - 这 4 个样本不是 patch application 问题，而是 checkout 阶段被跳过。
   - 结合之前日志，原因很可能是 Defects4J deprecated bug 或不可用 bug。

---

## 8. 当前关键结论

本阶段可以明确判断：

**CACHE preprocessing 中 correct patch 的 generation 流程已经基本闭环。**

理由包括：

- 535 个 patch 中成功处理 531 个；
- patch application failure 已经清零；
- method extraction failure 仍然为 0；
- 仅剩 4 个 checkout skipped，属于 Defects4J 数据源层面的问题，不再是脚本主流程问题。

更准确地说：

> 当前版本的 `genOverfittingPatches.py` 已经能够稳定生成绝大多数 correct patch 的 method-level 数据，Day3 中最大的 apply_patch 瓶颈已经被 reverse fallback 修复。

---

## 9. 当前阶段定位

如果从 CACHE 复现流程来看，当前阶段已经从：

```text
preprocessing debugging
```

推进到了：

```text
preprocessing mostly completed
```

更具体地说：

- 环境配置：已完成；
- Defects4J checkout：基本稳定；
- patch metadata parsing：基本稳定；
- original file location：基本稳定；
- patch application：已通过 reverse fallback 修复；
- method extraction：稳定；
- method-level output generation：基本完成；
- 后续 embedding / vocabulary / training：待继续推进。

---

## 10. 与 Day3 结论的修正

Day3 的结论是：

> 大量 apply_patch 失败可能是 patch-version mismatch。

Day4 的新结果进一步说明：

> 大量 apply_patch 失败更具体地说，是 developer correct patches 与 checkout 源码之间存在 patch direction mismatch；通过 `patch -R` fallback 可以基本修复。

因此，Day3 的归因方向是对的，但 Day4 将问题从“版本不匹配”进一步定位到了“patch 方向不匹配”。

---

## 11. 下一步计划

当前不建议继续在 `apply_patch` 上投入太多时间，因为该阶段已经取得 531 / 535 的成功率。  
下一步应该转向确认输出文件和推进后续 CACHE 主实验。

### 11.1 确认输出文件数量

需要检查 `files` 和 `methods` 输出目录：

```bash
find . -maxdepth 3 -type d -name "files" -o -name "methods"
```

统计 method-level 输出：

```bash
find . -path "*methods*" -name "buggy.java" | wc -l
find . -path "*methods*" -name "fixed.java" | wc -l
```

如果两个数量都接近 531，则说明 preprocessing 输出数据已经真正落盘。

### 11.2 继续查找后续训练入口

接下来需要根据 README 或仓库脚本继续定位：

- AST mining
- vocabulary construction
- embedding generation
- dataset building
- model training
- evaluation scripts

目标是从 preprocessing 阶段进入 CACHE 的 representation learning / patch correctness classification 阶段。

---

## 12. 阶段总结

本阶段最大的进展是解决了 Day3 中最主要的技术阻塞点。

从最终结果看：

- 脚本完整遍历 535 个 patch；
- 成功样本从 181 提升到 531；
- 350 个 apply patch failure 被全部消除；
- reverse fallback 共触发并成功 380 次；
- method extraction failure 保持为 0；
- 仅剩 4 个 checkout skipped。

因此，目前可以认为：

> CACHE preprocessing 的 correct patch 数据生成阶段已经基本完成，后续工作应从脚本修复转向输出验证和模型实验复现。
