# Amazon Application Recovery Controller (ARC)

## Overview

Amazon Application Recovery Controller (ARC) is a fully managed AWS service that helps you prepare for and execute faster recovery for applications running on the AWS Global Cloud Infrastructure. ARC provides recovery capabilities spanning both single-AZ and multi-Region impairments, enabling teams to go from incident detection to recovery in seconds or minutes instead of hours.

[Amazon Application Recovery Controller](https://aws.amazon.com/application-recovery-controller/){evidence: "Amazon Application Recovery Controller (ARC) helps you recover applications quickly—whether the impairment is in a single Availability Zone or an entire Region. ARC provides fully managed recovery capabilities so your team spends less time writing failover scripts and more time building resilient architectures."}

---

## 1. Core Components

### Routing Controls

Routing controls are the foundational mechanism for multi-Region failover in ARC. They function as simple on/off switches that control Route 53 health checks, which in turn govern DNS-based traffic routing. When you flip a routing control to "Off," its associated health check becomes unhealthy, and Route 53 routes traffic away from that endpoint.

[About routing control - ARC Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/routing-control.about.html){evidence: "Routing controls support failover across any AWS service that has a DNS endpoint. You can update routing control states to fail over traffic for disaster recovery, or when you detect latency drops for your application, or other issues."}

Key characteristics of routing controls:

- **Not health checks themselves** — They don't monitor endpoint health. They are manual overrides that control health check states.
- **Fail over entire stacks** — Unlike component-level health checks, a routing control can shift traffic for an entire application stack at once.
- **Safety rules** — You can configure guardrails (e.g., "at least one replica must be active") to prevent accidental misconfiguration during failover.

[About routing control - ARC Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/routing-control.about.html){evidence: "A routing control is a simple on-off switch that controls a health check. Typically, you change the state to redirect traffic, and that state change moves the traffic to go to a particular endpoint for an entire application stack, or prevents routing to the whole application stack."}

### Clusters

An ARC cluster is a set of five redundant Regional endpoints distributed across five different AWS Regions. These endpoints ensure high availability for routing control operations — even if one or more endpoints are unavailable, you can still update routing control states through the remaining endpoints.

[Getting started with routing control - ARC Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/getting-started-cli-routing-config.html){evidence: "An ARC cluster is a set of five endpoints, one in each of five different AWS Regions. The ARC infrastructure supports these endpoints to work in coordination so that they guarantee high availability and sequential consistency of failover operations."}

### Control Panels

Control panels are logical groupings of routing controls. They help organize routing controls by application or environment, making it easier to manage failover for complex architectures with multiple layers.

[ARC routing control documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/routing-control.html){evidence: "The routing control components in ARC are: clusters, control panels, routing controls, and routing control health checks."}

### Readiness Checks

Readiness checks continuously monitor AWS resource quotas, capacity, and network routing policies to verify that your application replicas are prepared for failover. They audit resources across AZs or Regions to identify configuration drift or capacity shortfalls before a disaster strikes.

[What is ARC? - ARC Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/what-is-route53-recovery.html){evidence: "ARC readiness check continually monitors AWS resource quotas, capacity, and network routing policies, and can notify you about changes that may affect your ability to failover to a replica application and recover from Region impairment."}

**Note:** Readiness check is no longer open to new customers as of April 30, 2026. Existing customers can continue using the feature.

[ARC Readiness Check Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/introduction-components-readiness.html){evidence: "The readiness check feature in Amazon Application Recovery Controller (ARC) will no longer be open to new customers starting on April 30, 2026. Existing customers can continue to use the service as normal."}

### Recovery Groups

A recovery group represents your application in ARC. It contains cells (which map to independent application replicas — typically one per Region or AZ). Readiness checks are associated with recovery groups to validate that each cell has equivalent configuration and can handle failover traffic.

[Creating recovery groups - ARC Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/recovery-readiness.recovery-groups.html){evidence: "A recovery group represents your application in Amazon Application Recovery Controller (ARC)."}

### Zonal Shift

Zonal shift enables you to quickly recover from single-AZ impairments by temporarily shifting traffic for a supported resource (such as an ALB, NLB, or EKS cluster) away from an impaired AZ to healthy AZs in the same Region.

[What is ARC? - ARC Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/what-is-route53-recovery.html){evidence: "You can use ARC zonal shift to quickly isolate and recover from single Availability Zone (AZ) impairments. Zonal shift temporarily shifts traffic for a supported resource away from an impaired AZ to healthy AZs in the same AWS Region."}

Key details:

- **Manual and temporary** — You specify an expiration up to 3 days (extendable).
- **Immediate isolation** — Helps recover from bad code deployments or AZ-level AWS impairments.

### Zonal Autoshift

Zonal autoshift is the automated counterpart to zonal shift. AWS proactively monitors internal telemetry and, when it detects an AZ impairment that could impact customers, it automatically shifts traffic away from the affected AZ — no operator intervention required.

[What is ARC? - ARC Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/what-is-route53-recovery.html){evidence: "ARC zonal autoshift authorizes AWS to shift traffic away from an impaired AZ for supported resources, on your behalf, to healthy AZs in the same AWS Region. AWS starts a zonal autoshift when internal telemetry indicates that there is an impairment in one AZ in an AWS Region that could potentially impact customers."}

Zonal autoshift operates on the principle of **static stability** — your application must be pre-scaled across multiple AZs to absorb the full loss of capacity in any single zone.

[Configuring ARC zonal autoshift observer notifications](https://aws.amazon.com/de/blogs/networking-and-content-delivery/configuring-amazon-application-recovery-controller-zonal-autoshift-observer-notifications/){evidence: "Zonal autoshift allows AWS to automatically shift traffic away from an AZ when AWS detects a potential failure there. It operates on the principle of static stability, where your application is pre-scaled across multiple AZs to handle the complete loss of capacity in any single zone."}

ARC also conducts **practice runs** to validate that shifting traffic away from an AZ is safe for your application before a real autoshift event occurs.

[ARC zonal autoshift documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/arc-zonal-autoshift.html){evidence: "The zonal shifts that ARC starts for practice runs help you to ensure that shifting away traffic from an Availability Zone during an autoshift is safe for your application."}

### Region Switch

Region switch is ARC's newest capability for orchestrating full-stack, multi-Region recovery. It provides a centralized, automated, and observable solution built around the concept of **plans**.

[Region switch in ARC](https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch.html){evidence: "Region switch is built around the concept of a plan, which you design and configure for your specific recovery needs. Each plan includes workflows that are made up of steps. Each step runs one or more execution blocks, which Region switch runs in parallel or in sequence, to complete an application recovery."}

Key features:

- **Execution blocks** — Modular actions for resource switching, traffic redirection, database failover, etc.
- **Cross-account support** — Orchestrate recovery across multiple AWS accounts.
- **Automatic triggers** — Execute plans based on Amazon CloudWatch alarms, or run them manually.
- **Active/passive and active/active** — Supports both configurations.
- **Regional data plane** — Plan execution doesn't depend on the impaired Region.

[Region switch in ARC](https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch.html){evidence: "Automatic failover or switchover, by triggering plan execution based on Amazon CloudWatch alarms."}

---

## 2. How ARC Orchestrates Failover

### Multi-Region Failover

For multi-Region applications, ARC provides two approaches:

**Routing Control (manual/API-driven):** You create routing controls associated with Route 53 health checks for each Region's endpoint. During failover, you update routing control states (Off for impaired Region, On for recovery Region) via the cluster data plane API. The data plane is distributed across five Regions for extreme reliability — you're never dependent on the impaired Region.

[Orchestrate DR automation blog](https://aws.amazon.com/fr/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/){evidence: "Route 53 ARC offers extreme reliability with its data plane to fail over the application during a regional impairment. Route53 ARC maintains the routing control states in a cluster, which is a set of five Regional endpoints. We can interact with any one of the cluster endpoints to update the state of a routing control, and it gets propagated across the five Regions of the cluster."}

**Region Switch (fully orchestrated):** Region switch plans define the complete sequence of failover actions — stopping traffic to the primary, promoting databases, enabling resources in the recovery Region, and redirecting traffic. Plans can be triggered manually or automatically via CloudWatch alarms.

[Region switch plans - ARC Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch-plans.html){evidence: "A plan enables you to build workflows to recover your applications by running a series of Region switch execution blocks that activate or deactivate your application and its resources, including cross-account resources, in the AWS Region that you specify."}

### Multi-AZ Failover

For single-Region multi-AZ applications, ARC uses zonal shift and zonal autoshift to redirect traffic away from an impaired AZ. This works at the load balancer or resource level — for example, shifting an ALB's traffic targets away from one AZ to the remaining healthy AZs. For EKS clusters, this updates east-to-west network traffic to only consider Pod endpoints in healthy AZs.

[Amazon EKS supports ARC](https://aws.amazon.com/jp/blogs/containers/amazon-eks-now-supports-amazon-application-recovery-controller/){evidence: "The ARC zonal shift and zonal autoshift capabilities achieve multi-AZ recovery for a supported resource by shifting the ingress traffic away from an impaired AZ to other healthy AZs. When the shift ends, ARC adds back the previously impacted AZ so that it can again receive ingress traffic."}

### Control Plane vs. Data Plane

ARC maintains a clear separation:

- **Control plane** (us-west-2): Used for creating and deleting ARC resources (clusters, routing controls, etc.)
- **Data plane** (5 Regions): Used for updating routing control states — the critical path during failover

[Orchestrate DR automation blog](https://aws.amazon.com/fr/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/){evidence: "The Control plane is located in the us-west-2 (Oregon) Region, which enables us to create and delete resources in the ARC Cluster, and the Data plane is available in 5 regions, which provides the service's core functionality. To be precise, any 'creation & deletion' of ARC Routing Controls are Control Plane operations, and any 'updates' to ARC Routing Controls are a Data Plane operation."}

---

## 3. Active-Active vs Active-Standby Patterns with ARC

### Active-Standby (Active/Passive)

In an active-standby pattern, one Region handles all traffic while the other remains provisioned but idle. ARC routing controls enable rapid switchover when the primary Region is impaired.

[Building highly resilient applications - Part 2: Multi-Region stack](https://aws.amazon.com/blogs/networking-and-content-delivery/building-highly-resilient-applications-using-amazon-route-53-application-recovery-controller-part-2-multi-region-stack){evidence: "The multi-Region stack design, as shown in Figure 1, supports an active-standby setup. In this design, the primary (active) Region is US East (N Virginia) or us-east-1, and the recovery (standby) Region is us-west-2."}

Typical active-standby failover sequence:

1. Turn off routing controls for the primary Region (stop incoming traffic)
2. Promote standby databases (e.g., Aurora Global Database failover)
3. Turn on routing controls for the recovery Region (start receiving traffic)

### Active-Active

In an active-active configuration, both Regions serve traffic simultaneously. Recovery involves shifting users away from the impaired Region — no database promotion is needed if both Regions have writable replicas.

[About routing control - ARC Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/routing-control.about.html){evidence: "If you want to enable faster recoveries, another option that you can choose for your architecture is an active-active implementation. With this approach, your replicas are active at the same time. This means that you can recover from failures by moving users away from an impaired application replica by just rerouting traffic to another active replica."}

ARC Region switch explicitly supports both patterns:

[Region switch in ARC](https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch.html){evidence: "Support for active/passive and active/active configurations. You can failover and failback if you have an active/passive multi-Region configurations, or shift-away and return if your application is set up as active/active in multiple Regions."}

### Trade-offs

| Aspect | Active-Standby | Active-Active |
| --- | --- | --- |
| Recovery speed | Minutes (DB promotion needed) | Seconds (just reroute traffic) |
| Cost | Lower (standby under-provisioned) | Higher (both Regions fully scaled) |
| Data consistency | Simpler (single writer) | Complex (conflict resolution needed) |
| ARC approach | Failover + failback | Shift-away + return |

---

## 4. Integration with Route 53, Global Accelerator, and ELB

### Route 53 Integration

ARC routing controls are deeply integrated with Route 53 through **routing control health checks**. The pattern works as follows:

1. Create routing controls in an ARC cluster
2. Create Route 53 health checks associated with those routing controls
3. Associate those health checks with DNS failover records (e.g., Alias records with failover routing policies) pointing to your Regional endpoints (typically load balancers)

When you update a routing control state, ARC updates the associated health check to healthy/unhealthy, and Route 53's DNS routing responds accordingly.

[About routing control - ARC Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/routing-control.about.html){evidence: "Routing control redirects traffic by using health checks in Amazon Route 53 that are configured with DNS records associated with the top-level resource of the cells in your recovery group, such as an Elastic Load Balancing load balancer. You can redirect traffic from one cell to another, for example, by updating a routing control state to Off (to stop traffic flow to one cell) and updating another routing control state to On (to start traffic flow to another)."}

You can also use **weighted routing policies** with Route 53 for more complex traffic distribution scenarios (e.g., gradually shifting traffic during a planned failover).

[About routing control - ARC Documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/routing-control.about.html){evidence: "You can also set up more complex traffic failover scenarios by using ARC routing control together with Route 53 health checks and DNS record sets, using DNS records with weighted routing policies."}

**DNS TTL consideration:** AWS recommends setting TTLs of 60 or 120 seconds for failover records to ensure traffic shifts complete quickly after a routing control state change.

[ARC best practices](https://docs.aws.amazon.com/r53recovery/latest/dg/route53-arc-best-practices.regional.html){evidence: "Setting a TTL of 60 or 120 seconds is a common choice for this scenario."}

### Elastic Load Balancing (ELB) Integration

ELB load balancers (ALB and NLB) serve as the typical top-level resources that ARC routes traffic to or away from. In multi-Region architectures, each Region has its own load balancer, and Route 53 failover records point to these load balancers using the ARC routing control health checks.

For multi-AZ scenarios, ALBs and NLBs are directly supported by zonal shift and zonal autoshift — ARC can shift load balancer targets away from an impaired AZ without any DNS changes.

[ARC zonal shift documentation](https://docs.aws.amazon.com/r53recovery/latest/dg/what-is-route53-recovery.html){evidence: "Starting a zonal shift helps your application recover quickly, for example, from a developer's bad code deployment or from an AWS impairment in a single AZ. Shifting traffic away from the impaired AZ reduces the impact for clients who are using your application in the impaired AZ."}

### Global Accelerator Integration

AWS Global Accelerator can also be used alongside ARC for routing control. Global Accelerator provides static anycast IP addresses and routes traffic to optimal endpoints. When combined with ARC routing controls, you can manage failover at the network layer (via Global Accelerator endpoint groups) rather than relying solely on DNS-based failover, which can be subject to TTL caching delays.

### Cross-Region Failover and Graceful Failback Solution

AWS provides a reference architecture that combines ARC with Route 53 and Systems Manager for automated failover:

[Cross-Region Failover and Graceful Failback on AWS](https://docs.aws.amazon.com/solutions/cross-region-failover-and-graceful-failback-on-aws/){evidence: "Amazon Route53 Application Recovery Controller routes traffic between multiple Regions and automates failover through integration with AWS Systems Manager documents."}

---

## 5. Step Functions DR Automation with ARC

AWS provides a detailed blueprint for orchestrating disaster recovery automation using Step Functions with ARC. This pattern eliminates manual runbook execution and ensures consistent, repeatable failover.

### Architecture

The solution deploys failover and failback Step Function state machines in both primary and standby Regions. These state machines use Lambda functions to interact with ARC's routing control APIs and DynamoDB global tables to store ARC cluster configuration.

[Orchestrate DR automation blog](https://aws.amazon.com/fr/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/){evidence: "These sample step functions use a custom Lambda function and global DynamoDB tables to automate the Route 53 ARC Routing Controls into on/off states, which plays an important role in managing the failover and failback of AWS service entry points."}

### Key Components

1. **DynamoDB Global Tables** — Store ARC cluster endpoints, control panel ARN, and the ordered sequence of routing controls for failover/failback
2. **Lambda Functions** — Execute ARC API calls to update routing control states, cycling through cluster endpoints for reliability
3. **Step Functions** — Orchestrate the ordered sequence of actions (stop traffic → failover DB → enable standby)
4. **Embedded child Step Functions** — Handle specific resource failovers (e.g., RDS global cluster promotion)

### Failover Sequence (Automated)

The Step Functions DR automation follows this precise order:

[Orchestrate DR automation blog](https://aws.amazon.com/fr/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/){evidence: "1. Stop accepting traffic in the primary Region (us-east-1) to prevent writes to the database while you're failing it over. 2. Fail over your database to the recovery Region (us-west-2) and make sure it's ready to accept writes. 3. Route user traffic to us-west-2, making it the new active Region."}

### Best Practices

- **Random endpoint selection** — The Lambda function cycles through ARC's 5 cluster endpoints, choosing a random one and retrying if it fails
- **Console independence** — All configuration stored in DynamoDB, eliminating dependency on AWS Console during an incident
- **Extensibility** — Child step functions can be added for additional resources (ElastiCache, OpenSearch, MSK)
- **Region independence** — The failover state machine can be triggered from the standby Region, ensuring it's not dependent on the impaired Region

[Orchestrate DR automation blog](https://aws.amazon.com/fr/blogs/networking-and-content-delivery/orchestrate-disaster-recovery-automation-using-amazon-route-53-arc-and-aws-step-functions/){evidence: "However, a robust failover mechanism should be independent of the Region we are trying to get out of, so it is recommended to programmatically manage routing state changes using Amazon Route 53 ARC API operations via one of the AWS SDKs."}

---

## 6. Real-World Architecture Patterns

### Pattern 1: Multi-Region Active-Standby with Full-Stack Failover

- Primary Region (us-east-1) serves all traffic
- Standby Region (us-west-2) with identical infrastructure, scaled down or at parity
- ARC routing controls manage traffic via Route 53 failover records
- Aurora Global Database with cross-Region replication
- Step Functions automate the failover sequence
- Readiness checks validate standby readiness continuously

[Building highly resilient applications - Part 2](https://aws.amazon.com/blogs/networking-and-content-delivery/building-highly-resilient-applications-using-amazon-route-53-application-recovery-controller-part-2-multi-region-stack){evidence: "Similarly, we leveraged Route 53 ARC routing controls and Route 53 health checks to allow application failover. The primary and recovery Regions each have cell boundaries defined around Availability Zones (AZs) in the respective Regions."}

### Pattern 2: Multi-AZ with Zonal Autoshift

- Single Region with 3 AZs
- Application pre-scaled to handle loss of one AZ (N+1 or N/3 capacity per AZ)
- Zonal autoshift enabled for ALBs/NLBs
- AWS automatically detects AZ impairment and shifts traffic
- Practice runs validate safety continuously

### Pattern 3: Hierarchical Cell Architecture

ARC supports nested cells within recovery groups:

- A Region-level cell contains sub-cells for each AZ
- Routing controls exist at both Region and AZ granularity
- This enables both intra-Region (AZ-level) and inter-Region failover from the same ARC configuration

### Pattern 4: Toyota's Multi-Service Region Switch

Toyota uses ARC Region switch to orchestrate disaster recovery across 10+ services, replacing manual runbooks with automated, single-click orchestration.

[Amazon Application Recovery Controller](https://aws.amazon.com/application-recovery-controller/){evidence: "ARC Region Switch transformed our disaster recovery from a manual, time-consuming process into an automated orchestration across 10+ services. Pre-failover validation and single-click orchestration freed our Operations team from tedious runbook maintenance, eliminating any chances of human error during execution."}

### Pattern 5: EKS Multi-AZ Recovery

Amazon EKS supports ARC zonal shift and zonal autoshift natively. When triggered, the shift updates in-cluster east-to-west network traffic to only consider Pod endpoints in healthy AZs.

[Amazon EKS zonal shift documentation](https://docs.aws.amazon.com/en_us/eks/latest/userguide/zone-shift.html){evidence: "You can start a zonal shift for an EKS cluster, or you can allow AWS to shift traffic for you by enabling zonal autoshift. This shift updates the flow of east-to-west network traffic in your cluster to only consider network endpoints for Pods running on worker nodes in healthy AZs."}

### Pattern 6: Region Switch with CloudWatch Alarm Triggers

For fully automated multi-Region failover:

- CloudWatch alarms monitor application health metrics
- When alarms breach thresholds, they automatically trigger Region switch plan execution
- Plans orchestrate the complete failover sequence without human intervention
- Real-time dashboards provide visibility into recovery progress

[Region switch in ARC](https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch.html){evidence: "Full-featured dashboards that give you real-time visibility into the recovery process."}

---

## 7. Pricing

ARC pricing is pay-as-you-go with no upfront charges or long-term commitments:

### Single-Region Recovery

| Feature | Cost |
| --- | --- |
| **Zonal shift** | No additional charge |
| **Zonal autoshift** | No additional charge |

### Multi-Region Recovery

| Feature | Cost |
| --- | --- |
| **Region switch plan** | $70 per plan per month |
| **Routing control cluster** | $2.50 per hour per cluster (~$1,800/month) |
| **Readiness check** | $0.045 per hour per check (~$32.40/month) |

[Amazon Application Recovery Controller Pricing](https://aws.amazon.com/application-recovery-controller/pricing/){evidence: "There is no additional charge for using zonal shift or zonal autoshift."}

[Amazon Application Recovery Controller Pricing](https://aws.amazon.com/application-recovery-controller/pricing/){evidence: "Region switch plan | $70 per plan per month | Prices are prorated for partial months. No upfront costs or long-term commitments."}

[Amazon Application Recovery Controller Pricing](https://aws.amazon.com/application-recovery-controller/pricing/){evidence: "Cluster | $2.50 per hour per cluster | Each cluster (maximum 2 per account) can host multiple routing controls."}

[Amazon Application Recovery Controller Pricing](https://aws.amazon.com/application-recovery-controller/pricing/){evidence: "Readiness check | $0.045 per hour per readiness check | For example, if you have two readiness checks configured (one for Auto Scaling Groups and one for DynamoDB tables), your total readiness check cost is $0.09 per hour."}

### Pricing Notes

- Each cluster can host multiple routing controls — no per-control charge
- Maximum 2 clusters per account
- Region switch plans are prorated for partial months
- Zonal shift/autoshift is free, making it an excellent low-cost resilience addition
- Readiness check is being retired for new customers (April 30, 2026)

---

## Summary

**Key Takeaways:**

1. **ARC is a comprehensive recovery platform** spanning both multi-AZ (zonal shift/autoshift) and multi-Region (routing controls, Region switch) failure scenarios, all under one service umbrella.
2. **Zonal shift and autoshift are free and powerful** — they provide immediate, automated AZ-level recovery with zero additional cost, making them a no-brainer for any multi-AZ application.
3. **Routing controls provide extreme reliability** via a 5-Region distributed data plane, ensuring you can always execute failover even when the impaired Region is completely unavailable.
4. **Region switch is the evolution** — it replaces custom Step Functions + Lambda automation with a fully managed orchestration service that handles cross-account, multi-service recovery with built-in observability and CloudWatch alarm triggers.
5. **Route 53 is the traffic backbone** — ARC's routing controls work through Route 53 health checks and DNS failover records, meaning any service with a DNS endpoint can be failed over.
6. **Both active-active and active-standby are first-class patterns** — ARC supports failover/failback for active-standby and shift-away/return for active-active configurations.
7. **Cost ranges from free (zonal) to ~$1,800/month (routing control cluster)** — with Region switch plans at $70/month offering a more affordable entry point for managed multi-Region orchestration.
8. **Automation is essential** — AWS recommends automating failover via Step Functions, Lambda, or Region switch plans rather than relying on console-based manual actions during an incident.

