---
title: "모듈 설계 고도화: 범용 vs 서비스 전용, 버전 관리"
weight: 1
---

## 범용 모듈 vs 서비스 전용 모듈

모듈을 설계할 때 가장 먼저 결정해야 할 것은 **어느 수준의 추상화**를 제공할 것인가입니다.

| 구분 | 범용 모듈 | 서비스 전용 모듈 |
|------|----------|----------------|
| 대상 | 모든 팀이 사용 | 특정 서비스/팀만 사용 |
| 유연성 | 높음 (변수가 많음) | 낮음 (의견이 많이 반영됨) |
| 사용 편의성 | 낮음 (설정할 게 많음) | 높음 (기본값이 많음) |
| 유지보수 | 어려움 (하위호환 유지) | 쉬움 (사용처가 적음) |
| 예시 | `terraform-aws-modules/vpc` | `company/webapp-infra` |

**범용 모듈 예시 (유연하지만 복잡):**

```hcl
# 범용 VPC 모듈 - 모든 옵션을 노출
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name                 = var.name
  cidr                 = var.cidr
  azs                  = var.availability_zones
  private_subnets      = var.private_subnets
  public_subnets       = var.public_subnets
  enable_nat_gateway   = var.enable_nat_gateway
  single_nat_gateway   = var.single_nat_gateway
  enable_dns_hostnames = var.enable_dns_hostnames
  # ... 수십 개의 변수
}
```

**서비스 전용 모듈 예시 (간단하고 의견이 담겨있음):**

```hcl
# 우리 회사 표준 VPC 모듈 - 필수 입력만 요구
module "vpc" {
  source = "git::https://github.com/company/terraform-modules.git//vpc?ref=v2.1.0"

  name        = var.service_name    # 이것만 필요
  environment = var.environment     # dev / stage / prod
  # 나머지는 회사 표준으로 고정됨
}
```

## 인터페이스 단순화 원칙

좋은 모듈은 **사용하기 쉬운 인터페이스**를 제공합니다.

```hcl
# 나쁜 예: 내부 구현 세부사항을 모두 노출
variable "vpc_cidr_block" { ... }
variable "subnet_1_cidr" { ... }
variable "subnet_2_cidr" { ... }
variable "subnet_3_cidr" { ... }
variable "nat_gateway_eip_allocation_id" { ... }

# 좋은 예: 비즈니스 의미를 가진 변수만 노출
variable "environment" {
  description = "배포 환경 (dev/stage/prod)"
  type        = string
}
variable "vpc_size" {
  description = "VPC 크기 (small/medium/large)"
  type        = string
  default     = "medium"
}
```

## 모듈 버전 관리 전략 (Git Tag 기반)

모듈을 Git 태그로 버전 관리하면 **안전한 업그레이드와 롤백**이 가능합니다.

```bash
# 새 버전 태그 생성
git tag -a v1.2.0 -m "Add RDS support"
git push origin v1.2.0
```

**모듈 참조 시 버전 고정:**

```hcl
module "vpc" {
  # 태그로 버전 고정 (필수)
  source = "git::https://github.com/company/terraform-modules.git//modules/vpc?ref=v1.2.0"
}

# Terraform Registry 사용 시
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1"    # 5.1.x 허용, 5.2 이상 금지
}
```

{{< callout type="warning" >}}
`ref=main` 또는 버전 없이 모듈을 참조하면 예고 없이 변경사항이 반영될 수 있습니다. **항상 버전을 고정**하세요.
{{< /callout >}}

## Semantic Versioning 적용

| 버전 유형 | 예시 | 의미 |
|----------|------|------|
| Major | `v2.0.0` | Breaking change 포함 |
| Minor | `v1.1.0` | 하위호환 기능 추가 |
| Patch | `v1.0.1` | 버그 수정 |

**Breaking change란:**

```hcl
# v1.x: 변수명 변경 전
variable "vpc_name" { ... }

# v2.0.0 (Breaking): 변수명이 변경됨 → 기존 사용처 수정 필요
variable "name" { ... }
```

## Breaking Change 관리 방법

Breaking change를 피할 수 없다면 **마이그레이션 가이드와 충분한 공지 기간**을 제공합니다.

```hcl
# 전환 기간 동안 두 변수를 모두 지원
variable "vpc_name" {
  description = "DEPRECATED: name 변수를 사용하세요. v3.0에서 제거됩니다."
  type        = string
  default     = null
}

variable "name" {
  description = "VPC 이름"
  type        = string
  default     = null
}

locals {
  # 하위호환성 처리
  name = coalesce(var.name, var.vpc_name)
}
```

## 내부 모듈 카탈로그 운영 방법

팀이 5명 이상이 되면 모듈 목록과 사용법을 중앙에서 관리해야 합니다.

**모듈 카탈로그 구조 예시:**

```
terraform-modules/ (리포지토리)
├── modules/
│   ├── vpc/           v2.1.0
│   ├── eks/           v1.3.0
│   ├── rds/           v1.5.2
│   ├── s3/            v1.1.0
│   └── iam-role/      v1.0.0
├── examples/
│   ├── complete-vpc/
│   └── three-tier-app/
└── CATALOG.md         ← 모듈 목록과 사용법 문서
```

**CATALOG.md 예시:**

```markdown
## 모듈 카탈로그

| 모듈 | 최신 버전 | 담당자 | 상태 |
|------|---------|--------|------|
| vpc  | v2.1.0  | @infra-team | 안정 |
| eks  | v1.3.0  | @platform-team | 안정 |
| rds  | v1.5.2  | @infra-team | 안정 |
```

{{< callout type="info" >}}
Terraform Cloud나 Terraform Enterprise를 사용하는 조직은 **Private Registry** 기능으로 내부 모듈을 중앙 관리하고, UI에서 버전별 문서를 확인할 수 있습니다.
{{< /callout >}}
