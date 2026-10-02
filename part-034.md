# Part 034: Terraform Lock File (.terraform.lock.hcl)
# Terraform Lock File และการจัดการ Provider Versions

## Steps 331-340: ทำความเข้าใจ Lock File อย่างละเอียด

---

## Step 331: Lock File คืออะไร?

### ความหมาย

`.terraform.lock.hcl` คือ **dependency lock file** ที่ Terraform สร้างโดยอัตโนมัติเมื่อรัน `terraform init` เพื่อ **pin ตัวเลข version** ของ providers ที่ใช้ในโปรเจกต์

### ทำไม Lock File ถึงสำคัญ?

```
ปัญหาที่ Lock File แก้:

ไม่มี Lock File:
  Dev machine    → terraform init → downloads provider 5.20.0
  CI/CD machine  → terraform init → downloads provider 5.21.0  ← ต่างกัน!
  Teammate PC    → terraform init → downloads provider 5.22.0  ← ต่างกัน!

  ผลลัพธ์: Infrastructure ที่ apply อาจแตกต่างกันในแต่ละ environment

มี Lock File:
  Dev machine    → terraform init → downloads provider 5.20.0
  CI/CD machine  → terraform init → downloads provider 5.20.0  ← เหมือนกัน!
  Teammate PC    → terraform init → downloads provider 5.20.0  ← เหมือนกัน!

  ผลลัพธ์: Reproducible และ predictable infrastructure
```

### Lock File ถูกสร้างเมื่อไหร่?

```bash
# สร้างครั้งแรกเมื่อรัน terraform init
terraform init

# อัปเดตเมื่อรัน terraform init -upgrade
terraform init -upgrade

# อัปเดตด้วย terraform providers lock
terraform providers lock
```

---

## Step 332: รูปแบบและ Contents ของ Lock File

### ตัวอย่าง Lock File จริง

```hcl
# .terraform.lock.hcl
# This file is maintained automatically by "terraform init".
# Manual edits may be lost in future updates.

provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.20.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123def456ghi789jkl012mno345pqr678stu901vwx234yz567890abcdef01=",
    "zh:1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
    "zh:abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890",
    # ... more hashes for different platforms
  ]
}

provider "registry.terraform.io/hashicorp/random" {
  version     = "3.5.1"
  constraints = ">= 3.0.0"
  hashes = [
    "h1:xyz789abc123def456ghi789jkl012mno345pqr678stu901vwx234yz567890=",
    "zh:fedcba0987654321fedcba0987654321fedcba0987654321fedcba0987654321",
  ]
}

provider "registry.terraform.io/hashicorp/null" {
  version     = "3.2.1"
  constraints = ">= 3.0.0"
  hashes = [
    "h1:abc789def123ghi456jkl789mno012pqr345stu678vwx901yz234ab567cd890=",
  ]
}
```

### ส่วนประกอบของ Lock File

```
Lock File Structure:

provider "registry.terraform.io/<namespace>/<type>" {
  ┌─ version     ← Exact version ที่ถูก selected
  ├─ constraints ← Version constraints จาก required_providers
  └─ hashes      ← Cryptographic hashes สำหรับ verification
     ├─ h1:...   ← Hash ของ zip file (cross-platform)
     └─ zh:...   ← Hash ของ extracted directory (platform-specific)
}
```

### ตัวอย่าง Lock File สมบูรณ์พร้อม Annotations

```hcl
# .terraform.lock.hcl
# This file is maintained automatically by "terraform init".
# Manual edits may be lost in future updates.

# Provider: AWS
# Version ที่ถูก lock: 5.20.0
# Constraint จาก versions.tf: ~> 5.0 (อนุญาต 5.0 ถึง < 6.0)
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.20.0"
  constraints = "~> 5.0"
  hashes = [
    # h1: hash ของ provider package (เหมือนกันทุก platform)
    "h1:2GduHfuuJC0GFAhRrHpwKELBWUuJz8GjB+f+oXRoVnI=",
    
    # zh: hash สำหรับ platform-specific binaries
    # linux/amd64
    "zh:0843a15d1cf14f6e60d9d2c33b0b94d7b9d9e9b4a1f3c5d2b8e7f6a5c4d3b2a1",
    # linux/arm64  
    "zh:1954b26e2df27f7e70d0e1d4c0b8e5f4a3d2c1b0a9f8e7d6c5b4a3f2e1d0c9b8",
    # darwin/amd64
    "zh:2a65c37f3eg38h8f81e2e5d1c4b7a6e5d4c3b2a1f0e9d8c7b6a5f4e3d2c1b0a9",
    # darwin/arm64 (M1/M2)
    "zh:3b76d48g4fh49i9g92f3f6e2d5c8b7f6e5d4c3b2a1f0e9d8c7b6a5f4e3d2c1b",
    # windows/amd64
    "zh:4c87e59h5gi50j0h03g4g7f3e6d9c8g7f6e5d4c3b2a1f0e9d8c7b6a5f4e3d2c",
  ]
}

# Provider: Random
provider "registry.terraform.io/hashicorp/random" {
  version     = "3.5.1"
  constraints = ">= 3.0.0, < 4.0.0"
  hashes = [
    "h1:sZTKBT0yEo9B9tKynmaW6gt68lADKd/6tW8yQuqud+0=",
    "zh:0d251a537cc9e78ca7a8ea5fd61dd0f23e69a0f67e9a3e2c1f8d0a3b7c5e2d1f",
    "zh:1e362b648de8b9f7c12a6d9e7b3a8f7d5c6b4a2f0e1d8c7b6a5f4e3d2c1b0a9",
  ]
}
```

---

## Step 333: Provider Version Pinning

### Version Constraints ใน versions.tf

```hcl
# versions.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    # ─── Pessimistic Constraint (~>) ──────────────────────
    # ~> 5.0  = >= 5.0, < 6.0 (อนุญาต patch updates)
    # ~> 5.20 = >= 5.20, < 5.21 (เฉพาะ patch)
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }

    # ─── Range Constraint ─────────────────────────────────
    # >= 3.0.0, < 4.0.0 (อนุญาต ≥ 3.0.0 และ < 4.0.0)
    random = {
      source  = "hashicorp/random"
      version = ">= 3.0.0, < 4.0.0"
    }

    # ─── Exact Constraint (=) ─────────────────────────────
    # = 3.2.1 (เฉพาะ version นี้เท่านั้น - ไม่แนะนำ)
    null = {
      source  = "hashicorp/null"
      version = "= 3.2.1"
    }

    # ─── Not Equal Constraint (!=) ────────────────────────
    # != 3.2.0 (ทุก version ยกเว้น 3.2.0)
    archive = {
      source  = "hashicorp/archive"
      version = ">= 2.0.0, != 2.2.0"  # ข้าม 2.2.0 เพราะมี bug
    }
  }
}
```

### ความแตกต่างระหว่าง Constraint กับ Lock

```
Constraint (versions.tf):        Lock (.terraform.lock.hcl):
─────────────────────────        ─────────────────────────────
กำหนด RANGE ที่ยอมรับ           กำหนด EXACT version ที่ใช้
~> 5.0                           5.20.0
>= 3.0.0, < 4.0.0               3.5.1

ทำงานร่วมกัน:
- Constraint = "ต้องอยู่ในช่วงนี้"
- Lock = "version ที่เลือกไว้แล้วในช่วงนี้"
```

---

## Step 334: Hash Verification

### ประเภทของ Hashes

```
Hash Types ใน Lock File:

h1: (HashZip)
- Hash ของ ZIP file ที่ดาวน์โหลดมา
- เหมือนกันทุก platform
- Format: h1:<base64-encoded-hash>=

zh: (HashDir)  
- Hash ของ directory หลัง extract
- แตกต่างตาม platform (linux/darwin/windows + amd64/arm64)
- Format: zh:<hex-hash>
```

### การ Verify Hashes

```bash
# Terraform verify hashes โดยอัตโนมัติเมื่อ terraform init
# ถ้า hash ไม่ตรง จะ error:

# Error: Failed to install provider
#
# Error while installing hashicorp/aws v5.20.0: the current package for
# registry.terraform.io/hashicorp/aws 5.20.0 doesn't match any of the
# checksums previously recorded in the dependency lock file.

# แก้โดย:
# 1. ถ้า intentional upgrade:
terraform init -upgrade

# 2. ถ้าไม่แน่ใจ - ลบ lock file แล้ว init ใหม่
rm .terraform.lock.hcl
terraform init

# 3. ถ้าต้องการ add hashes สำหรับ platform ใหม่:
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64
```

---

## Step 335: Platforms (linux/darwin/windows hashes)

### ทำไม Hashes ถึงแตกต่างตาม Platform?

```
Provider binary แตกต่างตาม OS และ Architecture:

hashicorp/aws 5.20.0:
├── linux_amd64      → terraform-provider-aws_5.20.0_linux_amd64.zip
├── linux_arm64      → terraform-provider-aws_5.20.0_linux_arm64.zip
├── darwin_amd64     → terraform-provider-aws_5.20.0_darwin_amd64.zip
├── darwin_arm64     → terraform-provider-aws_5.20.0_darwin_arm64.zip (M1/M2)
└── windows_amd64    → terraform-provider-aws_5.20.0_windows_amd64.zip
```

### Lock File สำหรับ Multiple Platforms

```bash
# Lock สำหรับ platform เดียว (default - platform ปัจจุบัน)
terraform init

# Lock สำหรับหลาย platforms พร้อมกัน
terraform providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64 \
  -platform=windows_amd64

# ทำไมต้อง lock หลาย platforms?
# - Dev ใช้ macOS (darwin_arm64 / darwin_amd64)
# - CI/CD ใช้ Linux (linux_amd64)
# - ถ้า lock file มีแค่ darwin hash และ CI/CD รัน terraform init
#   จะ error เพราะไม่มี linux hash
```

### ตัวอย่าง Lock File ที่มี Multiple Platforms

```hcl
# .terraform.lock.hcl (complete with all platforms)

provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.20.0"
  constraints = "~> 5.0"
  hashes = [
    # h1: Single hash สำหรับทุก platform
    "h1:2GduHfuuJC0GFAhRrHpwKELBWUuJz8GjB+f+oXRoVnI=",

    # zh: Platform-specific hashes
    # Linux AMD64
    "zh:0b3b3dc8b4f7c6d5e4f3g2h1i0j9k8l7m6n5o4p3q2r1s0t9u8v7w6x5y4z3a2",
    # Linux ARM64
    "zh:1c4c4ed9c5g8d7e6f5g4h3i2j1k0l9m8n7o6p5q4r3s2t1u0v9w8x7y6z5a4b3",
    # Darwin AMD64 (Intel Mac)
    "zh:2d5d5fe0d6h9e8f7g6h5i4j3k2l1m0n9o8p7q6r5s4t3u2v1w0x9y8z7a6b5c4",
    # Darwin ARM64 (M1/M2 Mac)
    "zh:3e6e6gf1e7i0f9g8h7i6j5k4l3m2n1o0p9q8r7s6t5u4v3w2x1y0z9a8b7c6d5",
    # Windows AMD64
    "zh:4f7f7hg2f8j1g0h9i8j7k6l5m4n3o2p1q0r9s8t7u6v5w4x3y2z1a0b9c8d7e6",
  ]
}
```

---

## Step 336: Committing Lock File to VCS

### ✅ ควร Commit Lock File

```bash
# Lock file ควร commit เข้า Git เสมอ!

git add .terraform.lock.hcl
git commit -m "feat: pin provider versions"

# ทำไม?
# 1. ทุกคนใน team ใช้ provider version เดียวกัน
# 2. CI/CD reproducible
# 3. สามารถ rollback provider version ได้
# 4. Track changes ใน provider versions
```

### .gitignore สำหรับ Terraform

```bash
# .gitignore

# ─── Commit เหล่านี้ ──────────────────────────────────────
# .terraform.lock.hcl  ← อย่าใส่ใน gitignore!

# ─── ไม่ควร Commit ────────────────────────────────────────

# Terraform state files (sensitive data)
*.tfstate
*.tfstate.*
*.tfstate.backup

# Terraform working directory
.terraform/

# Terraform plan files (อาจมี sensitive data)
*.tfplan
tfplan

# Sensitive variable files
*.tfvars
!example.tfvars       # ยกเว้น example file
terraform.tfvars.json

# Crash logs
crash.log
crash.*.log

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# CLI config
.terraformrc
terraform.rc
```

### Lock File ใน .gitignore ที่ผิด (Anti-pattern)

```bash
# ❌ อย่าทำแบบนี้! 
# บางคน add .terraform.lock.hcl เข้า .gitignore โดยเข้าใจผิด

# .gitignore (WRONG!)
.terraform/
.terraform.lock.hcl   # ❌ ไม่ควรอยู่ที่นี่!
*.tfstate
```

---

## Step 337: terraform providers lock Command

### การใช้ terraform providers lock

```bash
# Lock providers สำหรับ current platform
terraform providers lock

# Lock สำหรับ specific platforms
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64

# Lock เฉพาะ specific providers
terraform providers lock \
  -platform=linux_amd64 \
  registry.terraform.io/hashicorp/aws \
  registry.terraform.io/hashicorp/random

# Lock แล้วแสดง verbose output
terraform providers lock -platform=linux_amd64 2>&1 | tee lock_output.txt

# Output ตัวอย่าง:
# - Fetching hashicorp/aws 5.20.0 for linux_amd64...
# - Retrieved hashicorp/aws 5.20.0 for linux_amd64
# - Obtained hashicorp/aws checksums for linux_amd64
# ...
# Success! Terraform has updated the lock file.
```

### ขั้นตอน Setup Lock File ที่ดี

```bash
#!/bin/bash
# scripts/init_lock_file.sh
# สร้าง lock file ที่รองรับทุก platform

set -e

echo "🔐 Setting up Terraform lock file for all platforms..."

# ตรวจสอบว่ามี terraform
if ! command -v terraform &> /dev/null; then
  echo "❌ Terraform not found"
  exit 1
fi

# Init ก่อน
terraform init -backend=false

# Lock สำหรับทุก platform
terraform providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64 \
  -platform=windows_amd64

echo "✅ Lock file updated for all platforms"
echo "📋 Don't forget to commit .terraform.lock.hcl!"
git diff .terraform.lock.hcl
```

---

## Step 338: Updating Lock File

### terraform init -upgrade

```bash
# อัปเดต providers ให้เป็น latest version ที่ตรง constraints
terraform init -upgrade

# ตัวอย่าง output:
# Upgrading modules...
# - hashicorp/aws: 5.20.0 -> 5.25.0
# - hashicorp/random: 3.5.1 -> 3.6.0
# 
# Warning: new provider versions may introduce breaking changes

# อัปเดต specific provider
terraform init -upgrade -no-color 2>&1 | grep "aws"
```

### Workflow การอัปเดต Providers

```bash
#!/bin/bash
# scripts/upgrade_providers.sh

set -e

echo "📦 Current provider versions:"
terraform providers

echo ""
echo "🔄 Upgrading providers..."
terraform init -upgrade

echo ""
echo "📦 New provider versions:"
terraform providers

echo ""
echo "🧪 Running terraform plan to check for changes..."
terraform plan -out=upgrade_plan.tfplan

echo ""
echo "📋 Changes after provider upgrade:"
terraform show -no-color upgrade_plan.tfplan

echo ""
echo "❓ Do you want to apply these changes? (y/N)"
read -r response

if [[ "$response" =~ ^[Yy]$ ]]; then
  terraform apply upgrade_plan.tfplan
  echo "✅ Provider upgrade applied"
else
  echo "⏭  Skipping apply. Provider versions updated in lock file."
  echo "📝 Commit .terraform.lock.hcl to save the upgrade"
fi

# Cleanup
rm -f upgrade_plan.tfplan
```

### Selective Provider Upgrade

```bash
# ไม่มี built-in way ของการ upgrade เฉพาะ provider เดียว
# แต่ทำได้ด้วย:

# 1. เปลี่ยน version constraint ใน versions.tf
# จาก: ~> 5.0
# เป็น: ~> 5.25  (บังคับใช้ version ใหม่)

# 2. แล้วรัน
terraform init -upgrade

# หรือ 3. ลบ provider entry ใน lock file แล้ว init ใหม่
# (แต่ต้องระวัง - ให้ทำใน test environment ก่อน)
```

---

## Step 339: Lock File ใน Team Collaboration

### Team Workflow กับ Lock File

```
Team Workflow:

Developer A (macOS M2):
  git clone repo
  terraform init          → ใช้ versions จาก .terraform.lock.hcl
                           → download darwin_arm64 binaries
  
Developer B (Ubuntu):
  git clone repo
  terraform init          → ใช้ versions จาก .terraform.lock.hcl
                           → download linux_amd64 binaries
  
CI/CD (Linux):
  git checkout
  terraform init          → ใช้ versions จาก .terraform.lock.hcl
                           → download linux_amd64 binaries

ทุกคนใช้ version เดียวกัน (จาก lock file)!
```

### Lock File Conflicts ใน Git

```bash
# เมื่อ 2 คน อัปเดต lock file พร้อมกัน จะเกิด merge conflict

# ตัวอย่าง conflict:
<<<<<<< HEAD
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.21.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:aaa111...",
=======
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.22.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:bbb222...",
>>>>>>> feature/upgrade-providers

# วิธีแก้ Lock File Conflicts:
```

### วิธีแก้ Lock File Conflicts

```bash
# วิธีที่ 1: Accept ฝั่งใดฝั่งหนึ่ง แล้ว reinit
# (แนะนำสำหรับ simple conflicts)

# Accept ฝั่ง main
git checkout --ours .terraform.lock.hcl
terraform init  # ยืนยันว่า lock file ถูกต้อง

# หรือ accept ฝั่ง feature
git checkout --theirs .terraform.lock.hcl
terraform init

# วิธีที่ 2: Regenerate lock file ใหม่
# (แนะนำสำหรับ complex conflicts)
git checkout main -- .terraform.lock.hcl  # เริ่มจาก main
terraform init -upgrade  # อัปเดตเป็น latest
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64

# วิธีที่ 3: ใช้ higher version เสมอ
# ถ้าต้องเลือกระหว่าง 5.21.0 กับ 5.22.0 → ใช้ 5.22.0

# หลังแก้ conflict:
git add .terraform.lock.hcl
git commit -m "fix: resolve lock file conflict, use aws 5.22.0"
```

---

## Step 340: Lock File สำหรับ Air-gapped Environments

### Air-gapped Environment คืออะไร?

**Air-gapped environment** คือ environment ที่ **ไม่มีการเชื่อมต่อ Internet** ซึ่งพบบ่อยใน:
- Enterprise environments ที่มีข้อกำหนด security เข้มงวด
- Government/Military systems
- Industrial control systems

### Provider Mirror Protocol

Terraform รองรับ 2 ประเภทของ mirror:

#### 1. filesystem_mirror

```hcl
# ~/.terraform.d/plugins/ หรือ ตาม OS:
# Windows: %APPDATA%\terraform.d\plugins\
# macOS/Linux: ~/.terraform.d/plugins/

# รูปแบบ directory structure สำหรับ filesystem mirror:
# ~/.terraform.d/plugins/
# └── registry.terraform.io/
#     └── hashicorp/
#         └── aws/
#             ├── 5.20.0/
#             │   ├── linux_amd64/
#             │   │   └── terraform-provider-aws_5.20.0_linux_amd64.zip
#             │   └── darwin_arm64/
#             │       └── terraform-provider-aws_5.20.0_darwin_arm64.zip
#             └── terraform-provider-aws_5.20.0_SHA256SUMS
```

```hcl
# ~/.terraformrc หรือ %APPDATA%\terraform.rc

provider_installation {
  filesystem_mirror {
    path    = "/opt/terraform-providers"  # path ของ local mirror
    include = ["registry.terraform.io/*/*"]
  }
  
  # Fallback to official registry (ถ้า provider ไม่อยู่ใน mirror)
  # ลบ/comment บรรทัดนี้สำหรับ fully air-gapped
  direct {
    exclude = ["registry.terraform.io/*/*"]  # ไม่ download จาก internet
  }
}
```

#### 2. network_mirror

```hcl
# ~/.terraformrc

provider_installation {
  network_mirror {
    url     = "https://internal-mirror.company.com/terraform/"
    include = ["registry.terraform.io/hashicorp/*"]
  }
  
  # ไม่มี fallback - fully air-gapped
}
```

### การสร้าง Provider Bundle สำหรับ Air-gapped

```bash
#!/bin/bash
# scripts/create_provider_bundle.sh
# สร้าง provider bundle สำหรับ air-gapped environment

set -e

BUNDLE_DIR="/tmp/tf-providers-bundle"
PROVIDERS_DIR="$BUNDLE_DIR/providers"

echo "📦 Creating provider bundle for air-gapped environment..."

# สร้าง directories
mkdir -p "$PROVIDERS_DIR"

# Init เพื่อดาวน์โหลด providers
terraform init

# Copy providers จาก .terraform/providers
cp -r .terraform/providers/* "$PROVIDERS_DIR/"

# สร้าง lock file copy
cp .terraform.lock.hcl "$BUNDLE_DIR/"

# สร้าง mirror script
cat > "$BUNDLE_DIR/setup_mirror.sh" << 'EOF'
#!/bin/bash
# Setup local provider mirror

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
MIRROR_PATH="$HOME/.terraform-mirrors"

mkdir -p "$MIRROR_PATH"
cp -r "$SCRIPT_DIR/providers/"* "$MIRROR_PATH/"

cat >> ~/.terraformrc << TFRC
provider_installation {
  filesystem_mirror {
    path    = "$MIRROR_PATH"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
TFRC

echo "✅ Local provider mirror configured at: $MIRROR_PATH"
EOF

chmod +x "$BUNDLE_DIR/setup_mirror.sh"

# สร้าง README
cat > "$BUNDLE_DIR/README.md" << 'EOF'
# Terraform Provider Bundle

## Setup
1. Copy this bundle to the air-gapped machine
2. Run: ./setup_mirror.sh
3. Run: terraform init (will use local mirror)

## Included Providers
$(ls providers/)

## Usage
After running setup_mirror.sh, all terraform operations
will use the local provider mirror instead of the internet.
EOF

# สร้าง tar bundle
cd /tmp
tar -czf "tf-providers-$(date +%Y%m%d).tar.gz" tf-providers-bundle/

echo "✅ Bundle created: /tmp/tf-providers-$(date +%Y%m%d).tar.gz"
echo "📋 Copy this to the air-gapped machine and run setup_mirror.sh"
```

### network_mirror Setup (Internal Artifactory/Nexus)

```hcl
# ตัวอย่าง: ใช้ JFrog Artifactory เป็น Provider Mirror

# ~/.terraformrc
provider_installation {
  network_mirror {
    # Artifactory endpoint
    url = "https://artifactory.company.com/artifactory/terraform-providers/"
    
    # Credentials (ถ้าต้อง auth)
    # ใช้ environment variables แทน hardcode
    # TERRAFORM_HTTP_AUTH_HEADER = "Authorization: Bearer <token>"
    
    # ดาวน์โหลดจาก mirror เฉพาะ hashicorp providers
    include = [
      "registry.terraform.io/hashicorp/*",
      "registry.terraform.io/datadog/*",
    ]
  }
  
  # สำหรับ providers อื่นที่ไม่ได้อยู่ใน mirror
  direct {
    include = [
      "registry.terraform.io/community/*",
    ]
  }
}
```

### Terraform Enterprise / HCP Terraform Network Mirror

```hcl
# สำหรับ Terraform Enterprise
# provider_installation ถูก configure ใน TFE settings

# แต่สำหรับ CLI ให้ใช้:
# ~/.terraformrc

credentials "tfe.company.com" {
  token = "your-tfe-token"
}

provider_installation {
  network_mirror {
    url = "https://tfe.company.com/api/registry/v1/providers/"
  }
}
```

---

## สรุป: Lock File Best Practices

### Checklist สำหรับทีม

```bash
# ✅ Lock File Best Practices Checklist:

# 1. Commit lock file เข้า Git เสมอ
git add .terraform.lock.hcl
git commit -m "chore: update provider lock file"

# 2. Lock สำหรับทุก platform ที่ใช้ใน team
terraform providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64

# 3. อย่า edit lock file ด้วยมือ
# ← ใช้ terraform init หรือ terraform providers lock แทน

# 4. Update lock file อย่างสม่ำเสมอ (monthly)
terraform init -upgrade
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64
git add .terraform.lock.hcl
git commit -m "chore: upgrade providers to latest versions"

# 5. Review lock file changes ใน Code Review
# ดูว่า provider version เปลี่ยนแปลงอะไรบ้างใน CHANGELOG
```

### Version Constraint Strategy แนะนำ

```hcl
# versions.tf - แนะนำ

terraform {
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    # ✅ แนะนำ: Pessimistic constraint
    # อนุญาต minor และ patch updates แต่ไม่อนุญาต major
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }

    # ✅ แนะนำ: Lock ที่ minor version
    # อนุญาตเฉพาะ patch updates
    google = {
      source  = "hashicorp/google"
      version = "~> 5.10"
    }

    # ❌ ไม่แนะนำ: ไม่มี constraint
    # random = {
    #   source = "hashicorp/random"
    # }

    # ❌ ไม่แนะนำ: Exact version (ทำให้ upgrade ยาก)
    # null = {
    #   source  = "hashicorp/null"
    #   version = "= 3.2.1"
    # }
  }
}
```

---

*จบ Part 034: Terraform Lock File (.terraform.lock.hcl)*

*ต่อไป: Part 035 - Terraform Configuration Best Practices*
