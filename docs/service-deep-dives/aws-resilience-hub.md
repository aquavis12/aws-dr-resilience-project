# AWS Resilience Hub: Comprehensive Deep Dive

## 1. Next-Generation Features (Launched May 28, 2026)

AWS launched the next generation of Resilience Hub on **May 28, 2026**, introducing a fundamentally redesigned service with AI-powered capabilities and a new conceptual model.

### AI-Powered Failure Mode Analysis

The centerpiece of the next-gen Resilience Hub is a **GenAI-driven assessment engine** that conducts on-demand resilience evaluations across five critical failure modes:

- **Single Points of Failure** — identifies components without redundancy
- **Excessive Load** — detects resources at risk of being overwhelmed
- **Excessive Latency** — flags paths where latency could cascade
- **Misconfiguration** — surfaces configuration errors affecting resilience
- **Shared Fate** — identifies components that could fail together

[AWS Resilience Hub Pricing](https://awscloud.com/resilience-hub/pricing/){evidence: "A GenAI-driven assessment engine conducts an on-demand resilience evaluation of your service across five critical failure modes (i.e., Single Points of Failure, Excessive Load, Excessive Latency, Misconfiguration, and Shared Fate)"}

The next-gen Hub also supports **generative failure-mode analysis** that goes beyond rule-based checks to use large language models for identifying non-obvious failure scenarios and automatically generating remediation recommendations.

[AWS Outlines AI-powered Resilience Framework for Testing](https://letsdatascience.com/news/aws-outlines-ai-powered-resilience-framework-for-testing-4a669b43){evidence: "The post references the next generation of AWS Resilience Hub providing native dependency discovery and generative failure-mode analysis, and shows how the framework can use Amazon Bedrock AgentCore to host custom AI agents that automate experiment generation and validation."}

### Dependency Discovery via DNS Query Logs

The next-gen Resilience Hub introduces **Automated Dependency Assessment** — an optional capability that discovers service dependencies by analyzing actual runtime communication patterns, including DNS query logs. This reveals hidden dependencies that static configuration analysis cannot detect (e.g., third-party APIs, cross-service calls, undocumented integrations).

[Getting started with Next generation Resilience Hub](https://docs.aws.amazon.com/resilience-hub/latest/userguide/next-gen-getting-started.html){evidence: "Step 5: Enable dependency discovery (optional)"}

This feature is priced as an add-on at **$10 per service per month** when enabled, reflecting its value in uncovering blind spots in application architectures.

[AWS Resilience Hub Pricing](https://awscloud.com/resilience-hub/pricing/){evidence: "Automated Dependency Assessment is available as an optional add-on for $10 per service per month."}

### New Application Model: Services Replace Applications

The next-gen Resilience Hub replaces the concept of "applications" with **"services"** — a logical group of AWS resources that operate as a unit. This shift reflects modern microservices architectures and enables more granular resilience management.

Services can be described using:
- **AWS CloudFormation Stacks, resource tags, and Terraform state files** (each supporting cross-region and cross-account resources)
- **Kubernetes workloads** managed on Amazon EKS
- **Hybrid combinations** of resource collections and EKS clusters

[AWS Resilience Hub Pricing](https://awscloud.com/resilience-hub/pricing/){evidence: "Service: a logical group of AWS resources that you define to operate as a unit. You can describe a service in AWS Resilience Hub using 3 different methodologies: AWS CloudFormation Stacks, resource tags, and Terraform state files. Each collection supports cross-region and cross-account resources."}

### Modular Resilience Policies

The next-gen model introduces a **modular resilience policy** framework where policies are configured as the first step before onboarding services. This allows organizations to define standardized resilience tiers (e.g., "Tier 1 Critical," "Tier 2 Standard") and apply them consistently across services.

[Getting started with Next generation Resilience Hub](https://docs.aws.amazon.com/resilience-hub/latest/userguide/next-gen-getting-started.html){evidence: "Step 1: Configure a resilience policy"}

The workflow sequence — configure policy first, then create system/service, then run assessments — enables policy reuse across multiple services and supports organizational governance.

### Organization-Wide Reporting

Multi-account, centralized assessment has been enhanced in the next-gen version, building on capabilities initially introduced for the original Resilience Hub. Organizations can now assess resilience posture across their entire AWS Organization from a central management account.

[Centralized Multi-Account Application Resilience Assessment Using AWS Resilience Hub](https://aws.amazon.com/blogs/mt/centralized-multi-account-application-resilience-assessment-using-aws/){evidence: "AWS Resilience Hub enables you to define your resilience goals, assess your resilience posture against those goals, and implement recommendations for improvement based on the AWS Well-Architected Framework."}

---

## 2. Core Capabilities

### Continuous Resilience Validation

AWS Resilience Hub provides a central place to continuously assess and validate your resilience posture. It integrates into CI/CD pipelines (via APIs and AWS CodePipeline) to validate every build before production release.

[AWS Resilience Hub – Resilience management](https://docs.aws.amazon.com/resilience-hub/latest/userguide/arh-mgmt.html){evidence: "After you deploy an application into production, you can add AWS Resilience Hub to your CI/CD pipeline to validate every build before it is released into production."}

[Continually assessing application resilience with AWS Resilience Hub and AWS CodePipeline](https://aws.amazon.com/jp/blogs/architecture/continually-assessing-application-resilience-with-aws-resilience-hub-and-aws-codepipeline/){evidence: "Using AWS Resilience Hub, you can assess your applications to uncover potential resilience enhancements. This will allow you to validate your applications recovery time (RTO), recovery point (RPO) objectives and optimize business continuity while reducing recovery costs."}

Key aspects of continuous validation include:
- On-demand and scheduled resilience assessments
- CI/CD pipeline integration via APIs
- Automated daily assessments when drift detection is enabled
- Pre-production gate checks before deployment

### RTO/RPO Assessment

Resilience Hub evaluates applications against user-defined Recovery Time Objective (RTO) and Recovery Point Objective (RPO) targets across four disruption types:

1. **Application disruption** — loss of a required software service or process
2. **Infrastructure disruption** — loss of hardware (e.g., EC2 instances)
3. **Availability Zone disruption** — one or more AZs unavailable
4. **Region disruption** — one or more Regions unavailable

[Maximizing Multi-Region Resilience with AWS Resilience Hub](https://aws.amazon.com/blogs/mt/maximizing-multi-region-resilience-with-aws-resilience-hub){evidence: "AWS Resilience Hub protects applications through continuous resilience validation. It evaluates Recovery Time Objective (RTO) and Recovery Point Objective (RPO) targets and identifies infrastructure issues pre-emptively. This optimizes business continuity and reduces costs."}

For each disruption type, Resilience Hub provides estimated workload RTO and RPO values, comparing them against the defined policy to determine compliance ("Policy Met" or "Policy Breached").

[Identifying resilience drift using AWS Resilience Hub](https://aws.amazon.com/blogs/mt/identifying-resilience-drift-using-aws-resilience-hub/){evidence: "AWS Resilience Hub offers estimated workload RTO and RPO for 4 different disruptions – Application (loss of a required software service or process), Infrastructure (loss of hardware, such as Amazon EC2 instances), Availability Zone (one or more Availability Zones are unavailable), and Region (one or more Regions are unavailable)."}

### Resilience Testing

Resilience Hub integrates with **AWS Fault Injection Service (FIS)** for chaos engineering experiments that validate recovery capabilities under realistic failure conditions.

[How to use Resilience Hub's Fault Injection Experiments](https://aws.amazon.com/blogs/mt/how-to-use-resilience-hubs-fault-injection-experiments-to-test-applications-resilience){evidence: "Resilience Hub integrates with AWS FIS, a chaos engineering service, to provide fault-injection simulations of real-world failures. These include network errors, application processing errors, or too many open connections to a database."}

Operational recommendations from Resilience Hub include:
- **Amazon CloudWatch Alarms** — monitoring triggers for resilience-impacting metrics
- **Standard Operating Procedures (SOPs)** — recovery runbooks via AWS Systems Manager Documents
- **FIS Chaos Experiments** — pre-built fault injection templates

[Automate Standard Operating Procedures (SOPs) execution with AWS Resilience Hub](https://aws.amazon.com/cn/blogs/mt/automate-standard-operating-procedures-sops-execution-with-aws-resilience-hub/){evidence: "AWS Resilience Hub provides both Resiliency and Operational recommendations. Operational recommendations are comprised of Amazon CloudWatch Alarms, Standard Operating Procedures (SOPs) utilizing AWS Systems Manager Documents and chaos experiments using AWS Fault Injection Service (FIS)."}

These recommendations are provided as deployable AWS CloudFormation templates, enabling one-click implementation.

### Drift Detection

Resilience drift detection (launched August 2023) proactively identifies configuration changes that degrade an application's resilience posture. When enabled, Resilience Hub:

1. Runs daily automated assessments
2. Compares current state against the last known "Policy Met" baseline
3. Alerts via Amazon SNS when compliance status changes from "Met" to "Breached"
4. Provides specific remediation recommendations

[Identifying resilience drift using AWS Resilience Hub](https://aws.amazon.com/blogs/mt/identifying-resilience-drift-using-aws-resilience-hub/){evidence: "Resilience drift detection allows you to proactively detect and react to changes to the resilience posture of your application on AWS. Once you opt in for resilience drift detection, you are notified when your application is no longer meeting its resilience policy (i.e. the application will potentially not meet its RTO and/or RPO)."}

Example scenario: If an EBS backup schedule is accidentally changed, drift detection identifies the RPO impact (e.g., estimated RPO drifts from 1 hour to 2 hours) and recommends modifying the AWS Backup plan frequency to restore compliance.

---

## 3. Integration with Other AWS DR Services

AWS Resilience Hub operates as the **assessment and governance layer** within a broader ecosystem of disaster recovery services. It works alongside — not as a replacement for — operational DR tools.

### AWS Elastic Disaster Recovery (AWS DRS)

AWS DRS provides near-real-time block-level replication for Amazon EC2 instances, delivering crash-consistent recovery with RPO of seconds and RTO of 5–20 minutes. While DRS handles the mechanics of server replication and failover, Resilience Hub validates that your DRS configuration actually meets your stated RTO/RPO targets.

[Streamlining access to powerful disaster recovery capabilities of AWS](https://aws.amazon.com/blogs/architecture/streamlining-access-to-powerful-disaster-recovery-capabilities-of-aws/){evidence: "AWS DRS provides a nearly continuous block-level replication, recovery orchestration, and automated server conversion capabilities. With these, you to achieve a crash-consistent recovery point objective of seconds, and a recovery time objective typically ranging between 5–20 minutes."}

### Amazon Application Recovery Controller (ARC)

ARC provides fully managed recovery capabilities for rapid application failover across Availability Zones and Regions. It features readiness checks, routing controls, and zonal shift capabilities for quick recovery.

[Amazon Application Recovery Controller](https://aws.amazon.com/application-recovery-controller/){evidence: "Amazon Application Recovery Controller (ARC) helps you recover applications quickly—whether the impairment is in a single Availability Zone or an entire Region. ARC provides fully managed recovery capabilities so your team spends less time writing failover scripts and more time building resilient architectures."}

Resilience Hub complements ARC by validating that your ARC configurations (readiness checks, routing controls) are properly set up to achieve your declared recovery objectives.

### AWS Backup

AWS Backup provides centralized backup management across 140+ AWS resource types. Resilience Hub assesses whether backup plans, schedules, and retention policies are configured to meet your RPO requirements.

[Streamlining access to powerful disaster recovery capabilities of AWS](https://aws.amazon.com/blogs/architecture/streamlining-access-to-powerful-disaster-recovery-capabilities-of-aws/){evidence: "AWS Backup takes this further, tying together many of these disparate backup technologies, giving a single plane of glass to configure data backup plans across resources. AWS Backup also added backup capabilities for AWS resources that previously didn't have them such as Amazon Elastic File System (Amazon EFS) and Amazon FSx."}

### How They Work Together

The integration model follows a layered approach:

| Layer | Service | Role |
|-------|---------|------|
| **Assessment & Governance** | Resilience Hub | Define policies, assess compliance, detect drift |
| **Data Protection** | AWS Backup | Centralized backup schedules, cross-region/cross-account vaults |
| **Server Recovery** | AWS DRS | Real-time EC2 replication, automated failover |
| **Application Recovery** | ARC | Routing controls, zonal shift, readiness checks |
| **Chaos Testing** | AWS FIS | Fault injection experiments to validate resilience |

Resilience Hub acts as the "brain" that assesses whether the combined configuration of Backup plans, DRS replication, and ARC routing controls collectively achieves the declared resilience targets.

---

## 4. Use Cases and Best Practices

### Key Use Cases

**1. Pre-Production Resilience Gates**
Integrate Resilience Hub into CI/CD pipelines to automatically assess each deployment before production release. Failed assessments can block deployments that would degrade resilience posture.

[AWS Resilience Hub – Resilience management](https://docs.aws.amazon.com/resilience-hub/latest/userguide/arh-mgmt.html){evidence: "In addition to architectural guidance for improving your application resiliency, the recommendations provide code for meeting your resiliency policy, implementing tests, alarms, and standard operating procedures (SOPs) that you can deploy and run with your application in your integration and delivery (CI/CD) pipeline."}

**2. Compliance and Regulatory Reporting**
Organizations in regulated industries use Resilience Hub to demonstrate continuous compliance with recovery objectives required by frameworks like SOC 2, ISO 22301, and industry-specific mandates.

[AWS Resilience Hub – Resilience management](https://docs.aws.amazon.com/resilience-hub/latest/userguide/arh-mgmt.html){evidence: "AWS Resilience Hub helps you to protect your applications from disruptions, and reduce recovery costs to optimize business continuity to help meet compliance and regulatory requirements."}

**3. Multi-Account Enterprise Governance**
Large enterprises use centralized assessments to maintain visibility across hundreds of accounts and services, identifying systemic resilience gaps.

**4. AI-Powered Proactive Discovery (Next-Gen)**
Using generative failure-mode analysis and dependency discovery to surface hidden risks before they manifest as production incidents.

[Architecting AI-powered resilience framework on AWS](https://aws.amazon.com/blogs/architecture/architecting-ai-powered-resilience-framework-on-aws/){evidence: "When your production system goes down, you often discover the hard way that your resilience testing missed critical dependencies. Building an AI-powered resilience framework on AWS helps you find those weaknesses before your customers do."}

**5. Automated Resilience Testing Frameworks**
Combining Resilience Hub with Amazon Bedrock AgentCore to build AI agents that automatically generate chaos experiments, execute them, and validate results — creating continuous resilience assurance loops.

### Best Practices

1. **Define resilience policies before onboarding** — Establish tiered RTO/RPO targets (e.g., Tier 1: 15min/5min, Tier 2: 1hr/1hr, Tier 3: 4hr/4hr) aligned with business impact analysis.

2. **Enable drift detection on all production workloads** — Configuration changes are the most common cause of resilience degradation; daily drift assessments catch issues early.

3. **Integrate into CI/CD pipelines** — Use Resilience Hub APIs as quality gates to prevent resilience-reducing deployments from reaching production.

4. **Deploy operational recommendations** — Implement the generated CloudWatch alarms, SOPs, and FIS experiments rather than just reviewing assessment reports.

5. **Use dependency discovery for critical services** — Enable the automated dependency assessment add-on for Tier 1 services to uncover runtime dependencies invisible to static analysis.

6. **Run regular chaos experiments** — Use the FIS integration to validate that your application actually recovers within stated RTO/RPO, not just that it's configured to.

[Post-deployment activities (AWS Prescriptive Guidance)](https://docs.aws.amazon.com/prescriptive-guidance/latest/resilience-lifecycle-framework/post-deployment.html){evidence: "Automating these assessments is a best practice because it helps ensure that you are continuously evaluating your resilience posture in production."}

---

## 5. Pricing Model

### Original Resilience Hub (Legacy)

| Component | Price |
|-----------|-------|
| First 3 applications for 6 months | **Free** |
| Per assessed application per month | **$15/application** |

Billing begins once the first resilience assessment is run and stops when the service is removed.

### Next-Generation Resilience Hub (Launched May 28, 2026)

The new pricing model is usage-based with three billing dimensions:

| Component | Price |
|-----------|-------|
| Per service per month (includes 2 failure mode assessments for ≤150 resources) | **$15/service** |
| Additional resources beyond 150 per assessment | **$0.10/resource** |
| Additional failure mode assessments (beyond 2 included) | **$0.10/resource** (minimum 50 resources billed) |
| Automated Dependency Assessment (optional add-on) | **$10/service/month** |

[AWS Resilience Hub Pricing](https://awscloud.com/resilience-hub/pricing/){evidence: "Customers who want the expanded capabilities of the next generation of Resilience Hub (launched on May 28, 2026), are charged based on three factors: the number of services created in Resilience Hub, the number of failure mode assessments run, and whether dependency discovery assessment is enabled."}

### Pricing Examples

**Example 1:** 10 services, each <150 resources, 2 assessments/month
- 10 × $15 = **$150/month**

**Example 2:** 10 services, 25 resources each, 4 assessments/month
- Base: 10 × $15 = $150
- Extra assessments: 10 services × 50 resources (minimum) × 2 extra assessments × $0.10 = $100
- **Total: $250/month**

**Example 3:** 1 service, 100 resources, 2 assessments + dependency discovery
- Base: 1 × $15 = $15
- Dependency assessment: 1 × $10 = $10
- **Total: $25/month**

[AWS Resilience Hub Pricing](https://awscloud.com/resilience-hub/pricing/){evidence: "Service: Creating a service in AWS Resilience Hub is $15 per service per month (billing begins after completing the first failure mode assessment) and includes two failure mode assessments for services with 150 or fewer resources. Each additional resource beyond 150 is charged $0.10 during each failure mode assessment."}

---

## Summary

**Key Takeaways:**

1. **Major evolution in May 2026** — The next-generation Resilience Hub represents a fundamental shift from rule-based assessment to AI-powered resilience analysis, using generative AI to evaluate five critical failure modes and discover hidden dependencies.

2. **Service-centric model** — The move from "applications" to "services" reflects modern microservices architectures and enables more granular, composable resilience management with modular policies.

3. **Continuous validation is the core value proposition** — Whether via CI/CD integration, daily drift detection, or on-demand assessments, Resilience Hub's primary function is continuously proving (not assuming) that workloads meet their recovery objectives.

4. **Complementary, not competitive, with DR services** — Resilience Hub is the assessment/governance layer that validates the configurations of operational tools like AWS DRS, ARC, and Backup. It tells you *whether* you'll recover; those services *perform* the recovery.

5. **Actionable output** — Beyond reporting, Resilience Hub generates deployable CloudFormation templates for alarms, SOPs, and chaos experiments, making it a source of automation rather than just dashboards.

6. **Predictable pricing** — The next-gen model at $15/service/month plus per-assessment costs makes the service accessible for organizations of all sizes, with the dependency discovery add-on ($10/service/month) available for critical workloads requiring deeper analysis.

7. **AI-agent integration** — The framework now supports integration with Amazon Bedrock AgentCore to build autonomous resilience testing agents that continuously generate and validate chaos experiments — pointing toward fully autonomous resilience assurance.
