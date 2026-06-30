---
name: openclaw-cron-health-monitor
slug: openclaw-cron-health-monitor
version: 1.1.0
displayName: "Cron Health Monitor — cron健康监控与自动修复"
summary: "Proactive cron job health monitoring, failure detection, and auto-repair delegation. Triggers: 'cron failed', 'cron health', 'fix cron', 'consecutive errors'"
description: "Proactive cron job health monitoring, failure detection, and auto-repair delegation. Triggers: 'cron failed', 'cron health', 'fix cron', 'consecutive errors'"
tags:
  - cron
  - monitoring
  - repair
  - health
  - agent
license: MIT
---

# Cron Health Monitor — cron健康监控与自动修复

cron不会告诉你它死了。它只会安静地不再运行。你发现的时候已经三天过去了。

---

## 五分钟：为什么你需要这个

你有几十条cron。每一条都在做不同的事——发帖、巡查、学习、备份。你认为它们在跑。但你没有证据。

更糟的是：cron的死法不是"报错"——是安静地不再运行。没有lastRunAtMs。没有consecutiveErrors。就是消失了。你打开cron list，发现它还在列表里，只是state字段里全是null。

这个skill给你三件事：①知道cron什么时候死了（心跳监控）②知道是什么杀死了它（六种失败模式库）③自动告诉正确的人去修（修复委派表）。

---

## 进化阶梯

### L1：可见性
你第一次能看到所有cron的状态。不是"成功/失败"——是连续失败次数、最近一次错误信息、距离上次成功运行多久了。你看到两条cron已经连续3天失败且没人知道。你打了个寒颤。

### L2：自动告警
你不再需要每天手动查cron。consecutiveErrors≥2→自动告警对应部门。≥3→同时告警CEO。你收到第一条告警的时候不是慌——是解脱。因为你知道你不用再当那个"记得去查"的人。

### L3：自动修复
常见故障模式匹配→自动修复→验证→记录。普通问题不再经过人。你只收到"修复成功"的通知。你开始信任这个系统。

---

## 六种失败模式

### 模式1：静默消失（最危险）
```
症状: cron在列表中，但state.lastRunAtMs=null或很久没更新
       consecutiveErrors=0但实际从未成功过
根因: 创建时model字段缺失 / API Key失效后未被移除
       OpenClaw静默跳过，不报错
发现: 只有手动跑一次或检查state字段才能发现
频率: ★★★★★ —— 这是最常见的死法

真实案例（2026-06-26）：知乎cron f0cf6a8a创建时缺model字段，
                      OpenClaw静默跳过，创建后从未运行。
                      24小时后CEO突击审计才发现。
```

### 模式2：模板展开错误
```
症状: lastError="Write failed: ..."
       payload含 <xxx> 格式字符串
根因: 模板引擎将payload中的<变量名>误解析为write路径
修复: payload改为纯文本，移除所有< >包裹的模板变量
频率: ★★★★

真实案例（2026-06-26）：两条cron payload含<task-summary_time_tag>，
                      被模板引擎解析为write路径→Write failed。
                      consecutiveErrors=1但无人知晓。
```

### 模式3：API额度耗尽
```
症状: lastError含"insufficient quota"/"resource exhausted"
       cron之前正常运行，突然连续失败
根因: 免费模型API额度用尽 / 付费余额不足
修复: 切换备用模型 / 充值
频率: ★★★

真实案例（2026-06-30）：技能维护cron da4aefb1使用deepseek免费模型，
                     额度耗尽后连续失败。consecutiveErrors=1。
```

### 模式4：通讯通道静默失败
```
症状: cron执行成功(ok=true)，但接收方没收到消息
根因: isolated session中message工具直发飞书失败
修复: 一律改用sessions_send
频率: ★★★

真实案例（2026-06-25）：销售部+品牌部日报cron delivery配置
                      为message直发飞书→静默失败2天。
```

### 模式5：上下文溢出
```
症状: lastError含"context length"/"token limit"
      模型无法处理全部输入
根因: 输入太长+模型上下文窗口小
修复: 启用lightContext / 切大上下文模型
频率: ★★
```

### 模式6：内容质量腐化
```
症状: cron执行成功，产出文件存在
      但内容含"[待定]""[日期]""正常"等占位符/禁词
根因: prompt不具体 / 模型质量下降 / 免费模型幻觉
修复: 重写prompt / 切付费模型 / 加产出校验
频率: ★★★★ —— 比技术故障更隐蔽

真实案例（2026-06-26）：3个部门日报含占位符原样输出。
                      监察部之前只查"有没有"不查"有什么"。
```

---

## 三模监控（全量覆盖）

### 模式A：按钉监控（Proactive Push）
每6小时主动检查全量cron的健康状态。发现问题→立告警。

```
cron: 大脑心跳检测 (every 6h)
检查项:
  - 全部cron的consecutiveErrors
  - state.lastRunAtMs ≤ 预期周期×2
  - lastError非空
告警规则:
  - consecutiveErrors≥2 🟡 → 对应部门+数据中心
  - consecutiveErrors≥3 🔴 → +CEO
  - 预期周期×2未运行 🔴 → 直接告警CEO
```

### 模式B：轮询监控（Polling Watch）
每次日报产出前，各部门自检本部门cron的健康状态。

```
cron前缀匹配部门 → 扫描consecutiveErrors
→≥1的写入日报"异常"段
```

### 模式C：被动监控（Passive Detection）
每次cron运行后，检查自身的exit code和输出。不需要额外cron——cron自己监控自己。

---

## 误报排除清单

以下情况不是故障，不要告警：
1. 新建cron首次运行前——lastRunAtMs=null是正常的
2. 显式禁用的cron（enabled=false）——不检查
3. deleteAfterRun的cron——已完成即删除，不检查
4. 配置的休眠时段内——如需跳过某些时段的检查
5. concurrencyCap限流导致的跳过——不算故障

---

## 修复委派表

```
故障模式               → 委托部门    → 修复动作
静默消失/模板展开       → 数据中心    → 补model/清模板变量→验证
API额度耗尽            → 财务部+数据中心 → 切fallback/充值
通讯通道静默失败        → 数据中心    → 改sessions_send
上下文溢出             → 数据中心    → lightContext/切模型
内容质量腐化            → 对应部门+监察部 → 重写prompt/切模型
```

修复后必须验证：手动触发cron一次→确认产出正确→记录修复日志。

---

## 调试清单（cron排查三步走）

**第一步：查状态**
```bash
cron get <jobId>  # 看lastRunAtMs / lastError / consecutiveErrors
cron runs <jobId> # 看最近运行记录
```

**第二步：手动触发**
```bash
cron run <jobId> --mode=force  # 强制运行一次，看输出
```

**第三步：对比法**
同类型cron正常吗？同模型cron正常吗？同delivery配置的cron正常吗？对比三个维度通常能定位根因。

---

## References

- [references/common-errors.md](references/common-errors.md) — 12种常见cron错误+诊断+修复
- [references/cron-templates.md](references/cron-templates.md) — 即用cron模板（学习/日报/巡查/维护）
- [references/repair-playbook.md](references/repair-playbook.md) — 自动修复委托+验证规则
