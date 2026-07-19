---
title: "Lab 10: State 복구 시나리오"
weight: 11
---

{{< badge content="Lab 10" type="warning" >}}

운영 중 실제로 발생하는 Terraform State 문제 상황 세 가지를 직접 재현하고 복구합니다. 문제를 직접 만들어보는 것이 가장 빠른 학습법입니다.

---

## State 문제가 발생하는 주요 원인

```mermaid
flowchart TD
    subgraph problems["State 문제 유형"]
        S1["시나리오 1\n콘솔에서 리소스 수동 삭제\nState ≠ 실제 AWS"]
        S2["시나리오 2\n코드에서 리소스 이름 변경\n기존 리소스 삭제 후 재생성"]
        S3["시나리오 3\n콘솔에서 설정 수동 변경\nDrift 발생"]
    end

    subgraph solutions["복구 명령"]
        R1["terraform state rm\n+ terraform import"]
        R2["terraform state mv"]
        R3["terraform plan -refresh-only\n+ terraform apply -refresh-only"]
    end

    S1 --> R1
    S2 --> R2
    S3 --> R3
```

{{< callout type="warning" >}}
**State 조작은 신중하게**: `state rm`, `state mv` 등의 명령은 되돌리기 어렵습니다. 실습 전에 `terraform state pull > backup.tfstate`로 State를 백업하는 습관을 들이세요.
{{< /callout >}}

---

## 실습 파일 구성

```
lab10-state-recovery/
├── versions.tf
├── providers.tf
├── main.tf
├── outputs.tf
└── scripts/
    ├── 00-setup.sh
    ├── 01-scenario1-manual-delete.sh
    ├── 02-scenario2-rename.sh
    ├── 03-scenario3-drift.sh
    └── 04-cleanup.sh
```

각 스크립트는 내부에서 `cd "$(dirname "$0")/.."`로 항상 `lab10-state-recovery/` 루트로 이동한 뒤 실행되므로, 어느 위치에서 호출해도 안전합니다. `03-scenario3-drift.sh`는 `02-scenario2-rename.sh`가 먼저 실행되어 리소스가 `aws_s3_bucket.main`으로 이관되어 있다는 것을 전제로 하므로, 반드시 시나리오 2 → 시나리오 3 순서로 실행하세요.

---

## 사전 준비 — 실습용 리소스 배포

세 시나리오 모두 같은 베이스 코드에서 시작합니다. 아래 4개의 `.tf` 파일이 그 베이스 코드이고, `scripts/00-setup.sh`가 배포와 State 백업까지 자동으로 수행합니다.

### versions.tf

```hcl
terraform {
  required_version = ">= 1.0.0"

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

### providers.tf

```hcl
provider "aws" {
  region = "ap-northeast-2"
}
```

### main.tf

```hcl
# 버킷 이름 중복 방지를 위한 무작위 접미사
resource "random_string" "suffix" {
  length  = 8
  special = false
  upper   = false
}

# 애플리케이션 버킷 — 시나리오 2(이름 변경)와 시나리오 3(Drift)의 대상
resource "aws_s3_bucket" "app" {
  bucket = "lab10-state-recovery-app-${random_string.suffix.result}"

  tags = {
    Name      = "lab10-app"
    ManagedBy = "terraform"
  }
}

# 로그 버킷 — 시나리오 1(콘솔 수동 삭제)의 대상
resource "aws_s3_bucket" "logs" {
  bucket = "lab10-state-recovery-logs-${random_string.suffix.result}"

  tags = {
    Name      = "lab10-logs"
    ManagedBy = "terraform"
  }
}
```

{{< callout type="info" >}}
S3 버킷 이름은 리전이 아니라 전 세계에서 유일해야 합니다. `random_string.suffix`를 이름 뒤에 붙여 여러 사람이 동시에 실습하거나, 같은 사람이 실습을 여러 번 반복해도 이름 충돌(`BucketAlreadyExists`)이 나지 않도록 합니다. 이 때문에 이후 명령 예시에서는 버킷 이름을 하드코딩하지 않고 `terraform output -raw app_bucket_name` / `terraform output -raw logs_bucket_name`으로 조회해서 사용합니다.
{{< /callout >}}

### outputs.tf

```hcl
output "app_bucket_name" {
  description = "애플리케이션 버킷 이름 (시나리오 2, 3 대상)"
  value       = aws_s3_bucket.app.id
}

output "logs_bucket_name" {
  description = "로그 버킷 이름 (시나리오 1 대상)"
  value       = aws_s3_bucket.logs.id
}
```

### 초기 배포

```bash
cd lab10-state-recovery
terraform init
terraform apply -auto-approve
```

또는 State 백업까지 한 번에 처리하는 스크립트를 사용합니다:

```bash
./scripts/00-setup.sh
# terraform init + apply 실행
# backups/backup-<타임스탬프>.tfstate로 State 백업
# app / logs 버킷 이름 출력
```

---

## 시나리오 1: 콘솔에서 리소스 수동 삭제

**상황**: 팀원이 AWS 콘솔에서 S3 버킷을 직접 삭제했습니다. Terraform State에는 여전히 존재합니다.

`scripts/01-scenario1-manual-delete.sh`를 실행하면 아래 "문제 재현"과 "문제 확인" 단계까지 자동으로 실행됩니다. 복구 방법(A/B)은 상황에 따라 선택하는 것이므로 스크립트가 안내만 출력하고, 실제 명령은 아래처럼 직접 실행합니다.

### 문제 재현

```bash
# AWS CLI로 버킷 삭제 (콘솔에서 직접 삭제한 것을 시뮬레이션)
logs_bucket="$(terraform output -raw logs_bucket_name)"
aws s3 rb "s3://${logs_bucket}" --force

# State에는 아직 존재
terraform state list
# aws_s3_bucket.app
# aws_s3_bucket.logs  ← State에 있지만 실제로는 삭제됨
```

### 문제 확인

```bash
terraform plan
```

```
aws_s3_bucket.logs: Refreshing state... [id=lab10-state-recovery-logs-a1b2c3d4]

Error: reading S3 Bucket (lab10-state-recovery-logs-a1b2c3d4): NoSuchBucket
```

또는 Terraform 버전에 따라:

```
  # aws_s3_bucket.logs will be created
  + resource "aws_s3_bucket" "logs" {
      + bucket = "lab10-state-recovery-logs-a1b2c3d4"
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

`plan`이 "새로 생성하겠다"고 나옵니다. 코드대로 다시 만들 것인지, 아니면 코드도 삭제할 것인지 선택해야 합니다.

### 복구 방법 A: 코드도 삭제 (리소스가 더 이상 필요 없는 경우)

```bash
# 1. State에서 제거
terraform state rm aws_s3_bucket.logs

# 2. main.tf에서 aws_s3_bucket.logs 블록 삭제

# 3. plan으로 검증
terraform plan
# No changes. ✅
```

### 복구 방법 B: AWS에서 다시 생성 (리소스가 필요한 경우)

```bash
# 코드는 그대로, apply로 재생성
terraform apply -auto-approve
# aws_s3_bucket.logs: Creating...
# Apply complete! Resources: 1 added. ✅
```

---

## 시나리오 2: 코드에서 리소스 이름 변경

**상황**: `aws_s3_bucket.app`을 `aws_s3_bucket.main`으로 리팩터링했습니다. 그냥 apply하면 기존 버킷이 삭제되고 새 버킷이 생성됩니다.

`scripts/02-scenario2-rename.sh`는 아래 재현 → 확인 → 복구 → 검증 4단계를 전부 자동으로 실행합니다(다른 시나리오 스크립트와 달리 복구까지 끝까지 진행됩니다).

### 문제 재현

`main.tf`에서 리소스 라벨을 `app` → `main`으로 바꿉니다. 이때 `outputs.tf`가 `aws_s3_bucket.app.id`를 참조하고 있으므로 **함께 수정**해야 `plan`/`apply`가 깨지지 않습니다. 실습 스크립트는 이 두 파일을 `sed`로 한 번에 바꿉니다:

```bash
sed -i.bak 's/resource "aws_s3_bucket" "app"/resource "aws_s3_bucket" "main"/' main.tf
sed -i.bak 's/aws_s3_bucket\.app\.id/aws_s3_bucket.main.id/' outputs.tf
rm -f main.tf.bak outputs.tf.bak
```

수동으로 바꾼다면 `main.tf`는 다음과 같은 모습이 됩니다:

```hcl
# 변경 전: resource "aws_s3_bucket" "app"
# 변경 후:
resource "aws_s3_bucket" "main" {
  bucket = "lab10-state-recovery-app-${random_string.suffix.result}"

  tags = {
    Name      = "lab10-app"
    ManagedBy = "terraform"
  }
}
```

그리고 `outputs.tf`의 `app_bucket_name` 출력도 `aws_s3_bucket.main.id`를 가리키도록 바꿔야 합니다.

### 문제 확인

```bash
terraform plan
```

```
  # aws_s3_bucket.app will be destroyed
  - resource "aws_s3_bucket" "app" {
      - bucket = "lab10-state-recovery-app-a1b2c3d4"
    }

  # aws_s3_bucket.main will be created
  + resource "aws_s3_bucket" "main" {
      + bucket = "lab10-state-recovery-app-a1b2c3d4"
    }

Plan: 1 to add, 0 to change, 1 to destroy.
```

{{< callout type="warning" >}}
이 상태에서 `apply`하면 기존 버킷이 삭제됩니다. 버킷에 데이터가 있다면 유실됩니다.
{{< /callout >}}

### 복구: `terraform state mv`로 이름만 변경

```bash
# 형식: terraform state mv <현재_주소> <새_주소>
terraform state mv aws_s3_bucket.app aws_s3_bucket.main
```

```
Move "aws_s3_bucket.app" to "aws_s3_bucket.main"
Successfully moved 1 object(s).
```

### 검증

```bash
terraform plan
# No changes. Your infrastructure matches the configuration. ✅
```

State의 이름만 바뀌었을 뿐, 실제 AWS 버킷은 그대로입니다.

---

## 시나리오 3: Drift 감지 및 복구

**상황**: 누군가 AWS 콘솔에서 S3 버킷 태그를 수동으로 변경했습니다. Terraform 코드와 실제 인프라 사이에 차이(Drift)가 생겼습니다.

{{< callout type="warning" >}}
이 시나리오는 **시나리오 2가 먼저 실행되어 리소스가 `aws_s3_bucket.main`으로 이관되어 있다는 것을 전제**로 합니다. 시나리오 2를 건너뛰고 바로 실행하면 `aws_s3_bucket.main`이 State에 없어 아래 명령이 실패합니다. `scripts/03-scenario3-drift.sh`도 같은 전제로 동작하며, 재현과 Drift 감지까지만 자동 실행하고 복구는 아래 명령을 직접 선택해 실행합니다.
{{< /callout >}}

### 문제 재현

```bash
# AWS CLI로 태그 수동 변경 (콘솔 수동 변경 시뮬레이션)
app_bucket="$(terraform output -raw app_bucket_name)"
aws s3api put-bucket-tagging \
  --bucket "${app_bucket}" \
  --tagging 'TagSet=[{Key=Name,Value=lab10-app-MANUAL},{Key=Owner,Value=ops-team}]'
```

### Drift 감지

```bash
# -refresh-only: 계획 없이 실제 상태를 State에 반영해 차이를 확인
terraform plan -refresh-only
```

```
  ~ aws_s3_bucket.main will be updated in-place
  ~ tags = {
      ~ "Name"  = "lab10-app" -> "lab10-app-MANUAL"
      + "Owner" = "ops-team"
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

Drift가 시각적으로 표시됩니다. 이제 두 가지 선택이 있습니다.

### 복구 방법 A: 코드 기준으로 되돌리기

```bash
# 일반 apply — 코드 상태로 덮어씀
terraform apply -auto-approve
# aws_s3_bucket.main: Modifying... [id=lab10-state-recovery-app-a1b2c3d4]
# Apply complete! ✅ 태그가 코드 기준으로 복구됨
```

### 복구 방법 B: 현재 실제 상태를 State에 반영 (콘솔 변경 수용)

```bash
# State를 실제 AWS 상태로 동기화 (코드는 수정 필요)
terraform apply -refresh-only -auto-approve
```

이후 코드를 실제 상태와 일치하도록 업데이트합니다:

```hcl
resource "aws_s3_bucket" "main" {
  bucket = "lab10-state-recovery-app-${random_string.suffix.result}"

  tags = {
    Name      = "lab10-app-MANUAL"
    Owner     = "ops-team"
    ManagedBy = "terraform"
  }
}
```

---

## 실행 절차

{{% steps %}}

### 사전 준비 및 State 백업

```bash
cd lab10-state-recovery
./scripts/00-setup.sh
# 내부적으로 terraform init + apply, State 백업(backups/backup-*.tfstate),
# app / logs 버킷 이름 출력까지 한 번에 처리합니다.
```

수동으로 하나씩 실행한다면:

```bash
terraform init
terraform apply -auto-approve

# 항상 작업 전 State 백업
terraform state pull > backup.tfstate
echo "백업 완료: $(wc -c < backup.tfstate) bytes"
```

### 시나리오 1 — 콘솔 수동 삭제 복구

```bash
./scripts/01-scenario1-manual-delete.sh   # 재현 + 확인까지 자동 실행

# 복구 (방법 A: State에서 제거 후 코드도 삭제)
terraform state rm aws_s3_bucket.logs
# main.tf에서 logs 블록 삭제
terraform plan   # No changes 확인
```

### 시나리오 2 — 리소스 이름 변경

```bash
./scripts/02-scenario2-rename.sh
# main.tf/outputs.tf를 sed로 "app" → "main"으로 변경
# → plan으로 삭제+생성 계획 확인 (위험!)
# → state mv로 이름만 변경
# → plan으로 No changes 검증
# 위 4단계를 모두 자동 실행합니다.
```

### 시나리오 3 — Drift 감지 및 복구

{{< callout type="warning" >}}
반드시 시나리오 2를 먼저 실행한 뒤 진행하세요. 이 시나리오는 `aws_s3_bucket.main`을 대상으로 합니다.
{{< /callout >}}

```bash
./scripts/03-scenario3-drift.sh   # Drift 재현 + 감지까지 자동 실행

# 복구 (코드 기준으로 되돌리기)
terraform apply -auto-approve
terraform plan -refresh-only   # No changes 확인
```

### 정리

```bash
./scripts/04-cleanup.sh
# 내부적으로 terraform destroy -auto-approve
```

{{% /steps %}}

---

## State 조작 명령어 총정리

| 명령어 | 용도 | 주의사항 |
|--------|------|----------|
| `terraform state list` | 현재 State에 등록된 리소스 목록 | — |
| `terraform state show <주소>` | 특정 리소스의 State 상세 확인 | — |
| `terraform state rm <주소>` | State에서 리소스 제거 (실제 AWS 리소스 유지) | 되돌리기 어려움 |
| `terraform state mv <from> <to>` | State 내 리소스 주소 변경 | 되돌리기 어려움 |
| `terraform state pull` | 현재 State를 stdout으로 출력 | 백업 시 활용 |
| `terraform state push` | 로컬 파일을 Remote State에 강제 업로드 | 매우 위험 — 신중하게 |
| `terraform plan -refresh-only` | Drift 감지 (계획만, 변경 없음) | — |
| `terraform apply -refresh-only` | State를 실제 AWS 상태로 동기화 | 코드 수정 필요 |
| `terraform apply -replace=<주소>` | 특정 리소스 강제 재생성 | 다운타임 발생 가능 |

---

## 주의사항

{{< callout type="warning" >}}
**작업 전 State 백업 필수**: `terraform state pull > backup-$(date +%Y%m%d-%H%M%S).tfstate` — State를 잘못 수정하면 전체 인프라 관리가 불가능해질 수 있습니다.
{{< /callout >}}

{{< callout type="warning" >}}
**`terraform state push`는 최후의 수단**: 잘못된 State 파일을 push하면 Terraform이 존재하지 않는 리소스를 관리하려 하거나, 실제 리소스를 모르는 상태가 됩니다. Remote State에서 버전 관리(S3 versioning)가 켜져 있어야 롤백이 가능합니다.
{{< /callout >}}

{{< callout type="info" >}}
**`-replace` vs 수동 삭제 후 apply**: `terraform apply -replace="aws_instance.web"`은 Terraform이 해당 리소스를 먼저 삭제하고 다시 만듭니다. 콘솔에서 직접 삭제하고 `apply`하는 것과 결과는 같지만, `-replace`는 State와 실제 리소스를 동기화 상태로 유지합니다.
{{< /callout >}}

{{< callout type="info" >}}
**Drift는 일상적인 문제**: 운영 환경에서는 긴급 패치, 콘솔 실수, 다른 팀의 수동 변경 등으로 Drift가 자주 발생합니다. `terraform plan -refresh-only`를 CI/CD에서 주기적으로 실행해 Drift를 조기에 감지하세요.
{{< /callout >}}

---

## 핵심 학습 포인트

**State는 Terraform의 세계관**: Terraform은 State에 있는 것만 관리합니다. AWS에 실제로 존재해도 State에 없으면 모르고, State에 있어도 AWS에서 삭제되면 다시 만들려 합니다. State를 이해하면 Terraform의 동작 방식이 전부 예측 가능해집니다.

**`state mv` = 리소스를 건드리지 않고 이름만 변경**: 코드 리팩터링 시 리소스 이름이 바뀌어도 `state mv`로 State 주소를 먼저 맞추면 삭제·재생성 없이 적용할 수 있습니다. 데이터가 있는 리소스(DB, S3)에서 특히 중요합니다.

**Drift 감지는 정기 작업으로**: `plan -refresh-only`는 인프라를 변경하지 않고 실제 상태와의 차이만 보여줍니다. 주간 스케줄러나 CI/CD에 넣어두면 수동 변경을 조기에 발견할 수 있습니다.

→ 다음 실습: [Lab 11 모듈 버전 관리와 레지스트리](../module-versioning/) — 팀 규모의 재사용 모듈 배포 전략
