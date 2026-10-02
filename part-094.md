# Part 94: Custom Security Rules & Policies (Steps 931-940)

## การเขียน Custom Security Rules สำหรับ Terraform

---

## Step 931: Overview - ทำไมต้องเขียน Custom Rules?

### Built-in Rules ไม่เพียงพอเสมอไป

```
ปัญหาที่ built-in rules ตอบไม่ได้:
┌─────────────────────────────────────────────────────────────┐
│ ✗ องค์กรมี tagging policy เฉพาะของตัวเอง                  │
│ ✗ ใช้ได้เฉพาะ approved instance types เท่านั้น             │
│ ✗ ชื่อ resource ต้องเป็นไปตาม naming convention           │
│ ✗ Deploy ได้เฉพาะ approved regions เท่านั้น                │
│ ✗ Cost limit per resource type                             │
│ ✗ Company-specific security requirements                   │
└─────────────────────────────────────────────────────────────┘
```

### Tools ที่รองรับ Custom Rules

```
┌──────────────┬─────────────────────────────────────────────┐
│ Tool         │ Custom Rule Language                        │
├──────────────┼─────────────────────────────────────────────┤
│ Checkov      │ Python (BaseResourceCheck) + YAML           │
│ TFSec/Trivy  │ Rego (OPA)                                  │
│ Terrascan    │ Rego (OPA)                                  │
│ Sentinel     │ HashiCorp Sentinel (HCL-like)               │
│ Conftest     │ Rego (OPA)                                  │
│ OPA          │ Rego                                        │
└──────────────┴─────────────────────────────────────────────┘
```

---

## Step 932: Custom Checkov Rules ด้วย Python

### Project Structure

```
my-terraform-project/
├── terraform/
│   ├── main.tf
│   └── ...
├── custom_checks/
│   ├── __init__.py
│   ├── check_mandatory_tags.py
│   ├── check_approved_instance_types.py
│   ├── check_naming_convention.py
│   └── check_approved_regions.py
└── .checkov.yaml
```

### BaseResourceCheck Structure

```python
# custom_checks/check_mandatory_tags.py
from checkov.common.models.enums import CheckResult, CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck
from typing import Dict, List, Any

class MandatoryTagsCheck(BaseResourceCheck):
    """
    Custom Checkov check: Ensure all resources have mandatory tags.
    
    Tags required:
    - Owner: team/person responsible
    - Environment: dev/staging/prod
    - Project: project name
    - CostCenter: cost center code
    """
    
    def __init__(self):
        # กำหนด check metadata
        name = "Ensure mandatory tags are present on all AWS resources"
        id = "CKV_CUSTOM_1"                    # ID ต้องไม่ซ้ำ
        categories = [CheckCategories.GENERAL_SECURITY]
        
        # Resource types ที่ต้องการตรวจสอบ
        supported_resources = [
            "aws_instance",
            "aws_s3_bucket", 
            "aws_db_instance",
            "aws_lb",
            "aws_elasticache_cluster",
            "aws_eks_cluster",
            "aws_lambda_function",
        ]
        
        super().__init__(
            name=name, 
            id=id, 
            categories=categories,
            supported_resources=supported_resources
        )
    
    def scan_resource_conf(self, conf: Dict[str, List[Any]]) -> CheckResult:
        """
        ตรวจสอบ resource configuration
        conf: dict ที่มี Terraform resource attributes
        """
        required_tags = {"Owner", "Environment", "Project", "CostCenter"}
        
        # ดึง tags จาก configuration
        tags = conf.get("tags", [{}])
        if isinstance(tags, list):
            tags = tags[0] if tags else {}
        
        if not tags or not isinstance(tags, dict):
            self.evaluated_keys = ["tags"]
            return CheckResult.FAILED
        
        # ตรวจสอบว่ามี tags ครบ
        existing_tags = set(tags.keys())
        missing_tags = required_tags - existing_tags
        
        if missing_tags:
            self.evaluated_keys = ["tags"]
            return CheckResult.FAILED
        
        return CheckResult.PASSED


# สร้าง instance เพื่อ register check
check = MandatoryTagsCheck()
```

### Custom Check: Approved Instance Types

```python
# custom_checks/check_approved_instance_types.py
from checkov.common.models.enums import CheckResult, CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck
from typing import Dict, List, Any

APPROVED_INSTANCE_TYPES = {
    # General Purpose - ประหยัด
    "t3.micro", "t3.small", "t3.medium", "t3.large",
    "t3.xlarge", "t3.2xlarge",
    # Memory Optimized - สำหรับ database
    "m5.large", "m5.xlarge", "m5.2xlarge", "m5.4xlarge",
    # Compute Optimized - สำหรับ processing
    "c5.large", "c5.xlarge", "c5.2xlarge",
    # Dev only
    "t2.micro", "t2.small",
}

class ApprovedInstanceTypesCheck(BaseResourceCheck):
    """
    ตรวจสอบว่าใช้ instance types ที่ได้รับอนุมัติเท่านั้น
    """
    
    def __init__(self):
        name = "Ensure only approved EC2 instance types are used"
        id = "CKV_CUSTOM_2"
        categories = [CheckCategories.GENERAL_SECURITY]
        supported_resources = ["aws_instance"]
        
        super().__init__(
            name=name,
            id=id,
            categories=categories,
            supported_resources=supported_resources
        )
    
    def scan_resource_conf(self, conf: Dict[str, List[Any]]) -> CheckResult:
        instance_type = conf.get("instance_type", [None])
        if isinstance(instance_type, list):
            instance_type = instance_type[0]
        
        if not instance_type:
            return CheckResult.UNKNOWN
        
        # Handle variables (e.g., "${var.instance_type}")
        if instance_type.startswith("${") or instance_type.startswith("var."):
            return CheckResult.UNKNOWN
        
        if instance_type in APPROVED_INSTANCE_TYPES:
            return CheckResult.PASSED
        
        return CheckResult.FAILED


check = ApprovedInstanceTypesCheck()
```

### Custom Check: Naming Convention

```python
# custom_checks/check_naming_convention.py
import re
from checkov.common.models.enums import CheckResult, CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck
from typing import Dict, List, Any

# Convention: {env}-{service}-{resource_type}-{seq}
# Examples:
#   prod-api-sg-001     (security group)
#   dev-payments-ec2-001 (EC2 instance)
#   staging-web-s3-001  (S3 bucket)
NAMING_PATTERN = r'^(dev|staging|prod|sandbox)-[a-z0-9-]+-[a-z0-9-]+-\d{3}$'

class NamingConventionCheck(BaseResourceCheck):
    """
    บังคับใช้ naming convention สำหรับ resources
    """
    
    # Mapping: resource_type -> attribute ที่เก็บชื่อ
    NAME_ATTRIBUTE = {
        "aws_instance": "tags.Name",
        "aws_s3_bucket": "bucket",
        "aws_security_group": "name",
        "aws_db_instance": "identifier",
        "aws_lb": "name",
        "aws_vpc": "tags.Name",
        "aws_subnet": "tags.Name",
    }
    
    def __init__(self):
        name = "Ensure resource names follow naming convention"
        id = "CKV_CUSTOM_3"
        categories = [CheckCategories.GENERAL_SECURITY]
        supported_resources = list(self.NAME_ATTRIBUTE.keys())
        
        super().__init__(
            name=name,
            id=id, 
            categories=categories,
            supported_resources=supported_resources
        )
    
    def scan_resource_conf(self, conf: Dict[str, List[Any]]) -> CheckResult:
        resource_name = self._get_resource_name(conf)
        
        if not resource_name:
            # ถ้าหา name ไม่เจอ ให้ PASS (เพื่อไม่ทำให้ noisy เกินไป)
            return CheckResult.UNKNOWN
        
        # Skip if it's a variable reference
        if resource_name.startswith("${") or resource_name.startswith("var."):
            return CheckResult.UNKNOWN
        
        if re.match(NAMING_PATTERN, resource_name):
            return CheckResult.PASSED
        
        return CheckResult.FAILED
    
    def _get_resource_name(self, conf: Dict) -> str:
        """ดึงชื่อ resource จาก config"""
        # Try 'name' attribute
        name = conf.get("name", [None])
        if isinstance(name, list):
            name = name[0] if name else None
        
        # Try 'bucket' for S3
        if not name:
            name = conf.get("bucket", [None])
            if isinstance(name, list):
                name = name[0] if name else None
        
        # Try 'identifier' for RDS
        if not name:
            name = conf.get("identifier", [None])
            if isinstance(name, list):
                name = name[0] if name else None
        
        return name


check = NamingConventionCheck()
```

### Custom Check: YAML Format (ใหม่กว่า)

```yaml
# custom_checks/check_s3_intelligent_tiering.yaml
metadata:
  name: "Ensure S3 buckets have Intelligent Tiering configured"
  id: "CKV2_CUSTOM_1"
  category: "GENERAL_SECURITY"
  
definition:
  and:
    - cond_type: "filter"
      resource_types:
        - "aws_s3_bucket"
      
    - cond_type: "attribute"
      resource_types:
        - "aws_s3_bucket_intelligent_tiering_configuration"
      attribute: "status"
      operator: "equals"
      value: "Enabled"
```

### รัน Custom Checks

```bash
# ===== วิธีที่ 1: Directory flag =====
checkov -d . \
  --external-checks-dir ./custom_checks \
  --framework terraform

# ===== วิธีที่ 2: .checkov.yaml config =====
cat > .checkov.yaml << 'EOF'
# .checkov.yaml
framework:
  - terraform
external-checks-dir:
  - ./custom_checks
check:
  - CKV_CUSTOM_1
  - CKV_CUSTOM_2
  - CKV_CUSTOM_3
EOF

checkov -d .

# ===== วิธีที่ 3: Run specific custom check only =====
checkov -d . \
  --external-checks-dir ./custom_checks \
  --check CKV_CUSTOM_1

# ===== วิธีที่ 4: Python runner =====
cat > run_checks.py << 'EOF'
from checkov.terraform.runner import Runner
from checkov.runner_registry import RunnerRegistry

# Import custom checks
import custom_checks.check_mandatory_tags
import custom_checks.check_approved_instance_types

runner = Runner()
report = runner.run(root_folder='./terraform')
print(report.print_json())
EOF
python run_checks.py
```

---

## Step 933: Custom TFSec/Trivy Rules ด้วย Rego

### โครงสร้างไฟล์ Custom Checks

```
my-terraform-project/
├── terraform/
│   └── main.tf
└── .trivy/
    └── checks/
        ├── mandatory-tags.rego
        ├── approved-regions.rego
        └── cost-limit.rego
```

### Rego Basics

```rego
# ===== Rego Language Basics =====

# Package declaration (ต้อง match กับ convention)
package rules.my_check

# Import statements
import future.keywords.in
import future.keywords.if
import future.keywords.every

# Variables are immutable
x := 1

# Rules
allow {
    input.user == "admin"
}

deny[msg] {
    not input.resource.encrypted
    msg := "Resource must be encrypted"
}

# Comprehension
all_names := {r.name | r := input.resources[_]}
```

### Custom TFSec/Trivy Check: Mandatory Tags

```rego
# .trivy/checks/mandatory-tags.rego
package rules.mandatory_tags

import future.keywords.in

# Metadata สำหรับ TFSec
__rego_metadata__ := {
    "id": "CUS-001",
    "avd_id": "AVD-CUS-0001",
    "title": "Ensure mandatory tags are present",
    "short_code": "mandatory-tags",
    "description": "All resources must have Owner, Environment, Project, and CostCenter tags",
    "severity": "HIGH",
    "resolution": "Add the required tags to the resource",
    "links": ["https://your-wiki.company.com/tagging-policy"],
    "service": "general",
    "provider": "aws",
}

# Required tags
required_tags := {"Owner", "Environment", "Project", "CostCenter"}

# Taggable resource types
taggable_resources := {
    "aws_instance",
    "aws_s3_bucket",
    "aws_db_instance",
    "aws_lb",
    "aws_vpc",
    "aws_subnet",
    "aws_security_group",
    "aws_eks_cluster",
    "aws_lambda_function",
}

deny[res] {
    resource_type := taggable_resources[_]
    block := input[resource_type][name]
    
    existing_tags := {k | block.tags[k]}
    missing := required_tags - existing_tags
    count(missing) > 0
    
    res := {
        "msg": sprintf(
            "Resource '%v' of type '%v' is missing required tags: %v",
            [name, resource_type, missing]
        ),
        "startline": block.__startline__,
        "endline": block.__endline__,
    }
}
```

### Custom Check: Approved AWS Regions

```rego
# .trivy/checks/approved-regions.rego
package rules.approved_regions

__rego_metadata__ := {
    "id": "CUS-002",
    "avd_id": "AVD-CUS-0002",
    "title": "Ensure resources are only deployed in approved regions",
    "short_code": "approved-regions",
    "description": "Only ap-southeast-1 and us-east-1 are approved for deployment",
    "severity": "CRITICAL",
    "resolution": "Move resources to an approved region",
}

approved_regions := {
    "ap-southeast-1",  # Singapore
    "ap-southeast-2",  # Sydney
    "us-east-1",       # N. Virginia (for global services)
}

deny[res] {
    provider := input.provider.aws[_]
    region := provider.region
    not approved_regions[region]
    
    res := {
        "msg": sprintf(
            "AWS region '%v' is not in the approved list: %v",
            [region, approved_regions]
        ),
    }
}
```

### Custom Check: Cost Limit

```rego
# .trivy/checks/cost-limit.rego
package rules.cost_limit

__rego_metadata__ := {
    "id": "CUS-003",
    "avd_id": "AVD-CUS-0003",
    "title": "Ensure expensive instance types require approval tag",
    "short_code": "expensive-instances",
    "description": "Large instance types must have CostApproval tag",
    "severity": "MEDIUM",
}

# Expensive instance types ที่ต้องมี approval
expensive_types := {
    "m5.4xlarge", "m5.8xlarge", "m5.16xlarge", "m5.24xlarge",
    "c5.4xlarge", "c5.9xlarge", "c5.18xlarge",
    "r5.4xlarge", "r5.8xlarge", "r5.16xlarge",
    "x1.16xlarge", "x1.32xlarge",
    "p3.2xlarge", "p3.8xlarge", "p3.16xlarge",
}

deny[res] {
    instance := input.aws_instance[name]
    instance_type := instance.instance_type
    expensive_types[instance_type]
    
    # ต้องมี CostApproval tag
    not instance.tags.CostApproval
    
    res := {
        "msg": sprintf(
            "Instance '%v' uses expensive type '%v' but lacks CostApproval tag",
            [name, instance_type]
        ),
        "startline": instance.__startline__,
    }
}
```

### รัน Custom TFSec Checks

```bash
# รัน TFSec พร้อม custom checks
tfsec \
  --custom-check-dir .trivy/checks \
  .

# รัน Trivy พร้อม custom checks
trivy config \
  --policy .trivy/checks \
  --namespaces "rules" \
  .
```

---

## Step 934: Custom Terrascan Policies

### โครงสร้าง Policy Directory

```
custom-policies/
├── terraform/
│   ├── aws/
│   │   ├── security/
│   │   │   ├── mandatory_tags.rego
│   │   │   └── mandatory_tags.json
│   │   └── cost/
│   │       ├── cost_limit.rego
│   │       └── cost_limit.json
│   └── general/
│       ├── naming_convention.rego
│       └── naming_convention.json
└── README.md
```

### Policy Metadata JSON

```json
{
  "name": "mandatory_tags",
  "file": "mandatory_tags.rego",
  "policy_type": "terraform",
  "resource_type": "aws_instance",
  "template_args": {
    "prefix": "mandatory",
    "suffix": "tags"
  },
  "severity": "HIGH",
  "description": "Ensure EC2 instances have mandatory tags",
  "reference_id": "CUS-TERRA-001",
  "category": "INFRASTRUCTURE",
  "version": 2,
  "id": "CUS_TERRA_001"
}
```

### Policy Rego File

```rego
# mandatory_tags.rego
package rules.mandatory_tags

import future.keywords

# Terrascan input structure
# input.plan_root_module.resources[] = [{
#   "name": "my_instance",
#   "type": "aws_instance",
#   "config": {...attributes...}
# }]

resource_type := "aws_instance"

required_tags := {"Owner", "Environment", "Project"}

deny[msg] {
    resource := input.plan_root_module.resources[_]
    resource.type == resource_type
    
    tags := resource.config.tags
    existing := {k | tags[k]}
    missing := required_tags - existing
    count(missing) > 0
    
    msg := sprintf(
        "EC2 instance '%v' is missing required tags: %v",
        [resource.name, missing]
    )
}
```

### Testing Terrascan Policies

```bash
# สร้าง test input
cat > test-input.json << 'EOF'
{
  "plan_root_module": {
    "resources": [
      {
        "name": "my_instance",
        "type": "aws_instance",
        "config": {
          "instance_type": "t3.medium",
          "tags": {
            "Owner": "team-platform"
          }
        }
      }
    ]
  }
}
EOF

# รัน OPA test
opa eval \
  --input test-input.json \
  --data mandatory_tags.rego \
  "data.rules.mandatory_tags.deny"

# Output:
# {
#   "result": [{
#     "expressions": [{
#       "value": ["EC2 instance 'my_instance' is missing required tags: {'Environment', 'Project'}"]
#     }]
#   }]
# }
```

---

## Step 935: Custom Sentinel Policies (Terraform Cloud)

### Sentinel Overview

Sentinel เป็น Policy-as-Code framework ของ HashiCorp ใช้กับ:
- Terraform Cloud/Enterprise
- Vault
- Consul
- Nomad

### Sentinel Policy Structure

```
sentinel-policies/
├── required-tags.sentinel
├── approved-modules.sentinel
├── cost-limit.sentinel
└── sentinel.hcl  (policy set configuration)
```

### sentinel.hcl (Policy Set)

```hcl
# sentinel.hcl
policy "required-tags" {
  source            = "./required-tags.sentinel"
  enforcement_level = "hard-mandatory"  # hard-mandatory, soft-mandatory, advisory
}

policy "approved-modules" {
  source            = "./approved-modules.sentinel"
  enforcement_level = "hard-mandatory"
}

policy "cost-limit" {
  source            = "./cost-limit.sentinel"
  enforcement_level = "soft-mandatory"  # สามารถ override ได้ด้วย approval
}
```

### Sentinel Policy: Required Tags

```hcl
# required-tags.sentinel
# Policy: Ensure all resources have required tags

import "tfplan/v2" as tfplan

# ===== Constants =====
required_tags = ["Owner", "Environment", "Project", "CostCenter"]

# Resource types ที่ต้องการ tags
taggable_resources = [
    "aws_instance",
    "aws_s3_bucket",
    "aws_db_instance",
    "aws_lb",
    "aws_vpc",
]

# ===== Helper Functions =====

# ตรวจสอบ resource ว่ามี required tags
has_required_tags = func(resource) {
    tags = resource.change.after.tags
    
    all required_tags as tag {
        tags contains key(tag)
    }
}

# ===== Main Rule =====

# หา resources ทั้งหมดที่ขาด tags
violating_resources = filter tfplan.resource_changes as _, rc {
    rc.change.actions contains "create" or
    rc.change.actions contains "update"
    
    rc.type in taggable_resources
    not has_required_tags(rc)
}

# Print violations
for violating_resources as address, rc {
    tags = rc.change.after.tags else {}
    existing = keys(tags)
    missing = required_tags filter tag { tag not in existing }
    
    print("Resource", address, "is missing required tags:", missing)
}

# Main rule
main = rule {
    length(violating_resources) is 0
}
```

### Sentinel Policy: Approved Module Versions

```hcl
# approved-modules.sentinel
# Policy: Only approved module versions can be used

import "tfconfig/v2" as tfconfig

# Approved modules และ versions
approved_modules = {
    "terraform-aws-modules/vpc/aws": {
        "min_version": "5.0.0",
        "max_version": "6.0.0",
    },
    "terraform-aws-modules/eks/aws": {
        "min_version": "20.0.0",
        "max_version": "21.0.0",
    },
}

# ตรวจสอบ module calls
violating_modules = filter tfconfig.module_calls as _, module {
    source = module.source
    
    # ตรวจสอบว่า module อยู่ใน approved list
    source in keys(approved_modules)
    
    version = module.version_constraint
    approved = approved_modules[source]
    
    # Version ต้องอยู่ใน approved range
    not (version >= approved.min_version and version <= approved.max_version)
}

for violating_modules as name, module {
    print("Module", name, "version", module.version_constraint, 
          "is not in the approved range")
}

main = rule {
    length(violating_modules) is 0
}
```

### Sentinel Policy: Cost Limit

```hcl
# cost-limit.sentinel
# Policy: Resources must not exceed cost limits

import "tfplan/v2" as tfplan

# Cost limits per resource type (monthly USD estimate)
cost_limits = {
    "aws_instance": {
        "t3.micro": 8.47,
        "t3.small": 16.94,
        "t3.medium": 33.87,
        "t3.large": 67.74,
        "t3.xlarge": 135.49,
        "max_allowed": 200.00,
    },
}

# Expensive instances ที่ต้องมี approval tag
expensive_threshold_per_month = 500.00

main = rule {
    true  # Simplified - ในความเป็นจริงใช้ infracost integration
}
```

---

## Step 936: Custom OPA Policies สำหรับ Conftest

### Conftest คืออะไร?

```bash
# Conftest เป็น tool สำหรับรัน OPA policies บน config files
# รองรับ: Terraform, K8s, Dockerfile, Serverless, etc.

# ติดตั้ง
brew install conftest

# หรือ binary
CONFTEST_VERSION="0.49.0"
wget https://github.com/open-policy-agent/conftest/releases/download/v${CONFTEST_VERSION}/conftest_${CONFTEST_VERSION}_Linux_x86_64.tar.gz
tar -xzf conftest_*.tar.gz
sudo mv conftest /usr/local/bin/
```

### โครงสร้าง Conftest Project

```
my-terraform-project/
├── terraform/
│   └── main.tf
├── policy/
│   ├── terraform/
│   │   ├── mandatory-tags.rego
│   │   ├── approved-regions.rego
│   │   ├── naming-convention.rego
│   │   └── network-security.rego
│   └── kubernetes/
│       └── pod-security.rego
└── conftest.toml  (optional config)
```

### Conftest Policy: Mandatory Tags

```rego
# policy/terraform/mandatory-tags.rego
package main

# ===== Required Tags =====
required_tags := {"Owner", "Environment", "Project", "CostCenter"}

# ===== Resources to Check =====
taggable_resources := {
    "aws_instance",
    "aws_s3_bucket",
    "aws_db_instance",
    "aws_lb",
}

# ===== DENY Rule (blocks execution) =====
deny[msg] {
    resource_type := taggable_resources[_]
    resource := input.resource[resource_type][name]
    
    existing_tags := {k | resource.tags[k]}
    missing := required_tags - existing_tags
    count(missing) > 0
    
    msg := sprintf(
        "[MANDATORY-TAGS] Resource '%v' (%v) is missing required tags: %v",
        [name, resource_type, missing]
    )
}

# ===== WARN Rule (warning only, doesn't block) =====
warn[msg] {
    resource := input.resource.aws_instance[name]
    not resource.tags.Description
    
    msg := sprintf(
        "[OPTIONAL] EC2 instance '%v' has no Description tag",
        [name]
    )
}

# ===== VIOLATION Rule (for detailed reporting) =====
violation[msg] {
    resource := input.resource.aws_s3_bucket[name]
    not resource.tags.DataClassification
    
    msg := sprintf(
        "[DATA-CLASSIFICATION] S3 bucket '%v' must have DataClassification tag",
        [name]
    )
}
```

### Conftest Policy: Network Security

```rego
# policy/terraform/network-security.rego
package main

import future.keywords.in

# ===== No Open SSH to 0.0.0.0/0 =====
deny[msg] {
    sg := input.resource.aws_security_group[sg_name]
    ingress := sg.ingress[_]
    
    ingress.from_port <= 22
    ingress.to_port >= 22
    ingress.protocol in ["tcp", "-1"]
    
    # Check for open CIDR
    cidr := ingress.cidr_blocks[_]
    cidr in ["0.0.0.0/0", "::/0"]
    
    msg := sprintf(
        "[NETWORK-SEC] Security Group '%v' allows SSH (port 22) from %v",
        [sg_name, cidr]
    )
}

# ===== No Open RDP to 0.0.0.0/0 =====
deny[msg] {
    sg := input.resource.aws_security_group[sg_name]
    ingress := sg.ingress[_]
    
    ingress.from_port <= 3389
    ingress.to_port >= 3389
    ingress.protocol in ["tcp", "-1"]
    
    cidr := ingress.cidr_blocks[_]
    cidr in ["0.0.0.0/0", "::/0"]
    
    msg := sprintf(
        "[NETWORK-SEC] Security Group '%v' allows RDP (port 3389) from %v",
        [sg_name, cidr]
    )
}

# ===== No /8 or /16 CIDR blocks (too broad) =====
deny[msg] {
    sg := input.resource.aws_security_group[sg_name]
    ingress := sg.ingress[_]
    cidr := ingress.cidr_blocks[_]
    
    # Check CIDR prefix
    parts := split(cidr, "/")
    prefix := to_number(parts[1])
    prefix <= 16
    
    # ยกเว้น private networks
    not startswith(cidr, "10.")
    not startswith(cidr, "172.16.")
    not startswith(cidr, "192.168.")
    
    msg := sprintf(
        "[NETWORK-SEC] Security Group '%v' has too broad CIDR '%v' (/%v). Use more specific CIDRs.",
        [sg_name, cidr, prefix]
    )
}
```

### รัน Conftest

```bash
# สร้าง Terraform plan output
terraform init
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json

# รัน Conftest บน plan
conftest test tfplan.json \
  --policy ./policy/terraform/

# รัน บน Terraform files โดยตรง (ไม่ต้อง plan)
conftest test ./terraform/main.tf \
  --policy ./policy/terraform/ \
  --parser terraform

# รัน แสดง all results
conftest test tfplan.json \
  --policy ./policy/ \
  --all-namespaces

# รัน เฉพาะ package
conftest test tfplan.json \
  --policy ./policy/ \
  --namespace main

# Output JSON
conftest test tfplan.json \
  --policy ./policy/ \
  --output json

# ใน CI/CD
conftest test tfplan.json \
  --policy ./policy/ \
  --no-color \
  --output json \
  > conftest-results.json
```

---

## Step 937: Common Custom Rule Examples

### 1. Mandatory Tags (ครบทุก tool)

```python
# Checkov
REQUIRED_TAGS = {"Owner", "Environment", "Project", "CostCenter"}
```

```rego
# TFSec/Trivy/OPA
required_tags := {"Owner", "Environment", "Project", "CostCenter"}
```

### 2. Approved AMI IDs Only

```python
# custom_checks/check_approved_ami.py
from checkov.common.models.enums import CheckResult, CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck

# AMI IDs ที่ผ่าน security hardening แล้ว
APPROVED_AMIS = {
    # Amazon Linux 2
    "ami-0123456789abcdef0",  # ap-southeast-1
    "ami-0123456789abcdef1",  # us-east-1
    # Ubuntu 22.04 Hardened
    "ami-0fedcba9876543210",  # ap-southeast-1
    "ami-0fedcba9876543211",  # us-east-1
}

class ApprovedAMICheck(BaseResourceCheck):
    def __init__(self):
        super().__init__(
            name="Ensure only approved AMIs are used",
            id="CKV_CUSTOM_AMI",
            categories=[CheckCategories.GENERAL_SECURITY],
            supported_resources=["aws_instance", "aws_launch_template"]
        )
    
    def scan_resource_conf(self, conf):
        ami_id = conf.get("ami", [None])
        if isinstance(ami_id, list):
            ami_id = ami_id[0]
        
        if not ami_id or ami_id.startswith("${"):
            return CheckResult.UNKNOWN
        
        return CheckResult.PASSED if ami_id in APPROVED_AMIS else CheckResult.FAILED

check = ApprovedAMICheck()
```

### 3. CIDR Restrictions

```rego
# policy/terraform/cidr-restrictions.rego
package main

import future.keywords.in

# ห้ามใช้ CIDR ที่กว้างเกินไป
deny[msg] {
    sg := input.resource.aws_security_group[sg_name]
    ingress := sg.ingress[_]
    cidr := ingress.cidr_blocks[_]
    
    parts := split(cidr, "/")
    prefix := to_number(parts[1])
    
    # ห้าม /8, /16 สำหรับ internet CIDR
    prefix <= 16
    not _is_private_cidr(cidr)
    
    msg := sprintf(
        "Security group '%v' has overly broad CIDR '%v'. Use /24 or smaller.",
        [sg_name, cidr]
    )
}

_is_private_cidr(cidr) {
    startswith(cidr, "10.")
}

_is_private_cidr(cidr) {
    startswith(cidr, "172.16.")
}

_is_private_cidr(cidr) {
    startswith(cidr, "192.168.")
}
```

### 4. Naming Convention Enforcement

```rego
# policy/terraform/naming-convention.rego
package main

import future.keywords.in

# Pattern: {env}-{service}-{type}-{3-digit-seq}
valid_pattern := `^(dev|staging|prod|sandbox)-[a-z][a-z0-9-]*-[a-z0-9-]+-\d{3}$`

# Resources ที่ต้องตรวจสอบ
check_naming(resource_type, name_attr) = true {
    resource := input.resource[resource_type][name]
    resource_name := resource[name_attr]
    not regex.match(valid_pattern, resource_name)
}

deny[msg] {
    sg := input.resource.aws_security_group[tf_name]
    sg_name := sg.name
    not regex.match(valid_pattern, sg_name)
    
    msg := sprintf(
        "Security group '%v' (name: '%v') must follow pattern: {env}-{service}-sg-{seq}",
        [tf_name, sg_name]
    )
}

deny[msg] {
    instance := input.resource.aws_instance[tf_name]
    
    # ตรวจสอบ tags.Name แทน เพราะ EC2 ใช้ tag
    not instance.tags.Name
    
    msg := sprintf(
        "EC2 instance '%v' must have a Name tag following naming convention",
        [tf_name]
    )
}
```

### 5. Approved Regions Only

```rego
# policy/terraform/approved-regions.rego
package main

approved_regions := {
    "ap-southeast-1",  # Singapore (primary)
    "ap-southeast-2",  # Sydney (DR)
}

deny[msg] {
    provider := input.provider.aws
    region := provider.region
    not approved_regions[region]
    
    msg := sprintf(
        "AWS region '%v' is not approved. Use: %v",
        [region, approved_regions]
    )
}
```

### 6. Required Backup Policy

```python
# custom_checks/check_backup_policy.py
from checkov.common.models.enums import CheckResult, CheckCategories
from checkov.terraform.checks.resource.base_resource_check import BaseResourceCheck

class BackupPolicyCheck(BaseResourceCheck):
    """ตรวจสอบว่า RDS มี backup policy ที่เหมาะสม"""
    
    MIN_BACKUP_RETENTION_DAYS = 7
    MAX_BACKUP_RETENTION_DAYS = 35
    
    def __init__(self):
        super().__init__(
            name="Ensure RDS has adequate backup retention period",
            id="CKV_CUSTOM_BACKUP",
            categories=[CheckCategories.BACKUP_AND_RECOVERY],
            supported_resources=["aws_db_instance", "aws_rds_cluster"]
        )
    
    def scan_resource_conf(self, conf):
        retention = conf.get("backup_retention_period", [0])
        if isinstance(retention, list):
            retention = retention[0] if retention else 0
        
        try:
            retention_days = int(retention)
        except (TypeError, ValueError):
            return CheckResult.UNKNOWN
        
        if self.MIN_BACKUP_RETENTION_DAYS <= retention_days <= self.MAX_BACKUP_RETENTION_DAYS:
            return CheckResult.PASSED
        
        return CheckResult.FAILED


check = BackupPolicyCheck()
```

---

## Step 938: Testing Custom Rules

### Testing Checkov Custom Rules

```python
# tests/test_custom_checks.py
import os
import unittest
from checkov.terraform.runner import Runner
from checkov.runner_registry import RunnerRegistry

# Import checks
import sys
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
from custom_checks.check_mandatory_tags import MandatoryTagsCheck

class TestMandatoryTagsCheck(unittest.TestCase):
    
    def setUp(self):
        self.check = MandatoryTagsCheck()
    
    def test_pass_with_all_tags(self):
        """Test resource with all required tags PASSES"""
        resource_config = {
            "tags": [{
                "Owner": "team-platform",
                "Environment": "prod",
                "Project": "my-project",
                "CostCenter": "CC-001"
            }]
        }
        result = self.check.scan_resource_conf(resource_config)
        self.assertEqual(result, "passed")
    
    def test_fail_with_missing_tags(self):
        """Test resource missing tags FAILS"""
        resource_config = {
            "tags": [{
                "Owner": "team-platform",
                # Missing: Environment, Project, CostCenter
            }]
        }
        result = self.check.scan_resource_conf(resource_config)
        self.assertEqual(result, "failed")
    
    def test_fail_with_no_tags(self):
        """Test resource with no tags FAILS"""
        resource_config = {}
        result = self.check.scan_resource_conf(resource_config)
        self.assertEqual(result, "failed")
    
    def test_fail_with_empty_tags(self):
        """Test resource with empty tags FAILS"""
        resource_config = {"tags": [{}]}
        result = self.check.scan_resource_conf(resource_config)
        self.assertEqual(result, "failed")


if __name__ == '__main__':
    unittest.main()
```

### Integration Test ด้วย Terraform Files

```python
# tests/test_integration.py
import os
import pytest
from checkov.terraform.runner import Runner

def test_good_terraform_passes():
    """Good Terraform code should pass all checks"""
    runner = Runner()
    
    # สร้าง temp terraform file
    tf_content = '''
resource "aws_instance" "good" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t3.medium"
  
  tags = {
    Owner       = "team-platform"
    Environment = "prod"
    Project     = "my-project"
    CostCenter  = "CC-001"
  }
}
'''
    
    with open('/tmp/test_good.tf', 'w') as f:
        f.write(tf_content)
    
    report = runner.run(
        root_folder='/tmp',
        files=['/tmp/test_good.tf'],
        runner_filter_checks=['CKV_CUSTOM_1']
    )
    
    assert len(report.failed_checks) == 0
    assert len(report.passed_checks) > 0


def test_bad_terraform_fails():
    """Bad Terraform code should fail checks"""
    runner = Runner()
    
    tf_content = '''
resource "aws_instance" "bad" {
  ami           = "ami-0123456789abcdef0"
  instance_type = "t3.medium"
  # Missing required tags!
}
'''
    
    with open('/tmp/test_bad.tf', 'w') as f:
        f.write(tf_content)
    
    report = runner.run(
        root_folder='/tmp',
        files=['/tmp/test_bad.tf'],
        runner_filter_checks=['CKV_CUSTOM_1']
    )
    
    assert len(report.failed_checks) > 0
```

### Testing Rego Policies ด้วย OPA

```bash
# ติดตั้ง OPA
curl -L -o /usr/local/bin/opa \
  https://openpolicyagent.org/downloads/latest/opa_linux_amd64
chmod +x /usr/local/bin/opa

# สร้าง test file
cat > policy/terraform/mandatory-tags_test.rego << 'EOF'
package main

# ===== Test ผ่าน =====
test_resource_with_all_tags_passes {
    count(deny) == 0 with input as {
        "resource": {
            "aws_instance": {
                "my_instance": {
                    "instance_type": "t3.medium",
                    "tags": {
                        "Owner": "team-platform",
                        "Environment": "prod",
                        "Project": "my-project",
                        "CostCenter": "CC-001"
                    }
                }
            }
        }
    }
}

# ===== Test ล้มเหลว =====
test_resource_missing_tags_fails {
    count(deny) > 0 with input as {
        "resource": {
            "aws_instance": {
                "my_instance": {
                    "instance_type": "t3.medium",
                    "tags": {
                        "Owner": "team-platform"
                    }
                }
            }
        }
    }
}

# ===== Test ข้อความ error =====
test_error_message_contains_resource_name {
    msgs := deny with input as {
        "resource": {
            "aws_instance": {
                "my_bad_instance": {
                    "tags": {}
                }
            }
        }
    }
    some msg in msgs
    contains(msg, "my_bad_instance")
}
EOF

# รัน tests
opa test policy/ -v

# Output:
# PASS: 3/3
# ✅ test_resource_with_all_tags_passes
# ✅ test_resource_missing_tags_fails  
# ✅ test_error_message_contains_resource_name
```

---

## Step 939: การบรรจุ Custom Rules ใน CI/CD

### GitHub Actions ด้วย Custom Rules

```yaml
# .github/workflows/custom-security.yml
name: Custom Security Rules

on:
  pull_request:
    paths: ['**.tf']

jobs:
  custom-checks:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install Checkov
        run: pip install checkov
      
      - name: Run Checkov with Custom Rules
        run: |
          checkov -d ./terraform \
            --external-checks-dir ./custom_checks \
            --output json \
            --output-file checkov-results.json \
            --framework terraform \
            --compact || true
          
          # ตรวจสอบ custom check results
          python3 << 'PYTHON'
          import json
          
          with open('checkov-results.json') as f:
              results = json.load(f)
          
          custom_failures = [
              r for r in results.get('results', {}).get('failed_checks', [])
              if r['check_id'].startswith('CKV_CUSTOM_')
          ]
          
          if custom_failures:
              print(f"❌ Custom rule failures: {len(custom_failures)}")
              for failure in custom_failures:
                  print(f"  - {failure['check_id']}: {failure['resource']} in {failure['file_path']}")
              exit(1)
          else:
              print("✅ All custom rules passed!")
          PYTHON
      
      - name: Run Conftest
        run: |
          # ติดตั้ง conftest
          wget https://github.com/open-policy-agent/conftest/releases/download/v0.49.0/conftest_0.49.0_Linux_x86_64.tar.gz
          tar xzf conftest_*.tar.gz
          sudo mv conftest /usr/local/bin/
          
          # สร้าง plan
          cd terraform
          terraform init -backend=false
          terraform plan -out=tfplan.binary
          terraform show -json tfplan.binary > tfplan.json
          cd ..
          
          # รัน policies
          conftest test terraform/tfplan.json \
            --policy ./policy/terraform/ \
            --output json \
            > conftest-results.json || true
      
      - name: Upload Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: security-results
          path: |
            checkov-results.json
            conftest-results.json
```

---

## Step 940: Best Practices สำหรับ Custom Rules

### 1. Organization ของ Rules

```
security-policies/
├── README.md
├── CHANGELOG.md
├── checkov/
│   ├── __init__.py
│   ├── tagging/
│   │   ├── check_mandatory_tags.py
│   │   └── test_mandatory_tags.py
│   └── network/
│       ├── check_cidr_restrictions.py
│       └── test_cidr_restrictions.py
├── rego/
│   ├── tagging/
│   │   ├── mandatory_tags.rego
│   │   └── mandatory_tags_test.rego
│   └── network/
│       ├── cidr_restrictions.rego
│       └── cidr_restrictions_test.rego
└── sentinel/
    ├── required_tags.sentinel
    └── sentinel.hcl
```

### 2. Versioning Rules

```python
# ตัวอย่าง: ระบุ version ใน check
class MandatoryTagsCheck(BaseResourceCheck):
    VERSION = "1.2.0"
    LAST_UPDATED = "2024-01-15"
    AUTHOR = "security-team@company.com"
    TICKET = "SEC-123"  # JIRA ticket reference
```

### 3. Documentation

```rego
# mandatory_tags.rego
#
# Rule: CUS-001 - Mandatory Resource Tags
# Version: 1.0.0
# Author: security-team@company.com
# Ticket: SEC-123
# 
# Description:
#   All cloud resources must have the following tags for governance:
#   - Owner: Team or person responsible for the resource
#   - Environment: dev/staging/prod
#   - Project: Project codename
#   - CostCenter: Finance cost center code
#
# Rationale:
#   Tags are used for cost allocation, security response, and 
#   operational visibility.
#
# Remediation:
#   Add the required tags to all resources:
#   tags = {
#     Owner       = "team-name"
#     Environment = "prod"
#     Project     = "my-project"
#     CostCenter  = "CC-001"
#   }
```

### 4. Test Coverage >=80%

```bash
# ตรวจสอบ test coverage ของ Rego policies
opa test policy/ -v --coverage

# ตรวจสอบ test coverage ของ Python checks
pytest custom_checks/ --cov=custom_checks --cov-report=term-missing
```

---

## สรุป Custom Rules Cheatsheet

```
┌─────────────────────────────────────────────────────────────┐
│ Tool       │ Language  │ Command                           │
├────────────┼───────────┼───────────────────────────────────┤
│ Checkov    │ Python    │ checkov -d . --external-checks-dir │
│ TFSec      │ Rego      │ tfsec --custom-check-dir .         │
│ Trivy      │ Rego      │ trivy config --policy ./policies   │
│ Terrascan  │ Rego      │ terrascan scan --policy-path ./    │
│ Conftest   │ Rego      │ conftest test . --policy ./policy  │
│ Sentinel   │ Sentinel  │ (ใน Terraform Cloud)              │
└─────────────────────────────────────────────────────────────┘
```

---

*จบ Part 94: Custom Security Rules & Policies*
