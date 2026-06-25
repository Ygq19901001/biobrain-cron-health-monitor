# Cron Health Monitor ❤️‍🩹

**Proactive cron job health monitoring, failure detection, and auto-repair delegation for OpenClaw agents.**

[![SkillHub](https://img.shields.io/badge/SkillHub-openclaw--cron--health--monitor-blue?style=flat)](https://skillhub.cn/skills/openclaw-cron-health-monitor)
![Version](https://img.shields.io/badge/version-1.0.0-green)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

Your cron jobs shouldn't die silently. Detect failures before they cascade.

## What You Get

- **5 Failure Patterns** — Message failed, Timeout, Context overflow, Auth error, Model unavailable — each with distinct alerting
- **Tiered Alerting** — P0 (immediate), P1 (within 1h), P2 (daily digest), P3 (weekly summary)
- **Weekly Audit** — Automated full cron inspection every Friday
- **Auto-Repair Delegation** — Detected failures automatically dispatched to the right department for repair

## Quick Install

```bash
skillhub install openclaw-cron-health-monitor
```

Or clone directly:

```bash
git clone https://github.com/Ygq19901001/biobrain-cron-health-monitor.git
```

## Files

```
├── SKILL.md                          # Main skill definition
├── references/
│   ├── common-errors.md              # Failure pattern catalog
│   └── repair-playbook.md            # Auto-repair procedures
```

## Requirements

- OpenClaw agent runtime with cron access
- Cron jobs configured with failure alert settings

## Links

- [SkillHub Page](https://skillhub.cn/skills/openclaw-cron-health-monitor)
- [GitHub Issues](https://github.com/Ygq19901001/biobrain-cron-health-monitor/issues)
