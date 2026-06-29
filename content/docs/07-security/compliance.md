---
title: "컴플라이언스 대응: 표준 태깅, 감사 가능성, 변경 추적"
weight: 3
---

## 표준 태깅 전략

태그는 인프라 거버넌스의 기초입니다. 태그가 없으면 비용 추적, 리소스 소유권 파악, 컴플라이언스 감사가 불가능해집니다.

### 필수 태그 목록

| 태그 키 | 설명 | 예시 값 |
|---------|------|---------|
| `Environment` | 환경 구분 | `dev`, `stage`, `prod` |
| `Owner` | 담당 팀 또는 담당자 | `platform-team`, `user@company.com` |
| `CostCenter` | 비용 센터 코드 | `CC-1234` |
| `Project` | 프로젝트명 | `payment-service` |
| `ManagedBy` | 관리 방식 | `terraform` |
| `Repo` | 소스 코드 저장소 | `github.com/org/repo` |
| `CreatedDate` | 생성일 | `2024-01-15` |

## Terraform locals로 공통 태그 중앙화

```hcl
# locals.tf
locals {
  common_tags = {
    Environment = var.environment
    Owner       = var.team_name
    CostCenter  = var.cost_center
    Project     = var.project_name
    ManagedBy   = "terraform"
    Repo        = "github.com/org/${var.project_name}"
    CreatedDate = formatdate("YYYY-MM-DD", timestamp())
  }
}

# variables.tf
variable "environment" {
  type        = string
  description = "배포 환경"
  validation {
    condition     = contains(["dev", "stage", "prod"], var.environment)
    error_message = "environment는 dev, stage, prod 중 하나여야 합니다."
  }
}

variable "team_name" {
  type        = string
  description = "담당 팀명"
}

variable "cost_center" {
  type        = string
  description = "비용 센터 코드 (예: CC-1234)"
  validation {
    condition     = can(regex("^CC-[0-9]{4}$", var.cost_center))
    error_message = "cost_center는 CC-XXXX 형식이어야 합니다."
  }
}
```

```hcl
# 리소스에서 common_tags 활용
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-vpc-${var.environment}"
  })
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  tags = merge(local.common_tags, {
    Name = "${var.project_name}-web-${var.environment}"
    Role = "web-server"
  })
}
```

{{< callout type="info" >}}
`merge(local.common_tags, {...})` 패턴을 사용하면 공통 태그 + 리소스별 태그를 깔끔하게 합칠 수 있습니다. 리소스별 태그가 공통 태그를 덮어씁니다.
{{< /callout >}}

## 감사 가능성 확보 방법

감사 가능성(Auditability)이란 **누가, 언제, 무엇을, 왜 변경했는지** 추적할 수 있는 능력입니다.

### Git 커밋으로 변경 이력 확보

```bash
# 좋은 커밋 메시지 예시
git commit -m "feat(ec2): prod 웹서버 인스턴스 타입 t3.medium → m5.large 변경

변경 이유: 트래픽 증가로 인한 성능 저하 대응
승인자: 팀장 홍길동 (Slack 승인 #infra-changes 2024-01-15 14:30)
Jira: INFRA-1234"
```

### Terraform 변경 계획 저장

```bash
# Plan 결과를 파일로 저장 (감사 목적)
terraform plan -out=tfplan-$(date +%Y%m%d-%H%M%S).binary

# Plan 결과를 텍스트로 저장
terraform plan 2>&1 | tee plan-$(date +%Y%m%d-%H%M%S).log
```

## CloudTrail + Terraform 변경 추적

AWS CloudTrail은 API 호출을 모두 기록합니다. Terraform이 AWS 리소스를 생성/수정/삭제하면 CloudTrail에 기록됩니다.

```hcl
# CloudTrail 설정 (Terraform으로 관리)
resource "aws_cloudtrail" "main" {
  name                          = "${var.project_name}-cloudtrail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail_logs.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::*"]
    }
  }

  tags = local.common_tags
}

resource "aws_s3_bucket" "cloudtrail_logs" {
  bucket = "${var.project_name}-cloudtrail-logs"

  # CloudTrail 로그 버킷은 삭제 방지
  lifecycle {
    prevent_destroy = true
  }

  tags = local.common_tags
}
```

### CloudTrail 로그에서 Terraform 변경 찾기

```bash
# AWS CLI로 Terraform 관련 API 호출 조회
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=terraform-ci-user \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z \
  --query 'Events[*].{Time:EventTime,Event:EventName,Resource:Resources[0].ResourceName}'
```

## 승인 기록 보존 방법

### GitHub PR을 통한 승인 추적

```yaml
# .github/branch-protection.yml (GitHub 브랜치 보호 설정)
# main 브랜치에 직접 push 금지
# PR 필수: 최소 2명 승인
# Status check 필수: terraform-plan, security-scan
```

### Terraform Cloud 감사 로그

Terraform Cloud를 사용하면 모든 실행 이력이 자동으로 기록됩니다.

```hcl
# terraform.tf
terraform {
  cloud {
    organization = "your-org"
    workspaces {
      name = "production"
    }
  }
}
```

## 컴플라이언스 체크리스트

### 태깅 컴플라이언스

- [ ] 모든 리소스에 `Environment` 태그 있음
- [ ] 모든 리소스에 `Owner` 태그 있음
- [ ] 모든 리소스에 `CostCenter` 태그 있음
- [ ] 모든 리소스에 `ManagedBy = "terraform"` 태그 있음
- [ ] 태그 값이 허용된 값 목록에 있음 (validation 적용)

### 감사 가능성 체크리스트

- [ ] CloudTrail이 모든 리전에서 활성화됨
- [ ] CloudTrail 로그가 S3에 최소 1년 보존됨
- [ ] S3 로그 버킷에 MFA Delete 활성화됨
- [ ] Git 커밋 메시지에 변경 이유 포함됨
- [ ] PR 승인 기록이 Git 히스토리에 남아 있음
- [ ] Terraform plan 결과가 CI/CD 로그에 보존됨

### 보안 컴플라이언스 체크리스트

- [ ] 모든 S3 버킷에 퍼블릭 액세스 차단 적용됨
- [ ] 모든 EBS 볼륨에 암호화 적용됨
- [ ] 모든 RDS 인스턴스에 암호화 적용됨
- [ ] SSH 포트(22)가 전체 공개(0.0.0.0/0)되어 있지 않음
- [ ] IAM 역할에 최소 권한 원칙 적용됨
- [ ] Secrets Manager 또는 SSM Parameter Store로 비밀정보 관리됨
