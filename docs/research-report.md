# AWS Disaster Recovery & Resilience: Comprehensive Project Report

## Executive Summary

AWS provides a layered ecosystem of disaster recovery (DR) and resilience services that address the full spectrum of business continuity requirements — from low-cost backup-only approaches for non-critical workloads to zero-downtime active-active architectures for mission-critical global applications. This report covers five core AWS services — AWS Resilience Hub, Amazon Application Recovery Controller (ARC), AWS Elastic Disaster Recovery (DRS), and AWS Backup — and maps them to the four established DR strategies: Backup & Restore, Pilot Light, Warm Standby, and Active-Active (Multi-Site).

The fundamental trade-off underlying all DR decisions is cost versus recovery speed: "The tighter your RTO and RPO requirements, the more expensive and complex your DR strategy becomes. A business must balance the cost of downtime against the cost of maintaining DR infrastructure." [1](https://kindatechnical.com/aws-certification-cloud-practitioner/aws-infrastructure-for-disaster-recovery-strategies.html)

---

## The Four AWS DR Strategies

AWS organizes disaster recovery into four tiers, arranged from lowest cost and slowest recovery to highest cost and fastest recovery. These strategies are defined in the AWS Well-Architected Reliability Pillar (REL13-BP02): "Define a disaster recovery (DR) strategy that meets your workload's recovery objectives. Choose a strategy such as backup and restore, standby (active/passive), or active/active." [2](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html)

| Strategy | RTO | RPO | Cost | DR Region State |
|----------|-----|-----|------|-----------------|
| Backup & Restore | < 24 hours | Hours | $ | No running resources |
| Pilot Light | Tens of minutes | Minutes | $$ | Core data tier running |
| Warm Standby | Minutes | Seconds–Minutes | $$$ | Scaled-down full stack |
| Active-Active | Near zero | Near zero | $$$$ | Full production |

![Comparison of RTO and RPO across AWS DR strategies (log scale). Backup & Restore offers hours of recovery time, while DRS-based strategies achieve sub-second RPO and minutes of RTO.](dr_strategy_rto_rpo_comparison.png)

*Figure 1: Comparison of RTO and RPO across AWS DR strategies (log scale). Backup & Restore offers hours of recovery time, while DRS-based strategies achieve sub-second RPO and minutes of RTO.*

### Backup & Restore

The simplest strategy: automated backups replicated to a DR Region via AWS Backup or S3 Cross-Region Replication, with infrastructure rebuilt from IaC templates on demand. "Backup and restore is a suitable approach for mitigating against data loss or corruption. This approach can also be used to mitigate against a regional disaster by replicating data to other AWS Regions." [3](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html)

### Pilot Light

Core infrastructure (database replicas) stays running; compute is pre-configured but offline. "Provision a copy of your core workload infrastructure in the recovery Region. Replicate your data into the recovery Region and create backups of it there. Resources required to support data replication and backup, such as databases and object storage, are always on. Other elements such as application servers or serverless compute are not deployed, but can be created when needed with the necessary configuration and application code." [2](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html)

### Warm Standby

A scaled-down but fully functional replica runs continuously. "Maintain a scaled-down but fully functional version of your workload always running in the recovery Region. Business-critical systems are fully duplicated and are always on, but with a scaled down fleet. Data is replicated and live in the recovery Region. When the time comes for recovery, the system is scaled up quickly to handle the production load." [2](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html)

The key distinction: "pilot light cannot process requests without additional action taken first, while warm standby can handle traffic (at reduced capacity levels) immediately." [2](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html)

### Active-Active (Multi-Site)

Full production infrastructure runs simultaneously in 2+ Regions, both serving live traffic. This provides near-zero RTO and RPO but approximately doubles infrastructure costs.

---

## AWS Resilience Hub (Next-Generation)

AWS launched the next generation of Resilience Hub on May 28, 2026, introducing a redesigned service with AI-powered capabilities. The service provides continuous resilience validation, evaluating RTO and RPO targets and identifying infrastructure issues pre-emptively. [4](https://aws.amazon.com/blogs/mt/maximizing-multi-region-resilience-with-aws-resilience-hub)

### Key Next-Gen Features

**AI-Powered Failure Mode Analysis**: A GenAI-driven assessment engine evaluates services across five critical failure modes — Single Points of Failure, Excessive Load, Excessive Latency, Misconfiguration, and Shared Fate. [5](https://awscloud.com/resilience-hub/pricing/)

**Dependency Discovery via DNS Query Logs**: Automated dependency assessment discovers service dependencies by analyzing actual runtime communication patterns, including DNS query logs. This reveals hidden dependencies that static analysis cannot detect. Priced at $10 per service per month as an add-on. [5](https://awscloud.com/resilience-hub/pricing/)

**New Application Model (Services)**: The concept of "applications" is replaced with "services" — logical groups of AWS resources that operate as a unit. Services can be described via CloudFormation Stacks, resource tags, Terraform state files, or EKS workloads. [5](https://awscloud.com/resilience-hub/pricing/)

**Modular Resilience Policies**: Organizations define standardized resilience tiers (e.g., "Tier 1 Critical," "Tier 2 Standard") and apply them consistently across services.

**Organization-Wide Reporting**: Multi-account, centralized assessment across an entire AWS Organization from a central management account.

### Core Capabilities

- **Continuous Validation**: Integrates with CI/CD pipelines via APIs and AWS CodePipeline to validate every build before production release. [6](https://aws.amazon.com/jp/blogs/architecture/continually-assessing-application-resilience-with-aws-resilience-hub-and-aws-codepipeline/)
- **RTO/RPO Assessment**: Evaluates across 4 disruption types — application, infrastructure, AZ, and Region disruption.
- **Resilience Testing**: Integrates with AWS Fault Injection Service (FIS) for chaos engineering experiments. [7](https://aws.amazon.com/blogs/mt/how-to-use-resilience-hubs-fault-injection-experiments-to-test-applications-resilience)
- **Drift Detection**: Scheduled assessments detect configuration drift that degrades resilience posture.

### Pricing

Base assessment: $15 per service per month. Dependency discovery add-on: $10 per service per month.

---

## Amazon Application Recovery Controller (ARC)

ARC helps prepare for and execute faster recovery for applications running on AWS. It provides capabilities for both single-AZ and multi-Region impairments. [8](https://aws.amazon.com/application-recovery-controller/)

### Core Components

**Routing Controls**: On/off switches that control Route 53 health checks to govern DNS-based traffic routing. "A routing control is a simple on-off switch that controls a health check. Typically, you change the state to redirect traffic, and that state change moves the traffic to go to a particular endpoint for an entire application stack." [9](https://docs.aws.amazon.com/r53recovery/latest/dg/routing-control.about.html)

**Clusters**: A set of five redundant Regional endpoints ensuring high availability for routing control operations. [10](https://docs.aws.amazon.com/r53recovery/latest/dg/getting-started-cli-routing-config.html)

**Zonal Shift**: Quickly isolates impaired AZs by temporarily shifting traffic away from them to healthy AZs. [11](https://docs.aws.amazon.com/r53recovery/latest/dg/what-is-route53-recovery.html)

**Zonal Autoshift**: AWS proactively monitors internal telemetry and automatically shifts traffic away from impaired AZs — no operator intervention required. Operates on the principle of static stability. [12](https://aws.amazon.com/de/blogs/networking-and-content-delivery/configuring-amazon-application-recovery-controller-zonal-autoshift-observer-notifications/)

**Region Switch**: Orchestrates full-stack, multi-Region recovery with execution blocks, cross-account support, and automatic triggers based on CloudWatch alarms. [13](https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch.html)

### Pricing

![Amazon Application Recovery Controller monthly pricing by feature. Zonal shift/autoshift is free, while routing control clusters cost ~$1,800/month for 5-Region distributed data plane reliability.](arc_pricing_comparison.png)

*Figure 2: Amazon Application Recovery Controller monthly pricing by feature. Zonal shift/autoshift is free, while routing control clusters cost ~$1,800/month for 5-Region distributed data plane reliability.*

| Feature | Price |
|---------|-------|
| Zonal Shift / Autoshift | Free |
| Routing Control Cluster | $2.50/hour (~$1,800/mo) |
| Region Switch Plan | $70/plan/month |
| Routing Control Health Check | $0.75/month |

---

## AWS Elastic Disaster Recovery (DRS)

AWS DRS performs continuous block-level replication of all attached volumes — including the OS, databases, applications, and files — to a low-cost staging area in the target AWS Region. [14](https://docs.aws.amazon.com/prescriptive-guidance/latest/backup-recovery/on-prem-dr-to-aws.html)

### How It Works

1. **Replication Agent**: Lightweight agent on source servers recognizes write operations and immediately copies changed blocks to the staging area. All data is compressed and encrypted in transit.
2. **Staging Area**: Low-cost t3.small EC2 replication servers (each handles up to 15 staging disks) plus EBS volumes mirroring source disks.
3. **Point-in-Time Recovery**: Default schedule retains snapshots at minute (every 10 min for 1 hour), hourly (for 24 hours), and daily (for 7 days, configurable to 365 days) granularity.
4. **Recovery Launch**: "If there is a disaster, you can instruct Elastic Disaster Recovery to quickly launch thousands of your machines in their fully provisioned state within minutes." [14](https://docs.aws.amazon.com/prescriptive-guidance/latest/backup-recovery/on-prem-dr-to-aws.html)

### RPO & RTO

| Metric | Value |
|--------|-------|
| RPO | Sub-second (with "Use most recent data" option) |
| RTO | 5–20 minutes (depending on server conversion time) |

### Failover & Failback

**Failover**: Select servers → choose recovery point → DRS creates cloned volumes → conversion server makes them AWS-bootable → recovery instances launch → redirect traffic.

**Failback**: After primary is restored, DRS initiates reverse replication back to the source. A Failback Client is booted on the source server to receive replicated data. [15](https://docs.aws.amazon.com/drs/latest/userguide/failback-performing.html)

### Supported Sources

On-premises physical servers, VMware/Hyper-V virtual machines, other cloud providers (Azure, GCP), and AWS EC2 instances (for cross-region/cross-account DR).

### Pricing

| Component | Cost |
|-----------|------|
| Replication per server | $0.028/hour (~$20/mo) |
| Staging area EBS | Standard EBS pricing |
| Recovery instances | EC2/EBS charges during drill/recovery only |
| Conversion server | < $0.05 per recovery (runs ~2 min) |

---

## AWS Backup

AWS Backup is a fully managed service that centralizes and automates data protection across 20+ AWS services and hybrid workloads. [16](https://aws.amazon.com/backup/features/)

### Key Features

**Backup Plans**: Centralized policies with customizable schedules (as frequent as hourly), retention rules, lifecycle management (warm → cold storage), and tag-based resource assignment.

**Cross-Region & Cross-Account Copy**: Fundamental for DR — backups can be automatically copied across Regions and accounts as part of scheduled plans. "You can copy backups across different AWS Regions and accounts from a central console to meet compliance and disaster recovery needs." [16](https://aws.amazon.com/backup/features/)

**Vault Lock (WORM)**: Enforces immutability — once locked, backups cannot be deleted or altered, even by root accounts. Essential for ransomware protection and regulatory compliance.

**Logically Air-Gapped Vaults**: Isolated backup vaults shared via AWS RAM that provide defense against compromised accounts.

**Audit Manager**: Built-in compliance frameworks (e.g., backup frequency, cross-region copy, retention) with automated evaluation.

### Supported Resources (20+ Services)

| Category | Services |
|----------|----------|
| Storage | S3, EBS, EFS, FSx (4 types), Storage Gateway |
| Compute | EC2, SAP HANA on EC2 |
| Databases | RDS, Aurora, DynamoDB, Neptune, DocumentDB, Timestream, Redshift |
| Containers | EKS (cluster state + persistent volumes) |
| Application Stacks | CloudFormation (full stack including IAM, VPC) |
| Hybrid | VMware (on-premises, VMware Cloud on AWS) |

### Pricing

![AWS Backup monthly cost breakdown for a typical scenario with EFS backups, cross-region copies, restores, and audit evaluations totaling $68.97/month.](aws_backup_cost_breakdown.png)

*Figure 3: AWS Backup monthly cost breakdown for a typical scenario with EFS backups, cross-region copies, restores, and audit evaluations totaling $68.97/month.*

Approximate pricing: ~$0.05/GB-month for warm storage, ~$0.01/GB-month for cold storage, plus data transfer for cross-region copies.

---

## Service-to-Strategy Mapping

| Strategy | Primary Services | Supporting Services |
|----------|-----------------|---------------------|
| Backup & Restore | AWS Backup, S3 CRR | CloudFormation/CDK, Resilience Hub |
| Pilot Light | AWS DRS, RDS Read Replicas | ARC (routing control), Resilience Hub |
| Warm Standby | AWS DRS, ARC, Aurora Global DB | ASG, Resilience Hub, FIS |
| Active-Active | ARC, DynamoDB Global Tables, Global Accelerator | Aurora Global DB, Resilience Hub |

---

## Decision Framework

### When to Choose Each Strategy

| Criterion | Backup & Restore | Pilot Light | Warm Standby | Active-Active |
|-----------|-----------------|-------------|--------------|---------------|
| Acceptable RTO | Hours | Tens of min | Minutes | Near zero |
| Acceptable RPO | Hours | Minutes | Seconds | Near zero |
| Monthly budget | $50–200 | $200–800 | $1K–5K | $5K–20K+ |
| Operational complexity | Low | Medium | Medium-High | High |
| Team maturity | Any | Some DR experience | Dedicated SRE | Experienced SRE + chaos testing |
| Compliance needs | Basic data retention | SOC 2 | HIPAA/PCI-DSS | Financial/regulatory zero-downtime |
| Best for | Dev/test, non-critical | Business-critical apps | Mission-critical | Global, zero-tolerance |

### Key Design Principles

1. **Assess with Resilience Hub** — Start every DR strategy by defining resilience policies and running assessments against your actual architecture.
2. **Layer services** — Use AWS Backup as a baseline for ALL strategies (compliance layer), then add DRS/ARC for faster recovery.
3. **Test regularly** — Use FIS experiments and DRS drills to validate recovery works before you need it.
4. **Automate failover** — Move from manual to automated failover as maturity increases (ARC Region Switch + Step Functions).

---

## Conclusion

The AWS resilience ecosystem provides a complete toolkit spanning from basic data protection (AWS Backup) through server-level replication (DRS), traffic-level failover orchestration (ARC), and continuous governance (Resilience Hub). The optimal strategy depends on balancing business requirements (RTO/RPO), budget, and operational maturity. Most production organizations benefit from a tiered approach — using Backup & Restore for non-critical workloads, Pilot Light or Warm Standby for business-critical systems, and Active-Active only for zero-tolerance global applications.

## References

\[1\] https://kindatechnical.com/aws-certification-cloud-practitioner/aws-infrastructure-for-disaster-recovery-strategies.html

\[2\] https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html

\[3\] https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html

\[4\] https://aws.amazon.com/blogs/mt/maximizing-multi-region-resilience-with-aws-resilience-hub

\[5\] https://awscloud.com/resilience-hub/pricing/

\[6\] https://aws.amazon.com/jp/blogs/architecture/continually-assessing-application-resilience-with-aws-resilience-hub-and-aws-codepipeline/

\[7\] https://aws.amazon.com/blogs/mt/how-to-use-resilience-hubs-fault-injection-experiments-to-test-applications-resilience

\[8\] https://aws.amazon.com/application-recovery-controller/

\[9\] https://docs.aws.amazon.com/r53recovery/latest/dg/routing-control.about.html

\[10\] https://docs.aws.amazon.com/r53recovery/latest/dg/getting-started-cli-routing-config.html

\[11\] https://docs.aws.amazon.com/r53recovery/latest/dg/what-is-route53-recovery.html

\[12\] https://aws.amazon.com/de/blogs/networking-and-content-delivery/configuring-amazon-application-recovery-controller-zonal-autoshift-observer-notifications/

\[13\] https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch.html

\[14\] https://docs.aws.amazon.com/prescriptive-guidance/latest/backup-recovery/on-prem-dr-to-aws.html

\[15\] https://docs.aws.amazon.com/drs/latest/userguide/failback-performing.html

\[16\] https://aws.amazon.com/backup/features/
