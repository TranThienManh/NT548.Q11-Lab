# Lab 01 - Dùng Terraform và CloudFormation để quản lý và triển khai hạ tầng AWS

> **Môn học:** NT548.Q11 - DevOps  
> **Mục tiêu:** Sử dụng Terraform và CloudFormation để quản lý và triển khai tự động hạ tầng AWS

---

## Kiến Trúc Hạ Tầng

```
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                           AWS Cloud                                                                   │
│                                                                                                                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐       │
│  │                                                      VPC (10.0.0.0/16)                                                    │       │
│  │                                                                                                                           │       │
│  │  ┌────────────────────────────────────────────────┐              ┌────────────────────────────────────────────────┐      │       │
│  │  │            Public Subnet (10.0.1.0/24)         │              │            Private Subnet (10.0.2.0/24)        │      │       │
│  │  │                                                │              │                                                │      │       │
│  │  │    ┌────────────────┐       ┌───────────────┐  │              │    ┌────────────────┐                          │      │       │
│  │  │    │ Public EC2     │       │ NAT Gateway   │  │              │    │ Private EC2    │                          │      │       │
│  │  │    │ Instance       │       │               │  │              │    │ Instance       │                          │      │       │
│  │  │    │ (SSH: port 22  │       │ (Elastic IP)  │◄─┼──────────────┼────│ (SSH từ       │                          │      │       │
│  │  │    │  từ IP cụ thể) │       │               │  │              │    │  Public EC2)   │                          │      │       │
│  │  │    └────────────────┘       └───────────────┘  │              │    └────────────────┘                          │      │       │
│  │  │            │                        │          │              │            │                                   │      │       │
│  │  │            │                        │          │              │            ▼                                   │      │       │
│  │  │            │                        │          │              │    ┌────────────────┐                          │      │       │
│  │  │            │                        │          │              │    │ Private Route  │                          │      │       │
│  │  │            │                        │          │              │    │ Table → NAT GW │                          │      │       │
│  │  │            │                        │          │              │    └────────────────┘                          │      │       │
│  │  └────────────┼────────────────────────┼──────────┘              └────────────────────────────────────────────────┘      │       │
│  │               │                        │                                                                                  │       │
│  │               ▼                        │                                                                                  │       │
│  │       ┌────────────────┐               │                                                                                  │       │
│  │       │ Public Route   │               │                                                                                  │       │
│  │       │ Table → IGW    │               │                                                                                  │       │
│  │       └───────┬────────┘               │                                                                                  │       │
│  │               │                        │                                                                                  │       │
│  │               ▼                        │                                                                                  │       │
│  │       ┌────────────────┐               │                                                                                  │       │
│  │       │ Internet       │◄──────────────┘                                                                                  │       │
│  │       │ Gateway        │                                                                                                  │       │
│  │       └───────┬────────┘                                                                                                  │       │
│  └───────────────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────┘       │
│                  │                                                                                                                    │
│                  ▼                                                                                                                    │
│              Internet                                                                                                                 │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Cấu Trúc Dự Án

```
Lab01/
├── README.md                     # Tài liệu hướng dẫn
├── terraform/                    # Terraform Infrastructure as Code
│   ├── main.tf                   # Root module - khai báo các modules
│   ├── variables.tf              # Định nghĩa biến đầu vào
│   ├── outputs.tf                # Định nghĩa giá trị đầu ra
│   ├── providers.tf              # Cấu hình AWS Provider
│   ├── terraform.tfvars          # Giá trị biến
│   └── modules/                  # Terraform Modules
│       ├── vpc/                  # Module VPC, Subnets, IGW, NAT, Route Tables
│       │   ├── main.tf
│       │   ├── variables.tf
│       │   └── outputs.tf
│       ├── security/             # Module Security Groups
│       │   ├── main.tf
│       │   ├── variables.tf
│       │   └── outputs.tf
│       └── ec2/                  # Module EC2 Instances
│           ├── main.tf
│           ├── variables.tf
│           └── outputs.tf
└── cloudformation/               # CloudFormation Infrastructure as Code
    └── cloudformation.yml        # CloudFormation Template
```
---

## Yêu Cầu Môi Trường

- Tài khoản AWS với quyền triển khai infrastructure
- AWS CLI đã cài đặt và cấu hình
- Terraform CLI (version 1.0+)
- Git

---

## Hướng Dẫn Cài Đặt

### 1. Cài đặt AWS CLI

**Windows:**
```powershell
winget install Amazon.AWSCLI
```

**macOS:**
```bash
brew install awscli
```

**Linux:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Cấu hình AWS CLI:**
```bash
aws configure
# Nhập Access Key ID, Secret Access Key, Region, Output format
```

### 2. Cài đặt Terraform

**Windows:**
```powershell
winget install Hashicorp.Terraform
```

**macOS:**
```bash
brew install terraform
```

**Linux:**
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

---

## Triển Khai với Terraform

### Bước 1: Cấu hình biến

Chỉnh sửa file `terraform/terraform.tfvars`:

```hcl
prefix              = "terraform-lab1"
region              = "us-east-1"
vpc_cidr            = "10.0.0.0/16"
public_subnet_cidr  = "10.0.1.0/24"
private_subnet_cidr = "10.0.2.0/24"
allowed_ip          = "YOUR_PUBLIC_IP/32"    # Thay bằng IP của bạn
ami_id              = "ami-xxxxx"            # AMI phù hợp với region
instance_type       = "t2.micro"
key_name            = "your-key-name"
```

### Bước 2: Triển khai

```bash
cd terraform

# Khởi tạo Terraform
terraform init

# Xem trước thay đổi
terraform plan

# Triển khai hạ tầng
terraform apply

# Xem outputs
terraform output
```

### Bước 3: Xóa hạ tầng (khi hoàn thành)

```bash
terraform destroy
```

---

## Triển Khai với CloudFormation

### Bước 1: Triển khai Stack trên UI AWS

### Bước 2: Kiểm tra trạng thái

```bash
# Kiểm tra trạng thái stack
aws cloudformation describe-stacks --stack-name cloudformation-lab1

# Xem outputs
aws cloudformation describe-stacks --stack-name cloudformation-lab1 --query "Stacks[0].Outputs"

```

### Bước 3: Xóa Stack (khi hoàn thành)

```bash
aws cloudformation delete-stack --stack-name cloudformation-lab1
```

---

## Kiểm Tra Kết Quả Triển Khai

### 1. SSH vào Public EC2

```bash
ssh -i your-key.pem ec2-user@<PUBLIC_EC2_IP>
```

### 2. SSH từ Public EC2 vào Private EC2

```bash
# Copy key vào Public EC2
scp -i your-key.pem your-key.pem ec2-user@<PUBLIC_EC2_IP>:~/

# SSH vào Public EC2
ssh -i your-key.pem ec2-user@<PUBLIC_EC2_IP>

# Từ Public EC2, SSH vào Private EC2
ssh -i your-key.pem ec2-user@<PRIVATE_EC2_IP>
```

### 3. Kiểm tra kết nối Internet từ Private EC2

```bash
# Từ Private EC2, ping Google DNS (thông qua NAT Gateway)
ping -c3 8.8.8.8
---
