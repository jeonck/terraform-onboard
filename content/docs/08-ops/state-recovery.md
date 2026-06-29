---
title: "State 문제 복구: taint, import, state rm 실전 가이드"
weight: 2
---

{{< callout type="error" >}}
State 직접 조작은 항상 위험합니다. 조작 전에 반드시 백업하고, 프로덕션에서는 두 명 이상이 확인한 후 진행하세요.
{{< /callout >}}

## State 관련 문제 유형

| 문제 | 증상 | 해결 방법 |
|------|------|----------|
| 리소스가 state에 없음 | Plan에서 신규 생성으로 표시 | `terraform import` |
| state에 있지만 실제로 없음 | apply 시 "not found" 오류 | `terraform state rm` |
| 리소스 주소 변경 | 기존 것 삭제 후 새로 생성으로 표시 | `terraform state mv` |
| 강제 재생성 필요 | 설정은 맞지만 리소스 교체 필요 | `terraform apply -replace` |
| State Lock 해제 필요 | `Error acquiring the state lock` | `terraform force-unlock` |

## terraform state 명령어

```bash
# 모든 관리 리소스 목록
terraform state list

# 특정 리소스 상세 보기
terraform state show aws_instance.web

# 리소스 주소 변경 (이름 변경, 모듈 이동 시)
terraform state mv aws_instance.web aws_instance.web_server

# 리소스를 State에서 제거 (실제 리소스는 삭제되지 않음)
terraform state rm aws_instance.legacy

# State 파일 풀기 (다른 state 파일의 리소스 가져오기)
terraform state pull > current.tfstate
```

```bash
# 리소스 이동 예시: 모듈로 리팩토링할 때
# 기존: 루트 모듈에 있던 리소스
terraform state mv aws_instance.web module.compute.aws_instance.web

# 이름 변경 시
terraform state mv aws_s3_bucket.old_name aws_s3_bucket.new_name
```

## terraform state show 활용

```bash
# State에서 리소스 속성 상세 확인
terraform state show aws_instance.web

# 출력 예시
# aws_instance.web:
# resource "aws_instance" "web" {
#     ami                          = "ami-0c55b159cbfafe1f0"
#     arn                          = "arn:aws:ec2:ap-northeast-2:..."
#     id                           = "i-0abc12345def67890"
#     instance_type                = "t3.medium"
#     ...
# }
```

## terraform apply -replace (구 taint)

`terraform taint`는 Terraform 0.15.2부터 deprecated되었습니다. 대신 `terraform apply -replace`를 사용합니다.

```bash
# 구 방식 (deprecated)
terraform taint aws_instance.web
terraform apply

# 신 방식 (권장)
terraform apply -replace="aws_instance.web"

# 여러 리소스 동시 교체
terraform apply -replace="aws_instance.web" -replace="aws_eip.web"
```

### 언제 교체가 필요한가

```bash
# 교체가 필요한 상황들
# 1. EC2 인스턴스 내부 소프트웨어 문제 → 새 인스턴스로 교체
# 2. user_data 변경 → 새 인스턴스에만 적용 가능
# 3. 특정 설정이 인스턴스 내부에서 꼬인 경우

terraform apply -replace="aws_instance.app_server"
```

## State 파일 직접 수정 시 주의사항

{{< callout type="error" >}}
State 파일을 텍스트 편집기로 직접 수정하는 것은 **최후의 수단**입니다. 잘못 수정하면 인프라 전체가 관리 불가 상태가 될 수 있습니다.
{{< /callout >}}

```bash
# State 파일 백업 후 편집이 불가피한 경우
terraform state pull > backup-$(date +%Y%m%d-%H%M%S).tfstate

# JSON으로 편집 후 push
terraform state pull > current.tfstate
# ... 편집 ...
terraform state push current.tfstate
```

## State 백업과 복구 전략

### 자동 백업 설정

```hcl
# S3 backend 설정 (버전 관리 포함)
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-northeast-2"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"

    # versioning으로 자동 백업 (S3 버킷에서 별도 활성화 필요)
  }
}

# S3 버킷 버전 관리 활성화
resource "aws_s3_bucket_versioning" "state" {
  bucket = "my-terraform-state"
  versioning_configuration {
    status = "Enabled"
  }
}
```

### 수동 백업

```bash
# apply 전 수동 백업 (중요 변경 시)
terraform state pull > "backup-$(date +%Y%m%d-%H%M%S)-before-migration.tfstate"
echo "백업 완료: 파일을 안전한 곳에 보관하세요"
```

### S3에서 이전 버전 복구

```bash
# S3에서 State 이전 버전 목록 확인
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix prod/terraform.tfstate \
  --query 'Versions[*].{VersionId:VersionId,LastModified:LastModified}'

# 특정 버전 복구
aws s3api get-object \
  --bucket my-terraform-state \
  --key prod/terraform.tfstate \
  --version-id "BYBZnebad5vkLRPKYQSROmWJ7xk8oQ3r" \
  recovered.tfstate

terraform state push recovered.tfstate
```

## 실전 시나리오: State 문제 복구 Step-by-Step

### 시나리오: 수동으로 생성된 리소스를 Terraform 관리로 편입

```bash
# 1. 현재 State 확인
terraform state list

# 2. 가져올 리소스 ID 확인 (AWS 콘솔 또는 CLI)
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=prod-web-server" \
  --query 'Reservations[*].Instances[*].InstanceId'
# 결과: i-0abc12345def67890

# 3. Terraform 코드에 리소스 블록 추가
# (실제 콘솔 설정과 동일하게 작성)

# 4. import 실행
terraform import aws_instance.web i-0abc12345def67890

# 5. Plan으로 차이 확인
terraform plan
# 이상적으로는 "No changes" 또는 사소한 차이만 있어야 함

# 6. 코드와 실제 상태의 차이 수정
# (Plan에서 표시된 차이점을 코드에 반영)

# 7. 최종 확인
terraform plan  # No changes.가 나올 때까지 반복
```

### 시나리오: 리소스가 콘솔에서 수동 삭제됨

```bash
# 1. Plan 실행 (리소스가 없다는 오류 발생)
terraform plan
# Error: aws_instance.web does not exist

# 2. State에서 해당 리소스 제거
terraform state rm aws_instance.web

# 3. 다시 Plan (이제 신규 생성으로 표시됨)
terraform plan

# 4. Apply로 리소스 재생성
terraform apply
```
