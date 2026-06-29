---
title: "참고자료"
weight: 12
---

## 공식 문서 링크 모음

| 분류 | 링크 | 설명 |
|------|------|------|
| Terraform 공식 문서 | https://developer.hashicorp.com/terraform/docs | 전체 문서 |
| Terraform CLI 레퍼런스 | https://developer.hashicorp.com/terraform/cli | CLI 명령어 |
| AWS Provider 문서 | https://registry.terraform.io/providers/hashicorp/aws/latest | AWS 리소스 |
| Terraform Registry | https://registry.terraform.io | 모듈 & Provider |
| Terraform 언어 레퍼런스 | https://developer.hashicorp.com/terraform/language | HCL 문법 |
| Terraform 베스트 프랙티스 | https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices | 권장 패턴 |

## 유용한 도구 목록

| 도구 | 설명 | 설치 |
|------|------|------|
| **tflint** | Terraform 린터 (오류 패턴, deprecated 사용 감지) | `brew install tflint` |
| **tfsec** | IaC 보안 정적 분석 | `brew install tfsec` |
| **checkov** | IaC 보안 & 컴플라이언스 스캐너 | `pip install checkov` |
| **infracost** | Terraform 비용 추정 | `brew install infracost` |
| **terragrunt** | Terraform 래퍼 (DRY 구성 지원) | `brew install terragrunt` |
| **terraform-docs** | Terraform 모듈 문서 자동 생성 | `brew install terraform-docs` |
| **tfenv** | Terraform 버전 관리 | `brew install tfenv` |
| **trivy** | 컨테이너 + IaC 보안 스캔 | `brew install trivy` |

### 주요 도구 빠른 시작

```bash
# tflint 초기화 및 실행
tflint --init
tflint .

# terraform-docs로 README 자동 생성
terraform-docs markdown . > README.md

# tfenv로 버전 관리
tfenv install 1.7.0
tfenv use 1.7.0
tfenv list
```

## 추천 모듈 레지스트리

공식 및 검증된 Terraform 모듈 목록입니다. 직접 만들기 전에 먼저 확인하세요.

| 모듈 | 출처 | 설명 |
|------|------|------|
| `terraform-aws-modules/vpc/aws` | terraform-aws-modules | VPC + Subnet + NAT |
| `terraform-aws-modules/eks/aws` | terraform-aws-modules | EKS 클러스터 |
| `terraform-aws-modules/rds/aws` | terraform-aws-modules | RDS 인스턴스 |
| `terraform-aws-modules/alb/aws` | terraform-aws-modules | Application Load Balancer |
| `terraform-aws-modules/iam/aws` | terraform-aws-modules | IAM 역할/정책 |
| `terraform-aws-modules/s3-bucket/aws` | terraform-aws-modules | S3 버킷 (보안 설정 포함) |

```hcl
# 검증된 VPC 모듈 사용 예시
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.project_name}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-northeast-2a", "ap-northeast-2c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = var.environment != "prod"  # prod는 HA NAT

  tags = local.common_tags
}
```

## 핵심 CLI 명령어 Cheat Sheet

### 기본 워크플로우

| 명령어 | 설명 |
|--------|------|
| `terraform init` | Provider 다운로드, backend 초기화 |
| `terraform init -upgrade` | Provider 버전 강제 업그레이드 |
| `terraform validate` | 문법 검증 |
| `terraform fmt` | 코드 포맷 정리 |
| `terraform fmt -recursive` | 하위 디렉토리 포함 포맷 |
| `terraform plan` | 변경 계획 미리보기 |
| `terraform plan -out=tfplan` | Plan 결과 파일로 저장 |
| `terraform apply` | 실제 변경 적용 |
| `terraform apply tfplan` | 저장된 Plan 파일로 apply |
| `terraform apply -auto-approve` | 확인 없이 자동 apply (CI/CD용) |
| `terraform destroy` | 모든 리소스 삭제 |

### State 관리

| 명령어 | 설명 |
|--------|------|
| `terraform state list` | State에 있는 모든 리소스 목록 |
| `terraform state show <주소>` | 리소스 상세 속성 확인 |
| `terraform state mv <from> <to>` | 리소스 주소 변경 |
| `terraform state rm <주소>` | State에서 리소스 제거 (실제 삭제 안 됨) |
| `terraform state pull` | Remote State를 stdout으로 출력 |
| `terraform state push <파일>` | State 파일을 Remote로 업로드 |
| `terraform force-unlock <ID>` | State Lock 강제 해제 |

### 고급 명령어

| 명령어 | 설명 |
|--------|------|
| `terraform import <주소> <ID>` | 기존 리소스를 State에 편입 |
| `terraform apply -replace=<주소>` | 특정 리소스 강제 교체 |
| `terraform plan -refresh-only` | 실제 상태만 갱신 (Drift 확인) |
| `terraform apply -refresh-only` | 실제 상태를 State에 반영 |
| `terraform plan -target=<주소>` | 특정 리소스만 plan |
| `terraform output` | 모든 output 값 출력 |
| `terraform output -json` | JSON 형식으로 output 출력 |
| `terraform graph` | 의존성 그래프 출력 (DOT 형식) |
| `terraform console` | 대화형 표현식 평가 |

### 환경 변수

| 변수 | 설명 |
|------|------|
| `TF_LOG=DEBUG` | 상세 디버그 로그 활성화 |
| `TF_LOG=INFO` | 정보 수준 로그 |
| `TF_VAR_<변수명>=<값>` | 변수 환경 변수로 설정 |
| `TF_CLI_ARGS_plan="-compact-warnings"` | plan 기본 인자 설정 |
| `AWS_PROFILE=<프로필>` | AWS CLI 프로필 선택 |

## 자주 쓰는 Provider 목록

| Provider | Registry ID | 용도 |
|----------|------------|------|
| AWS | `hashicorp/aws` | AWS 리소스 전반 |
| Kubernetes | `hashicorp/kubernetes` | K8s 리소스 |
| Helm | `hashicorp/helm` | Helm Chart 배포 |
| GitHub | `integrations/github` | GitHub 저장소/팀 관리 |
| Datadog | `DataDog/datadog` | 모니터링 설정 |
| PagerDuty | `PagerDuty/pagerduty` | 알림 설정 |
| Vault | `hashicorp/vault` | 비밀정보 관리 |
| Random | `hashicorp/random` | 임의 값 생성 |
| Null | `hashicorp/null` | 프로비저너, 의존성 |
| Archive | `hashicorp/archive` | ZIP 파일 생성 |

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}
```
