---
title: "팀에서 쓰는 Terraform 폴더 구조 표준안"
weight: 3
---

## 왜 폴더 구조가 중요한가

Terraform 코드가 늘어날수록 "어디에 무엇이 있는지"를 빠르게 파악할 수 있어야 합니다. 폴더 구조는 팀의 약속이고, 온보딩 속도와 사고 방지에 직접적인 영향을 줍니다.

---

## Small Project 구조

단일 환경, 소규모 팀 (1~3명)에 적합합니다.

```
terraform-project/
├── main.tf          # 주요 리소스 정의
├── variables.tf     # 입력 변수 선언
├── outputs.tf       # 출력값 선언
├── terraform.tfvars # 변수 값 (Git에 커밋 가능한 값)
├── versions.tf      # required_providers, terraform 버전
└── README.md
```

각 파일의 역할:

| 파일 | 역할 |
|------|------|
| `main.tf` | 리소스와 모듈 호출 |
| `variables.tf` | 변수 선언 (이름, 타입, 설명, 기본값) |
| `outputs.tf` | 배포 후 출력할 값 |
| `terraform.tfvars` | 변수 실제 값 |
| `versions.tf` | Terraform 및 Provider 버전 고정 |

---

## Multi-Environment 구조

dev / staging / prod를 디렉토리로 분리하는 방식입니다.

```
terraform-project/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── backend.tf         # S3 remote state 설정
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── ...
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── ...
│       └── terraform.tfvars
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── ec2/
    │   └── ...
    └── rds/
        └── ...
```

{{< callout type="info" >}}
각 환경 디렉토리에서 독립적으로 `terraform init`, `plan`, `apply`를 실행합니다. 환경 간 state가 분리되어 prod에서 실수로 dev를 건드리는 일이 없어집니다.
{{< /callout >}}

---

## Monorepo vs Repo 분리 전략

| 구분 | Monorepo | Repo 분리 |
|------|----------|----------|
| **구조** | 모든 인프라 코드를 하나의 저장소에 | 서비스별 또는 환경별로 저장소 분리 |
| **장점** | 한 곳에서 전체 인프라 파악 가능, 모듈 공유 쉬움 | 팀별 독립 배포, 접근 권한 세분화 가능 |
| **단점** | 규모 커지면 CI/CD 복잡, 권한 분리 어려움 | 모듈 버전 관리 필요, 코드 파악 분산 |
| **추천 규모** | 소~중규모 팀, 단일 플랫폼 팀 | 대규모 조직, 독립적인 다수 팀 |

---

## 추천 표준 폴더 구조 (중규모 팀)

실무에서 가장 많이 사용하는 구조입니다.

```
terraform/
├── modules/                    # 공용 재사용 모듈
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── compute/
│   └── database/
├── environments/
│   ├── _shared/                # 모든 환경이 공유하는 설정
│   │   └── provider.tf
│   ├── dev/
│   │   ├── backend.tf          # 환경별 remote state 설정
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
├── .github/
│   └── workflows/
│       └── terraform.yml       # CI/CD 파이프라인
├── .gitignore
└── README.md
```

### .gitignore 필수 설정

```
# Terraform
.terraform/
.terraform.lock.hcl             # 팀 협의에 따라 커밋 여부 결정
terraform.tfstate               # 절대 커밋 금지
terraform.tfstate.backup
*.tfvars                        # 민감값 포함 가능 → 환경변수로 관리
!example.tfvars                 # 예시 파일은 포함
crash.log
override.tf
override.tf.json
```

{{< callout type="warning" >}}
**`terraform.tfstate`는 절대로 Git에 커밋하지 마세요.** 민감한 패스워드, 키, 시크릿이 평문으로 포함될 수 있습니다. Remote state를 사용해야 합니다.
{{< /callout >}}

---

## 실제 팀에서 쓰는 예시

```
company-infra/
├── modules/
│   ├── vpc/
│   ├── eks-cluster/
│   ├── rds-postgres/
│   └── s3-logging/
├── environments/
│   ├── dev/
│   │   ├── networking/      # 레이어별 분리 (대규모 state 관리)
│   │   │   └── main.tf
│   │   ├── compute/
│   │   │   └── main.tf
│   │   └── database/
│   │       └── main.tf
│   └── prod/
│       ├── networking/
│       ├── compute/
│       └── database/
└── scripts/
    ├── plan-all.sh
    └── apply-all.sh
```

레이어별 분리(networking / compute / database)는 state 범위를 작게 유지해 blast radius를 줄입니다. 네트워크 변경이 서버 state에 영향을 주지 않습니다.
