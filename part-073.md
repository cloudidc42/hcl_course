# Part 073: Terratest Integration Testing (ขั้นตอนที่ 721-730)

## บทนำ (Introduction)

Terratest เป็น Go library สำหรับการทำ automated testing ของ Infrastructure code
พัฒนาโดย Gruntwork ช่วยให้สามารถ test Terraform, Packer, Docker, Kubernetes ได้

---

## ขั้นตอนที่ 721: Terratest Framework Overview

### เปรียบเทียบ Terraform test vs Terratest

| Feature | terraform test | Terratest |
|---------|---------------|-----------|
| ภาษา | HCL | Go |
| Setup | Built-in | Library ภายนอก |
| Mock Support | Yes (1.7+) | ต้องใช้ AWS เจริญ |
| Provider Support | Terraform providers | Any HTTP API |
| Docker Testing | ไม่ได้ | ได้ |
| K8s Testing | ไม่ได้ | ได้ |
| ความยืดหยุ่น | ต่ำ | สูง |

### ติดตั้ง Go สำหรับ Terratest

```bash
# ติดตั้ง Go (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install golang-go

# หรือดาวน์โหลดจาก https://golang.org
wget https://go.dev/dl/go1.21.0.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# ตรวจสอบ
go version
# go version go1.21.0 linux/amd64
```

---

## ขั้นตอนที่ 722: Go Language Basics สำหรับ Terratest

### สิ่งที่ต้องรู้ใน Go

```go
package main

import (
    "fmt"
    "testing"
)

// 1. Functions
func greet(name string) string {
    return fmt.Sprintf("Hello, %s!", name)
}

// 2. Structs
type Config struct {
    Region      string
    Environment string
}

// 3. Error handling
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil
}

// 4. Testing function (ต้องเริ่มด้วย Test)
func TestGreet(t *testing.T) {
    result := greet("World")
    expected := "Hello, World!"
    
    if result != expected {
        t.Errorf("Expected %s but got %s", expected, result)
    }
}

// 5. defer (เรียกเมื่อ function จบ - สำคัญมากสำหรับ cleanup)
func TestWithCleanup(t *testing.T) {
    defer cleanup() // จะเรียกตอน test จบ แม้จะ fail
    
    // test code...
}

// 6. t.Helper() - ทำให้ error message ชี้ไปยังที่เรียก function นี้
func assertNotEmpty(t *testing.T, value string, message string) {
    t.Helper()
    if value == "" {
        t.Errorf(message)
    }
}
```

---

## ขั้นตอนที่ 723: Setting Up Terratest Project

### โครงสร้าง Project

```
terraform-module/
├── main.tf
├── variables.tf
├── outputs.tf
└── test/
    ├── go.mod
    ├── go.sum
    ├── terraform_basic_test.go
    ├── terraform_vpc_test.go
    └── fixtures/
        ├── basic/
        │   ├── main.tf
        │   └── outputs.tf
        └── complete/
            ├── main.tf
            └── outputs.tf
```

### สร้าง go.mod

```bash
# ใน directory test/
mkdir -p test && cd test
go mod init github.com/mycompany/terraform-module/test
```

```go
// test/go.mod
module github.com/mycompany/terraform-module/test

go 1.21

require (
    github.com/gruntwork-io/terratest v0.46.7
    github.com/stretchr/testify v1.8.4
    github.com/aws/aws-sdk-go v1.49.0
)
```

```bash
# Download dependencies
go mod tidy
```

### test/fixtures/basic/main.tf

```hcl
# test/fixtures/basic/main.tf
# Minimal configuration for testing module

variable "region" {
  type    = string
  default = "us-east-1"
}

variable "environment" {
  type    = string
  default = "test"
}

provider "aws" {
  region = var.region
}

module "vpc" {
  source = "../../"  # path to module being tested

  cidr_block   = "10.0.0.0/16"
  environment  = var.environment
  project_name = "test-project"
}

output "vpc_id" {
  value = module.vpc.vpc_id
}

output "public_subnet_ids" {
  value = module.vpc.public_subnet_ids
}
```

---

## ขั้นตอนที่ 724: terraform.Options และ Basic Test

### Basic Terratest Pattern

```go
// test/terraform_basic_test.go
package test

import (
    "testing"
    "fmt"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestTerraformBasic(t *testing.T) {
    t.Parallel() // รัน test แบบ parallel กับ test อื่น

    // กำหนด options
    terraformOptions := &terraform.Options{
        // Path ไปยัง Terraform code ที่จะ test
        TerraformDir: "../examples/basic",

        // Input variables
        Vars: map[string]interface{}{
            "region":      "us-east-1",
            "environment": "test",
            "project":     "terratest-example",
        },

        // Environment variables (สำหรับ AWS credentials)
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": "us-east-1",
        },

        // ไม่แสดง Terraform output ใน test (ลด noise)
        // NoColor: true,
    }

    // Destroy resources หลัง test จบ (แม้จะ fail)
    defer terraform.Destroy(t, terraformOptions)

    // Init และ Apply
    terraform.InitAndApply(t, terraformOptions)

    // ดึง output values
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    
    // Assert
    assert.NotEmpty(t, vpcID, "VPC ID should not be empty")
    assert.Regexp(t, "^vpc-", vpcID, "VPC ID should start with 'vpc-'")
}
```

### terraform.Options ทุก field

```go
terraformOptions := &terraform.Options{
    // Required
    TerraformDir: "./fixture",

    // Variables (-var flag)
    Vars: map[string]interface{}{
        "region": "us-east-1",
        "count":  3,  // number
    },

    // Var files (-var-file flag)
    VarFiles: []string{"test.tfvars", "override.tfvars"},

    // Backend config (-backend-config flag)
    BackendConfig: map[string]interface{}{
        "bucket": "my-terraform-state",
        "key":    "test/terraform.tfstate",
    },

    // Targets (-target flag)
    Targets: []string{"aws_vpc.main", "aws_subnet.public"},

    // State path
    StateFilePath: "/tmp/terraform.tfstate",

    // Environment variables
    EnvVars: map[string]string{
        "AWS_DEFAULT_REGION": "us-east-1",
    },

    // Retry settings
    MaxRetries:         3,
    TimeBetweenRetries: 5 * time.Second,
    RetryableTerraformErrors: map[string]string{
        "RequestError: send request failed": "Transient AWS error",
    },

    // Parallelism
    Parallelism: 10,

    // Lock timeout
    LockTimeout: "10m",

    // No color
    NoColor: true,

    // Logger
    Logger: logger.Discard, // ปิด log

    // Upgrade providers
    Upgrade: true,

    // Reconfigure backend
    Reconfigure: true,
}
```

---

## ขั้นตอนที่ 725: terraform.InitAndApply และ Destroy

### Complete Lifecycle Test

```go
// test/terraform_lifecycle_test.go
package test

import (
    "testing"
    "time"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/retry"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestCompleteLifecycle(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/complete",
        Vars: map[string]interface{}{
            "environment": "test",
            "region":      "us-east-1",
        },
    })

    // Ensure cleanup happens even if test fails
    defer terraform.Destroy(t, terraformOptions)

    // Step 1: Initialize Terraform
    terraform.Init(t, terraformOptions)

    // Step 2: Create a plan
    planFilePath := terraform.Plan(t, terraformOptions)
    assert.NotEmpty(t, planFilePath)

    // Step 3: Show plan (optional)
    planOutput := terraform.Show(t, terraformOptions)
    assert.Contains(t, planOutput, "aws_vpc.main")

    // Step 4: Apply
    terraform.Apply(t, terraformOptions)

    // Step 5: Get outputs
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    publicSubnets := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
    privateSubnets := terraform.OutputList(t, terraformOptions, "private_subnet_ids")

    // Step 6: Assert
    assert.NotEmpty(t, vpcID)
    assert.Equal(t, 2, len(publicSubnets))
    assert.Equal(t, 2, len(privateSubnets))

    // Step 7: Apply again (idempotency check)
    exitCode := terraform.ApplyAndIdempotent(t, terraformOptions)
    assert.Equal(t, 0, exitCode, "Second apply should be idempotent (no changes)")
}
```

### ตรวจสอบ Idempotency

```go
func TestIdempotency(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/basic",
        Vars: map[string]interface{}{
            "environment": "test",
        },
    }

    defer terraform.Destroy(t, terraformOptions)

    // First apply
    terraform.InitAndApply(t, terraformOptions)

    // Second apply - should have no changes
    stdout := terraform.Apply(t, terraformOptions)
    
    // ตรวจสอบว่าไม่มี changes
    assert.Contains(t, stdout, "0 to add, 0 to change, 0 to destroy")
}
```

---

## ขั้นตอนที่ 726: terraform.Output และ AWS Helpers

### Getting Different Output Types

```go
func TestOutputs(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/basic",
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // String output
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    
    // List output
    subnetIDs := terraform.OutputList(t, terraformOptions, "subnet_ids")
    
    // Map output
    tags := terraform.OutputMap(t, terraformOptions, "resource_tags")
    
    // Map of strings
    instanceIPs := terraform.OutputMapOfObjects(t, terraformOptions, "instance_ips")
    
    // JSON output
    configJSON := terraform.OutputJson(t, terraformOptions, "config")
    
    // Assertions
    assert.NotEmpty(t, vpcID)
    assert.Equal(t, 3, len(subnetIDs))
    assert.Equal(t, "production", tags["Environment"])
    _ = configJSON
    _ = instanceIPs
}
```

### AWS Package Helpers

```go
// test/terraform_aws_test.go
package test

import (
    "testing"
    "strings"

    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestAWSResources(t *testing.T) {
    t.Parallel()

    region := "us-east-1"
    
    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/aws-resources",
        Vars: map[string]interface{}{
            "region": region,
        },
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // ดึง outputs
    bucketName := terraform.Output(t, terraformOptions, "bucket_name")
    instanceID := terraform.Output(t, terraformOptions, "instance_id")
    paramName  := terraform.Output(t, terraformOptions, "parameter_name")

    // --- S3 Tests ---
    // ตรวจสอบ S3 bucket มีอยู่จริง
    aws.AssertS3BucketExists(t, region, bucketName)
    
    // ตรวจสอบ versioning
    aws.AssertS3BucketVersioningExists(t, region, bucketName)
    
    // ตรวจสอบ encryption
    aws.AssertS3BucketServerSideEncryptionEnabled(t, region, bucketName)

    // --- EC2 Tests ---
    // ดึงข้อมูล instance
    instance := aws.GetEc2InstanceIdsByTag(t, region, "Environment", "test")
    assert.Contains(t, instance, instanceID)
    
    // ตรวจสอบ instance state
    instanceState := aws.GetInstanceState(t, region, instanceID)
    assert.Equal(t, "running", instanceState)

    // --- SSM Parameter Store Tests ---
    // ดึงค่า parameter
    paramValue := aws.GetParameter(t, region, paramName)
    assert.NotEmpty(t, paramValue)
    assert.Equal(t, "expected-value", paramValue)

    // --- VPC Tests ---
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    
    // ดึง subnets ใน VPC
    subnets := aws.GetSubnetsForVpc(t, vpcID, region)
    assert.Greater(t, len(subnets), 0)
    
    // ตรวจสอบว่ามี public subnet
    publicSubnets := aws.GetPublicSubnetsForVpc(t, vpcID, region)
    assert.Greater(t, len(publicSubnets), 0)
}
```

---

## ขั้นตอนที่ 727: HTTP Testing

```go
// test/terraform_http_test.go
package test

import (
    "fmt"
    "testing"
    "time"

    "github.com/gruntwork-io/terratest/modules/http-helper"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestHTTPEndpoint(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/web-server",
        Vars: map[string]interface{}{
            "environment": "test",
        },
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // ดึง URL จาก output
    serverURL := terraform.Output(t, terraformOptions, "server_url")
    httpsURL  := fmt.Sprintf("https://%s", serverURL)

    // ตรวจสอบ HTTP endpoint พร้อม retry
    http_helper.HttpGetWithRetry(
        t,
        fmt.Sprintf("http://%s", serverURL),
        nil,   // TLS config
        200,   // expected status code
        "Hello, World!",  // expected body (substring)
        30,    // max retries
        5*time.Second,  // time between retries
    )

    // ตรวจสอบ HTTPS
    tlsConfig := &tls.Config{
        InsecureSkipVerify: false, // ใช้ proper TLS verification
    }
    
    http_helper.HttpGetWithRetryWithCustomValidation(
        t,
        httpsURL,
        tlsConfig,
        30,
        10*time.Second,
        func(statusCode int, body string) bool {
            return statusCode == 200 && strings.Contains(body, "healthy")
        },
    )
}

// ทดสอบ Load Balancer
func TestLoadBalancer(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/load-balancer",
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    albDNS := terraform.Output(t, terraformOptions, "alb_dns_name")
    url := fmt.Sprintf("http://%s", albDNS)

    // รอจนกว่า ALB จะพร้อม (อาจใช้เวลา 2-3 นาที)
    http_helper.HttpGetWithRetry(
        t,
        url,
        nil,
        200,
        "",   // ไม่ตรวจสอบ body content
        60,
        10*time.Second,
    )

    // ทดสอบ health check endpoint
    healthURL := fmt.Sprintf("%s/health", url)
    statusCode, body := http_helper.HttpGet(t, healthURL, nil)
    
    assert.Equal(t, 200, statusCode)
    assert.Contains(t, body, `"status":"ok"`)
}
```

---

## ขั้นตอนที่ 728: retry.DoWithRetry

```go
// test/terraform_retry_test.go
package test

import (
    "fmt"
    "testing"
    "time"

    "github.com/gruntwork-io/terratest/modules/retry"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestWithRetry(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/database",
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    dbEndpoint := terraform.Output(t, terraformOptions, "db_endpoint")
    dbPort := terraform.Output(t, terraformOptions, "db_port")

    // Retry จนกว่า database จะพร้อม
    description := fmt.Sprintf("Waiting for database at %s:%s to be ready", dbEndpoint, dbPort)
    
    result, err := retry.DoWithRetryE(
        t,
        description,
        30,              // max retries
        10*time.Second,  // time between retries
        func() (string, error) {
            // ลอง connect ไปยัง database
            conn, err := connectToDatabase(dbEndpoint, dbPort)
            if err != nil {
                return "", fmt.Errorf("database not ready: %w", err)
            }
            defer conn.Close()
            return "connected", nil
        },
    )

    assert.NoError(t, err)
    assert.Equal(t, "connected", result)
}

// Custom retry สำหรับ ECS Service
func WaitForECSServiceStable(t *testing.T, region, clusterName, serviceName string) {
    t.Helper()
    
    retry.DoWithRetry(
        t,
        fmt.Sprintf("Waiting for ECS service %s to be stable", serviceName),
        20,
        30*time.Second,
        func() string {
            // ตรวจสอบ ECS service status ผ่าน AWS SDK
            isStable := checkECSServiceStable(region, clusterName, serviceName)
            if !isStable {
                t.Logf("ECS service not yet stable, retrying...")
                return "not stable"
            }
            return "stable"
        },
    )
}
```

---

## ขั้นตอนที่ 729: SSH Testing และ Docker Testing

### SSH Testing สำหรับ EC2

```go
// test/terraform_ssh_test.go
package test

import (
    "testing"
    "time"
    "fmt"

    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/gruntwork-io/terratest/modules/ssh"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/retry"
    "github.com/stretchr/testify/assert"
)

func TestSSHToEC2(t *testing.T) {
    t.Parallel()

    region := "us-east-1"
    keyPairName := fmt.Sprintf("test-key-%s", random.UniqueId())

    // สร้าง key pair สำหรับ test
    keyPair := aws.CreateAndImportEC2KeyPair(t, region, keyPairName)
    defer aws.DeleteEC2KeyPair(t, keyPair)

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/ec2",
        Vars: map[string]interface{}{
            "key_pair_name": keyPairName,
            "region":        region,
        },
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // ดึง public IP
    publicIP := terraform.Output(t, terraformOptions, "public_ip")

    // สร้าง SSH Host
    host := ssh.Host{
        Hostname:    publicIP,
        SshUserName: "ec2-user",
        SshKeyPair:  keyPair,
    }

    // รอจนกว่า SSH จะพร้อม
    retry.DoWithRetry(
        t,
        "Waiting for SSH",
        10,
        30*time.Second,
        func() string {
            err := ssh.CheckSshConnectionE(t, host)
            if err != nil {
                return fmt.Sprintf("SSH not ready: %v", err)
            }
            return ""
        },
    )

    // รัน command ผ่าน SSH
    output := ssh.CheckSshCommand(t, host, "whoami")
    assert.Equal(t, "ec2-user", strings.TrimSpace(output))

    // ตรวจสอบ services
    output = ssh.CheckSshCommand(t, host, "systemctl is-active nginx")
    assert.Equal(t, "active", strings.TrimSpace(output))

    // ตรวจสอบ files
    output = ssh.CheckSshCommand(t, host, "ls /var/www/html/")
    assert.Contains(t, output, "index.html")

    // ตรวจสอบ disk space
    output = ssh.CheckSshCommand(t, host, "df -h / | tail -1 | awk '{print $5}' | tr -d '%'")
    diskUsage, _ := strconv.Atoi(strings.TrimSpace(output))
    assert.Less(t, diskUsage, 80, "Disk usage should be less than 80%")
}
```

### Docker Testing

```go
// test/docker_test.go
package test

import (
    "testing"
    "fmt"

    "github.com/gruntwork-io/terratest/modules/docker"
    "github.com/stretchr/testify/assert"
)

func TestDockerBuild(t *testing.T) {
    t.Parallel()

    tag := "my-app:test"
    buildOptions := &docker.BuildOptions{
        Tags: []string{tag},
    }

    // Build Docker image
    docker.Build(t, "../docker", buildOptions)

    // Run container
    runOptions := &docker.RunOptions{
        Command: []string{"echo", "hello"},
    }
    
    output := docker.Run(t, tag, runOptions)
    assert.Equal(t, "hello", strings.TrimSpace(output))
}

func TestDockerWithTerraform(t *testing.T) {
    t.Parallel()

    // Build image
    imageTag := fmt.Sprintf("myapp:%s", random.UniqueId())
    docker.Build(t, "../", &docker.BuildOptions{
        Tags: []string{imageTag},
    })

    // Push to ECR (ถ้าต้องการ)
    // aws.CreateECRRepo(t, region, repoName)
    // docker.Tag(t, imageTag, ecrURL)
    // docker.Push(t, ecrURL)
    
    // Deploy ด้วย Terraform
    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/ecs",
        Vars: map[string]interface{}{
            "docker_image": imageTag,
        },
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    serviceURL := terraform.Output(t, terraformOptions, "service_url")
    
    http_helper.HttpGetWithRetry(
        t,
        serviceURL,
        nil,
        200,
        "OK",
        30,
        10*time.Second,
    )
}
```

---

## ขั้นตอนที่ 730: Test Stages และ Parallelism

### Test Stages ด้วย skip flags

```go
// test/terraform_staged_test.go
package test

import (
    "os"
    "testing"

    "github.com/gruntwork-io/terratest/modules/terraform"
)

// ใช้ environment variables เพื่อ control ว่า stage ไหนจะรัน
// SKIP_setup=true = ข้าม setup stage
// SKIP_validate=true = ข้าม validation stage
// SKIP_teardown=true = ข้าม teardown (ไว้ debug)

func TestWithStages(t *testing.T) {
    t.Parallel()

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/complete",
    }

    // Stage 1: Setup (init & apply)
    if os.Getenv("SKIP_setup") != "true" {
        defer terraform.Destroy(t, terraformOptions)
        terraform.InitAndApply(t, terraformOptions)
    }

    // Stage 2: Validate
    if os.Getenv("SKIP_validate") != "true" {
        vpcID := terraform.Output(t, terraformOptions, "vpc_id")
        assert.NotEmpty(t, vpcID)
    }
    
    // Stage 3: Advanced tests
    if os.Getenv("SKIP_advanced") != "true" {
        // ทำ advanced tests
    }
}

// รัน test แบบ parallel หลายๆ test พร้อมกัน
func TestParallelDeployments(t *testing.T) {
    t.Parallel()
    
    tests := []struct {
        name        string
        environment string
        region      string
    }{
        {"dev-us-east-1", "dev", "us-east-1"},
        {"dev-us-west-2", "dev", "us-west-2"},
        {"staging-us-east-1", "staging", "us-east-1"},
    }

    for _, tc := range tests {
        tc := tc // capture range variable

        t.Run(tc.name, func(t *testing.T) {
            t.Parallel() // รัน sub-tests แบบ parallel

            terraformOptions := &terraform.Options{
                TerraformDir: "../examples/basic",
                Vars: map[string]interface{}{
                    "environment": tc.environment,
                    "region":      tc.region,
                },
            }

            defer terraform.Destroy(t, terraformOptions)
            terraform.InitAndApply(t, terraformOptions)

            vpcID := terraform.Output(t, terraformOptions, "vpc_id")
            assert.NotEmpty(t, vpcID)
        })
    }
}
```

### Complete Terratest Example: VPC Module

```go
// test/vpc_module_test.go
package test

import (
    "fmt"
    "testing"
    "time"

    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/gruntwork-io/terratest/modules/random"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestVPCModule(t *testing.T) {
    t.Parallel()

    // ใช้ random ID เพื่อ avoid naming conflicts
    uniqueID := random.UniqueId()
    region   := "us-east-1"
    
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/vpc",
        Vars: map[string]interface{}{
            "environment":  "test",
            "project_name": fmt.Sprintf("test-%s", uniqueID),
            "region":       region,
            "vpc_cidr":     "10.0.0.0/16",
            "public_subnets": []string{
                "10.0.1.0/24",
                "10.0.2.0/24",
            },
            "private_subnets": []string{
                "10.0.10.0/24",
                "10.0.11.0/24",
            },
        },
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": region,
        },
    })

    defer terraform.Destroy(t, terraformOptions)

    terraform.InitAndApply(t, terraformOptions)

    // --- Test Outputs ---
    vpcID           := terraform.Output(t, terraformOptions, "vpc_id")
    publicSubnets   := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
    privateSubnets  := terraform.OutputList(t, terraformOptions, "private_subnet_ids")
    natGatewayIPs   := terraform.OutputList(t, terraformOptions, "nat_gateway_ips")

    // --- VPC Assertions ---
    require.NotEmpty(t, vpcID, "VPC ID should not be empty")
    assert.Regexp(t, `^vpc-[a-f0-9]+$`, vpcID, "VPC ID format incorrect")

    // --- Subnet Assertions ---
    assert.Equal(t, 2, len(publicSubnets), "Should have 2 public subnets")
    assert.Equal(t, 2, len(privateSubnets), "Should have 2 private subnets")

    // --- NAT Gateway Assertions ---
    assert.Equal(t, 2, len(natGatewayIPs), "Should have 2 NAT gateways")
    
    for _, ip := range natGatewayIPs {
        assert.Regexp(t, `^\d+\.\d+\.\d+\.\d+$`, ip, "NAT Gateway IP format incorrect")
    }

    // --- AWS API Verification ---
    
    // ตรวจสอบ VPC ด้วย AWS API โดยตรง
    vpc := aws.GetVpcById(t, vpcID, region)
    assert.Equal(t, "10.0.0.0/16", aws.GetCidrBlockAssociationStates(t, *vpc.CidrBlock))
    assert.True(t, *vpc.EnableDnsHostnames, "DNS hostnames should be enabled")
    assert.True(t, *vpc.EnableDnsSupport, "DNS support should be enabled")

    // ตรวจสอบ subnets
    subnets := aws.GetSubnetsForVpc(t, vpcID, region)
    assert.Equal(t, 4, len(subnets), "Should have 4 subnets total")

    publicSubnetsInfo := aws.GetPublicSubnetsForVpc(t, vpcID, region)
    assert.Equal(t, 2, len(publicSubnetsInfo), "Should have 2 public subnets")

    // --- Tag Verification ---
    vpcTags := aws.GetTagsForVpc(t, vpcID, region)
    assert.Equal(t, "test", vpcTags["Environment"], "Environment tag should be 'test'")
    assert.Equal(t, fmt.Sprintf("test-%s", uniqueID), vpcTags["Project"])
    assert.Equal(t, "terraform", vpcTags["ManagedBy"])

    // --- Idempotency Check ---
    terraform.Apply(t, terraformOptions)  // second apply
    // ถ้าไม่มี changes จะไม่ error
}
```

### Cost Considerations

```go
// test/cost_aware_test.go
package test

import (
    "os"
    "testing"
)

// Skip expensive tests ใน certain environments
func skipIfNotIntegration(t *testing.T) {
    t.Helper()
    if os.Getenv("RUN_INTEGRATION_TESTS") != "true" {
        t.Skip("Skipping integration test - set RUN_INTEGRATION_TESTS=true to run")
    }
}

func TestExpensiveResources(t *testing.T) {
    skipIfNotIntegration(t)
    // ... test NAT Gateway, RDS, ElastiCache ที่มีค่าใช้จ่ายสูง
}

// Cost-friendly tests (ใช้ small/free-tier resources เท่านั้น)
func TestFreeTierResources(t *testing.T) {
    t.Parallel()
    // S3, VPC, Security Groups = ฟรี
    // t3.nano instance = ถูก
}
```

---

## สรุป (Summary)

| Feature | Code | Notes |
|---------|------|-------|
| Init & Apply | `terraform.InitAndApply(t, opts)` | รัน init แล้ว apply |
| Destroy | `defer terraform.Destroy(t, opts)` | ต้องมี defer เสมอ |
| Get Output | `terraform.Output(t, opts, "key")` | String output |
| Get List | `terraform.OutputList(t, opts, "key")` | List output |
| HTTP Test | `http_helper.HttpGetWithRetry(...)` | ทดสอบ HTTP endpoint |
| Retry | `retry.DoWithRetry(t, desc, n, wait, fn)` | Retry logic |
| SSH Test | `ssh.CheckSshCommand(t, host, cmd)` | ทดสอบผ่าน SSH |

---

*จบ Part 073 - ในส่วนถัดไปจะเรียนรู้เรื่อง Terraform Cloud & Remote Operations*
