---
title: "변수·출력·로컬값: 실무에서 가장 많이 쓰는 기본기"
weight: 1
---

## 변수가 없는 코드의 문제

처음 Terraform을 작성하다 보면 이런 코드가 나옵니다.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  tags = {
    Name        = "my-web-server"
    Environment = "production"
  }
}
```

dev와 prod에 같은 코드를 복사해서 쓰게 되고, 한 곳에서 값을 바꾸면 다른 곳은 잊어버립니다. **변수, 로컬값, 출력값은 이 문제를 해결하는 핵심 도구입니다.**

---

## 변수 설계

### 변수명 규칙

실무에서 쓰는 변수명은 **snake_case**, 소문자, 명사 중심으로 짓습니다.

```hcl
# 좋은 예
variable "instance_type" {}
variable "vpc_cidr_block" {}
variable "enable_deletion_protection" {}

# 나쁜 예
variable "InstanceType" {}    # PascalCase - 비권장
variable "t" {}               # 너무 짧음
variable "ec2_instance_type_for_web_server_in_production" {}  # 너무 김
```

### 기본값 vs 필수값

```hcl
# 필수값: 기본값 없음 → 항상 외부에서 전달
variable "environment" {
  description = "배포 환경 (dev, staging, prod)"
  type        = string
  # default 없음 = 반드시 값을 넘겨야 함
}

# 기본값 있음 → 선택적 입력
variable "instance_type" {
  description = "EC2 인스턴스 타입"
  type        = string
  default     = "t3.micro"

  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "허용된 인스턴스 타입: t3.micro, t3.small, t3.medium"
  }
}

# 민감 변수: plan/apply 출력에서 값 숨김
variable "db_password" {
  description = "데이터베이스 비밀번호"
  type        = string
  sensitive   = true
}
```

### tfvars 사용법

`terraform.tfvars` 파일에 값을 분리하면 환경별 관리가 쉬워집니다.

```hcl
# environments/dev/terraform.tfvars
environment    = "dev"
instance_type  = "t3.micro"
min_capacity   = 1
max_capacity   = 3
enable_logging = false
```

```hcl
# environments/prod/terraform.tfvars
environment    = "prod"
instance_type  = "t3.medium"
min_capacity   = 3
max_capacity   = 10
enable_logging = true
```

apply 시 명시적으로 파일을 지정합니다.

```bash
terraform apply -var-file="environments/dev/terraform.tfvars"
terraform apply -var-file="environments/prod/terraform.tfvars"
```

---

## 로컬값으로 반복 제거

같은 문자열이 여러 곳에 반복된다면 `locals` 블록으로 중앙화합니다.

```hcl
# 나쁜 예: 하드코딩 반복
resource "aws_s3_bucket" "logs" {
  bucket = "mycompany-prod-logs"
  tags = {
    Project     = "mycompany"
    Environment = "prod"
    ManagedBy   = "terraform"
  }
}

resource "aws_s3_bucket" "backup" {
  bucket = "mycompany-prod-backup"
  tags = {
    Project     = "mycompany"
    Environment = "prod"
    ManagedBy   = "terraform"
  }
}
```

```hcl
# 좋은 예: locals 사용
locals {
  project     = "mycompany"
  environment = var.environment
  name_prefix = "${local.project}-${local.environment}"

  common_tags = {
    Project     = local.project
    Environment = local.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_s3_bucket" "logs" {
  bucket = "${local.name_prefix}-logs"
  tags   = local.common_tags
}

resource "aws_s3_bucket" "backup" {
  bucket = "${local.name_prefix}-backup"
  tags   = local.common_tags
}
```

---

## 출력값 설계와 모듈 간 연결

`output` 블록은 두 가지 용도로 씁니다.

1. 배포 후 중요한 값을 화면에 출력
2. 모듈 간 데이터를 전달 (상위 모듈이 하위 모듈 output을 참조)

```hcl
# outputs.tf
output "vpc_id" {
  description = "생성된 VPC의 ID"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "퍼블릭 서브넷 ID 목록"
  value       = aws_subnet.public[*].id
}

output "db_endpoint" {
  description = "데이터베이스 엔드포인트"
  value       = aws_db_instance.main.endpoint
  sensitive   = true  # 민감 정보는 출력 숨김
}
```

모듈 간 연결:

```hcl
# 네트워크 모듈의 output을 서버 모듈에 전달
module "network" {
  source = "./modules/network"
}

module "server" {
  source     = "./modules/server"
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.public_subnet_ids
}
```

---

## 안티패턴 vs 좋은 패턴

{{< callout type="error" >}}
**안티패턴: 하드코딩과 중복**

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  tags = { Environment = "prod", Team = "platform" }
}

resource "aws_instance" "api" {
  ami           = "ami-0abcdef1234567890"  # 중복
  instance_type = "t3.micro"              # 중복
  tags = { Environment = "prod", Team = "platform" }  # 중복
}
```
{{< /callout >}}

{{< callout type="info" >}}
**좋은 패턴: 변수 + locals로 중앙화**

```hcl
locals {
  ami          = "ami-0abcdef1234567890"
  common_tags  = { Environment = var.environment, Team = "platform" }
}

resource "aws_instance" "web" {
  ami           = local.ami
  instance_type = var.instance_type
  tags          = local.common_tags
}

resource "aws_instance" "api" {
  ami           = local.ami
  instance_type = var.instance_type
  tags          = local.common_tags
}
```
{{< /callout >}}
