# AWS Backup Service & Backup and Restore DR Strategy

## 1. AWS Backup Service Overview

AWS Backup is a fully managed service that centralizes and automates data protection across AWS services and hybrid workloads. It provides core data protection features, ransomware detection and recovery capabilities, and compliance insights and analytics for data protection policies and operations.

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "AWS Backup is a fully managed service that centralizes and automates data protection across AWS services and hybrid workloads. It provides core data protection features, ransomware detection and recovery capabilities, and compliance insights and analytics for data protection policies and operations."}

### Centralized Policy-Based Backup

AWS Backup provides a single console, APIs, and CLI to centrally manage backups across all supported AWS services. Instead of managing backups individually for each service, administrators define unified backup policies that apply organization-wide.

[AWS Backup Centralized Backup Plans and Policies](https://kindatechnical.com/aws-certification-associate-sysops-administrator/aws-backup-centralized-backup-plans-and-policies.html){evidence: "Instead of managing backups individually for each service, AWS Backup lets you define backup plans with schedules, retention rules, and cross-Region copy rules that apply to all supported resources through a single interface."}

With AWS Organizations integration, a single backup policy can be defined and applied across all accounts and Regions in the organization.

[AWS Backup - EBS User Guide](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-lifecycle-backup.html){evidence: "per resource, you define a single backup policy and apply it across accounts and Regions."}

### Backup Plans

Backup plans are the core configuration construct in AWS Backup. They allow administrators to define:

- **Backup schedules**: Customizable or predefined schedules based on best practices, with backups as frequently as every hour
- **Retention policies**: Automatic retention and expiration of backups to minimize storage costs
- **Lifecycle rules**: Automatic transition of backups from warm storage to cold storage for cost optimization
- **Tag-based resource assignment**: Apply plans to AWS resources by tagging them for consistent classification

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "You can create backups managed by backup plans, enabling you to define your backup requirements and apply these policies to the AWS resources you want to protect. Backup plans simplify and scale your data protection strategy across your applications and organization."}

[Protecting Amazon S3 Using AWS Backup](https://community.aws/tutorials/protecting-amazon-s3-using-aws-backup){evidence: "AWS Backup provides unlimited retention options and the ability to create backups as frequently as every hour."}

### Backup Vaults

A backup vault is a logical container that stores and manages encrypted backups. Each vault is associated with an AWS KMS encryption key that encrypts all backups stored within it.

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "The AWS Backup vault is a logical container that stores and manages your encrypted backups. When creating a backup vault, you must specify the AWS Key Management Service (AWS KMS) encryption key that encrypts the backups placed in this vault. All copied backups are encrypted with the key of the target vault."}

Key vault properties:
- Resource-based access policies control access across all users
- Encryption keys for backup data are independent of keys used for production resources, providing defense-in-depth
- Vault Lock capability for immutability (WORM compliance)

### Cross-Region Copy

AWS Backup enables copying backups across different AWS Regions either manually (on-demand) or automatically as part of a scheduled backup plan. This is fundamental for disaster recovery.

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "You can copy backups across different AWS Regions and accounts from a central console to meet compliance and disaster recovery needs. You can copy backups either manually as an on-demand copy, or automatically as part of a scheduled backup plan to multiple different Regions and accounts, and recover those backups in a new Region or account."}

[Protecting Amazon S3 Using AWS Backup](https://community.aws/tutorials/protecting-amazon-s3-using-aws-backup){evidence: "AWS Backup provides unlimited retention options and the ability to create backups as frequently as every hour. You can also use AWS Backup to copy backups across AWS Regions and accounts."}

### Cross-Account Copy

Cross-account backup provides an additional layer of protection against accidental or malicious deletion, disasters, or ransomware by isolating backups in a separate AWS account.

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "You can create data protection policies and use AWS Organizations to enforce the protection policies throughout all the accounts in that organization. This provides multi-account backups and gives an additional layer of protection should the source account experience disruption from accidental or malicious deletion, disasters, or ransomware."}

### Lifecycle Management

AWS Backup supports lifecycle policies that automatically transition backups between storage tiers:

- **Warm storage**: Default tier for active backups with fast restore capability
- **Cold storage**: Low-cost tier for long-term retention (supported for EBS, EFS, DynamoDB, Timestream, SAP HANA on EC2, and VMware)
- **Minimum retention in cold**: Backups transitioned to cold storage must be retained for a minimum of 90 days

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "Configure lifecycle policies that automatically transition backups from warm storage to cold storage, helping lower backup storage costs by storing backups in a low-cost cold storage tier."}

---

## 2. Supported Resources

AWS Backup supports a comprehensive set of 20+ AWS services spanning storage, compute, database, and hybrid workloads:

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "AWS Backup provides a backup console, public APIs, and a command line interface to centrally manage backups across the AWS storage, compute, database, and hybrid services your applications run on, including Amazon Simple Storage Service (S3), Amazon Elastic Block Store (EBS), Amazon FSx, Amazon Elastic File System (EFS), AWS Storage Gateway, Amazon Elastic Compute Cloud (EC2), Amazon Relational Database Service (RDS), Amazon Aurora, Amazon DynamoDB, Amazon Neptune, Amazon DocumentDB (with MongoDB compatibility), Amazon Timestream, Amazon Redshift, SAP HANA on Amazon EC2 and the entire application stack defined by AWS CloudFormation, as well as hybrid applications like VMware workloads running on premises and in VMware CloudTM on AWS and AWS Outposts."}

### Full Supported Services List

| Category | Services |
|----------|----------|
| **Storage** | Amazon S3, Amazon EBS, Amazon EFS, Amazon FSx (Windows File Server, Lustre, NetApp ONTAP, OpenZFS), AWS Storage Gateway |
| **Compute** | Amazon EC2 (including Windows applications), SAP HANA on EC2 |
| **Databases** | Amazon RDS, Amazon Aurora, Amazon DynamoDB, Amazon Neptune, Amazon DocumentDB (MongoDB compatibility), Amazon Timestream, Amazon Redshift |
| **Containers** | Amazon EKS (cluster state, persistent volumes, Kubernetes resources) |
| **Application Stacks** | AWS CloudFormation (entire application stack including IAM roles and VPC security groups) |
| **Hybrid** | VMware workloads (on-premises, VMware Cloud on AWS, VMware on AWS Outposts) |

For EC2 backups, AWS Backup stores not just EBS volumes but also metadata including instance type, VPC configuration, security groups, IAM role, monitoring configuration, and tags.

[Disaster Recovery Workloads on AWS](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html){evidence: "AWS Backup also adds additional capabilities for EC2 backup—in addition to the instance's individual EBS volumes, AWS Backup also stores and tracks the following metadata: instance type, configured virtual private cloud (VPC), security group, IAM role, monitoring configuration, and tags."}

---

## 3. Backup & Restore as a DR Strategy

### Architecture

Backup and Restore is the simplest and lowest-cost of the four AWS DR strategies (Backup & Restore → Pilot Light → Warm Standby → Multi-Site Active/Active). The architecture involves:

1. **Regular backups** of data and configuration from the primary Region
2. **Cross-Region replication** of backups to the disaster recovery Region
3. **Infrastructure as Code (IaC)** templates stored and ready for deployment
4. **Full environment rebuild** from backups when disaster strikes

[Disaster Recovery Workloads on AWS](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html){evidence: "Backup and restore is a suitable approach for mitigating against data loss or corruption. This approach can also be used to mitigate against a regional disaster by replicating data to other AWS Regions, or to mitigate lack of redundancy for workloads deployed to a single Availability Zone. In addition to data, you must redeploy the infrastructure, configuration, and application code in the recovery Region."}

The architecture requires:
- Periodic or continuous backup execution
- Cross-Region backup copies for regional DR
- IaC (CloudFormation/CDK) for infrastructure redeployment
- AMI copies for EC2 instance restoration
- Application code pipeline (CodePipeline) for automated redeployment

[Disaster Recovery Workloads on AWS](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html){evidence: "To enable infrastructure to be redeployed quickly without errors, you should always deploy using infrastructure as code (IaC) using services such as AWS CloudFormation or the AWS Cloud Development Kit (AWS CDK). Without IaC, it may be complex to restore workloads in the recovery Region, which will lead to increased recovery times and possibly exceed your RTO."}

### RTO/RPO Expectations

Backup and Restore offers the **highest RTO and RPO** among all DR strategies, but at the **lowest cost**:

- **RPO**: Typically **24 hours or less**; can be as low as **5 minutes** with continuous/automated backups
- **RTO**: Typically **hours** (depends on data volume and infrastructure complexity)

[AWS Well-Architected Framework - REL13-BP02](https://docs.aws.amazon.com/wellarchitected/2022-03-31/framework/rel_planning_for_recovery_disaster_recovery.html){evidence: "hours, RTO in 24 hours or less): Back up your data and applications into the recovery Region. Using automated or continuous backups will enable point in time recovery, which can lower RPO to as low as 5 minutes in some cases."}

[Disaster Recovery DR Architecture on AWS Part II](https://aws.amazon.com/ko/blogs/architecture/disaster-recovery-dr-architecture-on-aws-part-ii-backup-and-restore-with-rapid-recovery/){evidence: "As shown in Figure 1, backup and restore is associated with higher RTO (recovery time objective) and RPO (recovery point objective). This results in longer downtimes and greater loss of data between when the disaster event occurs and recovery."}

### When to Use Backup & Restore

- Workloads that can tolerate hours of downtime during recovery
- Lower-priority applications where cost optimization is paramount
- As a baseline data protection strategy complementing higher-tier DR for critical systems
- When regulatory compliance requires off-site backup copies

### Cost Profile

Backup and Restore is the **lowest-cost DR strategy** because:
- No always-on infrastructure in the DR Region (unlike Pilot Light/Warm Standby)
- Pay only for backup storage and cross-Region data transfer
- Infrastructure is only provisioned at time of disaster
- No ongoing compute costs in the secondary Region

The trade-off is longer recovery time, as the entire environment must be rebuilt from scratch during a disaster event.

---

## 4. AWS Backup Audit Manager

AWS Backup Audit Manager provides automated compliance monitoring and reporting for data protection activities across your AWS environment.

[Streamline and automate compliance monitoring with AWS Backup Audit Manager](https://aws.amazon.com/blogs/storage/streamline-and-automate-compliance-monitoring-and-reporting-with-aws-backup-audit-manager/){evidence: "AWS Backup Audit Manager provides built-in compliance controls that continuously check for drifts while allowing you to customize those controls to define your AWS Backup data protection policies."}

### Key Capabilities

- **Frameworks**: Collections of controls that evaluate compliance posture
- **Controls**: Pre-built or customizable procedures that audit backup requirements (e.g., backup frequency, retention period)
- **Daily Reports**: Automatic compliance reports with insights on data protection status
- **Drift Detection**: Continuous evaluation to detect violations against defined guardrails
- **Remediation Prompts**: Automatic alerts when policy violations are detected

[AWS Backup Audit Manager Documentation](https://docs.aws.amazon.com/en_us/aws-backup/latest/devguide/aws-backup-audit-manager.md){evidence: "You can use AWS Backup Audit Manager to audit the compliance of your AWS Backup policies against controls that you define. A *control* is a procedure designed to audit the compliance of a backup requirement, such as the backup frequency or the backup retention period."}

### Controls

AWS Backup Audit Manager supports up to **50 controls per account per Region**. Controls can evaluate:
- Whether recovery points were created within a specified timeframe (1-744 hours or 1-31 days)
- Backup frequency requirements
- Retention period compliance
- Cross-Region copy requirements
- Encryption requirements

[Controls and remediation](https://docs.aws.amazon.com/en_us/aws-backup/latest/devguide/controls-and-remediation.html){evidence: "You can use up to 50 controls per account per Region."}

[Choosing your controls](https://docs.aws.amazon.com/aws-backup/latest/devguide/choosing-controls.html){evidence: "Evaluates if a recovery point was created within specified time frame. Value in hours [ 1 to 744 ] or days [ 1 to 31 ]."}

### Legal Hold

AWS Backup supports legal hold to prevent backup deletion during legal proceedings, even when retention periods have expired. Legal holds remain in place until explicitly released.

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "AWS Backup supports legal hold, which is used when an organization must retain certain data either for preservation, auditing, or as evidence in legal proceedings and e-Discovery. You can use legal holds to prevent backups from being deleted even if their retention period is over, and remain in place until they are explicitly released."}

---

## 5. Integration with AWS Organizations (Multi-Account Management)

AWS Backup provides deep integration with AWS Organizations for enterprise-scale backup management:

### Centralized Policy Deployment

Organizations can centrally deploy data protection and immutable backup policies across all member accounts.

[How to implement a centralized immutable backup solution with AWS Backup](https://aws.amazon.com/de/blogs/storage/how-to-implement-a-centralized-immutable-backup-solution-with-aws-backup/){evidence: "Additionally, with the use of AWS Organizations and AWS Backup, companies are able to centrally deploy data protection and immutable backup policies to configure, manage, and govern backup activity across all AWS accounts and supported resources to protect critical backup recovery points."}

### Cross-Account Management Capabilities

The cross-account management feature provides two key capabilities:

1. **Backup policy creation** across the entire AWS Organization
2. **Cross-account monitoring** of backup, restore, and copy jobs across all member accounts

[Delegated administrator support for AWS Backup](https://aws.amazon.com/jp/blogs/storage/delegated-administrator-support-for-aws-backup/){evidence: "The cross-account management feature has two capabilities. First, the ability to create backup policies across an AWS Organizations and second is the ability to monitor cross-account backup, restore and copy jobs across all the member accounts where backup policies have been applied to."}

### Delegated Administration

AWS Backup supports delegated administrators, enabling backup management to be handled by a dedicated backup administration account without requiring access to the management account.

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "You can delegate backup policy management in AWS Organizations and cross account monitoring in AWS Backup. This enables delegating backup management to a dedicated backup administration account, removing the need for member accounts to access management accounts for backup administration."}

### Separation of Duties

AWS Backup enforces separation between resource owners and backup operators:
- Backup operators can back up resources without direct access to those resources
- Resource owners cannot impact retention of backups
- Backup operators cannot mutate or exfiltrate data

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "With AWS Backup, a backup operator can back up all supported resources on AWS without requiring the backup operator to have direct access to those resources. This provides a separation of control where resource owners can't impact the retention of backups, and backup operators can't mutate or exfiltrate data."}

---

## 6. Cyber Resilience Features

### AWS Backup Vault Lock

Vault Lock provides WORM (Write Once, Read Many) protection for backups, preventing any user—including root account—from deleting backups or modifying lifecycle settings.

[Enhance the security posture of your backups with AWS Backup Vault Lock](https://aws.amazon.com/jp/blogs/storage/enhance-the-security-posture-of-your-backups-with-aws-backup-vault-lock/){evidence: "Customers can use AWS Backup Vault Lock to prevent any user from deleting their backups or making changes to their backup lifecycle settings. AWS Backup Vault Lock improves your security postures and ensures a mechanism for restore, even in a worst-case scenario like total account compromise."}

Vault Lock has been assessed by Cohasset Associates for use in environments subject to **SEC 17a-4, CFTC, and FINRA** regulations.

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "AWS Backup Vault Lock has been assessed by Cohasset Associates for use in environments subject to SEC 17a-4, CFTC, and FINRA regulations."}

### Logically Air-Gapped Vaults

Logically air-gapped vaults represent the highest level of backup protection in AWS Backup, designed specifically for cyber resilience against ransomware and destructive events.

Key properties:
- **Locked by default** in Compliance mode (immutable)
- **AWS-owned encryption keys** for additional protection layer
- **Cross-account sharing** via AWS Resource Access Manager (RAM)
- **Direct restore** from a different account than the one that created the backup
- **Multi-party approval** for access when backup account becomes inaccessible

[Building cyber resiliency with AWS Backup logically air-gapped vault](https://aws.amazon.com/blogs/storage/building-cyber-resiliency-with-aws-backup-logically-air-gapped-vault/){evidence: "With the logically air-gapped vault, the immutable backup copies are locked by default and further protected through encryption using AWS-owned keys."}

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "You can store immutable backups in logically air-gapped vault, which is a type of AWS Backup Vault. Logically air-gapped vaults are locked in Compliance mode by default, isolated with encryption using either AWS owned keys or AWS KMS customer managed keys. They allow secure sharing of access via AWS Resource Access Manager (RAM), across accounts and organizations, supporting direct restore to help reduce recovery time from a data loss event."}

[Cyber resilience on AWS reference approach](https://aws.amazon.com/blogs/architecture/cyber-resilience-on-aws-a-reference-approach-for-recovery-from-ransomware-and-destructive-events/){evidence: "The post walks through a reference pattern for isolating recovery from production, describes how AWS Backup logically air-gapped vaults provide deletion-protected backup storage, and presents a validation pipeline that checks whether a backup is recoverable and safe to use."}

### Restore Testing

AWS Backup provides automated restore testing to validate recovery readiness:
- Periodic evaluation of restore viability
- Monitoring of restore job duration times
- Recovery readiness drills for compliance/regulatory requirements

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "You can perform automated and periodic evaluation of restore viability as well as monitor restore job duration times with the restore testing feature. You can conduct recovery readiness test drills to prepare for possible data downtime or data loss events, satisfying compliance or regulatory requirements."}

### Malware Scanning

AWS Backup integrates with Amazon GuardDuty Malware Protection to scan backups for malware before restoration, ensuring clean recovery points.

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "Protect your backups from malware with Amazon GuardDuty Malware Protection for AWS Backup. Automate malware scanning of newly created backups through backup plans or run on-demand scans of existing backups to identify clean recovery points for recovery."}

### Zero Trust Architecture

AWS Backup supports Zero Trust principles through the combination of delegated admin, Organizations integration, Audit Manager, and Vault Lock.

[AWS Backup Features](https://aws.amazon.com/backup/features/){evidence: "NIST defines Zero Trust as an evolving set of cybersecurity controls that shifts from static, network-based perimeters to active defense in depth focused on users, assets, and resources. Use AWS Backup delegated admin with AWS Organizations, AWS Backup Audit Manager, and AWS Backup Vault Lock to help build your defense in depth as part of a Zero Trust architecture."}

---

## 7. Pricing

AWS Backup uses a pay-as-you-go pricing model with no upfront costs or minimum fees.

[AWS Backup Pricing](https://aws.amazon.com/backup/pricing/){evidence: "With AWS Backup, pay only for the backup storage you use, backup data transferred between AWS Regions, backup data you restore, and backup evaluations. There is no minimum fee and there are no setup charges."}

### Pricing Components

| Component | Pricing Model |
|-----------|---------------|
| **Backup Storage (Warm)** | ~$0.05/GB-Month (varies by resource type and region) |
| **Backup Storage (Cold)** | Lower rate; 90-day minimum retention |
| **Logically Air-Gapped Vault Storage** | ~$0.0575/GB-Month (e.g., EBS in EU West) |
| **Restore** | Per-GB restored (e.g., $0.02/GB for EFS warm) |
| **Cross-Region Data Transfer** | $0.02-$0.04/GB depending on resource group |
| **Audit Manager Evaluations** | $0.00125 per backup evaluation |
| **Restore Testing** | $1.50 per recovery point tested + restore storage charges |
| **Backup Search** | Indexing + storage + per-search charges |

[AWS Backup Pricing](https://aws.amazon.com/backup/pricing/){evidence: "AWS Backup storage pricing is based on the amount of storage space your backup data consumes. The storage amount billed in a month is based on the average storage space used throughout the month (billed as GB-Month)."}

[AWS Backup Pricing: Full Cost Breakdown](https://www.eon.io/blog/aws-backup-pricing){evidence: "Most teams model AWS Backup pricing as $0.05/GB-month. That estimate breaks quickly once restores, cross-region copies, and S3-level charges show up on the bill."}

### Pricing Example: Comprehensive Scenario

For a typical setup with 400-800 GB EFS backups, 10 cross-region copies (40 GB each), 10 restores, and 1500 audit evaluations across 2 regions:

- Storage: $30
- Cross-Region transfer: $16
- Destination storage: $12
- Restores: $0.20
- Item-level restores: $1.02
- Audit Manager: $9.75
- **Total: $68.97/month**

[AWS Backup Pricing](https://aws.amazon.com/backup/pricing/){evidence: "Total monthly AWS Backup bill = $30 + $16 + $12 + $0.20 + $1.02 + $9.75 = $68.97"}

### Pricing Example: Logically Air-Gapped Vault

For 3,200 GB EBS + 2,000 GB EFS with cross-region copies to a logically air-gapped vault:

- Primary backup storage: $104
- Cross-Region transfers: $144
- Air-gapped vault storage: $149.50
- **Total: $397.50/month**

[AWS Backup Pricing](https://aws.amazon.com/backup/pricing/){evidence: "Your monthly AWS Backup bill = $104 (primary backups) + $144 (cross-Region transfers) + 149.50 (logically air-gapped copies) = $397.50"}

### Key Pricing Notes

- Cold storage has a **90-day minimum** retention requirement; early deletion incurs pro-rated charges
- S3 backups incur additional charges for GET/LIST requests and EventBridge events
- EKS cluster backups incur a per-namespace creation fee
- No charge for failed copy jobs
- Backup storage free tiers for EBS, DocumentDB, and Neptune do not apply in logically air-gapped vaults

---

## Summary

**Key Takeaways:**

1. **Centralized Management**: AWS Backup provides a single pane of glass for backup management across 20+ AWS services, eliminating the need for service-specific backup configurations.

2. **Backup & Restore DR Strategy**: This is the lowest-cost DR approach with typical RPO of 24 hours (as low as 5 minutes with continuous backups) and RTO measured in hours. Best suited for workloads that can tolerate extended recovery times.

3. **Enterprise-Scale with Organizations**: Deep integration with AWS Organizations enables centralized policy enforcement, delegated administration, and cross-account backup with separation of duties.

4. **Cyber Resilience**: Vault Lock (WORM compliance for SEC 17a-4/CFTC/FINRA) and logically air-gapped vaults provide defense against ransomware, insider threats, and account compromise through immutability and isolation.

5. **Compliance Automation**: AWS Backup Audit Manager provides continuous compliance monitoring with customizable controls, automatic drift detection, and daily compliance reports.

6. **Cost-Effective**: Pay-as-you-go pricing starting at ~$0.05/GB-month with no upfront costs. Lifecycle policies and cold storage tiering further optimize costs for long-term retention.

7. **Restore Confidence**: Automated restore testing validates recovery readiness, while GuardDuty integration ensures malware-free recovery points.

8. **Architecture Best Practice**: Backup & Restore should always be paired with Infrastructure as Code (CloudFormation/CDK) for rapid environment reconstruction, and organizations should consider higher-tier DR strategies (Pilot Light, Warm Standby) for mission-critical workloads with stricter RTO/RPO requirements.
