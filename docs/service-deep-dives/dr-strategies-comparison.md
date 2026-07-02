# AWS Disaster Recovery Strategies: Comprehensive Comparison & Decision Framework

## Introduction

AWS defines four disaster recovery (DR) strategies that represent a spectrum of trade-offs between cost, complexity, and recovery speed. These strategies are governed by two critical metrics:

- **RTO (Recovery Time Objective):** The maximum acceptable time from when a disaster occurs to when the system is fully operational again.
- **RPO (Recovery Point Objective):** The maximum acceptable amount of data loss, measured in time.

As the AWS Well-Architected Framework states: "Disaster recovery strategies available to you within AWS can be broadly categorized into four approaches, ranging from the low cost and low complexity of making backups to more complex strategies using multiple active Regions." [Disaster Recovery of Workloads on AWS: Recovery in the Cloud](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html){evidence: "Disaster recovery strategies available to you within AWS can be broadly categorized into four approaches, ranging from the low cost and low complexity of making backups to more complex strategies using multiple active Regions."}

The fundamental principle underlying all DR decisions is this: "The tighter your RTO and RPO requirements, the more expensive and complex your DR strategy becomes. A business must balance the cost of downtime against the cost of maintaining DR infrastructure." [AWS Infrastructure for Disaster Recovery Strategies](https://kindatechnical.com/aws-certification-cloud-practitioner/aws-infrastructure-for-disaster-recovery-strategies.html){evidence: "The tighter your RTO and RPO requirements, the more expensive and complex your DR strategy becomes. A business must balance the cost of downtime against the cost of maintaining DR infrastructure."}

---

## Strategy 1: Backup & Restore

### Architecture

Backup & Restore is the simplest DR strategy. During normal operations, automated backups of data (database snapshots, EBS snapshots, AMIs, configuration data) are stored in durable storage — typically Amazon S3 in another Region. When disaster strikes, new infrastructure is provisioned from scratch using Infrastructure as Code (IaC) and data is restored from backups.

The recovery Region has **no running resources** during normal operations — only backup storage exists there. [AWS Prescriptive Guidance – Defining Your DR Strategy](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-database-disaster-recovery/defining.html){evidence: "Provision all required application resources in the DR Region and restore the database from a copied snapshot."}

### RTO & RPO

| Metric | Target |
|--------|--------|
| **RTO** | Less than 24 hours |
| **RPO** | Hours (depends on backup frequency) |

The AWS Well-Architected Reliability Pillar specifies: "Back up your data and applications into the recovery Region. Using automated or continuous backups will permit point in time recovery (PITR), which can lower RPO to as low as 5 minutes in some cases. In the event of a disaster, you will deploy your infrastructure (using infrastructure as code to reduce RTO), deploy your code, and restore the backed-up data to recover from a disaster in the recovery Region." [REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "Back up your data and applications into the recovery Region. Using automated or continuous backups will permit point in time recovery (PITR), which can lower RPO to as low as 5 minutes in some cases. In the event of a disaster, you will deploy your infrastructure (using infrastructure as code to reduce RTO), deploy your code, and restore the backed-up data to recover from a disaster in the recovery Region."}

### Cost

**$ (Lowest)** — You only pay for backup storage during normal operations. No running compute or database instances in the recovery Region.

"This results in longer downtimes and greater loss of data between when the disaster event occurs and recovery. However, backup and restore can still be the right strategy for your workload because it is the easiest and least expensive strategy to implement." [DR Architecture on AWS, Part II: Backup and Restore](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-ii-backup-and-restore-with-rapid-recovery){evidence: "This results in longer downtimes and greater loss of data between when the disaster event occurs and recovery. However, backup and restore can still be the right strategy for your workload because it is the easiest and least expensive strategy to implement."}

### When to Use

- Non-critical workloads and development environments
- Workloads where several hours of downtime is acceptable
- Organizations with tight budgets that need basic disaster protection
- Systems where the cost of downtime is lower than the cost of maintaining standby infrastructure

---

## Strategy 2: Pilot Light

### Architecture

Pilot Light keeps the most critical core components — primarily databases — running continuously in the DR Region, while application servers and other non-core infrastructure remain pre-configured but not deployed. The "pilot light" analogy comes from gas heating: a small flame stays burning so the full system can ignite quickly.

"Provision a copy of your core workload infrastructure in the recovery Region. Replicate your data into the recovery Region and create backups of it there. Resources required to support data replication and backup, such as databases and object storage, are always on. Other elements such as application servers or serverless compute are not deployed, but can be created when needed with the necessary configuration and application code." [REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "Provision a copy of your core workload infrastructure in the recovery Region. Replicate your data into the recovery Region and create backups of it there. Resources required to support data replication and backup, such as databases and object storage, are always on. Other elements such as application servers or serverless compute are not deployed, but can be created when needed with the necessary configuration and application code."}

### RTO & RPO

| Metric | Target |
|--------|--------|
| **RTO** | Tens of minutes |
| **RPO** | Minutes |

The AWS Prescriptive Guidance confirms: for Pilot Light, RPO is "Tens of minutes" and RTO is "Tens of minutes." [AWS Prescriptive Guidance – Defining Your DR Strategy](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-database-disaster-recovery/defining.html){evidence: "Pilot light — Tens of minutes — Tens of minutes — Provision a copy of your application infrastructure and switch the resources in the application stack off."}

### Cost

**$$ (Low-Medium)** — You pay for minimal database replication (e.g., RDS Cross-Region Read Replicas) and AMI/configuration storage, but not for idle compute instances.

### When to Use

- Business-critical workloads needing faster recovery than Backup & Restore
- Organizations that cannot justify the cost of a full warm standby
- Core databases must remain synchronized but compute can be provisioned on-demand
- Applications where tens-of-minutes recovery time is acceptable

### Key Distinction from Warm Standby

"The distinction is that pilot light cannot process requests without additional action taken first, while warm standby can handle traffic (at reduced capacity levels) immediately. Pilot light will require you to turn on servers, possibly deploy additional (non-core) infrastructure, and scale up, while warm standby only requires you to scale up (everything is already deployed and running)." [REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "The distinction is that pilot light cannot process requests without additional action taken first, while warm standby can handle traffic (at reduced capacity levels) immediately. Pilot light will require you to turn on servers, possibly deploy additional (non-core) infrastructure, and scale up, while warm standby only requires you to scale up (everything is already deployed and running)."}

---

## Strategy 3: Warm Standby

### Architecture

Warm Standby maintains a **scaled-down but fully functional** copy of the production environment running continuously in the DR Region. Unlike Pilot Light where application servers are stopped, Warm Standby keeps everything running at minimum capacity. Data is replicated in real time. When disaster strikes, the environment is scaled up to full production capacity.

"Maintain a scaled-down but fully functional version of your workload always running in the recovery Region. Business-critical systems are fully duplicated and are always on, but with a scaled down fleet. Data is replicated and live in the recovery Region. When the time comes for recovery, the system is scaled up quickly to handle the production load." [REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "Maintain a scaled-down but fully functional version of your workload always running in the recovery Region. Business-critical systems are fully duplicated and are always on, but with a scaled down fleet. Data is replicated and live in the recovery Region. When the time comes for recovery, the system is scaled up quickly to handle the production load."}

When fully scaled, this strategy is known as **hot standby**. [REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "The more scaled-up the warm standby is, the lower RTO and control plane reliance will be. When fully scales this is known as hot standby."}

### RTO & RPO

| Metric | Target |
|--------|--------|
| **RTO** | Minutes |
| **RPO** | Seconds to minutes (real-time replication) |

### Cost

**$$$ (Moderate-High)** — You pay for running a reduced environment 24/7, including smaller EC2 instance types, fewer instances, and replicated databases.

The AWS Prescriptive Guidance categorizes Warm Standby cost as "High" because you must "Provision a copy of the entire application infrastructure in the DR Region, but keep the copy scaled down compared with the primary Region." [AWS Prescriptive Guidance – Defining Your DR Strategy](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-database-disaster-recovery/defining.html){evidence: "Provision a copy of the entire application infrastructure in the DR Region, but keep the copy scaled down compared with the primary Region. The DR Region will be able to accept traffic at a smaller volume compared with the primary Region."}

### When to Use

- Important workloads that need recovery in minutes
- Systems that justify the ongoing cost of a running standby environment
- Applications where immediate (reduced-capacity) traffic handling is needed during failover
- Organizations with moderate-to-aggressive RTO requirements

---

## Strategy 4: Active-Active (Multi-Site)

### Architecture

The most robust strategy runs the full production workload simultaneously in two or more AWS Regions. All environments handle live traffic at all times using routing policies (geolocation, latency-based, or weighted). Data is replicated bidirectionally between Regions.

"Your workload is deployed to, and actively serving traffic from, multiple AWS Regions. This strategy requires you to synchronize data across Regions. Possible conflicts caused by writes to the same record in two different regional replicas must be avoided or handled, which can be complex." [REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "Your workload is deployed to, and actively serving traffic from, multiple AWS Regions. This strategy requires you to synchronize data across Regions. Possible conflicts caused by writes to the same record in two different regional replicas must be avoided or handled, which can be complex."}

The architecture supports three write patterns:

1. **Read-local/Write-local** — All reads and writes served from the local Region; "last writer wins" reconciliation handles conflicts.
2. **Read-local/Write-global** — One Region designated as the global write Region; others forward writes.
3. **Read-local/Write-partitioned** — Each record assigned a "home Region" for writes based on partition key or origin.

[DR Architecture on AWS, Part IV: Multi-site Active/Active](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iv-multi-site-active-active){evidence: "Each Region hosts a highly available, multi-Availability Zone (AZ) workload stack. In each Region, data is replicated live between the data stores and also backed up. This protects against disasters that include data deletion or corruption, since the data backup can be restored to the last known good state."}

### RTO & RPO

| Metric | Target |
|--------|--------|
| **RTO** | Near zero (potentially zero) |
| **RPO** | Near zero |

The AWS Prescriptive Guidance states the RTO for Multi-site active/active is "Zero or near zero" and RPO is "Near zero." [AWS Prescriptive Guidance – Defining Your DR Strategy](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-database-disaster-recovery/defining.html){evidence: "Multi-site or active/active — Near zero — Zero or near zero — Provision a complete copy of your infrastructure into the DR Region."}

### Cost

**$$$$ (Highest)** — Full production capacity running in multiple Regions simultaneously, plus the operational complexity of managing data synchronization and conflict resolution.

"This must be weighed against the potential cost and complexity of operating active stacks in multiple sites." [DR Architecture on AWS, Part IV: Multi-site Active/Active](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iv-multi-site-active-active){evidence: "the multi-site active/active strategy will give you the lowest RTO (recovery time objective) and RPO (recovery point objective). However, this must be weighed against the potential cost and complexity of operating active stacks in multiple sites."}

### When to Use

- Mission-critical workloads where any downtime results in significant business impact
- Financial trading platforms, healthcare systems, global e-commerce
- Applications requiring low-latency access from globally distributed users
- Workloads with regulatory requirements for zero downtime

"Multi-site active/active is the most operationally complex of the DR strategies, and should only be selected when business requirements necessitate it." [REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "Multi-site active/active is the most operationally complex of the DR strategies, and should only be selected when business requirements necessitate it."}

---

## Comparative Summary Table

| Strategy | RTO | RPO | Cost | Complexity | DR Region State |
|----------|-----|-----|------|------------|-----------------|
| **Backup & Restore** | Hours (≤24h) | Hours | $ (Lowest) | Low | No running resources |
| **Pilot Light** | Tens of minutes | Minutes | $$ (Low) | Medium | Core data infra only |
| **Warm Standby** | Minutes | Seconds–Minutes | $$$ (Medium) | Medium-High | Scaled-down full stack |
| **Active-Active** | Near zero | Near zero | $$$$ (Highest) | High | Full production |

[AWS Infrastructure for Disaster Recovery Strategies](https://kindatechnical.com/aws-certification-cloud-practitioner/aws-infrastructure-for-disaster-recovery-strategies.html){evidence: "Strategy: Backup and Restore | RTO: Hours to days | RPO: Hours | Cost: Lowest | Complexity: Low. Pilot Light | RTO: Minutes to hours | RPO: Minutes | Cost: Low | Complexity: Medium. Warm Standby | RTO: Minutes | RPO: Seconds to minutes | Cost: Medium | Complexity: Medium-High. Active-Active | RTO: Near zero | RPO: Near zero | Cost: Highest | Complexity: High."}

---

## Decision Criteria

### Business Criticality

Your choice of DR strategy should be driven primarily by how critical the workload is to your business:

"Depending on how critical the applications in your organization are to your business, you might decide on a uniform strategy for all applications or develop a more complex DR strategy based on the criticality of each application." [AWS Prescriptive Guidance – Defining Your DR Strategy](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-database-disaster-recovery/defining.html){evidence: "Depending on how critical the applications in your organization are to your business, you might decide on a uniform strategy for all applications or develop a more complex DR strategy based on the criticality of each application."}

**Tiered approach recommendations:**
- **Tier 1 (Mission-critical):** Active-Active or Warm Standby — e.g., payment processing, core APIs
- **Tier 2 (Business-important):** Warm Standby or Pilot Light — e.g., internal business apps, CRM
- **Tier 3 (Non-critical):** Backup & Restore — e.g., dev/test environments, batch processing

### Compliance & Regulatory Requirements

- **Data residency:** Some workloads have regulatory data residency requirements that limit multi-Region strategies. If your locality has only one AWS Region, multi-AZ strategies are the appropriate fallback.
- **Recovery mandates:** Financial services, healthcare, and government sectors often have mandated RTO/RPO objectives that eliminate lower-tier strategies.
- **Audit and testing:** All strategies require regular testing; higher-tier strategies require more frequent DR drills.

### Budget Constraints

The relationship between DR performance and cost is often exponential rather than linear. "The two primary metrics governing your DR spend are RTO and RPO... the relationship between performance and price is often exponential rather than linear." [Balancing resilience and spend: a guide to AWS disaster recovery cost optimization](https://hykell.com/knowledge-base/aws-disaster-recovery-cost-optimization/){evidence: "The two primary metrics governing your DR spend are RTO and RPO. RTO is the maximum tolerable duration of an outage, while RPO defines the maximum window of data loss you can accept. These objectives dictate your underlying architecture, and the relationship between performance and price is often exponential rather than linear."}

**Key budget considerations:**
- Compare cost of DR infrastructure vs. cost of downtime (lost revenue, SLA penalties, reputation damage)
- Consider On-Demand Capacity Reservations for Pilot Light to guarantee compute availability without running instances
- AWS Elastic Disaster Recovery (DRS) can achieve Warm Standby-like RPO/RTO at Pilot Light cost levels

### Operational Complexity

| Strategy | Operational Burden |
|----------|-------------------|
| Backup & Restore | Lowest — manage backup schedules, periodic restore testing |
| Pilot Light | Medium — manage replication, AMI updates, failover runbooks |
| Warm Standby | Medium-High — maintain running environment, CI/CD to both regions, scaling automation |
| Active-Active | High — conflict resolution, global routing, bidirectional replication, multi-region deployments |

"When selecting your DR strategy, you must weigh the benefits of lower RTO and RPO vs the costs of implementing and operating a strategy. The pilot light and warm standby strategies both offer a good balance of benefits and cost." [DR Architecture on AWS, Part III: Pilot Light and Warm Standby](https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iii-pilot-light-and-warm-standby){evidence: "When selecting your DR strategy, you must weigh the benefits of lower RTO (recovery time objective) and RPO (recovery point objective) vs the costs of implementing and operating a strategy. The pilot light and warm standby strategies both offer a good balance of benefits and cost."}

---

## AWS Services Mapped to DR Strategies

### Data Replication & Backup Services

| AWS Service | Backup & Restore | Pilot Light | Warm Standby | Active-Active |
|-------------|:---:|:---:|:---:|:---:|
| **AWS Backup** | ✅ Primary | ✅ Supplement | ✅ Supplement | ✅ Supplement |
| **S3 Cross-Region Replication** | ✅ | ✅ | ✅ | ✅ |
| **RDS Automated Snapshots** | ✅ | — | — | — |
| **RDS Cross-Region Read Replicas** | — | ✅ | ✅ | ✅ |
| **Aurora Global Database** | — | ✅ | ✅ | ✅ |
| **DynamoDB Global Tables** | — | ✅ | ✅ | ✅ |
| **EBS Snapshots (cross-region copy)** | ✅ | ✅ | — | — |

### Compute & Infrastructure Services

| AWS Service | Backup & Restore | Pilot Light | Warm Standby | Active-Active |
|-------------|:---:|:---:|:---:|:---:|
| **CloudFormation / Terraform** | ✅ Deploy on failover | ✅ Pre-configured | ✅ Deployed | ✅ Deployed |
| **EC2 AMIs (cross-region)** | ✅ | ✅ | — | — |
| **Auto Scaling Groups** | — | ✅ Scale up | ✅ Scale up | ✅ Running |
| **EC2 Capacity Reservations** | — | ✅ Reserve | ✅ | ✅ |
| **Lambda (Provisioned Concurrency)** | — | — | ✅ | ✅ |
| **AWS Elastic Disaster Recovery** | — | ✅ | ✅ | — |

### Traffic Routing & Failover Services

| AWS Service | Backup & Restore | Pilot Light | Warm Standby | Active-Active |
|-------------|:---:|:---:|:---:|:---:|
| **Route 53 (DNS Failover)** | ✅ Manual | ✅ Health-check | ✅ Health-check | ✅ Latency/Geo |
| **AWS Global Accelerator** | — | ✅ | ✅ | ✅ |
| **Application Recovery Controller** | — | ✅ | ✅ | ✅ |
| **CloudFront** | — | — | ✅ | ✅ |
| **Elastic Load Balancing** | — | ✅ Deploy | ✅ Running | ✅ Running |

[AWS Infrastructure for Disaster Recovery Strategies](https://kindatechnical.com/aws-certification-cloud-practitioner/aws-infrastructure-for-disaster-recovery-strategies.html){evidence: "S3 Cross-Region Replication: Automatically copies objects to a bucket in another Region for backup/restore and pilot light strategies. RDS Cross-Region Read Replicas: Maintains a read replica of your database in another Region that can be promoted during failover. DynamoDB Global Tables: Provides multi-Region, multi-active database replication with sub-second replication lag. Aurora Global Database: Replicates data across Regions with less than one second of replication lag and supports promotion of a secondary Region in under a minute. Route 53 Health Checks and DNS Failover: Monitors endpoint health and automatically routes traffic away from unhealthy Regions. AWS Backup: Centralized backup service that automates backup scheduling, retention, and cross-Region copying. CloudFormation: Infrastructure as code that lets you rapidly recreate your entire infrastructure stack in another Region from templates."}

### AWS Elastic Disaster Recovery (DRS) — Special Mention

AWS Elastic Disaster Recovery deserves special attention as it bridges the gap between Pilot Light and Warm Standby:

"If you are considering the pilot light or warm standby strategy for disaster recovery, AWS Elastic Disaster Recovery could provide an alternative approach with improved benefits. Elastic Disaster Recovery can offer an RPO and RTO target similar to warm standby, but maintain the low-cost approach of pilot light." [REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "Elastic Disaster Recovery can offer an RPO and RTO target similar to warm standby, but maintain the low-cost approach of pilot light. Elastic Disaster Recovery replicates your data from your primary region to your recovery Region, using continual data protection to achieve an RPO measured in seconds and an RTO that can be measured in minutes."}

---

## AWS Well-Architected Reliability Pillar: REL13-BP02 Guidance

### Best Practice: Use Defined Recovery Strategies to Meet Recovery Objectives

The AWS Well-Architected Reliability Pillar's best practice REL13-BP02 provides authoritative guidance for DR planning:

**Desired Outcome:** "For each workload, there is a defined and implemented DR strategy that allows the workload to achieve DR objectives. DR strategies between workloads make use of reusable patterns." [REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "For each workload, there is a defined and implemented DR strategy that allows the workload to achieve DR objectives. DR strategies between workloads make use of reusable patterns (such as the strategies previously described)."}

### Common Anti-Patterns

The framework identifies three anti-patterns to avoid:

1. **Implementing inconsistent recovery procedures** for workloads with similar DR objectives
2. **Leaving the DR strategy to be implemented ad-hoc** when a disaster occurs
3. **Dependency on control plane operations** during recovery

[REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "Common anti-patterns: Implementing inconsistent recovery procedures for workloads with similar DR objectives. Leaving the DR strategy to be implemented ad-hoc when a disaster occurs. Dependency on control plane operations during recovery."}

### Risk Level

"Level of risk exposed if this best practice is not established: **High**. Without a planned, implemented, and tested DR strategy, you are unlikely to achieve recovery objectives in the event of a disaster." [REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "Level of risk exposed if this best practice is not established: High. Without a planned, implemented, and tested DR strategy, you are unlikely to achieve recovery objectives in the event of a disaster."}

### Implementation Steps (Summary)

1. **Determine a DR strategy** that satisfies recovery requirements — avoid over-engineering (more stringent than needed incurs unnecessary costs)
2. **Review implementation patterns** for the selected strategy
3. **Assess workload resources** and their DR Region configuration during normal operations
4. **Determine recovery actions** for failover — deploy missing resources, promote databases, scale up
5. **Implement traffic rerouting** — use Route 53, Global Accelerator, or Application Recovery Controller; prefer data plane over control plane operations
6. **Design a failback plan** — re-synchronize data from recovery Region to primary Region

### Key Principles from REL13-BP02

- **Use IaC (CloudFormation/Terraform)** to deploy and maintain infrastructure in both Regions
- **Rely on data plane, not control plane** during recovery (per REL11-BP04)
- **Automate failover steps** even if initiation is manual — "the manual initiation is like the push of a button"
- **Test regularly** using AWS Resilience Hub to continuously validate RTO/RPO targets
- **Protect against data corruption** — continuous replication alone is insufficient; include point-in-time backups

[REL13-BP02 – AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html){evidence: "A DR strategy relies on the ability to stand up your workload in a recovery site if your primary location becomes unable to run the workload. The most common recovery objectives are RTO and RPO, as discussed in REL13-BP01 Define recovery objectives for downtime and data loss."}

---

## Summary

**Key Takeaways:**

1. **Four strategies exist on a cost-recovery spectrum:** Backup & Restore ($, hours) → Pilot Light ($$, tens of minutes) → Warm Standby ($$$, minutes) → Active-Active ($$$$, near-zero). There is no one-size-fits-all answer.

2. **Start with business requirements, not technology:** Define your RTO/RPO targets based on the cost of downtime and data loss, then select the least expensive strategy that meets those targets.

3. **Tier your workloads:** Not every system requires Active-Active. Classify applications by criticality and assign appropriate strategies to each tier to optimize costs.

4. **AWS Elastic Disaster Recovery (DRS) bridges the gap** between Pilot Light and Warm Standby — achieving seconds-level RPO and minutes-level RTO at Pilot Light cost levels.

5. **The pilot light vs. warm standby distinction is operational:** Pilot Light cannot serve traffic without additional deployment actions; Warm Standby can serve (reduced) traffic immediately.

6. **REL13-BP02 is non-negotiable:** The Well-Architected Framework classifies the risk as HIGH if a defined, implemented, and tested DR strategy is not established. Ad-hoc disaster recovery during an actual event will almost certainly fail to meet objectives.

7. **Test, test, test:** Regardless of strategy chosen, regular DR testing (drills, game days) using services like AWS Resilience Hub is essential for confidence in actual recovery.

---

## Sources

1. AWS. "Disaster Recovery of Workloads on AWS: Recovery in the Cloud." https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html
2. AWS. "REL13-BP02 Use defined recovery strategies to meet the recovery objectives." https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/rel_planning_for_recovery_disaster_recovery.html
3. AWS. "Disaster Recovery (DR) Architecture on AWS, Part IV: Multi-site Active/Active." https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iv-multi-site-active-active
4. AWS. "Disaster Recovery (DR) Architecture on AWS, Part III: Pilot Light and Warm Standby." https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iii-pilot-light-and-warm-standby
5. AWS. "Disaster Recovery (DR) Architecture on AWS, Part II: Backup and Restore with Rapid Recovery." https://aws.amazon.com/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-ii-backup-and-restore-with-rapid-recovery
6. AWS. "Defining your DR strategy (Prescriptive Guidance)." https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-database-disaster-recovery/defining.html
7. KindaTechnical. "AWS Infrastructure for Disaster Recovery Strategies." https://kindatechnical.com/aws-certification-cloud-practitioner/aws-infrastructure-for-disaster-recovery-strategies.html
8. Hykell. "Balancing resilience and spend: a guide to AWS disaster recovery cost optimization." https://hykell.com/knowledge-base/aws-disaster-recovery-cost-optimization/
