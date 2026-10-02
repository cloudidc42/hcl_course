# หลักสูตร HCL & Terraform Misconfiguration Analysis
## ตั้งแต่ระดับพื้นฐาน จนถึง ระดับมืออาชีพและระดับโลก

> **ภาษาไทย + English** | **100+ Parts** | **1000+ Steps** | **ใช้งานได้จริง 100%**

---

## เกี่ยวกับหลักสูตร

หลักสูตรนี้ครอบคลุมการเรียนรู้ **HashiCorp Configuration Language (HCL)** และ **Terraform** อย่างครบถ้วนตั้งแต่พื้นฐานจนถึงระดับมืออาชีพ พร้อมทั้งการวิเคราะห์ **Terraform Misconfiguration** ที่เกิดขึ้นในสภาพแวดล้อมจริง

### จุดเด่นของหลักสูตร

- **100+ Parts** ครอบคลุมทุกหัวข้อ
- **1000+ Steps** เรียนแบบ step-by-step
- **เนื้อหาใช้งานได้จริง 100%** ทุก example สามารถ run ได้ทันที
- **Security-focused** เน้นการวิเคราะห์และป้องกัน misconfiguration
- **Real-world scenarios** จากประสบการณ์จริงในองค์กรระดับโลก
- **Best practices** ตาม CIS Benchmarks, NIST, SOC2, PCI-DSS

---

## ข้อกำหนดเบื้องต้น (Prerequisites)

| ระดับ | ความรู้ที่ต้องการ |
|-------|-----------------|
| Beginner | ความรู้พื้นฐาน Linux/Terminal, ความเข้าใจ Cloud computing เบื้องต้น |
| Intermediate | มีประสบการณ์กับ Cloud providers (AWS/Azure/GCP) |
| Advanced | เข้าใจ DevOps practices, CI/CD pipeline |
| Professional | ประสบการณ์ Infrastructure management |

### เครื่องมือที่ต้องติดตั้ง

```bash
# Terraform
terraform version  # >= 1.5.0

# AWS CLI
aws --version  # >= 2.0

# Git
git --version

# Optional: Security tools
checkov --version
tfsec --version
terrascan version
```

---

## โครงสร้างหลักสูตร (Course Structure)

### 🎯 Tier 1: HCL Fundamentals (Parts 001-015)
> พื้นฐาน HCL Language สำหรับมือใหม่

| Part | หัวข้อ | Steps |
|------|--------|-------|
| [001](./part-001.md) | Introduction to HCL & Infrastructure as Code | 1-10 |
| [002](./part-002.md) | HCL Syntax - โครงสร้างพื้นฐาน | 11-20 |
| [003](./part-003.md) | HCL Data Types: Primitives | 21-30 |
| [004](./part-004.md) | HCL Data Types: Complex Types | 31-40 |
| [005](./part-005.md) | HCL Expressions และ Operators | 41-50 |
| [006](./part-006.md) | HCL Built-in Functions | 51-60 |
| [007](./part-007.md) | HCL Conditional Expressions | 61-70 |
| [008](./part-008.md) | HCL For Expressions | 71-80 |
| [009](./part-009.md) | HCL Dynamic Blocks | 81-90 |
| [010](./part-010.md) | HCL Template Syntax & Heredocs | 91-100 |
| [011](./part-011.md) | HCL Input Variables | 101-110 |
| [012](./part-012.md) | HCL Output Values | 111-120 |
| [013](./part-013.md) | HCL Local Values | 121-130 |
| [014](./part-014.md) | HCL Modules Overview | 131-140 |
| [015](./part-015.md) | HCL Best Practices | 141-150 |

### 🔧 Tier 2: Terraform Core (Parts 016-035)
> Terraform fundamentals และ CLI mastery

| Part | หัวข้อ | Steps |
|------|--------|-------|
| [016](./part-016.md) | Introduction to Terraform | 151-160 |
| [017](./part-017.md) | Terraform Installation & Configuration | 161-170 |
| [018](./part-018.md) | Terraform Providers | 171-180 |
| [019](./part-019.md) | Provider Configuration Deep Dive | 181-190 |
| [020](./part-020.md) | Terraform Resources | 191-200 |
| [021](./part-021.md) | Resource Meta-Arguments | 201-210 |
| [022](./part-022.md) | Terraform Data Sources | 211-220 |
| [023](./part-023.md) | Terraform State Basics | 221-230 |
| [024](./part-024.md) | State Storage & Backends | 231-240 |
| [025](./part-025.md) | Terraform CLI: init, fmt, validate | 241-250 |
| [026](./part-026.md) | Terraform CLI: plan & apply | 251-260 |
| [027](./part-027.md) | Terraform CLI: destroy & import | 261-270 |
| [028](./part-028.md) | Terraform State Commands | 271-280 |
| [029](./part-029.md) | Terraform Workspaces | 281-290 |
| [030](./part-030.md) | Terraform Debugging & Troubleshooting | 291-300 |
| [031](./part-031.md) | Terraform Graph & Dependencies | 301-310 |
| [032](./part-032.md) | Terraform Refresh & Reconciliation | 311-320 |
| [033](./part-033.md) | Terraform Console & Expressions | 321-330 |
| [034](./part-034.md) | Terraform Lock File | 331-340 |
| [035](./part-035.md) | Terraform Configuration Best Practices | 341-350 |

### ☁️ Tier 3: Cloud Providers (Parts 036-060)
> AWS, Azure, GCP provider mastery

| Part | หัวข้อ | Steps |
|------|--------|-------|
| [036](./part-036.md) | AWS Provider Setup & Authentication | 351-360 |
| [037](./part-037.md) | AWS EC2 Instances | 361-370 |
| [038](./part-038.md) | AWS VPC & Networking | 371-380 |
| [039](./part-039.md) | AWS Security Groups & NACLs | 381-390 |
| [040](./part-040.md) | AWS S3 Buckets | 391-400 |
| [041](./part-041.md) | AWS IAM Roles, Users & Policies | 401-410 |
| [042](./part-042.md) | AWS RDS Databases | 411-420 |
| [043](./part-043.md) | AWS EKS Kubernetes Clusters | 421-430 |
| [044](./part-044.md) | AWS Lambda Functions | 431-440 |
| [045](./part-045.md) | AWS CloudFront CDN | 441-450 |
| [046](./part-046.md) | AWS Route53 DNS | 451-460 |
| [047](./part-047.md) | AWS ElastiCache & Database Caching | 461-470 |
| [048](./part-048.md) | AWS SQS, SNS & Event-Driven Architecture | 471-480 |
| [049](./part-049.md) | AWS ECR & ECS Container Services | 481-490 |
| [050](./part-050.md) | AWS Load Balancers (ALB/NLB/CLB) | 491-500 |
| [051](./part-051.md) | Azure Provider Setup & Authentication | 501-510 |
| [052](./part-052.md) | Azure Resource Groups & Management | 511-520 |
| [053](./part-053.md) | Azure Virtual Networks | 521-530 |
| [054](./part-054.md) | Azure Virtual Machines | 531-540 |
| [055](./part-055.md) | Azure AKS Kubernetes Service | 541-550 |
| [056](./part-056.md) | Azure Storage Accounts | 551-560 |
| [057](./part-057.md) | Azure Database Services | 561-570 |
| [058](./part-058.md) | GCP Provider Setup & Authentication | 571-580 |
| [059](./part-059.md) | GCP Compute Engine & GKE | 581-590 |
| [060](./part-060.md) | Multi-Cloud Patterns | 591-600 |

### 🚀 Tier 4: Advanced Terraform (Parts 061-080)
> Advanced patterns, testing, enterprise features

| Part | หัวข้อ | Steps |
|------|--------|-------|
| [061](./part-061.md) | Variables: Deep Dive & Validation | 601-610 |
| [062](./part-062.md) | Complex Variable Types & Structures | 611-620 |
| [063](./part-063.md) | Output Values: Advanced Patterns | 621-630 |
| [064](./part-064.md) | Terraform Modules: Deep Dive | 631-640 |
| [065](./part-065.md) | Module Versioning & Registry | 641-650 |
| [066](./part-066.md) | Module Composition Patterns | 651-660 |
| [067](./part-067.md) | Count & For_each Mastery | 661-670 |
| [068](./part-068.md) | Lifecycle Rules & Resource Control | 671-680 |
| [069](./part-069.md) | Provider Aliasing & Multi-Region | 681-690 |
| [070](./part-070.md) | Custom Conditions & Preconditions | 691-700 |
| [071](./part-071.md) | Moved Blocks & Refactoring | 701-710 |
| [072](./part-072.md) | Terraform Testing Framework | 711-720 |
| [073](./part-073.md) | Terratest Integration Testing | 721-730 |
| [074](./part-074.md) | Terraform Cloud & Remote Operations | 731-740 |
| [075](./part-075.md) | Terraform Enterprise Features | 741-750 |
| [076](./part-076.md) | Sentinel Policy as Code | 751-760 |
| [077](./part-077.md) | OPA (Open Policy Agent) with Terraform | 761-770 |
| [078](./part-078.md) | Terraform Performance & Scale | 771-780 |
| [079](./part-079.md) | Terraform State Advanced Operations | 781-790 |
| [080](./part-080.md) | Terraform Drift Detection | 791-800 |

### 🔒 Tier 5: Security & Misconfiguration Analysis (Parts 081-095)
> Security deep dive, vulnerability analysis, compliance

| Part | หัวข้อ | Steps |
|------|--------|-------|
| [081](./part-081.md) | Introduction to IaC Security | 801-810 |
| [082](./part-082.md) | S3 Bucket Misconfigurations | 811-820 |
| [083](./part-083.md) | IAM Security Issues & Privilege Escalation | 821-830 |
| [084](./part-084.md) | VPC & Network Misconfigurations | 831-840 |
| [085](./part-085.md) | Encryption at Rest Misconfigurations | 841-850 |
| [086](./part-086.md) | Encryption in Transit Issues | 851-860 |
| [087](./part-087.md) | Logging & Monitoring Misconfigurations | 861-870 |
| [088](./part-088.md) | Public Exposure Vulnerabilities | 871-880 |
| [089](./part-089.md) | Compliance Frameworks: CIS, NIST, SOC2 | 881-890 |
| [090](./part-090.md) | Checkov Security Scanner | 891-900 |
| [091](./part-091.md) | TFSec Static Analysis | 901-910 |
| [092](./part-092.md) | Terrascan Security Scanner | 911-920 |
| [093](./part-093.md) | Snyk IaC Security | 921-930 |
| [094](./part-094.md) | Custom Security Rules & Policies | 931-940 |
| [095](./part-095.md) | Security CI/CD Integration | 941-950 |

### 🌟 Tier 6: Professional & World-class (Parts 096-100+)
> GitOps, enterprise patterns, world-class architecture

| Part | หัวข้อ | Steps |
|------|--------|-------|
| [096](./part-096.md) | GitOps with Terraform | 951-960 |
| [097](./part-097.md) | Atlantis PR Automation | 961-970 |
| [098](./part-098.md) | Terraform Patterns & Anti-patterns | 971-980 |
| [099](./part-099.md) | Cost Optimization with Terraform | 981-990 |
| [100](./part-100.md) | World-class Architecture Patterns | 991-1000 |

---

## วิธีการเรียน (Learning Path)

```
[Beginner]
   └── Parts 001-015 (HCL Fundamentals)
         └── Parts 016-035 (Terraform Core)
               └── [Intermediate]
                     └── Parts 036-060 (Cloud Providers)
                           └── Parts 061-080 (Advanced Terraform)
                                 └── [Advanced]
                                       └── Parts 081-095 (Security & Misconfig)
                                             └── Parts 096-100 (Professional)
                                                   └── [Professional/World-class]
```

---

## การติดตั้ง (Quick Setup)

```bash
# Clone repository
git clone <repo-url>
cd hcl_course

# ตรวจสอบ Terraform version
terraform version

# ตั้งค่า AWS credentials
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_DEFAULT_REGION="ap-southeast-1"

# เริ่มต้นใช้งาน
cd examples/part-001
terraform init
terraform plan
```

---

## License

MIT License - สามารถนำไปใช้งาน ปรับปรุง และแจกจ่ายได้อย่างอิสระ

---

*หลักสูตรนี้ถูกพัฒนาเพื่อนักพัฒนาและ DevOps Engineers ชาวไทย ที่ต้องการพัฒนาทักษะ IaC ในระดับมืออาชีพ*
