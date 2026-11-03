---
name: oos-rollback-audit
description: 信号/门控被跨年 OOS 或盲测证伪后的生产落点审计链。五步：特征码定位生产落点→模块判读（自算 vs 传入）→命名歧义鉴别→Δ 性质判读（中性保留/有害回退）→注释降级+闸门复扫+状态机登记。凡「证伪/作废/降级」动作后必走此链，防「研究结论作废但生产还在用」的断链。
agent_created: true
---

# OOS 证伪 → 生产落点审计链（oos-rollback-audit）

## 触发条件（任一命中即走此链）

- 信号/门控/语境闸被**跨年 OOS**、**盲测**、**口径审计**证伪（Δ 不达门槛 / 盲测为负 / 未来泄漏实锤）
- 经验库/rules 表把某条规则从 verified/candidate 改判 rejected
- 用户说"XX 作废了"后，**必须确认生产链里它还在不在跑**

## 五步流程

### ① 特征码全盘定位生产落点

证伪结论落档 ≠ 生产链已清理。先构造该逻辑的**特征码**（阈值常数/变量名/关键表达式），grep 全盘：

```bash
grep -rn "0.003\|rel15\|同向门控" --include="*.py" core/ execution/ monitor_realtime.py signal_engine_v1.py
```

注意：研究脚本（`_` 前缀）与生产文件（core/、execution/、monitor、serve）要分开统计。**生产落点才是 P0**。

### ② 模块判读：自身计算 vs 外部传入

变量可疑时先看**值从哪来**：

- 模块**自身计算** → 检查计算口径是否 ≤i（如 `iloc[:local+1].max()` 合规 / `iloc[:].max()` 全日聚合泄漏）
- 模块**外部传入参数**（如 `ctx_veto.py` 的 `pos_frac` 是入参）→ 模块本身无罪，风险在**调用方**。再 grep 调用方，逐一看传入值的计算口径

### ③ 命名歧义鉴别（铁律：先读定义再下结论）

变量名 ≠ 语义。本次实证：系统内 `rel15` = **近15min动量**（前第3根 5m bar close，≤i 合法），leak_scan 想当然解释为 "vs 当日 15:00 收盘=未来泄漏" → 2 次误报，差点把合法代码当泄漏重跑。

处理原则：
- **先读变量赋值行**（`p3 = bars["close"].iloc[i-3]` → 15min 前，不是 15:00），再判定
- 同名不同义 = 命名风险源 → **改名优先**（如 rel15→rel3），其次加 `# leak-ok`
- leak_scan 词边界特性：`_rel15`（下划线前缀）**不触发** `\brel15\b`，生产局部变量带下划线可天然绕开

### ④ Δ 性质判读（决定回退 vs 保留）

| Δ(OOS) 结果 | 性质 | 动作 |
|---|---|---|
| Δ ≈ 0（±1pp 内） | **中性**（无害无益） | 保留代码 + 注释降级标注，物理回退交拍板 |
| Δ < 0 显著 | **有害**（负资产） | **开盘前物理回退**，P0 |
| 未来泄漏实锤 | **作弊** | 无条件回退 + 全量结论重审 |

中性场景别过度反应：改产线行为本身有风险，「无增益」不等于「该删」。

### ⑤ 收尾四件套

1. **注释降级**：生产代码里的 "verified YYYY-MM-DD" 标注必须补 OOS 证伪信息（docstring + 调用处注释），防错误知识继续传播
2. **闸门复扫**：`python leak_scan.py <改动文件>` + `python leak_scan.py --self` 均 0 HIGH 0 MED
3. **py_compile**：所有改动文件编译验证
4. **状态机登记**：`core/production_ready.py register(...)`——研究结论（含证伪/降级）进登记库，审计[20]可兜底扫描

## 本次实证案例（2026-09-08）

V1 惯性门控（`abs(chg_td)≥0.003 and chg_td*rel15≥0`）跨年 OOS 证伪（2023/24/25 7-8月 Δ≈+0~+1%，远低 +3pp 门槛）：

1. grep 定位：生产落点 `execution/intraday_roll15.py` L444（概率接管 `_day_inertia_master`）+ L475（v4 筛子 S2惯性）
2. 判读：`ctx_veto.py` pos_frac 外部传入（模块无罪，且无生产调用方）→ P0 解除
3. 歧义鉴别：`rel15` = 近15min动量 ≤i 合法，leak_scan 命名误读 → 文档澄清（leak_scan.py docstring 已补）
4. Δ≈0 中性 → 保留代码，注释降级 "verified 09-04 → OOS 证伪降级研究假设"
5. 收尾：leak_scan 0/0 + py_compile OK + P010/P011 登记 production_ready

## 关联

- 上游闸门：`leak_scan.py`（未来泄漏静态扫描）/ `pa-logic-defect-gate`（9 型逻辑过检）
- 状态机：`core/production_ready.py`（research→verified→pending_promotion→promoted/declined）
- 经验库：problem_experience.db（泄漏族 id 累计 25+ 次，本链是泄漏族「证伪后处理」的标准 SOP）
