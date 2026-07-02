# 🛡️ AWS Disaster Recovery & Resilience — Complete Project

[![AWS](https://img.shields.io/badge/AWS-Resilience-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/resilience/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge)](LICENSE)
[![Labs](https://img.shields.io/badge/Labs-6-green?style=for-the-badge)](labs/)

> A comprehensive hands-on project covering AWS DR strategies (Backup & Restore, Pilot Light, Warm Standby, Active-Active) using AWS Resilience Hub, Amazon ARC, AWS DRS, AWS Backup, and Service Screener v2.

## 🎯 What's Inside

| Resource | Description |
|----------|-------------|
| 📄 [Research Report](docs/research-report.md) | 5000+ word deep dive on all services with citations |
| 🧪 [Hands-On Labs](labs/) | 6 labs × (Simple + Complex) setups with step-by-step CLI |
| 📋 [Decision Framework](decision-framework/) | Interactive HTML tool for choosing DR strategies |
| 📊 [Presentation](presentation/) | 27-slide PPTX covering theory + architecture |
| 🔧 [IaC Templates](terraform/) | Terraform & CloudFormation for each strategy |
| 🤝 [OSS Contribution](oss-contribution/) | Proposed PRs to AWS Service Screener v2 |

## 🏗️ DR Strategies Covered

```
┌──────────────────────────────────────────────────────────────────────────┐
│  💲          💲💲           💲💲💲           💲💲💲💲                      │
│  Backup &    Pilot          Warm            Active-                      │
│  Restore     Light           Standby         Active                      │
│                                                                          │
│  RTO: <24h   RTO: ~10min    RTO: min        RTO: ~0                     │
│  RPO: hours  RPO: min       RPO: sec        RPO: ~0                     │
│                                                                          │
│  ◀──── Increasing Cost & Complexity ────▶                               │
│  ◀──── Decreasing RTO/RPO ──────────────▶                               │
└──────────────────────────────────────────────────────────────────────────┘
```

## 🔧 AWS Services

| Service | Role | Strategy |
|---------|------|----------|
| **AWS Resilience Hub** | Assessment & Governance | All strategies |
| **Amazon ARC** | Failover Orchestration | Warm Standby, Active-Active |
| **AWS DRS** | Server Replication | Pilot Light, Warm Standby |
| **AWS Backup** | Data Protection | Backup & Restore |
| **Service Screener v2** | Configuration Assessment | Pre-DR validation |

## 🧪 Labs Overview

| Lab | Service | Simple Setup | Complex Setup |
|-----|---------|-------------|---------------|
| 1 | AWS Backup | Single EC2 backup + restore | Multi-account, vault lock, cross-region |
| 2 | AWS DRS | 1 server replication + drill | Cross-region multi-server, automated failover |
| 3 | Amazon ARC | Zonal shift for ALB | Routing controls + Region Switch + DynamoDB Global |
| 4 | Resilience Hub | Basic policy + assessment | Next-gen, dependency discovery, CI/CD gate |
| 5 | End-to-End | EC2+RDS with Route 53 | 3-tier app, full disaster simulation |
| 6 | Service Screener | Single-region scan | Multi-region + Well-Architected integration |

## 🚀 Quick Start

```bash
# Clone the repo
git clone https://github.com/aquavis12/aws-dr-resilience-project.git
cd aws-dr-resilience-project

# Start with Lab 1 (Simple Setup)
cd labs/lab1-aws-backup
cat simple-setup.md

# Or run Service Screener for a baseline assessment
cd labs/lab6-service-screener
bash setup.sh
```

## 💰 Cost Estimates

| Setup Tier | Total Cost | Duration |
|-----------|-----------|----------|
| Simple (all labs) | $20–45 | 4–6 hours |
| Complex (all labs) | $135–340 | 2–3 days |

> ⚠️ Always set billing alerts and clean up resources after labs!

## 🤝 OSS Contribution: Service Screener v2

This project includes proposed contributions to [aws-samples/service-screener-v2](https://github.com/aws-samples/service-screener-v2):

- **AWS Backup checks** — vault lock, cross-region copy, backup plan frequency
- **AWS DRS checks** — replication health, lag monitoring, PIT retention
- **ARC checks** — routing controls, zonal shift readiness, practice runs

See [oss-contribution/](oss-contribution/) for implementation details.

## 📂 Repository Structure

```
aws-dr-resilience-project/
├── README.md
├── docs/
│   ├── research-report.md          # Comprehensive research (citations)
│   └── service-deep-dives/         # Per-service analysis
├── labs/
│   ├── lab1-aws-backup/            # Simple + Complex
│   ├── lab2-aws-drs/
│   ├── lab3-amazon-arc/
│   ├── lab4-resilience-hub/
│   ├── lab5-end-to-end/
│   └── lab6-service-screener/
├── decision-framework/
│   └── index.html                  # Interactive decision tool
├── presentation/
│   └── aws-dr-resilience.pptx      # 27-slide deck
├── terraform/
│   └── modules/                    # Per-strategy IaC
├── cloudformation/                 # CFN templates
├── scripts/                        # Helper scripts
├── oss-contribution/
│   └── service-screener-v2/        # Proposed PR code
└── diagrams/                       # Architecture diagrams
```

## 👤 Author

**Venkata Pavan Vishnu Rachapudi**  
AWS Community Builder (Security) | 14× AWS Certified  
[GitHub](https://github.com/aquavis12) | [LinkedIn](#)

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- [AWS Well-Architected Framework — Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/)
- [AWS Service Screener v2](https://github.com/aws-samples/service-screener-v2)
- [AWS DR Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/)
