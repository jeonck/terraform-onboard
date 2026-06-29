---
title: "비용 통제: 리소스 표준화와 비용 추정 도구 연계"
weight: 4
---

## Terraform으로 비용 통제하는 방법

인프라 비용은 Terraform이 만드는 리소스의 직접적인 결과입니다. 코드 리뷰 단계에서 비용 영향을 파악하면 예상치 못한 청구서를 막을 수 있습니다.

{{< callout type="info" >}}
비용 통제의 핵심은 배포 후 확인이 아니라 **PR 단계에서 예측**하는 것입니다. infracost는 코드 변경이 월 비용에 미치는 영향을 자동으로 계산합니다.
{{< /callout >}}

## infracost 도구 설치 및 사용법

```bash
# macOS 설치
brew install infracost

# 또는 직접 설치
curl -fsSL https://raw.githubusercontent.com/infracost/infracost/master/scripts/install.sh | sh

# API 키 등록 (무료)
infracost auth login

# 버전 확인
infracost --version
```

### 기본 사용법

```bash
# 현재 디렉토리 비용 추정
infracost breakdown --path .

# tfvars 파일 지정
infracost breakdown --path . --terraform-var-file prod.tfvars

# JSON 출력 (CI/CD 연동)
infracost breakdown --path . --format json > infracost-base.json

# 변경 전후 비용 비교
infracost diff --path . --compare-to infracost-base.json
```

### infracost 결과 예시

```
Name                                     Quantity  Unit      Monthly Cost

aws_instance.web
 ├─ Instance usage (Linux, on-demand, m5.large)  730  hours        $70.08
 └─ root_block_device
    └─ Storage (general purpose SSD, gp3)         50  GB            $4.00

aws_rds_instance.db
 ├─ Database instance (db.t3.medium, MySQL)      730  hours        $49.64
 └─ Storage (general purpose SSD, gp2)           100  GB            $11.50

OVERALL TOTAL                                                      $135.22

──────────────────────────────────
1 resource type wasn't estimated: aws_s3_bucket
```

## PR마다 비용 추정 결과 게시

```yaml
# .github/workflows/infracost.yml
name: Infracost 비용 추정

on:
  pull_request:
    paths:
      - "**.tf"
      - "**.tfvars"

jobs:
  infracost:
    name: 비용 영향 분석
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - name: Checkout base branch
        uses: actions/checkout@v4
        with:
          ref: "${{ github.event.pull_request.base.ref }}"

      - name: Infracost 설치
        uses: infracost/actions/setup@v3
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}

      - name: Base branch 비용 계산
        run: |
          infracost breakdown --path . \
            --format json \
            --out-file /tmp/infracost-base.json

      - name: PR branch checkout
        uses: actions/checkout@v4

      - name: PR branch 비용 계산 및 비교
        run: |
          infracost diff --path . \
            --format json \
            --compare-to /tmp/infracost-base.json \
            --out-file /tmp/infracost-diff.json

      - name: PR에 비용 비교 코멘트 게시
        run: |
          infracost comment github \
            --path /tmp/infracost-diff.json \
            --repo $GITHUB_REPOSITORY \
            --github-token ${{ github.token }} \
            --pull-request ${{ github.event.pull_request.number }} \
            --behavior update
```

PR에 자동으로 다음과 같은 코멘트가 게시됩니다:

```
💰 Infracost 비용 추정 결과

월 비용 변화: $70.08 → $140.16 (+$70.08, +100%)

변경된 리소스:
+ aws_instance.web_2  +$70.08/월 (신규 추가)

기존 비용: $70.08/월
예상 비용: $140.16/월
```

## 고비용 리소스 생성 통제 정책

### 인스턴스 타입 화이트리스트

```hcl
# variables.tf
variable "instance_type" {
  type        = string
  description = "EC2 인스턴스 타입"

  validation {
    condition = contains([
      # 개발/테스트용
      "t3.micro", "t3.small", "t3.medium",
      # 운영 소규모
      "t3.large", "t3.xlarge",
      # 운영 중규모
      "m5.large", "m5.xlarge", "m5.2xlarge",
      # 운영 대규모 (별도 승인 필요)
      "m5.4xlarge"
    ], var.instance_type)
    error_message = "허용되지 않은 인스턴스 타입입니다. 승인된 타입 목록을 확인하세요."
  }
}
```

### 환경별 인스턴스 크기 제한

```hcl
# locals.tf
locals {
  # 환경별 허용 인스턴스 타입
  allowed_instance_types = {
    dev   = ["t3.micro", "t3.small", "t3.medium"]
    stage = ["t3.medium", "t3.large", "m5.large"]
    prod  = ["t3.large", "m5.large", "m5.xlarge", "m5.2xlarge"]
  }

  # 현재 환경에서 허용된 타입인지 검증
  instance_type_allowed = contains(
    local.allowed_instance_types[var.environment],
    var.instance_type
  )
}

resource "null_resource" "instance_type_check" {
  lifecycle {
    precondition {
      condition     = local.instance_type_allowed
      error_message = "${var.environment} 환경에서는 ${var.instance_type}을 사용할 수 없습니다."
    }
  }
}
```

## 비용 태그 전략

비용 할당을 위한 태그 전략입니다.

```hcl
locals {
  cost_tags = {
    CostCenter   = var.cost_center       # 비용 센터 코드
    BillingTeam  = var.team_name         # 청구 담당 팀
    CostCategory = var.cost_category     # compute, storage, network, database
    AutoShutdown = var.auto_shutdown     # true/false (개발 환경 자동 종료)
  }
}
```

### 비용 카테고리 정의

```hcl
variable "cost_category" {
  type        = string
  description = "리소스 비용 카테고리"
  validation {
    condition = contains([
      "compute",    # EC2, ECS, Lambda
      "storage",    # S3, EBS, EFS
      "database",   # RDS, DynamoDB, ElastiCache
      "network",    # VPC, NAT Gateway, Load Balancer
      "security",   # WAF, Shield, GuardDuty
      "monitoring"  # CloudWatch, X-Ray
    ], var.cost_category)
    error_message = "유효한 cost_category 값을 입력하세요."
  }
}
```

## 예산 알림 설정

```hcl
# AWS Budgets로 비용 알림 설정
resource "aws_budgets_budget" "monthly" {
  name         = "${var.project_name}-monthly-budget"
  budget_type  = "COST"
  limit_amount = "1000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.budget_alert_email]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = [var.budget_alert_email]
  }

  tags = local.common_tags
}
```

{{< callout type="warning" >}}
NAT Gateway는 데이터 처리 비용이 GB당 $0.045로 예상외로 높습니다. 대용량 데이터를 처리하는 환경에서는 VPC Endpoint를 먼저 검토하세요.
{{< /callout >}}

## 비용 최적화 체크리스트

- [ ] 개발 환경 인스턴스에 자동 종료 스케줄 설정
- [ ] 미사용 EIP(탄력적 IP) 정기 정리
- [ ] S3 Intelligent-Tiering 또는 수명 주기 정책 적용
- [ ] RDS 개발 환경에 Multi-AZ 비활성화
- [ ] Reserved Instance 또는 Savings Plans 검토 (운영 환경)
- [ ] 비용 이상 알림(Anomaly Detection) 활성화
- [ ] infracost를 CI/CD에 통합하여 PR마다 비용 검토
