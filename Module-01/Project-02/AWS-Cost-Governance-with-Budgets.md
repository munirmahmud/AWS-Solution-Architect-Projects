# Module 01 — Project 02: AWS Cost Governance with Budgets

## Project Summary

Configured a proactive cost management system using AWS Budgets to monitor and alert on monthly AWS spending. This establishes a financial guardrail for the AWS account before any significant infrastructure is deployed.

## Architecture Overview

**Service Used:** AWS Budgets (Global Service — account-wide, no region scope)
**Budget Type:** Monthly Cost Budget
**Alert Mechanism:** Email notifications via SNS (managed by AWS Budgets internally)

```
AWS Account Spend
       ↓
  AWS Cost Explorer (data source)
       ↓
  AWS Budgets (monitors actual + forecasted spend)
       ↓
  Threshold Breached? → Email Alert → You take action
```

## Configuration Details

| Parameter | Value |
|---|---|
| Budget Type | Monthly Cost Budget |
| Budget Amount | $5–$10 (learning account guardrail) |
| Alert Threshold 1 | 85% of actual spend |
| Alert Threshold 2 | 100% of actual spend |
| Alert Threshold 3 | Forecasted to exceed 100% |
| Notification Method | Email |
| Data Refresh Lag | 8–24 hours |
| Free Tier | First 2 budgets free/month |

## Key Architectural Decisions

**Why AWS Budgets over CloudWatch Billing Alarms?**
Budgets supports forecasted spend alerts — it warns you *before* you breach the limit, not just after. It also supports usage-based budgets, RI/Savings Plans coverage budgets, and automated remediation actions. CloudWatch Billing Alarms are simpler but purely reactive.

**Why start at 85% threshold?**
Gives you a reaction window. Cost data lags up to 24 hours, so by the time an alert fires at 85%, you realistically have some runway before hitting 100%.

**Why keep the budget amount low?**
On a learning account, there's no business justification for high spend tolerance. Small budgets enforce discipline and surface misconfigurations (forgotten running instances, open NAT gateways) quickly.


## SA Trade-offs & Scalability Path

| Current Setup | Production / Enterprise Equivalent |
|---|---|
| Single monthly cost budget | Separate budgets per team, project, environment |
| Email notification only | SNS → Lambda → auto-remediation (stop EC2, restrict IAM) |
| Account-wide scope | Per-service, per-tag, or per-linked account scope via AWS Organizations |
| Manual response to alert | Budget Actions (hard enforcement via IAM policy attachment) |


## Security Considerations

- No sensitive data is exposed through budget alerts
- In enterprise setups, Budget Actions can automatically attach a **Deny policy** to IAM roles when thresholds are breached — effectively hard-stopping further spend
- Root account should have MFA enabled before relying on any cost alert system (covered in Project 12)


## Lessons Learned

- AWS Budgets is a **global service** — no region selection required
- Cost Explorer must be enabled as a prerequisite for forecasted alerts to function
- Budget alerts are **not real-time** — 8 to 24 hour data lag means this is a guardrail, not a kill switch
- Two budgets per month are free; beyond that, $0.02/day per budget


## Relationship to Other Projects

| Project | Connection |
|---|---|
| Level 01 — Project 1 (Billing Alarms) | Complementary reactive layer; together they form a complete cost alerting strategy |
| Level 01 — Project 12 (IAM User) | IAM policies are what Budget Actions enforce when automating spend control |
| Level 200+ (Budget Actions) | Next evolution — automated remediation instead of just alerting |
