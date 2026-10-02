# Part 017: Terraform Installation & Configuration (การติดตั้งและตั้งค่า)
## Steps 161-170: ขั้นตอนการติดตั้ง Terraform บนทุก Platform

---

## บทนำ (Introduction)

การติดตั้ง Terraform สามารถทำได้หลายวิธีขึ้นอยู่กับ Operating System ที่ใช้ บทนี้ครอบคลุมการติดตั้งบน Linux, macOS, Windows รวมถึงการจัดการหลาย versions ด้วย tfenv และ asdf

---

## Step 161: ติดตั้งบน Linux

### ติดตั้งด้วย apt (Debian/Ubuntu)

```bash
# Step 1: เพิ่ม HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Step 2: เพิ่ม repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list

# Step 3: Update และ install
sudo apt-get update && sudo apt-get install terraform

# ตรวจสอบ
terraform version
# Terraform v1.7.0
# on linux_amd64
```

### ติดตั้งด้วย yum (RHEL/CentOS/Amazon Linux)

```bash
# Step 1: เพิ่ม HashiCorp repository
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# Step 2: Install
sudo yum -y install terraform

# ตรวจสอบ
terraform version
```

### ติดตั้งด้วย dnf (Fedora)

```bash
# เพิ่ม repository
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/fedora/hashicorp.repo

# Install
sudo dnf install terraform
```

### ติดตั้งจาก Binary (ทุก Linux)

```bash
# Step 1: ดาวน์โหลด binary
TERRAFORM_VERSION="1.7.0"
wget "https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip"

# Step 2: Unzip
unzip "terraform_${TERRAFORM_VERSION}_linux_amd64.zip"

# Step 3: ย้ายไป /usr/local/bin
sudo mv terraform /usr/local/bin/

# Step 4: ตรวจสอบ permission
sudo chmod +x /usr/local/bin/terraform

# Step 5: ตรวจสอบ
terraform version

# ทางเลือก: ติดตั้งใน user directory
mkdir -p ~/.local/bin
mv terraform ~/.local/bin/
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### ติดตั้งบน Amazon Linux 2

```bash
# Amazon Linux 2 ใช้ yum
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform

# หรือ binary install
curl -fsSL https://releases.hashicorp.com/terraform/1.7.0/terraform_1.7.0_linux_amd64.zip -o terraform.zip
unzip terraform.zip
sudo mv terraform /usr/local/bin/
terraform version
```

---

## Step 162: ติดตั้งบน macOS

### ติดตั้งด้วย Homebrew (แนะนำ)

```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง Terraform
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# หรือจาก community tap
brew install terraform

# ตรวจสอบ
terraform version

# Update Terraform
brew upgrade hashicorp/tap/terraform
```

### ติดตั้งจาก Binary (macOS)

```bash
# Intel Mac (amd64)
TERRAFORM_VERSION="1.7.0"
curl -fsSL "https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_darwin_amd64.zip" -o terraform.zip
unzip terraform.zip
sudo mv terraform /usr/local/bin/

# Apple Silicon Mac (arm64)
curl -fsSL "https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_darwin_arm64.zip" -o terraform.zip
unzip terraform.zip
sudo mv terraform /usr/local/bin/

# ตรวจสอบ architecture
uname -m
# arm64 = Apple Silicon
# x86_64 = Intel
```

### macOS Shell Completion

```bash
# Bash
terraform -install-autocomplete

# Zsh
# เพิ่มใน ~/.zshrc
autoload -U +X bashcompinit && bashcompinit
complete -o nospace -C /usr/local/bin/terraform terraform

# Fish shell
terraform -install-autocomplete
```

---

## Step 163: ติดตั้งบน Windows

### ติดตั้งด้วย Chocolatey

```powershell
# ติดตั้ง Chocolatey ก่อน (run as Administrator)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# ติดตั้ง Terraform
choco install terraform

# ตรวจสอบ
terraform version

# Update
choco upgrade terraform
```

### ติดตั้งจาก Binary (Windows)

```powershell
# Step 1: ดาวน์โหลด
$version = "1.7.0"
$url = "https://releases.hashicorp.com/terraform/$version/terraform_${version}_windows_amd64.zip"
Invoke-WebRequest -Uri $url -OutFile "terraform.zip"

# Step 2: Extract
Expand-Archive -Path "terraform.zip" -DestinationPath "C:\terraform"

# Step 3: เพิ่ม PATH
$env:PATH += ";C:\terraform"
[System.Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";C:\terraform", "Machine")

# Step 4: ตรวจสอบ (เปิด terminal ใหม่)
terraform version
```

### ติดตั้งด้วย Winget

```powershell
# Windows Package Manager (Windows 11 / Windows 10 1809+)
winget install Hashicorp.Terraform
```

### การใช้งานบน WSL (Windows Subsystem for Linux)

```bash
# WSL Ubuntu
# ใช้วิธีเดียวกับ Linux Ubuntu

# แนะนำ: ใช้ WSL2 สำหรับ performance ที่ดีขึ้น
wsl --set-default-version 2

# ติดตั้ง Ubuntu ใน WSL
wsl --install -d Ubuntu

# จากนั้น follow Linux installation steps
```

---

## Step 164: tfenv - Terraform Version Manager

### ทำไมต้องใช้ tfenv?

```
ปัญหา: แต่ละโปรเจค/ทีมใช้ Terraform version ต่างกัน
- Project A: Terraform 1.5.0
- Project B: Terraform 1.7.0
- Legacy: Terraform 0.15.0

tfenv ช่วยสลับระหว่าง versions ได้ง่าย
```

### ติดตั้ง tfenv

```bash
# ติดตั้งบน macOS
brew install tfenv

# ติดตั้งบน Linux (manual)
git clone --depth=1 https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bash_profile
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.zshrc
source ~/.bash_profile  # หรือ source ~/.zshrc
```

### การใช้งาน tfenv

```bash
# List versions ที่ติดตั้งแล้ว
tfenv list

# List versions ที่ available
tfenv list-remote

# ติดตั้ง version ที่ต้องการ
tfenv install 1.7.0
tfenv install 1.6.6
tfenv install latest  # version ล่าสุด

# เลือก version ที่จะใช้ (global)
tfenv use 1.7.0

# ตรวจสอบ
terraform version

# ถอนการติดตั้ง version
tfenv uninstall 1.5.0

# Pin version สำหรับ project (สร้างไฟล์ .terraform-version)
echo "1.7.0" > .terraform-version

# tfenv จะใช้ version จาก .terraform-version โดยอัตโนมัติ
cd /path/to/project
terraform version  # ใช้ version ที่ระบุใน .terraform-version
```

### .terraform-version ไฟล์

```bash
# .terraform-version ใน project directory
cat .terraform-version
# 1.7.0

# tfenv จะ auto-detect และ switch version
# ถ้า version ยังไม่ได้ติดตั้ง จะ install อัตโนมัติ
```

---

## Step 165: asdf - Universal Version Manager

### ติดตั้ง asdf

```bash
# macOS
brew install asdf

# Linux
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.0
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.bashrc
echo '. "$HOME/.asdf/completions/asdf.bash"' >> ~/.bashrc
source ~/.bashrc
```

### ติดตั้ง Terraform Plugin สำหรับ asdf

```bash
# เพิ่ม terraform plugin
asdf plugin add terraform https://github.com/asdf-community/asdf-hashicorp.git

# ติดตั้ง version
asdf install terraform 1.7.0
asdf install terraform 1.6.6

# กำหนด global version
asdf global terraform 1.7.0

# กำหนด local version (สร้าง .tool-versions)
asdf local terraform 1.7.0

# ดู versions ที่ติดตั้ง
asdf list terraform

# ตรวจสอบ
terraform version
```

### .tool-versions ไฟล์

```
# .tool-versions
terraform 1.7.0
nodejs 20.0.0
python 3.11.0
```

---

## Step 166: ตรวจสอบการติดตั้ง

```bash
# ตรวจสอบ version
terraform version
# Terraform v1.7.0
# on linux_amd64
# + provider registry.terraform.io/hashicorp/aws v5.31.0

# ดู help
terraform help
terraform --help
terraform -help

# ดู subcommand help
terraform init --help
terraform plan --help
terraform apply --help

# ทดสอบ การทำงาน
mkdir test-terraform && cd test-terraform

cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.0"
}

output "hello" {
  value = "Hello, Terraform!"
}
EOF

terraform init   # Initialize
terraform plan   # Show plan
terraform apply  # Apply (พิมพ์ yes)
terraform output # Show outputs
```

---

## Step 167: Environment Variables

### TERRAFORM_LOG - Logging

```bash
# Log levels: TRACE, DEBUG, INFO, WARN, ERROR, OFF
export TF_LOG=DEBUG

# ดู log ระหว่าง run
terraform plan

# เปิด trace log (มาก output มาก)
export TF_LOG=TRACE

# ปิด logging
export TF_LOG=OFF
# หรือ unset
unset TF_LOG
```

### TERRAFORM_LOG_PATH - Log to File

```bash
# เก็บ log ลงไฟล์
export TF_LOG=DEBUG
export TF_LOG_PATH=/tmp/terraform.log

# Run command
terraform apply

# ดู log file
cat /tmp/terraform.log
tail -f /tmp/terraform.log  # Follow

# แยก provider log
export TF_LOG_PROVIDER=DEBUG
export TF_LOG_PROVIDER_PATH=/tmp/provider.log
```

### TF_CLI_ARGS - Default Arguments

```bash
# เพิ่ม arguments อัตโนมัติ
export TF_CLI_ARGS="-no-color"  # ปิด color output

# สำหรับ specific command
export TF_CLI_ARGS_plan="-compact-warnings"
export TF_CLI_ARGS_apply="-auto-approve"

# ตัวอย่างใน CI/CD
export TF_CLI_ARGS_init="-backend-config=bucket=my-tf-state"
export TF_CLI_ARGS_plan="-out=tfplan"
export TF_CLI_ARGS_apply="tfplan"
```

### TF_DATA_DIR - Plugin Directory

```bash
# Default: .terraform/
# Override:
export TF_DATA_DIR=/shared/.terraform

# ประโยชน์: share plugins ระหว่าง directories
# ลด disk space และ download time
```

### TF_WORKSPACE - Default Workspace

```bash
# กำหนด workspace เริ่มต้น
export TF_WORKSPACE=prod

# แทนที่จะพิมพ์
terraform workspace select prod
terraform plan

# แค่
terraform plan  # ใช้ workspace prod โดยอัตโนมัติ
```

### TF_VAR_* - Variable Values

```bash
# ส่งค่า variable
export TF_VAR_environment=prod
export TF_VAR_region=ap-southeast-1
export TF_VAR_database_password="$(aws secretsmanager get-secret-value \
  --secret-id prod/db/password \
  --query SecretString \
  --output text)"

terraform apply
```

### TF_CLI_CONFIG_FILE - Config File Location

```bash
# Default: ~/.terraformrc
# Override location:
export TF_CLI_CONFIG_FILE=/custom/path/.terraformrc
```

### ตัวอย่าง CI/CD Environment Variables

```bash
#!/bin/bash
# ci-deploy.sh

# Terraform settings
export TF_CLI_ARGS="-no-color"
export TF_IN_AUTOMATION=true  # reduces verbose output

# AWS credentials
export AWS_ACCESS_KEY_ID="${AWS_ACCESS_KEY_ID}"
export AWS_SECRET_ACCESS_KEY="${AWS_SECRET_ACCESS_KEY}"
export AWS_DEFAULT_REGION="ap-southeast-1"

# Application variables
export TF_VAR_environment="prod"
export TF_VAR_project_name="myapp"
export TF_VAR_db_password="${DB_PASSWORD}"  # from CI/CD secret

# Initialize and apply
terraform init \
  -backend-config="bucket=${TF_STATE_BUCKET}" \
  -backend-config="key=prod/terraform.tfstate" \
  -backend-config="region=ap-southeast-1"

terraform plan -out=tfplan
terraform apply tfplan
```

---

## Step 168: Terraform CLI Configuration File

### ~/.terraformrc (Linux/macOS) หรือ %APPDATA%/terraform.rc (Windows)

```hcl
# ~/.terraformrc

# Plugin cache directory (ลด download time)
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"

# Disable checkpoint (ข้าม version check)
disable_checkpoint = true

# Provider installation methods
provider_installation {
  # ใช้ local mirror ก่อน
  filesystem_mirror {
    path    = "/usr/share/terraform/providers"
    include = ["registry.terraform.io/hashicorp/*"]
  }
  
  # ถ้าไม่พบใน local mirror ดาวน์โหลดจาก registry
  direct {
    exclude = ["example.com/*/*"]
  }
}

# Credentials สำหรับ private registry
credentials "app.terraform.io" {
  token = "xxxxxxxxxxxxxxxxxxxxxx"
}

# Custom hostname override
host "my-private-registry.example.com" {
  services = {
    "modules.v1" = "https://my-private-registry.example.com/api/v1/modules/"
    "providers.v1" = "https://my-private-registry.example.com/api/v1/providers/"
  }
}
```

### Plugin Cache Directory

```bash
# สร้าง cache directory
mkdir -p ~/.terraform.d/plugin-cache

# เพิ่มใน ~/.terraformrc
echo 'plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"' >> ~/.terraformrc

# ผล: providers จะถูก cache ใน directory นี้
# ไม่ต้อง download ซ้ำเมื่อ init projects ใหม่ที่ใช้ provider เดียวกัน

# ดู cached providers
ls ~/.terraform.d/plugin-cache/
```

---

## Step 169: Network Requirements และ Proxy Settings

### Network Requirements

```bash
# Terraform ต้องการ access to:
# 1. registry.terraform.io - download providers/modules
# 2. releases.hashicorp.com - version checks
# 3. Cloud provider APIs (AWS, GCP, Azure)

# ตรวจสอบ network access
curl -I https://registry.terraform.io
curl -I https://releases.hashicorp.com

# Test AWS API access
aws sts get-caller-identity
```

### HTTP/HTTPS Proxy

```bash
# กำหนด proxy
export HTTPS_PROXY=http://proxy.example.com:8080
export HTTP_PROXY=http://proxy.example.com:8080
export NO_PROXY=localhost,127.0.0.1,.internal.company.com

# หรือใน ~/.terraformrc
# Terraform ใช้ HTTP_PROXY / HTTPS_PROXY environment variables โดยอัตโนมัติ

# ทดสอบ
terraform version  # ต้อง check version ผ่าน internet
```

### Airgapped/Offline Installation

```bash
# สำหรับ environment ที่ไม่มี internet access

# 1. Download providers ล่วงหน้า
terraform providers mirror /path/to/mirror

# 2. ตั้งค่า ~/.terraformrc ใช้ local mirror
cat > ~/.terraformrc << 'EOF'
provider_installation {
  filesystem_mirror {
    path    = "/path/to/mirror"
    include = ["registry.terraform.io/*/*"]
  }
  direct {
    exclude = ["registry.terraform.io/*/*"]
  }
}
EOF

# 3. Init จะใช้ local mirror แทน registry
terraform init
```

---

## Step 170: First terraform init

### ทำความเข้าใจ terraform init

```bash
# terraform init ทำอะไร?
# 1. ดาวน์โหลด providers ที่ระบุใน required_providers
# 2. ดาวน์โหลด modules
# 3. Configure backend
# 4. สร้าง .terraform directory
# 5. สร้าง/update .terraform.lock.hcl
```

### ตัวอย่าง First Project

```hcl
# main.tf
terraform {
  required_version = ">= 1.3.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "my-project/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

provider "aws" {
  region = "ap-southeast-1"
}

data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}
```

```bash
# ขั้นตอนแรก
terraform init

# Output:
# Initializing the backend...
# 
# Initializing provider plugins...
# - Finding hashicorp/aws versions matching "~> 5.0"...
# - Installing hashicorp/aws v5.31.0...
# - Installed hashicorp/aws v5.31.0 (signed by HashiCorp)
#
# Terraform has created a lock file .terraform.lock.hcl to record the provider
# selections it made above.
#
# Terraform has been successfully initialized!

# ดู ที่ถูกสร้าง
ls -la .terraform/
# providers/
# terraform.tfstate (if using local backend)

cat .terraform.lock.hcl
```

### terraform init Options

```bash
# อัพเกรด providers
terraform init -upgrade

# Reconfigure backend
terraform init -reconfigure

# Migrate state
terraform init -migrate-state

# Backend config ผ่าน CLI
terraform init \
  -backend-config="bucket=my-state-bucket" \
  -backend-config="key=prod/terraform.tfstate" \
  -backend-config="region=ap-southeast-1"

# ไม่ดาวน์โหลด modules (ใช้ cached)
terraform init -get=false

# ไม่ดาวน์โหลด providers
terraform init -get-plugins=false
```

---

## Environment Setup สำหรับ Production

### Complete .bashrc / .zshrc Setup

```bash
# ~/.bashrc หรือ ~/.zshrc

# Terraform version manager
export PATH="$HOME/.tfenv/bin:$PATH"

# Terraform aliases
alias tf="terraform"
alias tfi="terraform init"
alias tfp="terraform plan"
alias tfa="terraform apply"
alias tfd="terraform destroy"
alias tff="terraform fmt -recursive"
alias tfv="terraform validate"

# Terraform environment
export TF_LOG=WARN  # Production: ใช้ WARN
export TF_LOG_PATH="/tmp/terraform-$(date +%Y%m%d).log"

# Plugin cache
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
mkdir -p "$TF_PLUGIN_CACHE_DIR"

# Auto-select workspace from environment
tf-select-env() {
  local env=${1:-dev}
  terraform workspace select $env || terraform workspace new $env
  echo "Selected workspace: $(terraform workspace show)"
}

# Function: terraform plan with output file
tf-plan() {
  terraform plan -out=tfplan "$@"
}

# Function: terraform apply with saved plan
tf-apply() {
  if [ -f tfplan ]; then
    terraform apply tfplan
    rm -f tfplan
  else
    terraform apply "$@"
  fi
}
```

---

## สรุป (Summary)

### Quick Reference - Installation Commands

| Platform | Method | Command |
|----------|--------|---------|
| Ubuntu/Debian | apt | `sudo apt-get install terraform` |
| RHEL/CentOS | yum | `sudo yum install terraform` |
| macOS | brew | `brew install hashicorp/tap/terraform` |
| Windows | choco | `choco install terraform` |
| Any | tfenv | `tfenv install 1.7.0` |
| Any | binary | Download from releases.hashicorp.com |

### Key Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `TF_LOG` | Log level | `DEBUG`, `INFO`, `WARN` |
| `TF_LOG_PATH` | Log file path | `/tmp/tf.log` |
| `TF_CLI_ARGS` | Default args | `-no-color` |
| `TF_DATA_DIR` | Plugin directory | `/shared/.terraform` |
| `TF_WORKSPACE` | Active workspace | `prod` |
| `TF_VAR_*` | Variable values | `TF_VAR_env=prod` |
| `TF_IN_AUTOMATION` | CI/CD mode | `true` |

### ✅ Best Practices

1. **ใช้ tfenv** จัดการ versions ใน development
2. **Pin version** ด้วย `.terraform-version` ใน project
3. **Plugin cache** ลด download time
4. **Commit .terraform.lock.hcl** ใน git
5. **TF_IN_AUTOMATION=true** ใน CI/CD

### ⚠️ Common Issues

```bash
# Error: provider version mismatch
terraform init -upgrade

# Error: backend state lock
terraform force-unlock <lock-id>

# Error: provider not found
# ตรวจสอบ internet connection และ proxy settings

# Error: permission denied
chmod +x /usr/local/bin/terraform
```

---

*จบ Part 017 - Terraform Installation & Configuration*
