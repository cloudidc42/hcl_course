# Part 072: Terraform Testing Framework (ขั้นตอนที่ 711-720)

## บทนำ (Introduction)

Terraform 1.6+ มี native testing framework ที่ทรงพลัง ช่วยให้เราสามารถ test Infrastructure code
เหมือนกับ unit testing ใน software development ทั่วไป

---

## ขั้นตอนที่ 711: terraform test command (Terraform 1.6+)

### ทำไมต้อง Test Infrastructure Code?

```
ปัญหาที่พบบ่อยใน Infrastructure as Code:
1. Variable validation ไม่ถูกต้อง
2. Module ส่ง output ผิด
3. Resource configuration ไม่ตรงกับ expectation
4. Conditional logic ใน locals ผิดพลาด
```

### คำสั่งพื้นฐาน

```bash
# Run tests ทั้งหมด
terraform test

# Run test เฉพาะไฟล์
terraform test -filter=tests/basic.tftest.hcl

# Run แบบ verbose เห็น output ทั้งหมด
terraform test -verbose

# Run แบบ plan เท่านั้น (ไม่ apply จริง)
terraform test  # run blocks กำหนด command = plan หรือ apply
```

---

## ขั้นตอนที่ 712: Test File Structure

### โครงสร้างไฟล์ทดสอบ

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── modules/
│   └── vpc/
│       ├── main.tf
│       └── outputs.tf
└── tests/                    # หรือ *.tftest.hcl ไว้ที่ root
    ├── unit/
    │   ├── variables.tftest.hcl
    │   └── locals.tftest.hcl
    ├── integration/
    │   ├── vpc_test.tftest.hcl
    │   └── full_stack.tftest.hcl
    └── modules/
        └── vpc_module.tftest.hcl
```

### โครงสร้างพื้นฐานของ tftest.hcl

```hcl
# tests/basic.tftest.hcl

# Provider configuration สำหรับ test
provider "aws" {
  region = "us-east-1"
}

# Variables สำหรับ test run นี้
variables {
  environment = "test"
  region      = "us-east-1"
}

# Run block - หน่วยพื้นฐานของการทดสอบ
run "verify_vpc_created" {
  command = apply  # หรือ plan

  # Override variables สำหรับ run นี้โดยเฉพาะ
  variables {
    vpc_cidr = "10.0.0.0/16"
  }

  # Assert blocks - ตรวจสอบผลลัพธ์
  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block should be 10.0.0.0/16"
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS hostnames should be enabled"
  }
}
```

---

## ขั้นตอนที่ 713: Run Block และ Command

### run block options

```hcl
run "test_name" {
  # command: plan หรือ apply
  # plan  = ตรวจสอบ config ไม่ apply จริง
  # apply = apply จริงแล้วตรวจสอบ (ต้องการ AWS credentials)
  command = plan

  # module: ถ้าจะ test module เฉพาะ
  module {
    source = "./modules/networking"
  }

  # variables: override ตัวแปร
  variables {
    environment = "test"
  }

  # expect_failures: คาดหวังว่าจะ fail
  expect_failures = [
    var.invalid_cidr
  ]

  # assert blocks
  assert {
    condition     = <expression>
    error_message = "<message>"
  }
}
```

### ตัวอย่าง plan vs apply

```hcl
# tests/plan_only.tftest.hcl

# Test ด้วย plan (เร็วกว่า ไม่ต้องใช้ credentials จริง)
run "validate_configuration" {
  command = plan

  assert {
    condition     = length(aws_instance.servers) == 3
    error_message = "Should create exactly 3 servers"
  }

  assert {
    condition     = aws_instance.servers["web-1"].instance_type == "t3.micro"
    error_message = "Web servers should use t3.micro"
  }
}

# Test ด้วย apply (ต้องการ real AWS)
run "verify_actual_creation" {
  command = apply

  assert {
    condition     = aws_instance.servers["web-1"].id != ""
    error_message = "Instance should have an ID after creation"
  }

  assert {
    condition     = aws_instance.servers["web-1"].public_ip != ""
    error_message = "Instance should have a public IP"
  }
}
```

---

## ขั้นตอนที่ 714: Variables ใน Tests

### ระดับต่างๆ ของ Variables

```hcl
# tests/variable_test.tftest.hcl

# Level 1: Global variables สำหรับทั้งไฟล์
variables {
  aws_region  = "us-east-1"
  environment = "test"
}

# Level 2: Run-specific variables (override global)
run "test_dev_environment" {
  variables {
    environment = "dev"    # override global
    vpc_cidr    = "10.1.0.0/16"
  }

  command = plan

  assert {
    condition     = aws_vpc.main.tags["Environment"] == "dev"
    error_message = "Environment tag should be dev"
  }
}

run "test_prod_environment" {
  variables {
    environment = "prod"   # different override
    vpc_cidr    = "10.0.0.0/16"
  }

  command = plan

  assert {
    condition     = aws_vpc.main.tags["Environment"] == "prod"
    error_message = "Environment tag should be prod"
  }
}
```

### ใช้ Variables จาก .tfvars ไฟล์

```bash
# ใช้ tfvars ไฟล์กับ terraform test
terraform test -var-file="tests/test.tfvars"
```

```hcl
# tests/test.tfvars
environment       = "test"
aws_region        = "us-east-1"
instance_type     = "t3.nano"
enable_monitoring = false
```

---

## ขั้นตอนที่ 715: Assert Block

### assert block syntax

```hcl
assert {
  condition     = <boolean expression>
  error_message = "<string>"
}
```

### ตัวอย่าง assert patterns ต่างๆ

```hcl
# tests/assert_examples.tftest.hcl

run "comprehensive_assertions" {
  command = apply

  # 1. ตรวจสอบ equality
  assert {
    condition     = aws_s3_bucket.main.bucket == "my-test-bucket-12345"
    error_message = "Bucket name is incorrect"
  }

  # 2. ตรวจสอบ boolean
  assert {
    condition     = aws_s3_bucket_versioning.main.versioning_configuration[0].status == "Enabled"
    error_message = "Versioning should be enabled"
  }

  # 3. ตรวจสอบ list length
  assert {
    condition     = length(aws_subnet.public) == 2
    error_message = "Should have exactly 2 public subnets"
  }

  # 4. ตรวจสอบ map key existence
  assert {
    condition     = contains(keys(aws_instance.servers), "web-1")
    error_message = "Should have a server named web-1"
  }

  # 5. ตรวจสอบด้วย regex
  assert {
    condition     = can(regex("^arn:aws:iam::", aws_iam_role.main.arn))
    error_message = "IAM role ARN should start with arn:aws:iam::"
  }

  # 6. ตรวจสอบ null/non-null
  assert {
    condition     = aws_instance.web.public_ip != null
    error_message = "Public IP should not be null"
  }

  # 7. ตรวจสอบ tag
  assert {
    condition     = aws_instance.web.tags["Environment"] == var.environment
    error_message = "Environment tag should match variable"
  }

  # 8. ตรวจสอบ CIDR block
  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR should be 10.0.0.0/16"
  }

  # 9. ตรวจสอบ output
  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC ID output should not be empty"
  }

  # 10. ตรวจสอบ complex condition
  assert {
    condition = alltrue([
      for sg in aws_security_group.web : length(sg.ingress) > 0
    ])
    error_message = "All security groups should have at least one ingress rule"
  }

  # 11. ตรวจสอบ encryption
  assert {
    condition     = aws_ebs_volume.data.encrypted == true
    error_message = "EBS volume should be encrypted"
  }

  # 12. ตรวจสอบ port ใน security group
  assert {
    condition = anytrue([
      for rule in aws_security_group.web.ingress :
      rule.from_port == 443 && rule.to_port == 443
    ])
    error_message = "Security group should allow HTTPS (port 443)"
  }
}
```

---

## ขั้นตอนที่ 716: expect_failures สำหรับ Negative Testing

### ทดสอบ Validation Logic

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  type = string

  validation {
    condition = can(regex("^t[23]\\.(nano|micro|small|medium|large)$", var.instance_type))
    error_message = "Instance type must be a t2 or t3 type."
  }
}

variable "vpc_cidr" {
  type = string

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid IPv4 CIDR block."
  }
}
```

```hcl
# tests/validation_test.tftest.hcl

run "invalid_environment_should_fail" {
  command = plan

  variables {
    environment = "production"  # ผิด - ต้องเป็น "prod"
  }

  expect_failures = [
    var.environment  # คาดหวังว่า validation นี้จะ fail
  ]
}

run "invalid_instance_type_should_fail" {
  command = plan

  variables {
    instance_type = "m5.large"  # ผิด - ต้องเป็น t2/t3
  }

  expect_failures = [
    var.instance_type
  ]
}

run "invalid_cidr_should_fail" {
  command = plan

  variables {
    vpc_cidr = "not-a-cidr"  # CIDR ไม่ถูกต้อง
  }

  expect_failures = [
    var.vpc_cidr
  ]
}

run "valid_inputs_should_pass" {
  command = plan

  variables {
    environment   = "prod"
    instance_type = "t3.micro"
    vpc_cidr      = "10.0.0.0/16"
  }

  # ไม่มี expect_failures = ทุกอย่างต้องผ่าน
  assert {
    condition     = var.environment == "prod"
    error_message = "Valid environment should be accepted"
  }
}
```

---

## ขั้นตอนที่ 717: Module Testing

### Test Module โดยตรง

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = var.enable_dns_hostnames

  tags = merge(var.tags, {
    Name = var.name
  })
}

resource "aws_subnet" "public" {
  count = length(var.public_subnets)

  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true
}
```

```hcl
# tests/modules/vpc_test.tftest.hcl

# Test module โดยตรง (ไม่ต้องผ่าน root module)
run "basic_vpc_creation" {
  command = apply

  module {
    source = "./modules/vpc"  # path to module
  }

  variables {
    cidr_block           = "10.0.0.0/16"
    name                 = "test-vpc"
    enable_dns_hostnames = true
    public_subnets       = ["10.0.1.0/24", "10.0.2.0/24"]
    availability_zones   = ["us-east-1a", "us-east-1b"]
    tags = {
      Environment = "test"
    }
  }

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR should match input"
  }

  assert {
    condition     = length(aws_subnet.public) == 2
    error_message = "Should create 2 public subnets"
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS hostnames should be enabled"
  }

  assert {
    condition     = aws_vpc.main.tags["Environment"] == "test"
    error_message = "Environment tag should be set"
  }
}

run "vpc_with_dns_disabled" {
  command = plan

  module {
    source = "./modules/vpc"
  }

  variables {
    cidr_block           = "10.0.0.0/16"
    name                 = "test-vpc-no-dns"
    enable_dns_hostnames = false
    public_subnets       = ["10.0.1.0/24"]
    availability_zones   = ["us-east-1a"]
    tags                 = {}
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == false
    error_message = "DNS hostnames should be disabled"
  }
}
```

---

## ขั้นตอนที่ 718: Provider Mocking (Terraform 1.7+)

### ทำไมต้อง Mock Provider?

การ test จริงกับ AWS มีข้อเสีย:
- ใช้เวลานาน (5-15 นาที)
- มีค่าใช้จ่าย
- ต้องการ credentials
- ไม่เหมาะกับ unit testing

Mock provider ช่วยแก้ปัญหานี้

### Mock Provider Syntax

```hcl
# tests/mock_test.tftest.hcl

# Mock provider สำหรับ AWS
mock_provider "aws" {
  # กำหนด mock ค่า default สำหรับทุก resource
  mock_resource "aws_vpc" {
    defaults = {
      id         = "vpc-12345mock"
      arn        = "arn:aws:ec2:us-east-1:123456789012:vpc/vpc-12345mock"
      owner_id   = "123456789012"
      state      = "available"
      cidr_block = "10.0.0.0/16"
    }
  }

  mock_resource "aws_subnet" {
    defaults = {
      id                = "subnet-mock12345"
      availability_zone = "us-east-1a"
      state             = "available"
    }
  }

  mock_resource "aws_instance" {
    defaults = {
      id            = "i-mockinstance1234"
      public_ip     = "1.2.3.4"
      private_ip    = "10.0.1.5"
      public_dns    = "ec2-1-2-3-4.compute-1.amazonaws.com"
      private_dns   = "ip-10-0-1-5.ec2.internal"
      instance_state = "running"
    }
  }

  mock_resource "aws_security_group" {
    defaults = {
      id  = "sg-mock12345678"
      arn = "arn:aws:ec2:us-east-1:123456789012:security-group/sg-mock12345678"
    }
  }
}

# ใช้ mock provider ในการ test
run "unit_test_with_mock" {
  command = apply  # apply ด้วย mock - เร็วมาก ไม่ต้องการ AWS

  assert {
    condition     = aws_vpc.main.id == "vpc-12345mock"
    error_message = "VPC should use mocked ID"
  }

  assert {
    condition     = aws_instance.web.public_ip == "1.2.3.4"
    error_message = "Instance should use mocked IP"
  }
}
```

### override_resource สำหรับ specific resource override

```hcl
# tests/override_test.tftest.hcl

run "test_with_specific_overrides" {
  command = apply

  # Override ค่าของ resource เฉพาะ (ไม่ต้อง mock ทั้ง provider)
  override_resource {
    target = aws_vpc.main
    values = {
      id         = "vpc-overridden"
      cidr_block = "192.168.0.0/16"
    }
  }

  # Override data source
  override_data {
    target = data.aws_ami.latest
    values = {
      id   = "ami-mocked12345"
      name = "mocked-ami"
    }
  }

  assert {
    condition     = aws_vpc.main.id == "vpc-overridden"
    error_message = "VPC should use overridden ID"
  }
}
```

### ตัวอย่าง Full Mock Test Suite

```hcl
# tests/full_mock_suite.tftest.hcl

mock_provider "aws" {
  alias = "mock"

  mock_resource "aws_vpc" {
    defaults = {
      id       = "vpc-00000000000000001"
      owner_id = "123456789012"
      state    = "available"
    }
  }

  mock_resource "aws_internet_gateway" {
    defaults = {
      id = "igw-00000000000000001"
    }
  }

  mock_resource "aws_route_table" {
    defaults = {
      id = "rtb-00000000000000001"
    }
  }

  mock_resource "aws_security_group" {
    defaults = {
      id  = "sg-00000000000000001"
      arn = "arn:aws:ec2:us-east-1:123456789012:security-group/sg-00000000000000001"
    }
  }

  mock_data "aws_availability_zones" {
    defaults = {
      names = ["us-east-1a", "us-east-1b", "us-east-1c"]
    }
  }

  mock_data "aws_ami" {
    defaults = {
      id            = "ami-00000000000000001"
      name          = "mock-ami-2024"
      description   = "Mock AMI for testing"
      image_type    = "machine"
      state         = "available"
      virtualization_type = "hvm"
    }
  }
}

variables {
  environment  = "test"
  vpc_cidr     = "10.0.0.0/16"
  project_name = "test-project"
}

run "test_vpc_configuration" {
  command = apply

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR mismatch"
  }

  assert {
    condition     = aws_vpc.main.tags["Environment"] == "test"
    error_message = "Environment tag not set correctly"
  }
}

run "test_networking_outputs" {
  command = apply

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC ID output should not be empty"
  }

  assert {
    condition     = length(output.public_subnet_ids) == 2
    error_message = "Should output 2 public subnet IDs"
  }
}

run "test_security_group_rules" {
  command = plan

  assert {
    condition = anytrue([
      for rule in aws_security_group.web.ingress :
      rule.from_port == 80 && rule.protocol == "tcp"
    ])
    error_message = "Security group must allow HTTP traffic"
  }

  assert {
    condition = anytrue([
      for rule in aws_security_group.web.ingress :
      rule.from_port == 443 && rule.protocol == "tcp"
    ])
    error_message = "Security group must allow HTTPS traffic"
  }
}
```

---

## ขั้นตอนที่ 719: Test Organization และ Naming Conventions

### Directory Structure ที่แนะนำ

```
terraform-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── locals.tf
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── tests/
│   │       ├── unit.tftest.hcl         # unit tests ด้วย mocks
│   │       └── integration.tftest.hcl  # integration tests จริง
│   └── compute/
│       └── tests/
│           └── unit.tftest.hcl
└── tests/
    ├── unit/
    │   ├── variables_validation.tftest.hcl
    │   ├── locals_logic.tftest.hcl
    │   └── output_format.tftest.hcl
    ├── integration/
    │   ├── vpc_integration.tftest.hcl
    │   └── full_stack.tftest.hcl
    └── fixtures/
        ├── minimal.tfvars      # minimal valid configuration
        └── complete.tfvars     # complete configuration
```

### Naming Conventions

```hcl
# ตั้งชื่อ test run ให้สื่อความหมาย
run "given_valid_inputs_when_applying_then_vpc_is_created" { ... }

# หรือ สั้นกว่า แต่ชัดเจน
run "vpc_creation_with_valid_inputs" { ... }
run "invalid_environment_variable_fails_validation" { ... }
run "outputs_contain_valid_vpc_id" { ... }
run "security_group_allows_https_traffic" { ... }
```

### Test ที่ดีควรทดสอบอะไร?

```hcl
# tests/what_to_test.tftest.hcl

# 1. Variable Validation
run "test_valid_environments" {
  command = plan
  variables { environment = "prod" }
  assert {
    condition     = var.environment == "prod"
    error_message = "Valid environment should be accepted"
  }
}

# 2. Resource Creation
run "test_resource_exists_after_apply" {
  command = apply
  assert {
    condition     = aws_s3_bucket.main.id != ""
    error_message = "S3 bucket should be created"
  }
}

# 3. Output Values
run "test_output_format" {
  command = apply
  assert {
    condition     = can(regex("^arn:aws:", output.bucket_arn))
    error_message = "Bucket ARN should be valid AWS ARN"
  }
}

# 4. Tags are correct
run "test_required_tags" {
  command = apply
  assert {
    condition = alltrue([
      aws_instance.web.tags["Environment"] != null,
      aws_instance.web.tags["Project"] != null,
      aws_instance.web.tags["Owner"] != null,
    ])
    error_message = "Required tags must be present"
  }
}

# 5. Security Configuration
run "test_encryption_enabled" {
  command = plan
  assert {
    condition     = aws_ebs_volume.data.encrypted == true
    error_message = "EBS volumes must be encrypted"
  }
}

# 6. Networking Rules
run "test_no_public_access_to_database" {
  command = plan
  assert {
    condition = !anytrue([
      for rule in aws_security_group.db.ingress :
      rule.cidr_blocks != null && contains(rule.cidr_blocks, "0.0.0.0/0")
    ])
    error_message = "Database security group must not allow public access"
  }
}
```

---

## ขั้นตอนที่ 720: Running Tests & CI/CD Integration

### Command Line Options

```bash
# Run tests ทั้งหมด
terraform test

# Run test เฉพาะไฟล์
terraform test -filter=tests/unit/variables_validation.tftest.hcl

# Run test หลายไฟล์
terraform test \
  -filter=tests/unit/variables_validation.tftest.hcl \
  -filter=tests/unit/locals_logic.tftest.hcl

# Verbose output
terraform test -verbose

# ไม่ต้องการ color output (สำหรับ CI)
terraform test -no-color

# JSON output สำหรับ parsing
terraform test -json

# ระบุ var file
terraform test -var-file="tests/fixtures/minimal.tfvars"
```

### GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/terraform-test.yml
name: Terraform Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  TF_VERSION: "1.7.0"

jobs:
  unit-tests:
    name: Unit Tests (Mock)
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init

      - name: Run Unit Tests
        run: |
          terraform test \
            -filter=tests/unit/ \
            -no-color \
            -verbose

  integration-tests:
    name: Integration Tests (Real AWS)
    runs-on: ubuntu-latest
    needs: unit-tests
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    permissions:
      id-token: write   # สำหรับ OIDC authentication
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init
        env:
          TF_VAR_environment: test

      - name: Run Integration Tests
        run: |
          terraform test \
            -filter=tests/integration/ \
            -no-color \
            -var-file="tests/fixtures/ci.tfvars"
        env:
          TF_VAR_environment: test
          TF_VAR_region: us-east-1

  module-tests:
    name: Module Tests
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Test Networking Module
        working-directory: modules/networking
        run: |
          terraform init
          terraform test -no-color

      - name: Test Compute Module
        working-directory: modules/compute
        run: |
          terraform init
          terraform test -no-color
```

### ตัวอย่าง Complete Test Suite

```hcl
# tests/complete_suite.tftest.hcl
# Complete test suite สำหรับ VPC + EC2 + S3 deployment

provider "aws" {
  region = "us-east-1"
}

mock_provider "aws" {
  alias = "mock"

  mock_resource "aws_vpc" {
    defaults = {
      id                   = "vpc-test12345"
      cidr_block           = "10.0.0.0/16"
      enable_dns_hostnames = true
      state                = "available"
      owner_id             = "123456789012"
    }
  }

  mock_resource "aws_subnet" {
    defaults = {
      id                   = "subnet-test12345"
      state                = "available"
      map_public_ip_on_launch = false
    }
  }

  mock_resource "aws_security_group" {
    defaults = {
      id  = "sg-test12345678"
      arn = "arn:aws:ec2:us-east-1:123456789012:security-group/sg-test12345678"
    }
  }

  mock_resource "aws_instance" {
    defaults = {
      id             = "i-testinstance1234"
      instance_state = "running"
      public_ip      = "54.0.0.1"
      private_ip     = "10.0.1.10"
    }
  }

  mock_resource "aws_s3_bucket" {
    defaults = {
      id          = "test-bucket-name"
      bucket      = "test-bucket-name"
      bucket_domain_name = "test-bucket-name.s3.amazonaws.com"
      region      = "us-east-1"
    }
  }
}

variables {
  environment  = "test"
  project_name = "myapp"
  vpc_cidr     = "10.0.0.0/16"
  instance_type = "t3.micro"
}

# --- Unit Tests (ใช้ mock) ---

run "unit_vpc_cidr_validation" {
  command = plan

  assert {
    condition     = aws_vpc.main.cidr_block == var.vpc_cidr
    error_message = "VPC CIDR should match variable"
  }
}

run "unit_tags_applied" {
  command = plan

  assert {
    condition     = aws_vpc.main.tags["Environment"] == "test"
    error_message = "Environment tag should be set"
  }

  assert {
    condition     = aws_vpc.main.tags["Project"] == "myapp"
    error_message = "Project tag should be set"
  }

  assert {
    condition     = aws_vpc.main.tags["ManagedBy"] == "terraform"
    error_message = "ManagedBy tag should be terraform"
  }
}

run "unit_security_group_rules" {
  command = plan

  assert {
    condition = anytrue([
      for rule in aws_security_group.web.ingress :
      rule.from_port == 443 && rule.to_port == 443 && rule.protocol == "tcp"
    ])
    error_message = "Must allow HTTPS"
  }

  assert {
    condition = !anytrue([
      for rule in aws_security_group.web.ingress :
      rule.from_port == 22 && contains(coalesce(rule.cidr_blocks, []), "0.0.0.0/0")
    ])
    error_message = "Must not allow SSH from anywhere"
  }
}

run "unit_s3_encryption" {
  command = plan

  assert {
    condition     = aws_s3_bucket_server_side_encryption_configuration.main != null
    error_message = "S3 bucket must have encryption configured"
  }
}

# --- Validation Tests ---

run "validation_invalid_environment_rejected" {
  command = plan

  variables {
    environment = "unknown"
  }

  expect_failures = [var.environment]
}

run "validation_valid_environments_accepted" {
  command = plan

  variables { environment = "dev" }
  assert {
    condition     = var.environment == "dev"
    error_message = "dev should be valid"
  }
}

# --- Output Tests ---

run "outputs_vpc_id_not_empty" {
  command = apply

  assert {
    condition     = output.vpc_id != ""
    error_message = "VPC ID output should not be empty"
  }
}

run "outputs_subnet_ids_count" {
  command = apply

  assert {
    condition     = length(output.private_subnet_ids) >= 1
    error_message = "Should have at least 1 private subnet"
  }
}
```

---

## Test Coverage Strategies

```markdown
## Test Coverage Matrix

| Component         | Unit Test | Integration Test | E2E Test |
|-------------------|-----------|-----------------|----------|
| Variables         | ✅ Plan   | -               | -        |
| Locals            | ✅ Plan   | -               | -        |
| Resource Config   | ✅ Mock   | ✅ Real AWS     | -        |
| Module Interface  | ✅ Mock   | ✅ Real AWS     | -        |
| Outputs           | ✅ Mock   | ✅ Real AWS     | -        |
| Full Stack        | -         | -               | ✅ Real  |
| Performance       | -         | -               | ✅ Real  |

## Test Speed vs Cost Tradeoff

| Test Type   | Speed    | Cost    | Coverage |
|-------------|----------|---------|----------|
| Plan only   | <30s     | Free    | Syntax   |
| Mock Apply  | <60s     | Free    | Logic    |
| Real Apply  | 5-15min  | $$$     | Full     |
```

---

## สรุป (Summary)

Terraform Testing Framework ช่วยให้เรา:

1. **Unit test** ด้วย mock providers - เร็ว ฟรี
2. **Integration test** กับ real infrastructure - แม่นยำ
3. **Validate** variable validation logic
4. **Verify** outputs format และ values
5. **Test** security requirements
6. **Automate** ใน CI/CD pipeline

---

*จบ Part 072 - ในส่วนถัดไปจะเรียนรู้เรื่อง Terratest Integration Testing*
