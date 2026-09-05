# Multi-Tier Well-Architected AWS Infrastructure

A three-tier web application architecture built on AWS, demonstrating proper network segmentation, least-privilege security group design, and traffic isolation between public and private resources.

> **Status:** Built and validated manually via the AWS Console as a design/learning phase. A full Terraform (Infrastructure as Code) rebuild with enhancements is in progress.

## Architecture Overview

```
                                   Internet
                                      │
                                      ▼
                          ┌───────────────────────┐
                          │  Internet Gateway      │
                          └───────────┬───────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    │           Public Subnets            │
                    │        (2 Availability Zones)       │
                    │                                      │
                    │   ┌──────────────────────────────┐  │
                    │   │  Application Load Balancer     │  │
                    │   │  (sg-alb: 80 from 0.0.0.0/0)    │  │
                    │   └───────────────┬────────────────┘  │
                    │                   │                    │
                    │   ┌───────────────┴────────────────┐  │
                    │   │        NAT Gateway              │  │
                    │   └───────────────┬────────────────┘  │
                    └────────────────────┼────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │           Private Subnets                │
                    │        (2 Availability Zones)            │
                    │                                           │
                    │   ┌───────────────────────────────────┐  │
                    │   │   App Tier — EC2 (Amazon Linux)     │  │
                    │   │   (sg-app: 80 from sg-alb only)      │  │
                    │   └───────────────┬───────────────────┘  │
                    │                   │                        │
                    │   ┌───────────────┴───────────────────┐  │
                    │   │   Database Tier — RDS (MySQL)        │  │
                    │   │   (sg-db: 3306 from sg-app only)     │  │
                    │   └───────────────────────────────────┘  │
                    └───────────────────────────────────────────┘
```

## What This Project Demonstrates

- **VPC design from scratch** — custom CIDR block, 4 subnets (2 public, 2 private) spread across 2 Availability Zones for redundancy
- **Correct public/private routing** — public subnets route through an Internet Gateway; private subnets route outbound-only traffic through a NAT Gateway, with no direct internet exposure
- **Security-group-to-security-group access chaining** — each tier only accepts traffic from the tier immediately in front of it (ALB → App → Database), using security group references as the source instead of IP ranges or CIDR blocks
- **No public database or app servers** — the only resource reachable from the public internet is the Application Load Balancer; the app tier and database both sit in private subnets with no public IPs
- **SSH-free instance access** — used AWS Systems Manager Session Manager (via an IAM instance role) instead of key pairs or open port 22, removing an entire class of exposure
- **End-to-end connectivity validation** — confirmed the full request path (internet → ALB → target group → private app instance → private database) actually works, not just that resources exist

## Tech Stack

| Layer | Service |
|---|---|
| Networking | VPC, Subnets, Internet Gateway, NAT Gateway, Route Tables |
| Compute | EC2 (Amazon Linux 2023) |
| Database | RDS (MySQL) |
| Load Balancing | Application Load Balancer + Target Groups |
| Access | IAM Roles, Systems Manager Session Manager |
| Security | Security Groups (tier-chained, least-privilege) |

## Key Design Decisions

**Why a custom CIDR block instead of defaults?**
Chose a `/16` range to leave ample room for future subnetting, and to build the habit of deliberately sizing a network rather than accepting a default.

**Why security-group-to-security-group rules instead of IP ranges?**
An IP-based rule only knows about an address; a security-group-based rule knows about *identity*. If the app tier's instances change or scale, the database's access rule doesn't need to be touched — it's already scoped to "anything using `sg-app`," which is a more resilient and auditable pattern for how access actually works in a growing environment.

**Why Single-AZ RDS instead of Multi-AZ?**
Multi-AZ RDS runs a synchronously replicated standby instance in a second AZ for automatic failover — valuable for production, but it roughly doubles the database cost. Since this is a validation/learning environment rather than a system serving live traffic, Single-AZ was the appropriate tradeoff. In a production deployment handling real user traffic, Multi-AZ would be the correct choice.

**Why RDS (MySQL) instead of Aurora?**
Aurora is AWS's own MySQL/PostgreSQL-compatible engine, optimized for fast failover and cheap read scaling — valuable for high-availability or read-heavy workloads. Standard RDS was the better fit here because it has a genuine low-cost tier, and it teaches transferable, engine-agnostic database skills (subnet groups, standard connection handling) rather than AWS-specific internals that weren't the focus of this project.

**Why Session Manager instead of SSH?**
Removes the need for a bastion host, an open port 22, or key pair management entirely. Access is controlled purely through IAM, which is both more secure and a better demonstration of least-privilege identity-based access control.

## Debugging Notes (Real Issues Hit During Build)

- **Stale NAT Gateway route:** After deleting and recreating a NAT Gateway, the private route table's `0.0.0.0/0` route still pointed at the old (now-deleted) NAT Gateway ID. Routes don't auto-update to a replacement resource — this had to be manually repointed.
- **AL2023 package naming:** `yum install mysql` fails on Amazon Linux 2023, since the plain MySQL client package was removed. Resolved by installing `mariadb1011-client-utils`, which provides a wire-protocol-compatible `mysql` client.
- **IAM Role for SSM Agent:** Forgot to add an IAM Role for the SSM Agent on the EC2 Instance, which is necessary for connecting via SSM, despite having full Admin privileges on user account.
- **SSM agent didn't check in on first boot:** An EC2 instance launched before its NAT Gateway/route was fully live never successfully registered with Systems Manager. A reboot after the network path was confirmed live resolved it — the agent doesn't retry indefinitely on its own.

## Cost Management

This project was built and torn down across multiple short sessions rather than left running continuously, since key resources (NAT Gateway, ALB, RDS, EC2) all incur hourly charges outside the AWS free tier. Practice followed:

- VPC, subnets, route tables, and security groups (no cost) were left standing between sessions
- NAT Gateway, EC2, RDS, and ALB were created only for active work windows and fully deleted at the end of each session (including releasing Elastic IPs and skipping RDS snapshot/backup retention)
- An AWS Budget alert was configured to flag spend early rather than after the fact


## Author's Note

This build was done manually through the AWS Console first, intentionally, to build a real understanding of subnet routing, security group chaining, and tier isolation before automating any of it with Terraform. The Terraform version of this same architecture is the actual portfolio deliverable and will be linked here once complete.
