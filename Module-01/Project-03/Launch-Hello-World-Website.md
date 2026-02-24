# COM03-AWS100 — Launch a Hello World Website on the Internet

> **Level:** 100 (Introductory)  
> **Service Used:** Amazon EC2  
> **Project Status:** ✅ Completed


## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Services Used](#services-used)
- [Prerequisites](#prerequisites)
- [Implementation Steps](#implementation-steps)
- [Architecture Decision Record (ADR)](#architecture-decision-record-adr)
- [Security Considerations](#security-considerations)
- [Cost Analysis](#cost-analysis)
- [Lessons Learned](#lessons-learned)
- [What I'd Do Differently at Production Scale](#what-id-do-differently-at-production-scale)
- [Cleanup](#cleanup)

## Overview

This project involves deploying a simple "Hello World" website on an Amazon EC2 instance, making it publicly accessible over the internet via HTTP. The goal is to understand the end-to-end process of provisioning a virtual server, bootstrapping it with a web server, and controlling network access through Security Groups.

**Key Learning Objectives:**
- Launch and configure an EC2 instance
- Use User Data scripts to automate software installation at boot
- Configure Security Groups as virtual firewalls
- Understand how public IPs and DNS work in AWS
- Expose a web application to the internet on port 80

## Architecture

```
┌─────────────────────────────────────────────────┐
│                   AWS Cloud                      │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │              VPC (Default)                │   │
│  │                                          │   │
│  │  ┌────────────────────────────────────┐  │   │
│  │  │         Public Subnet              │  │   │
│  │  │                                    │  │   │
│  │  │  ┌──────────────────────────────┐  │  │   │
│  │  │  │   Security Group             │  │  │   │
│  │  │  │   • Port 80 → 0.0.0.0/0     │  │  │   │
│  │  │  │   • Port 22 → My IP Only    │  │  │   │
│  │  │  │                              │  │  │   │
│  │  │  │  ┌────────────────────────┐  │  │  │   │
│  │  │  │  │   EC2 Instance         │  │  │  │   │
│  │  │  │  │   t2.micro             │  │  │  │   │
│  │  │  │  │   Amazon Linux 2023    │  │  │  │   │
│  │  │  │  │   Apache HTTP Server   │  │  │  │   │
│  │  │  │  │   Hello World Page     │  │  │  │   │
│  │  │  │  └────────────────────────┘  │  │  │   │
│  │  │  └──────────────────────────────┘  │  │   │
│  │  └────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────┘   │
│                       │                          │
│              Internet Gateway                    │
└───────────────────────┼──────────────────────────┘
                        │
              Public Internet (Port 80)
                        │
                  Your Browser
```

**Traffic Flow:**  
`User Browser → Internet Gateway → Security Group (Port 80 check) → EC2 Instance → Apache → index.html`

## Services Used

| Service | Purpose |
|---------|---------|
| **Amazon EC2** | Virtual server to host the web application |
| **Security Groups** | Virtual firewall to control inbound/outbound traffic |
| **Amazon VPC** | Default network isolation for the EC2 instance |
| **Internet Gateway** | Enables internet connectivity for the public subnet |
| **Amazon EBS** | Root volume (8GB gp3) attached to the EC2 instance |

## Prerequisites

- AWS Account with appropriate IAM permissions (EC2 full access)
- A key pair (`.pem` file) created or imported in your target region
- Basic familiarity with the AWS Management Console

## Implementation Steps

### Step 1 — Choose a Region

Navigate to the AWS Management Console and select your preferred region (e.g., `us-east-1`).

> **Why it matters:** Region selection affects latency for end users, data residency compliance, and service availability. For learning projects, use the region closest to you.

### Step 2 — Launch an EC2 Instance

1. Navigate to **EC2 → Instances → Launch Instance**
2. Configure the following:

**AMI Selection:**
- Selected: **Amazon Linux 2023**
- Reason: AWS-optimized, includes `aws` CLI pre-installed, frequent security patches

**Instance Type:**
- Selected: **t2.micro**
- Reason: Free Tier eligible (750 hrs/month). Part of the burstable `t` family — earns CPU credits during low usage, suitable for variable/low-traffic workloads

> ⚠️ **Note:** `t2.micro` uses fixed CPU credit model. For consistent high-CPU workloads, use compute-optimized families (e.g., `c5`, `c6i`). `t3.micro` is a better burstable choice for new workloads — it supports unlimited burst mode.

### Step 3 — Configure the Security Group

Created a new Security Group with the following inbound rules:

| Type | Protocol | Port Range | Source | Reason |
|------|----------|------------|--------|--------|
| HTTP | TCP | 80 | 0.0.0.0/0 | Allow public web traffic |
| SSH | TCP | 22 | My IP only | Restrict management access |

**Outbound:** All traffic allowed (default)

> **SA Thinking:** Security Groups are **stateful** — response traffic is automatically allowed without explicit outbound rules. Never open SSH (port 22) to `0.0.0.0/0`; AWS public IPs are scanned by bots within minutes of launch.

### Step 4 — Add Key Pair

- Created a new ED25519 key pair
- Downloaded and stored the `.pem` file securely with permissions set to `400`

```bash
chmod 400 my-key.pem
```

> **Note:** In production, avoid key pairs entirely. Use **AWS Systems Manager Session Manager** for shell access — it eliminates open port 22, requires no key management, and provides full audit logging.

### Step 5 — Configure Storage

- Volume Type: **gp3** (default)
- Size: **8 GB**

> **Note:** `gp3` is the current-gen general purpose SSD. It delivers 3,000 IOPS and 125 MB/s baseline throughput regardless of volume size — more cost-effective than `gp2`, which scales IOPS with size.

### Step 6 — Add User Data (Bootstrap Script)

In **Advanced Details → User Data**, the following script was added to automate Apache installation on first boot:

**For Fedora based distribution**

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<html><h1>Hello World from $(hostname -f)</h1></html>" > /var/www/html/index.html
```

**For Debian based distribution**

```bash
#!/bin/bash
apt update -y
apt install -y apache2
systemctl start apache2
systemctl enable apache2
echo "<html><h1>Hello World from $(hostname -f)</h1></html>" > /var/www/html/index.html
```

**What each line does:**

| Command | Purpose |
|---------|---------|
| `yum update -y` | Apply all OS security patches |
| `yum install -y httpd` | Install Apache web server |
| `systemctl start httpd` | Start Apache immediately |
| `systemctl enable httpd` | Auto-start Apache on every reboot |
| `echo ... > index.html` | Create the Hello World page |

> **SA Thinking:** User Data is your first step toward **infrastructure automation**. This concept scales up to CloudFormation, Ansible, and immutable AMI baking via EC2 Image Builder. `systemctl enable` is critical — without it, Apache won't restart after a reboot.


### Step 7 — Launch and Verify

1. Clicked **Launch Instance**
2. Waited ~2 minutes for instance to reach `running` state and pass both status checks
3. Copied the **Public IPv4 address** from the instance details panel
4. Opened `http://<public-ip>` in a browser

**Result:** ✅ Hello World page loaded successfully

> ⚠️ Use `http://` (not `https://`). There is no SSL certificate configured — that would require ACM + ALB or CloudFront.

## Architecture Decision Record (ADR)

### ADR-001: Instance Type — t2.micro

| Field | Detail |
|-------|--------|
| **Decision** | Use `t2.micro` |
| **Context** | Learning/demo environment, minimal traffic expected |
| **Rationale** | Free Tier eligible; burstable performance sufficient for Hello World |
| **Trade-offs** | Not suitable for sustained CPU workloads; credit exhaustion causes throttling |
| **Alternatives Considered** | `t3.micro` (better credit model), `t3.small` (more RAM) |

### ADR-002: Web Server — Apache (httpd)

| Field | Detail |
|-------|--------|
| **Decision** | Use Apache HTTP Server |
| **Context** | Simple static page serving |
| **Rationale** | Available in Amazon Linux repos, minimal configuration needed for static HTML |
| **Trade-offs** | Heavier than Nginx for high-concurrency static serving |
| **Alternatives Considered** | Nginx (better performance for static), Python HTTP server (lighter for demos) |

### ADR-003: Bootstrap via User Data

| Field | Detail |
|-------|--------|
| **Decision** | Use EC2 User Data for automated server configuration |
| **Context** | Needed Apache installed without manual SSH intervention |
| **Rationale** | Enforces repeatability and automation from the start; no manual steps |
| **Trade-offs** | User Data runs once at launch; re-runs require instance replacement or manual intervention |
| **Alternatives Considered** | Manual SSH setup (not repeatable), AWS Systems Manager Run Command |


## Security Considerations

| Area | Implementation | Production Recommendation |
|------|---------------|--------------------------|
| **SSH Access** | Restricted to My IP only | Use SSM Session Manager — no port 22 needed |
| **HTTP vs HTTPS** | HTTP only (port 80) | Add ACM certificate + ALB or CloudFront for HTTPS |
| **IAM Role** | None attached | Attach IAM role with least-privilege permissions for any AWS API calls |
| **OS Patching** | `yum update -y` at launch | Use AWS Systems Manager Patch Manager for ongoing patching |
| **Key Pair** | `.pem` file stored locally | Eliminate key pairs; use SSM Session Manager |

## Cost Analysis

| Resource | Cost (Free Tier) | Cost (Beyond Free Tier) |
|----------|-----------------|------------------------|
| `t2.micro` EC2 | Free (750 hrs/month, 12 months) | ~$0.0116/hr (~$8.35/mo) |
| EBS gp3 (8 GB) | Free (30 GB/month, 12 months) | ~$0.08/GB/month (~$0.64/mo) |
| Data Transfer Out | Free (first 100 GB/month) | $0.09/GB after free tier |
| Public IPv4 | Free while instance running | $0.005/hr if Elastic IP unattached |
| **Estimated Total** | **$0.00** | **~$9/month** |

> **SA Cost Tip:** Always **terminate** (not just stop) EC2 instances after learning labs. A stopped instance still incurs EBS storage costs. An Elastic IP that is allocated but not attached to a running instance also incurs charges.


## Lessons Learned

1. **Security Groups are your first line of defense** — Understanding stateful vs. stateless firewalls is fundamental to every AWS architecture.

2. **User Data = the seed of automation** — Moving from manual SSH setup to declarative bootstrap scripts is a mindset shift that scales to IaC tools like CloudFormation and Terraform.

3. **`systemctl enable` is not optional** — Starting a service is not the same as enabling it. Always enable services you need to survive a reboot.

4. **Public IPs are ephemeral** — Stop and start an instance, and the public IP changes. Use Elastic IPs or (better) Route 53 with a domain for stable addressing.

5. **AMI choice has downstream implications** — Package managers (`yum` vs `apt`), default users (`ec2-user` vs `ubuntu`), and pre-installed tools differ. Know your AMI.


## What I'd Do Differently at Production Scale

| Concern | Learning Setup | Production Setup |
|---------|---------------|-----------------|
| **High Availability** | Single EC2 in one AZ | Auto Scaling Group across 2+ AZs |
| **Load Balancing** | Direct EC2 public IP | Application Load Balancer (ALB) |
| **HTTPS / TLS** | No SSL | ACM certificate on ALB or CloudFront |
| **DNS** | Raw IP address | Route 53 with custom domain |
| **Server Access** | SSH with key pair | SSM Session Manager, no port 22 |
| **Web Server** | EC2 + Apache | S3 + CloudFront (for purely static sites) |
| **Monitoring** | None | CloudWatch metrics + alarms |
| **Infrastructure** | Manual console clicks | CloudFormation / Terraform |
| **Security** | Basic SG rules | WAF, Shield, VPC with private subnets |


## Cleanup

To avoid ongoing charges, the following resources were terminated after project completion:

- [x] EC2 Instance — **Terminated**
- [x] Security Group — **Deleted**
- [x] Key Pair — **Deleted from AWS** (local `.pem` also removed)
- [x] EBS Volume — **Automatically deleted** (Delete on Termination was enabled)

> ✅ No orphaned resources remaining.

## References

- [Amazon EC2 Documentation](https://docs.aws.amazon.com/ec2/)
- [EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/)
- [Security Groups for EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html)
- [EC2 User Data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [Original Project Repository](https://github.com/munirmahmud/AWS-Solution-Architect-Projects/)
