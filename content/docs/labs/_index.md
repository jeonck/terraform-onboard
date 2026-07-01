---
title: "실습 랩"
weight: 11
---

코드를 직접 작성하고 실행해보는 핵심 실습 목록입니다. 각 Lab은 독립적으로 수행 가능하지만 순서대로 진행하면 학습 효과가 높습니다.

{{< callout type="info" >}}
**실습 환경 준비**: AWS 계정, Terraform CLI, AWS CLI가 필요합니다. 모든 실습은 AWS 프리 티어 또는 최소 비용으로 진행 가능합니다.
{{< /callout >}}

## Lab 목록

| Lab | 제목 | 소요 시간 | 단계 |
|-----|------|----------|------|
| [QuickLab](ec2-webserver) | **EC2 웹서버 퀵실습** | 15분 | 입문 |
| [Lab 01](s3-bucket) | **S3 버킷 첫 번째 실습** | 15분 | 입문 |
| [Lab 02](vpc-ec2) | **VPC + EC2 기본 스택** | 30분 | 기초 |
| [Lab 03](#lab-03) | 변수와 Outputs 실습 | 20분 | 기초 |
| [Lab 04](#lab-04) | 모듈 만들기 | 45분 | 실무 |
| [Lab 05](#lab-05) | dev/prod 환경 분리 | 30분 | 실무 |
| [Lab 06](#lab-06) | Remote State 설정 | 30분 | 협업 |
| [Lab 07](#lab-07) | 기존 리소스 Import | 30분 | 협업 |
| [Lab 08](#lab-08) | GitHub Actions 파이프라인 | 60분 | 자동화 |
| [Lab 09](#lab-09) | 보안 스캔 통합 | 30분 | 보안 |
| [Lab 10](#lab-10) | State 복구 시나리오 | 45분 | 운영 |

---

## Lab 01: 첫 번째 리소스 배포 {#lab-01}

**목표**: S3 버킷 하나를 직접 생성하고 삭제하며 Terraform 기본 실행 흐름 체험

**학습 내용**: `terraform init`, `plan`, `apply`, `destroy`

```bash
# 작업 디렉토리 생성
mkdir lab01-first-resource && cd lab01-first-resource

# main.tf 작성
cat > main.tf << 'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_s3_bucket" "my_first_bucket" {
  bucket = "my-terraform-lab01-${random_string.suffix.result}"
}

resource "random_string" "suffix" {
  length  = 8
  special = false
  upper   = false
}

output "bucket_name" {
  value = aws_s3_bucket.my_first_bucket.id
}
EOF

terraform init
terraform plan
terraform apply
terraform destroy
```

**예상 결과**: S3 버킷 생성 → 이름 출력 → 삭제 완료

---

## Lab 02: VPC + EC2 기본 스택 {#lab-02}

**목표**: 실제 인프라 구성 요소(VPC, Subnet, Security Group, EC2)를 단계별로 배포

**학습 내용**: 리소스 간 참조, 암시적 의존성, 출력값 활용

```bash
# VPC 생성 → Subnet → Security Group → EC2 순서로 작성
# 각 단계에서 terraform plan으로 변경 내용 확인
```

**실습 포인트**:
- `aws_vpc.main.id`처럼 참조로 의존성 표현
- 보안 그룹에서 최소 권한 원칙 적용
- Output으로 EC2 퍼블릭 IP 출력

---

## Lab 03: 변수와 Outputs 실습 {#lab-03}

**목표**: 하드코딩된 값을 변수로 교체하고, outputs으로 필요한 정보 노출

**학습 내용**: `variable`, `output`, `locals`, `tfvars` 파일

```bash
# 1. 하드코딩된 main.tf에서 시작
# 2. 변수 추출: instance_type, environment
# 3. locals로 네이밍 규칙 중앙화
# 4. dev.tfvars, prod.tfvars 분리
terraform plan -var-file=dev.tfvars
terraform plan -var-file=prod.tfvars
```

---

## Lab 04: 모듈 만들기 {#lab-04}

**목표**: 재사용 가능한 네트워크 모듈 작성 및 루트 모듈에서 호출

**학습 내용**: 모듈 입력/출력 설계, `module` 블록 사용

```
디렉토리 구조:
lab04/
├── modules/
│   └── network/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── main.tf  (모듈 호출)
```

---

## Lab 05: dev/prod 환경 분리 {#lab-05}

**목표**: 동일 모듈을 dev와 prod 환경에 다른 설정으로 배포

**학습 내용**: 디렉토리 기반 환경 분리, tfvars 활용

```
environments/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
└── prod/
    ├── main.tf
    └── terraform.tfvars
```

---

## Lab 06: Remote State 설정 {#lab-06}

**목표**: S3 + DynamoDB로 Remote Backend 구성하고 팀 협업 기반 마련

**학습 내용**: S3 backend 설정, State Locking, State 공유

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "lab06/terraform.tfstate"
    region         = "ap-northeast-2"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

**실습 포인트**: 두 터미널에서 동시 apply 시도 → Lock 동작 확인

---

## Lab 07: 기존 리소스 Import {#lab-07}

**목표**: 콘솔에서 수동 생성한 리소스를 Terraform State에 편입

**학습 내용**: `terraform import`, import block (Terraform 1.5+)

```bash
# 방법 1: CLI import
terraform import aws_s3_bucket.existing my-existing-bucket-name

# 방법 2: import block (Terraform 1.5+)
import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket-name"
}
```

---

## Lab 08: GitHub Actions 파이프라인 {#lab-08}

**목표**: PR 생성 시 자동 plan, main 머지 시 자동 apply 파이프라인 구성

**학습 내용**: GitHub Actions workflow, OIDC 인증, 환경 승인

```yaml
# .github/workflows/terraform.yml
on:
  pull_request:  # plan
  push:
    branches: [main]  # apply
```

**실습 포인트**: PR 코멘트로 plan 결과 자동 게시

---

## Lab 09: 보안 스캔 통합 {#lab-09}

**목표**: tfsec와 checkov를 CI/CD 파이프라인에 추가

**학습 내용**: IaC 정적 분석, 보안 이슈 수정

```bash
# 의도적으로 보안 이슈 있는 코드 작성 후
tfsec .
checkov -d .
# 결과를 보고 코드 수정
```

---

## Lab 10: State 복구 시나리오 {#lab-10}

**목표**: 실제 운영에서 발생하는 State 문제 상황을 재현하고 복구

**학습 내용**: `state rm`, `state mv`, `terraform import`, `apply -replace`

**시나리오 1**: 콘솔에서 리소스 수동 삭제 후 state 정리

**시나리오 2**: 리소스 코드 이름 변경 후 교체 없이 state mv로 처리

**시나리오 3**: 콘솔 수동 변경(Drift) 감지 및 복구

```bash
# Drift 감지
terraform plan -refresh-only

# State에서 리소스 제거
terraform state rm aws_instance.old_name

# 리소스 교체
terraform apply -replace="aws_instance.broken"
```
