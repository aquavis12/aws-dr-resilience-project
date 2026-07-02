# AWS Elastic Disaster Recovery (DRS)

## How DRS Works

### Continuous Block-Level Replication

AWS Elastic Disaster Recovery operates through a lightweight agent installed on each source server. This agent performs continuous block-level replication of all attached volumes — including the operating system, system state configuration, databases, applications, and files — to a low-cost staging area in the target AWS account and preferred Region.

[AWS Prescriptive Guidance - Backup and Recovery](https://docs.aws.amazon.com/prescriptive-guidance/latest/backup-recovery/on-prem-dr-to-aws.html){evidence: "Elastic Disaster Recovery continuously replicates your machines into a low-cost staging area in your target AWS account and preferred AWS Region. The block level replication is an exact replica of your servers' storage including the operating system, system state configuration, databases, applications, and files."}

The agent runs in memory on the source server and recognizes write operations to locally attached disks, immediately attempting to copy changed blocks across the network into the replication staging area subnet.

[AWS Storage Blog - Achieving Data Consistency](https://aws.amazon.com/blogs/storage/achieving-data-consistency-with-aws-elastic-disaster-recovery/){evidence: "Replication is performed at the block level, via an agent that is installed on each source machine that you want to protect. The agent runs in memory and recognizes write operations to locally attached disks."}

All data in transit is compressed and encrypted during replication.

[AWS Prescriptive Guidance - Backup and Recovery](https://docs.aws.amazon.com/prescriptive-guidance/latest/backup-recovery/on-prem-dr-to-aws.html){evidence: "Continuous block-level replication (compressed and encrypted)"}

### Staging Area Architecture

The staging area uses minimal, low-cost resources to maintain continuous replication:

- **Replication Servers**: Low-cost EC2 instances (default: t3.small) that receive replicated data. Each replication server handles up to 15 staging disks from multiple source servers. DRS launches and terminates these automatically as needed.
- **Staging Disks**: EBS volumes that mirror each source server disk, maintaining an exact block-level copy.
- **EBS Snapshots**: Used for point-in-time recovery, stored according to retention policies.

[AWS DRS Pricing](https://aws.amazon.com/disaster-recovery/pricing){evidence: "AWS DRS uses EC2 instances as Replication Servers, which facilitate ongoing data replication. By default, these are low-cost t3.small EC2 instances. Each Replication Server can handle replication of disks from multiple source servers, with a maximum of 15 staging disks."}

### Launch Settings and Server Conversion

When a failover or drill is initiated, DRS launches recovery instances from the replicated data. The orchestration process creates cloned volumes from the replication staging area and initiates an automated server conversion process that transforms all volumes originating outside of AWS into AWS-compatible volumes attached to bootable EC2 instances.

[Elastic Disaster Recovery Concepts](https://docs.aws.amazon.com/zh_tw/drs/latest/userguide/CloudEndure-Concepts.html){evidence: "When launching a recovery job, the AWS DRS orchestration process creates cloned volumes by using the replicated volumes in the replication staging area. During this process, AWS DRS also initiates a process that converts all volumes that originated outside of AWS into AWS-compatible volumes, which are attached to EC2 instances that can boot natively on AWS."}

A Conversion Server EC2 instance is launched temporarily for each recovery instance to perform this conversion, running for only a few minutes at a cost of less than $0.05 per instance.

### Point-in-Time Recovery

AWS DRS automatically creates and retains Point-in-Time (PIT) Snapshots with the following default retention schedule:

| Granularity | Frequency | Retention |
|------------|-----------|-----------|
| Minute | 1 snapshot per 10 minutes | Prior 1 hour |
| Hour | 1 snapshot per 1 hour | Prior 24 hours |
| Day | 1 snapshot per 1 day | Prior 7 days (configurable 1–365 days) |

[Elastic Disaster Recovery Concepts](https://docs.aws.amazon.com/zh_tw/drs/latest/userguide/CloudEndure-Concepts.html){evidence: "Minute - 1 PIT Snapshot per 10 minutes for the prior 1 hour. Hour - 1 PIT Snapshot per 1 hour for the prior 24 hours. Day - 1 PIT Snapshot per 1 day for the prior 7 days."}

The "Use most recent data" option creates an on-demand PIT Snapshot representing a sub-second RPO at the time the recovery job is submitted. If this fails (e.g., source server is unreachable), DRS falls back to the last consistent state from data on the Replication Server.

[Elastic Disaster Recovery Concepts](https://docs.aws.amazon.com/zh_tw/drs/latest/userguide/CloudEndure-Concepts.html){evidence: "When used, AWS Elastic Disaster Recovery will attempt to create an on-demand PIT Snapshot of all Source Servers within the Recovery Job, representing a sub-second RPO of the Source Server."}

---

## Supported DR Strategies

### Pilot Light Implementation

In a Pilot Light strategy using DRS, only the core infrastructure components are kept running in the recovery environment. DRS maintains continuous replication of server data to the staging area (EBS volumes and snapshots), but full recovery instances are only provisioned when a disaster or drill occurs. This minimizes idle costs while keeping data synchronized for rapid recovery.

[AWS Architecture Blog - DR Part III](https://aws.amazon.com/ko/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-iii-pilot-light-and-warm-standby/){evidence: "Both strategies might replicate data from the primary Region to data resources in the recovery Region, such as Amazon Relational Database Service (Amazon RDS) DB instances or Amazon DynamoDB tables. These data resources in the recovery Region are then ready to serve requests."}

DRS naturally aligns with the Pilot Light pattern because:
- The replication staging area uses minimal compute (low-cost replication servers)
- Full recovery instances are launched only on demand
- Data remains continuously synchronized with sub-second RPO
- Recovery can be initiated within minutes

[AWS DRS - Oracle JDE Guidance](https://aws.amazon.com/solutions/guidance/oracles-jd-edwards-enterpriseone-with-aws-elastic-disaster-recovery/){evidence: "This strategy helps you create a lower-cost disaster recovery environment where you initially provision a replication server for replicating data from the source database, and you provision the actual database server only when you start a disaster recovery drill or recovery."}

### Warm Standby Implementation

For Warm Standby, DRS can be combined with other AWS services where some components run at reduced capacity in the recovery region. While DRS handles the server-level replication and rapid launch, database services like Amazon RDS cross-region read replicas or DynamoDB Global Tables maintain the data tier in an active (though scaled-down) state. During failover, DRS launches full-capacity recovery instances while the pre-provisioned data tier is scaled up to handle production traffic.

[Cloudian - DR Strategies](https://cloudian.com/guides/disaster-recovery/disaster-recovery-on-aws-4-strategies-and-how-to-deploy-them/amp/){evidence: "Warm standby – involves running a full backup system in standby mode, with live data replicated from the production environment."}

---

## Failover and Failback Workflows

### Failover Process

1. **Initiate Recovery**: From the DRS console, select source servers and initiate a recovery launch (or use AWS CLI/API for automation).
2. **Select Recovery Point**: Choose "Use most recent data" for sub-second RPO, or select a specific Point-in-Time snapshot.
3. **Launch Recovery Instances**: DRS creates cloned volumes from staging area snapshots, runs the conversion server process, and launches fully provisioned EC2 instances.
4. **Redirect Traffic**: After recovery instances are running, redirect traffic from primary systems to the launched recovery instances (via DNS changes, load balancer updates, etc.).

[AWS DRS - Performing a Failover](https://docs.aws.amazon.com/zh_tw/drs/latest/userguide/failback-preparing-failover.html){evidence: "AWS Elastic Disaster Recovery helps you perform a failover by launching recovery instances in AWS. Once the Recovery instances are launched, you will need to redirect the traffic from your primary systems to the launched recovery instances."}

DRS can launch thousands of machines in their fully provisioned state within minutes.

[AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/backup-recovery/on-prem-dr-to-aws.html){evidence: "If there is a disaster, you can instruct Elastic Disaster Recovery to quickly launch thousands of your machines in their fully provisioned state within minutes."}

### Failback Process

Once the primary environment is restored, failback returns operations from AWS recovery instances back to the original infrastructure:

1. **Initiate Reverse Replication**: DRS begins replicating data from the recovery instances back to the source environment.
2. **Boot Failback Client**: A Failback Client is booted directly on the source server that will receive the replicated data.
3. **Complete Data Sync**: Wait for reverse replication to synchronize all changes made during the disaster period.
4. **Cutover**: Redirect traffic back to the original primary environment.

[AWS DRS - Failback to On-Premises](https://docs.aws.amazon.com/drs/latest/userguide/failback-performing.html){evidence: "To initiate this process, the Failback Client is booted directly on the source server that will receive the replicated data."}

For VMware environments, the DRS Mass Failback Automation Client enables one-click or custom failback for multiple vCenter machines simultaneously.

[AWS DRS - Mass Failback Automation](https://docs.aws.amazon.com/drs/latest/userguide/failback-failover-drsfa.html){evidence: "one-click or custom failback for multiple vCenter machines at once."}

### Automated Failover and Failback

DRS supports automation of the failover and failback process, helping achieve lower and more consistent RTO.

[AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/backup-recovery/on-prem-dr-to-aws.html){evidence: "You can also automate your DR failover and failback process with Elastic Disaster Recovery. Automating your failover and failback process can help you achieve a lower and more consistent recovery time objective (RTO)."}

---

## Cross-Region and Cross-Account DR Setups

### Cross-Region Architecture

When deploying DRS for cross-region disaster recovery, applications hosted in a primary AWS Region are protected by replicating data to a secondary Region used for recovery during drills or disasters.

[AWS Guidance - Cross-Region DR Design](https://docs.aws.amazon.com/guidance/latest/deploying-cross-region-disaster-recovery-with-aws-elastic-disaster-recovery/design-guidance.html){evidence: "When deploying Elastic Disaster Recovery as a cross-Region approach, you will be protecting applications that are hosted in your primary Region by replicating data to a secondary Region that would also be used for recovery during a drill or disaster."}

Key architectural considerations:
- All communication stays within the AWS Cloud backbone and never traverses the public internet
- VPC Peering or Transit Gateway connects the primary and recovery regions
- DRS should be initialized in both source and target regions in advance for operational continuity
- Regular failover and failback drills are recommended

[AWS Storage Blog - Private Cross-Region DR](https://aws.amazon.com/blogs/storage/private-cross-region-disaster-recovery-with-aws-elastic-disaster-recovery/){evidence: "Note that all of the communication is within the AWS Cloud backbone and never goes outside to the public Internet. VPC Peering is used in this architecture, but it can be replaced with an Amazon Transit Gateway if desired."}

[AWS Guidance - Cross-Region DR Failback](https://docs.aws.amazon.com/guidance/latest/deploying-cross-region-disaster-recovery-with-aws-elastic-disaster-recovery/failback.html){evidence: "To ensure operational continuity, initialize the AWS DRS in advance in both the source and target AWS Regions, and conduct regular failover and failback drills."}

### Cross-Account Architecture

Cross-account DR setups provide additional isolation and security. The architecture involves:
- VPC endpoints for private connectivity
- VPC peering connections between accounts
- Amazon Route 53 private hosted zones for DNS resolution
- Network isolation and security controls maintained throughout

[AWS Storage Blog - Cross-Account DR Part 1](https://aws.amazon.com/blogs/storage/cross-account-disaster-recovery-setup-using-aws-elastic-disaster-recovery-in-secured-networks-part-1-architecture-and-network-setup/){evidence: "In this first part, we focus on the architecture and network setup needed to build a secure cross-account disaster recovery (DR) solution that maintains network isolation, preserves IP addressing schemes, and makes sure of continuous operation during DR scenarios."}

[AWS Storage Blog - Cross-Account DR Part 2](https://aws.amazon.com/blogs/storage/cross-account-disaster-recovery-setup-using-aws-elastic-disaster-recovery-in-secured-networks-part-2-failover-and-failback-implementation/){evidence: "We walk through installing the AWS replication agent, performing recovery drills, initiating reverse replication, and completing the failback to the production environment."}

### Scalability Limits

DRS supports up to 300 source servers per AWS account due to account-wide API limitations. For larger environments, multiple staging area accounts can be created.

[AWS Guidance - Technical Prerequisites](https://docs.aws.amazon.com/guidance/latest/deploying-cross-region-disaster-recovery-with-aws-elastic-disaster-recovery/technical-prerequisites.html){evidence: "Due to AWS account wide API limitations, Elastic Disaster Recovery is limited to protecting 300 source servers per AWS account. In order to replicate more than 300 servers, you would be required to create multiple staging area AWS accounts."}

### PIT Snapshot Storage by Strategy

| Replication Strategy | Replication Target | EBS Snapshot Region | EBS Snapshot Account |
|---------------------|-------------------|--------------------|--------------------|
| Single Account | Any Region | Same as Replication Target | Same AWS Account |
| Extended Account | Any Region | Same as Replication Target | Staging Account |
| Multi-Account | Any Region | Same as Replication Target | Target Account |
| Reverse Replication | Any Region | Same as Source | Source Account |
| Any | Outpost | Stored locally on Outpost | Outpost Account |

[Elastic Disaster Recovery Concepts](https://docs.aws.amazon.com/zh_tw/drs/latest/userguide/CloudEndure-Concepts.html){evidence: "Replication Strategy: Single Account, Extended Account, Multi-Account, Reverse Replication, Any (Outpost)"}

---

## RPO and RTO Capabilities

### Recovery Point Objective (RPO): Sub-Second

AWS DRS achieves a crash-consistent RPO of seconds (sub-second under optimal conditions) through continuous block-level replication.

[Elastic Disaster Recovery Concepts](https://docs.aws.amazon.com/zh_tw/drs/latest/userguide/CloudEndure-Concepts.html){evidence: "The Recovery Point Objective (RPO) of Elastic Disaster Recovery is typically in the sub-second range."}

The agent continuously monitors blocks written to source volumes and immediately copies them to the staging area. As long as the network bandwidth can keep pace with the write rate, RPO remains at seconds.

[Elastic Disaster Recovery Concepts](https://docs.aws.amazon.com/zh_tw/drs/latest/userguide/CloudEndure-Concepts.html){evidence: "The AWS Replication Agent continuously monitors the blocks written to the source server volume(s), and immediately attempts to copy the blocks across the network and into the replication staging area subnet located in the customer's target AWS account. This continuous replication approach allows an RPO of seconds as long as the written data can be immediately copied across the network and into the replication Staging Area volumes."}

**Important**: DRS provides crash-consistent recovery points. Application data kept in memory is not replicated until it is written to the source server volumes.

[Elastic Disaster Recovery Concepts](https://docs.aws.amazon.com/zh_tw/drs/latest/userguide/CloudEndure-Concepts.html){evidence: "A crash-consistent recovery point allows the successful recovery of crash-consistent applications, such as databases. The recovery point will include any data that has been successfully written to the source server volume(s). Application data that is kept in memory is not replicated to the target replication Staging Area until it is written to the source server volume(s)."}

**Factors affecting RPO**:
- Outbound network bandwidth from source environment
- Inbound network bandwidth at the target staging area
- Staging area resource capacity (replication server instance type, EBS IOPS)
- Write burst rates on source volumes

### Recovery Time Objective (RTO): Minutes

RTO typically ranges between 5–20 minutes depending on the environment.

[Elastic Disaster Recovery Concepts](https://docs.aws.amazon.com/zh_tw/drs/latest/userguide/CloudEndure-Concepts.html){evidence: "AWS Elastic Disaster Recovery (AWS DRS) provides continuous block-level replication, recovery orchestration, and automated server conversion capabilities. These allow customers to achieve a crash-consistent recovery point objective (RPO) of seconds, and a recovery time objective (RTO) typically ranging between 5–20 minutes."}

**Factors affecting RTO**:
- **OS type**: Linux servers typically boot within ~5 minutes; Windows servers within ~20 minutes due to the more resource-intensive boot process
- **OS configuration**: Heavier workloads and additional startup services increase boot time
- **Target instance performance**: Higher-performance instance types yield faster boot
- **Target volume performance**: Higher-IOPS volume types speed up boot

[Elastic Disaster Recovery Concepts](https://docs.aws.amazon.com/zh_tw/drs/latest/userguide/CloudEndure-Concepts.html){evidence: "OS type: The average recovered Linux server normally boots within 5 minutes, while the average recovered Windows server normally boots within 20 minutes because it is tied to the more resource-intensive Windows boot process."}

---

## Integration with CloudEndure Lineage

### Evolution from CloudEndure Disaster Recovery

AWS Elastic Disaster Recovery is the direct successor to CloudEndure Disaster Recovery (CEDR). It is built on the same core technology but integrated natively into the AWS Management Console.

[AWS DRS General FAQ](https://docs.aws.amazon.com/drs/latest/userguide/General-Questions-FAQ.html){evidence: "AWS Elastic Disaster Recovery (AWS DRS) is the next generation of CloudEndure Disaster Recovery (CEDR) and is the recommended service to use for Disaster Recovery to AWS."}

[CloudEndure DR EOL FAQ](https://docs.cloudendure.com/Content/FAQ/FAQ/CloudEndure_DR_EOL_FAQ.htm){evidence: "AWS Elastic Disaster Recovery (AWS DRS) is now the recommended service for disaster recovery on AWS. AWS DRS is based on CloudEndure Disaster Recovery The CloudEndure solution that enables the recovery or continuation of vital technology infrastructure and systems in case of a crippling event."}

### Key Differences from CloudEndure DR

DRS provides similar capabilities to CEDR but with key improvements:
- Operated directly from the AWS Management Console (not a separate SaaS portal)
- Native integration with AWS IAM, CloudTrail, and other AWS services
- Support for AWS Organizations and multi-account strategies
- Enhanced automation capabilities through AWS APIs and SDK

[AWS - When to Choose DRS](https://aws.amazon.com/tr/disaster-recovery/when-to-choose-aws-drs/){evidence: "Elastic Disaster Recovery is the recommended service for disaster recovery to AWS. It provides similar capabilities as CloudEndure Disaster Recovery, and is operated from the AWS Management Console."}

### Migration Path from CEDR to DRS

AWS provides a structured upgrade path including the CEDR to DRS Upgrade Assessment Tool and the Server Upgrade Tool. The manual process involves:
1. Initialize DRS in the target region
2. Verify CloudEndure recovery works as expected
3. Pause CloudEndure replication and uninstall the CEDR agent
4. Install the AWS Replication Agent
5. Configure replication and launch settings in DRS
6. Wait for initial sync to reach "Healthy" state
7. Run a drill instance to verify functionality
8. Wait for the desired PIT retention period to accumulate
9. Remove servers from the CloudEndure console

[AWS Storage Blog - Upgrading from CEDR to DRS](https://aws.amazon.com/blogs/storage/using-aws-systems-manager-to-upgrade-from-cloudendure-disaster-recovery-to-aws-elastic-disaster-recovery/){evidence: "Using AWS Systems Manager to upgrade from CloudEndure Disaster Recovery to AWS Elastic Disaster Recovery"}

Note: CloudEndure Migration has been discontinued, and AWS Application Migration Service (now AWS Transform MGN) serves as the migration counterpart to DRS.

---

## Cost Model

### Pricing Structure

DRS uses simple, predictable, usage-based pricing with no upfront costs, no minimum fees, and no long-term contracts.

| Component | Cost |
|-----------|------|
| DRS service fee | $0.028 per source server per hour |

[AWS DRS Pricing](https://aws.amazon.com/disaster-recovery/pricing){evidence: "AWS Elastic Disaster Recovery (AWS DRS) has simple, predictable, usage-based pricing. With AWS Elastic Disaster Recovery, you pay only for the servers you are actively replicating to AWS. Your costs are based on a flat per hour fee. There are no resources to manage, no upfront costs, and no minimum fee."}

The flat hourly fee covers continuous data replication, test launches, recovery launches, and point-in-time recovery — regardless of disk count, storage size, number of launches, or target region.

[AWS DRS Pricing](https://aws.amazon.com/disaster-recovery/pricing){evidence: "Pricing includes continuous data replication, test launches, recovery launches, and point-in-time recovery."}

### Ongoing Replication Costs (Additional AWS Resources)

Beyond the DRS service fee, replication consumes:
- **EBS Volumes** (staging disks): Equally sized volumes for each source disk (mix of gp3 and sc1 based on performance needs)
- **EBS Snapshots**: Base snapshot per disk plus incremental snapshots for PIT recovery (affected by daily change rate and retention period)
- **EC2 Instances** (Replication Servers): Low-cost t3.small by default; higher-performance types for high write-rate servers

**Example**: 100 servers with 30TB total storage, 3.3% daily change rate, 7-day retention:
- DRS fee: $2,044/month
- EBS volumes: $1,425/month
- EBS snapshots: $1,847/month
- EC2 replication servers: $1,054/month
- **Total: ~$6,389/month**

[AWS DRS Pricing](https://aws.amazon.com/disaster-recovery/pricing){evidence: "Total monthly AWS DRS charge: 100 servers * $0.028 per server per hour * 730 hours = $2,044.00"}

### Drill and Recovery Costs

Full recovery infrastructure costs are incurred **only** during drills or actual recovery events. You pay for the EC2 instances and EBS volumes of recovery instances only while they are running.

**Example**: 8-hour DR drill for 100 servers:
- EBS volumes (recovery): $28.49
- EC2 instances (recovery): $94.46
- **Total: ~$123 for an 8-hour drill**

[AWS DRS Pricing](https://aws.amazon.com/disaster-recovery/pricing){evidence: "Total cost for an 8-hour DR drill for 100 servers = $122.94"}

### Cost Optimization

The key cost advantage is eliminating idle DR infrastructure:

[AWS DRS FAQs](https://aws.amazon.com/jp/disaster-recovery/faqs/){evidence: "When you use AWS DRS, you save costs by removing idle disaster recovery site resources and maintenance, and pay for your full recovery site only when you need it for drills or recovery."}

---

## Supported Source Servers

### On-Premises Servers

DRS supports a wide range of on-premises infrastructure:
- **Physical servers**: Bare metal servers with the AWS Replication Agent installed
- **VMware vSphere**: Virtual machines running on VMware infrastructure
- **Microsoft Hyper-V**: Hyper-V hosted workloads

[AWS Storage Blog - Protect Hyper-V Workloads](https://aws.amazon.com/blogs/storage/protect-hyper-v-workloads-with-aws-elastic-disaster-recovery/){evidence: "Elastic Disaster Recovery can protect Hyper-V workloads that run on-premises and allow you to achieve RTOs in the minutes and RPOs in the seconds."}

[AWS DRS - Failover/Failback VMware Blog](https://aws.amazon.com/it/blogs/mt/how-to-perform-failover-and-failback-using-aws-elastic-disaster-recovery-aws-drs-between-vmware-and-aws-environments/){evidence: "Failover from your VMware on-premises environment to AWS: Using AWS DRS for creating a full replica of on-premises servers, including the root volume and operating system, on AWS."}

### Other Cloud Providers

DRS can protect workloads running on other cloud platforms including Microsoft Azure and Google Cloud Platform.

[AWS DRS - Network Diagrams](https://docs.aws.amazon.com/drs/latest/userguide/Network-diagrams.html){evidence: "running on other cloud providers such as Microsoft Azure or Google Cloud."}

### AWS EC2 Instances

DRS also supports replication of existing AWS EC2 instances for cross-region or cross-account DR scenarios. This is the basis for Region-to-Region disaster recovery strategies.

[AWS Storage Blog - Cross-Region DR](https://aws.amazon.com/blogs/storage/cross-region-disaster-recovery-using-aws-elastic-disaster-recovery){evidence: "To address these challenges, AWS Elastic Disaster Recovery provides a scalable, cost-effective way to protect workloads. It continuously replicates applications and data to a designated AWS Region, allowing organizations to recover quickly without major infrastructure changes."}

[AWS Guidance - Cross-Region DR](https://docs.aws.amazon.com/guidance/latest/deploying-cross-region-disaster-recovery-with-aws-elastic-disaster-recovery/getting-started.html){evidence: "Elastic Disaster Recovery minimizes downtime and data loss with fast, reliable recovery of on-premises and cloud-based applications running on Amazon Elastic Compute Cloud (EC2) and Amazon Elastic Block Store (EBS) using affordable storage, minimal compute, and point-in-time recovery."}

### Operating System Support

DRS supports both Linux and Windows operating systems. The agent replicates any volume attached to the server, working with LVM and RAID configurations as well.

---

## Summary

**Key Takeaways:**

1. **Core Mechanism**: AWS DRS uses a lightweight agent for continuous block-level replication to a low-cost staging area, enabling sub-second RPO and 5–20 minute RTO without maintaining expensive idle DR infrastructure.

2. **Cost Efficiency**: At $0.028/server/hour for the service fee plus AWS resource costs, DRS eliminates traditional DR site overhead. Full recovery instances are only provisioned (and charged) during drills or actual disasters — an 8-hour drill for 100 servers costs approximately $123.

3. **Broad Source Support**: Protects physical servers, VMware, Hyper-V, other clouds (Azure, GCP), and existing AWS EC2 instances — providing a single DR solution across heterogeneous environments.

4. **Flexible Recovery**: Point-in-time recovery with configurable retention (up to 365 days) allows recovery from ransomware, corruption, or operational errors by selecting a specific pre-incident snapshot.

5. **Enterprise-Ready Architecture**: Supports cross-region, cross-account, and multi-account strategies with up to 300 servers per account. Private connectivity options ensure data never traverses the public internet.

6. **CloudEndure Heritage**: As the next-generation successor to CloudEndure DR, DRS brings the same proven replication technology into the native AWS console with full IAM, CloudTrail, and Organizations integration.

7. **DR Strategy Alignment**: Naturally implements Pilot Light (minimal staging area, launch on demand) and can combine with other services for Warm Standby patterns, giving organizations flexibility to balance cost against RTO/RPO requirements.
