# Beginner - Project 01: Create Three Billing Alarms

> **Service Used:** Amazon CloudWatch + Amazon SNS
> 
> *Your financial safety net — set this up before you build anything else in AWS.*

## 📐 Architecture Overview

```
💰 AWS Spend  →  📊 CloudWatch  →  📣 SNS Topic  →  📧 Your Email
                  3 Alarms           billing-alerts
                 $10 / $30 / $50
```

Three graduated alarms that email you when your AWS spend crosses a threshold. Simple to build, critical to have.


## 🚨 READ THIS BEFORE YOU START — Region Rules

> ⚠️ **This is the #1 mistake that wastes hours. Read carefully.**

ALL three services must be in **us-east-1 (N. Virginia)**:

| Service | Region | Why |
|---|---|---|
| CloudWatch Billing Alarm | `us-east-1` only | Billing metrics are global but only published to us-east-1 |
| SNS Topic | `us-east-1` | Must match the CloudWatch alarm region |
| SNS Subscription | `us-east-1` | Created inside the topic |

**If you create SNS in ap-southeast-1 (Singapore) and the alarm in us-east-1 — they cannot see each other and it will fail silently with no error message.**


## 🧭 Step-by-Step Guide

### Step 1 — Enable Billing Alerts (Prerequisite)

1. Sign in as **root user** (or IAM user with billing permissions)
2. Click your account name (top-right) → **Account**
3. Scroll to **Billing preferences**
4. Check **"Receive Billing Alerts"** → **Save preferences**

> ⚠️ Without this step, billing metrics will **not appear** in CloudWatch at all. This is a one-time account-level setting.


### Step 2 — Switch to us-east-1

1. Look at the region selector (top-right of AWS Console)
2. Select **US East (N. Virginia) — us-east-1**
3. **Keep this region for every remaining step**

> ⚠️ Billing metrics only exist in us-east-1. Any other region = no billing data visible in CloudWatch.


### Step 3 — Create SNS Topic & Subscription (in us-east-1)

1. Go to **SNS → Topics → Create topic**
2. Type: **Standard** | Name: `billing-alerts` → **Create topic**
3. Open the topic → **Subscriptions** tab → **Create subscription**
4. Protocol: **Email** | Endpoint: your email address → **Create subscription**
5. Check your inbox → click **"Confirm subscription"** from the AWS email

> ⚠️ If the subscription stays as **Pending confirmation**, the alarm will fire silently. Always confirm the email.

**Why SNS instead of emailing directly from CloudWatch?**
CloudWatch alarms don't send emails — they *publish to SNS*, and SNS delivers to subscribers. This decoupling means one alarm can notify a team email, trigger a Lambda, send SMS, or all three at once. You're learning the universal AWS alerting pattern.


### Step 4 — Create Alarm 1: $10 Early Warning

1. Go to **CloudWatch → Alarms → Create alarm → Select metric**
2. Browse → **Billing** → **Total Estimated Charge** → check `EstimatedCharges` (USD) → **Select metric**
3. Period: `6 hours`
4. Threshold type: **Static** | Condition: **Greater than** | Value: `10`
5. Missing data treatment: **Treat missing data as missing**
6. Notifications: In alarm → Select existing SNS topic → `billing-alerts`
7. Alarm name: `billing-alarm-10-usd` → **Create alarm**

> ✅ The alarm will show **INSUFFICIENT_DATA** at first. This is **normal** — billing data updates once per day. It is not an error.


### Step 5 — Create Alarm 2: $30 Caution

Repeat Step 4 with only one change:
- Value: `30`
- Alarm name: `billing-alarm-30-usd`


### Step 6 — Create Alarm 3: $50 Take Action

Repeat Step 4 with only one change:
- Value: `50`
- Alarm name: `billing-alarm-50-usd`

> ✅ You should now see **3 alarms** in CloudWatch → Alarms. You're done!


## 💰 Cost Breakdown

| Component | Free Tier? | Cost |
|---|---|---|
| CloudWatch Billing Alarm | ✅ Yes | $0.00 |
| SNS Topic | ✅ Yes | $0.00 |
| SNS Email (first 1,000/month free) | ✅ Yes | $0.00 |
| **Total** | | **$0.00** |

## ⚠️ Common Mistakes

| # | Mistake | Fix |
|---|---|---|
| 1 | SNS topic created in wrong region | Delete and recreate in us-east-1. Alarm and SNS must be in the same region. |
| 2 | Billing alerts not enabled in Account settings | Account → Billing preferences → enable Receive Billing Alerts |
| 3 | SNS subscription not confirmed | Check email for the AWS confirmation link. Unconfirmed = silent failures. |
| 4 | Alarm stuck on INSUFFICIENT_DATA | Normal. Billing data updates once per day. Wait 24 hours. |
| 5 | Single alarm set too high (e.g. $500) | Use graduated thresholds for early warning, caution, and action signals. |

## 🔭 Solutions Architect Lens

**Cost Optimization**
Always ask: what does the monitoring infrastructure itself cost? Here, $0. But if you chose SMS over email, SNS charges $0.75 per 100 messages — a misconfigured alarm flapping 100x/hour would generate real costs fast. Email is the right choice for billing alerts.

**Security**
Using SNS rather than a hardcoded email is best practice. People leave jobs. A SNS topic pointing to a team distribution list is far easier to maintain than updating individual alarm configs every time someone's email changes.

**Scalability of the Pattern**
The pattern you just built — `CloudWatch metric → alarm → SNS → action` — is universal in AWS. You'll use this exact pipeline for CPU alerts, API error rates, database connection counts, and more. SNS can also fan out to Lambda for auto-remediation. This is the backbone of AWS operational monitoring.

**Why Three Alarms?**
Graduated thresholds create a graduated response. `$10 = investigate`, `$30 = something is wrong`, `$50 = stop and act`. In enterprise environments these map to runbooks: the first alarm triggers a Slack message, the third might automatically stop non-production EC2 instances.

## ✅ Completion Checklist

- [ ] Billing alerts enabled in Account preferences
- [ ] Region set to us-east-1
- [ ] SNS topic `billing-alerts` created in us-east-1
- [ ] SNS email subscription confirmed
- [ ] Alarm `billing-alarm-10-usd` created
- [ ] Alarm `billing-alarm-30-usd` created
- [ ] Alarm `billing-alarm-50-usd` created
- [ ] All 3 alarms visible in CloudWatch → Alarms
