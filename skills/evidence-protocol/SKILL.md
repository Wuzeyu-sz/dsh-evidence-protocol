---
name: evidence-protocol
description: 证据协议族（作者自建，2026-08~09 生产事故蒸馏）：结论落地前用尺子自检（metric-selftest）、对照设计（ab-comparison-robustness）、事件级 R 验证（event-level-R-verify）、证伪后的生产落点清理（oos-rollback-audit）、派生字段一致性（derived-field-lifecycle-audit）、审计报告复核（audit-report-review）。当你要下「有差异 / 有 edge / 覆盖率是 X% / 已修复 / 已作废」类结论，或要复核别人的审计报告时走这里。
whenToUse: 写计数或回测脚本；比较两组或前后口径；判断信号 E[R]；结论被 OOS 证伪后清理生产链；源字段改写后核派生字段；复核外部审计报告并重排优先级。触发词：对照组、基线、显著性、样本外、OOS、E[R]、覆盖率、派生字段、证伪、复核报告。
version: 1.0.0
---

# 证据协议族（evidence-protocol）

六份协议是同一条结论链上的不同关卡，全部由本机生产事故逐条蒸馏，非转录他人技能。入口只给路由与共用铁律；每份正文逐字保留在 `references/`。

## 0. 路由

| 你手上的活 | 读哪份 |
|---|---|
| 要写计数/扫描/覆盖率脚本，或要报「共 N / 命中 M / 覆盖 X%」 | `references/metric-selftest.md`（正例关/负例关/数量级关） |
| 要做 A/B、分组、前后对比、参数扫描、归因 | `references/ab-comparison-robustness.md`（C1~C26） |
| 要算信号 E[R]、判「有没有正期望」 | `references/event-level-R-verify.md`（事件级 R + 四验） |
| 信号/门控被 OOS 或盲测证伪，要清生产落点 | `references/oos-rollback-audit.md`（特征码→落点→降级→复扫） |
| 源字段被改写，要核派生字段与唯一源头 | `references/derived-field-lifecycle-audit.md` |
| 要复核一份审计/优化报告并重排优先级 | `references/audit-report-review.md` |

优先级：**尺子 > 对照 > 指标 > 落点清理**。尺子不准，后面全废。

## 1. 共用铁律

1. 先验尺子再信数字：计数脚本必须过「正例必中 / 负例必不中 / 数量级合理」三关。
2. 同口径才可比：比较前把两边分母、量纲、含不含某类写成一行。
3. 三态显式化：有值 / 显式 null / 缺失不得混同。
4. 单点不外推：结论前给中位数 + 分位 + n。
5. 负例必须真红：判据有效性靠造故障验证，不靠「代码写了」。
6. 证伪要落到生产：结论作废 ≠ 生产已清理，必须按特征码确认落点。
7. 复核用第一手实测：报告里的数字一律重算，已落地项不得当待办。

## 2. 反模式（真实发生过）

- 实测的 P 乘假设的赔率算 E[R]，两边不同源 ⇒ 符号反转。
- 裸子串计数（`parse` 命中 `parse_float`）⇒ 覆盖率虚高。
- 派生字段不重算，同一行自相矛盾且不报错。
- OOS 证伪只改记录，生产闸门照跑。
- 多重检验只报最好那组。

## 3. 最小闭环

写断言 → 过尺子三关 → 定对照与基线 → 算指标（真实成本）→ 稳健性 → 收口（净胜率/净盈亏比/净 PF/终值/MaxDD + 样本量）→ 若为负：清生产落点。

## 4. references

| 文件 | 体积 |
|---|---|
| `references/ab-comparison-robustness.md` | 75,821 B |
| `references/event-level-R-verify.md` | 10,951 B |
| `references/metric-selftest.md` | 9,094 B |
| `references/derived-field-lifecycle-audit.md` | 7,044 B |
| `references/audit-report-review.md` | 5,691 B（发布版已把一处个人称呼改为「一次复核任务」） |
| `references/oos-rollback-audit.md` | 4,448 B |
