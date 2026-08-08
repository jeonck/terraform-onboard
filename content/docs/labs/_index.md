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
| [Lab 03](variables-outputs) | **변수와 Outputs 실습** | 20분 | 기초 |
| [Lab 04](modules) | **모듈 만들기** | 45분 | 실무 |
| [Lab 05](env-separation) | **dev/prod 환경 분리** | 30분 | 실무 |
| [Lab 06](remote-state) | **Remote State 설정** | 30분 | 협업 |
| [Lab 07](import) | **기존 리소스 Import** | 30분 | 협업 |
| [Lab 08](github-actions) | **GitHub Actions 파이프라인** | 60분 | 자동화 |
| [Lab 09](security-scan) | **보안 스캔 통합** | 30분 | 보안 |
| [Lab 10](state-recovery) | **State 복구 시나리오** | 45분 | 운영 |
| [Lab 11](module-versioning) | **모듈 버전 관리와 레지스트리** | 40분 | 실무 심화 |
| [Lab 12](policy-as-code) | **Policy as Code** | 40분 | 거버넌스 |
| [Lab 13](multi-region) | **멀티 리전 배포** | 45분 | 아키텍처 |
| [Lab 14](iam-least-privilege) | **IAM 최소권한과 컴플라이언스 태깅** | 40분 | 보안 |
| [Lab 15](monitoring-as-code) | **모니터링 as Code** | 40분 | 운영 |
| [Lab 16](terraform-test) | **Terraform 네이티브 테스트** | 40분 | 품질 |
| [Lab 17](drift-detection) | **드리프트 탐지 자동화** | 40분 | 운영 |
| [Lab 18](ecs-container) | **ECS 컨테이너 서비스 배포** | 45분 | 컨테이너 |
| [Lab 19](eks-kubernetes) | **Kubernetes(EKS) 프로비저닝** | 60분 | 컨테이너 |
| [Lab 20](onprem-k8s) | **온프렘형 3-Node K8s 클러스터 구축** | 60분 | 온프렘 |

---

## 실무 JD 기반 역량 매핑 {#jd-mapping}

Lab 11~20은 LinkedIn의 실제 Terraform 채용공고(Terraform Infrastructure Engineer, Senior Terraform Engineer, Cloud Terraform Engineer, DevSecOps Specialist, SRE 등 11건)를 분석해, 현업에서 반복적으로 요구되는 역량을 실습으로 옮긴 것입니다. 각 랩의 "이 랩이 입증하는 실무 역량" 섹션에서 원문 JD 인용을 확인할 수 있습니다.

| 채용공고 요구 역량 | 원문 표현 (JD 인용) | 대응 Lab |
|---|---|---|
| 재사용 모듈 설계·거버넌스 | "build and maintain reusable Terraform modules" | [Lab 04](modules), [Lab 11](module-versioning) |
| 모듈 레지스트리 운영 | "Private Module Registry" | [Lab 11](module-versioning) |
| 정책 기반 배포 통제 | "Terraform Sentinel" | [Lab 12](policy-as-code) |
| 멀티 리전·고가용성 설계 | "multi-region, or active-active cloud infrastructure patterns" | [Lab 13](multi-region) |
| 규제 환경 보안 (IAM·태깅) | "IAM... in HIPAA-regulated production" | [Lab 14](iam-least-privilege) |
| 모니터링·자가 복구 구성 | "self-scaling and self-healing configurations" | [Lab 15](monitoring-as-code) |
| 테스트 주도 인프라 개발 | "Test-Driven Development (TDD)" | [Lab 16](terraform-test) |
| 안전한 운영 자동화 | "safe, scalable cloud automation" | [Lab 10](state-recovery), [Lab 17](drift-detection) |
| 컨테이너 오케스트레이션 | "VPC, EC2, ECS, EKS, IAM, S3, and SQS" | [Lab 18](ecs-container), [Lab 19](eks-kubernetes) |
| Kubernetes 환경 구성·운영 | "Configure and manage Kubernetes environments" | [Lab 19](eks-kubernetes), [Lab 20](onprem-k8s) |
| 온프렘·하이브리드 인프라 | "on-premises and AWS infrastructure" | [Lab 20](onprem-k8s) |
| CI/CD 파이프라인 통합 | "Integrate Terraform into CI/CD pipelines" | [Lab 08](github-actions), [Lab 09](security-scan) |
| State 운영·복구 | "support production infrastructure changes" | [Lab 06](remote-state), [Lab 10](state-recovery) |

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
