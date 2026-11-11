# Python基础轮时测试题

用于短任务基础优化或复现案例后期的12函数题。它直接生成可检查的Python代码，便于把功能和完整轮时一起看。已有更适合的固定Python任务可以继续使用，无需换题重跑历史。

## 请求设置

温度0，关闭思考；新请求只带一条user消息，不附历史或工具。历史Strata接口使用`chat_template_kwargs.enable_thinking=false`、`stream=true`、`max_tokens=3000`，让模型自然结束；不是要求生成3000token。其他后端使用其实际支持的等价字段。

**常规阶段4保留确认方案中的检查点和缓存设置**，完整轮时与续接检查对应同一部署组合。只有专门复现历史隔离测速时才使用`strata_checkpoint=false`，该读数单列为历史条件，不代替正式检查点开启时的成绩。

题目在历史模板下约382输入token。先完成必要预热，再按当前阶段做少量相同独立请求。正式比较时可沿用1次预热、5次记录；不把5次变成每次小改的固定门槛。专家缓存暖态和前缀复用分开记录，基础独立请求结束后另做实际生成内容续接。

## 原题

```text
Write a Python module implementing exactly these pure functions. Return only Python code, no markdown, explanations, imports, or tests. Use straightforward readable loops and conditionals. Inputs meet the stated types. Do not mutate inputs.
1. clamp(x, lo, hi): return lo if x<lo, hi if x>hi, otherwise x; lo<=hi.
2. is_even(n): boolean, works for negative integers.
3. sum_squares(nums): sum of squares of integer items; empty -> 0.
4. count_vowels(text): count ASCII a,e,i,o,u, case insensitive.
5. unique_stable(items): list of distinct hashable items in first-seen order.
6. chunked(items, size): list of lists, final chunk may be short; size<=0 raises ValueError; empty items -> [].
7. rle_encode(text): list of (character,count) tuples for consecutive runs; empty -> [].
8. rle_decode(runs): inverse of rle_encode, counts are positive integers.
9. merge_intervals(intervals): input list of integer (start,end) tuples with start<=end; return sorted list of tuples. Merge exactly when next_start <= current_end; do NOT use current_end+1. Thus [(1,3),(3,4)] -> [(1,4)], but [(1,4),(5,8)] stays separate. Empty -> [].
10. transpose(matrix): rectangular list of lists; return list of lists of columns; [] or [[],[]] -> [].
11. binary_search(nums,target): sorted distinct integer list; return index or -1.
12. valid_brackets(text): ignore all characters other than ()[]{}; return whether brackets are correctly nested and balanced.
```

## 检查与读数

运行生成代码，检查空输入、负数、区间边界、括号嵌套、非法chunk大小及不修改输入。特别保留`merge_intervals([(1,4),(5,8)]) == [(1,4),(5,8)]`：连续整数端点不等于相交，题目已明确这一点。

记录请求对应的实际输入/输出、正常结束状态、完整窗口总时间和窗口数、Decode速度、专家命中与前缀复用。结果表可只用“版本、Python ms/轮、tok/s、功能、续接”五列，再补正在定位的那一项成本。

从请求日志或请求ID对齐原生计时，等该请求的汇总完整落盘后再取值，不直接抓文件最后一条就当本次结果。总Decode均速使用总输出/总Decode时间。常态轮时、短时最低值、滚动吞吐峰值分别列出。

诊断所需的有效token/轮可以记录，但固定K和草稿，先优化执行成本。K调优仍在操作者指定任务阶段进行。实施方法见[工程拆解](../references/optimization-engineering.md)，流程见[短任务毫秒优化](../references/millisecond-optimization.md)。
