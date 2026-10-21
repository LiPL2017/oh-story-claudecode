# Agent Note: check-doc-budget.sh 在 CRLF 工作区锚点全失配

Status: implemented
Date: 2026-10-21

## Problem

`scripts/check-doc-budget.sh` 的 `sectionOf` 用 `/^(#{1,6})\s+(.*)$/` 匹配标题行，而 JS 正则的 `.` 不匹配 `\r`：`core.autocrlf=true` 的 Windows 工作区检出 CRLF 后，所有标题行整行失配，全部 `文件#小节` 条目报「小节找不到」（本地实测 1848 条）、对应路径被 `broken` 跳过不再算总量。守卫在维护者本地形同虚设——CI（Linux LF 检出）正常，因此 02ee9d3 细纲路径漏调的 9 个分支预算缺口（超 16~296 字）一直没被发现，该提交进 main 后任何后续 push 的 CI 都会红。

## Decision

- `sectionOf` 解析前统一 `text.replace(/\r/g, "")`，一处改动同时覆盖标题匹配与围栏判断；脚本其余正则（`preloads` 的 `\r?\n`、`split(/^---\s*$/m)`）本就兼容 CRLF 不动。
- 顺带暴露并补齐 02ee9d3 漏调的预算：story-architect 任务包 3 个分支 +120~140、主会话细纲 6 个分支各 +200，读数不变只补数字；其中任务包三条读数有 ±45 字波动（generated 包里残留的临时目录路径长度随运行变化），余量按波动上限留足。why 字段注明是补调，不新增内容。

## Alternatives considered

- 改成正则加 `\r?`（`/^(#{1,6})\s+(.*?)\r?$/`）：改动同样小，但只修标题一处，围栏行等后续若有同类问题要逐个补；入口统一归一化一劳永逸。
- 不修脚本，只在 CI 跑守卫、Windows 本地用 LF 副本验证：流程可用，但 AGENTS.md 要求提交前本地跑守卫，维护者（win32）每次都要搭 LF 副本太绕，且「本地红但不知道是真红还是环境红」的状态会持续掩盖真实回归——这次 9 条缺口就是这样漏掉的。
- 把 9 条预算缺口留给上一轮作者自己处理：超出量小且 02ee9d3 的 why 已写明加规则的理由，只补数字不算替作者做新决策；留红则下次 push CI 必红，修复入口就在眼前。

## Consequences

收益：Windows 本地守卫读数与 CI 一致，锚点错误清零；main 上被掩盖的预算缺口补齐，CI 恢复可绿。代价：归一化在 `sectionOf` 输入上做一次字符串拷贝，文件最大约几万字符，量级可忽略；`replace(/\r/g, "")` 会把行中孤立的 `\r` 一并去掉，Markdown 正文不使用该字符，无实际影响。
