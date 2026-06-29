---
title: "Terraform State를 처음부터 제대로 이해하기"
weight: 4
---

## State란 무엇인가

Terraform State는 **Terraform이 관리하는 인프라의 현재 상태를 기록한 데이터베이스**입니다.

```mermaid
flowchart TD
    subgraph terraform["Terraform"]
        code["코드 (.tf 파일)\n원하는 상태"]
        state["State 파일\n(terraform.tfstate)\n현재 상태 기록"]
    end

    subgraph aws["실제 AWS"]
        ec2["EC2 Instance\ni-0abc123def"]
        s3["S3 Bucket\nmy-bucket"]
        vpc["VPC\nvpc-0xyz789"]
    end

    code -- "plan/apply" --> aws
    aws -- "상태 동기화" --> state
    state -- "비교 기준" --> code

    style terraform fill:#f0fdf4
    style aws fill:#eff6ff
```

Terraform은 `terraform apply`를 실행하면:
1. 현재 State와 코드를 비교
2. 실제 AWS API로 현재 인프라 조회
3. 세 가지를 비교해 무엇을 바꿔야 하는지 계산

---

## State 파일 내부 구조

```json
{
  "version": 4,
  "terraform_version": "1.5.7",
  "resources": [
    {
      "type": "aws_s3_bucket",
      "name": "my_first_bucket",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "id": "my-terraform-first-bucket-20240101",
            "arn": "arn:aws:s3:::my-terraform-first-bucket-20240101",
            "bucket": "my-terraform-first-bucket-20240101",
            "region": "ap-northeast-2",
            "tags": {
              "Environment": "dev",
              "ManagedBy": "terraform",
              "Name": "my-first-bucket"
            }
          }
        }
      ]
    }
  ]
}
```

State 파일에 저장되는 정보:
- 리소스의 실제 ID (AWS에서 부여한 ID)
- 모든 속성값
- 리소스 간 의존 관계
- **민감 정보 포함 가능** (RDS 패스워드, IAM 키 등)

---

## State가 왜 중요한가

### 변경 감지

State 없이는 Terraform이 "지금 AWS에 뭐가 있는지" 알 방법이 없습니다.

```bash
# State 기반으로 관리 중인 리소스 목록 확인
$ terraform state list
aws_s3_bucket.my_first_bucket
aws_vpc.main
aws_subnet.public[0]
aws_subnet.public[1]
```

### 리소스 식별

AWS에 같은 이름의 리소스가 있어도, State가 어떤 리소스를 관리하는지 정확히 알고 있습니다.

---

## State가 꼬이면 왜 위험한가

### 시나리오 1: 콘솔에서 수동 변경

```mermaid
sequenceDiagram
    participant Code as Terraform 코드
    participant State as State 파일
    participant AWS as 실제 AWS

    Code->>State: terraform apply (EC2 생성)
    State->>AWS: EC2 생성됨 (i-0abc123)
    Note over AWS: 누군가 콘솔에서 직접 EC2 삭제
    AWS--xState: State는 여전히 EC2 있다고 기록
    Code->>State: terraform plan
    State->>AWS: "EC2 없어? 다시 만들게!"
    Note over AWS: 예상치 못한 EC2 재생성
```

### 시나리오 2: State 파일 분실

State 파일을 잃어버리면 Terraform은 어떤 리소스를 자신이 관리하는지 모릅니다.
- `terraform apply` → 모든 리소스를 새로 생성하려 시도
- 이미 있는 리소스와 충돌 발생
- 최악의 경우 기존 리소스 삭제 후 재생성

### 시나리오 3: 두 사람이 동시에 apply

```
사람 A: terraform apply 시작 (State 읽음)
사람 B: terraform apply 시작 (같은 State 읽음)
사람 A: State 업데이트 완료
사람 B: State 업데이트 완료 → A의 변경 사항 덮어씀!
```

이것이 **State Locking**이 필요한 이유입니다.

---

## State 파일을 Git에 넣으면 안 되는 이유

{{< callout type="warning" >}}
**절대 `terraform.tfstate`를 Git에 커밋하지 마세요!**
{{< /callout >}}

### 이유 1: 민감 정보 노출

```json
{
  "attributes": {
    "password": "super-secret-db-password",  // RDS 패스워드 평문 저장!
    "secret_key": "wJalrXUtnFEMI/K7MDENG",  // AWS Secret Key!
    "private_key_pem": "-----BEGIN RSA PRIVATE KEY-----\n..."  // TLS 인증서!
  }
}
```

### 이유 2: 동시 편집 충돌

여러 팀원이 각자 State를 가지면 충돌이 발생합니다. Git merge conflict와 비슷하지만, 인프라에서 발생하면 리소스가 중복 생성되거나 삭제됩니다.

### 이유 3: 대용량 파일

수백 개의 리소스가 있는 State 파일은 매우 클 수 있으며, 변경할 때마다 Git 이력이 쌓입니다.

**올바른 방법**: Remote Backend 사용 (S3 + DynamoDB 또는 Terraform Cloud)

```hcl
# 팀 협업 시에는 이렇게 Remote Backend를 설정합니다
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-northeast-2"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

Remote Backend에 대한 상세 내용은 [4단계: 팀 협업](/docs/04-team)에서 다룹니다.

---

## 핵심 규칙 정리

{{< callout type="warning" >}}
**State 관련 절대 규칙**

1. `terraform.tfstate`를 Git에 커밋하지 않는다
2. `.gitignore`에 반드시 추가한다
3. 팀 협업 시에는 반드시 Remote Backend를 사용한다
4. State 파일을 직접 편집하지 않는다 (`terraform state` 명령어 사용)
5. `terraform state rm`은 실제 리소스를 삭제하지 않지만 관리에서 제외시킨다 (주의)
{{< /callout >}}

```bash
# .gitignore에 반드시 추가할 항목
terraform.tfstate
terraform.tfstate.backup
.terraform/
*.tfvars        # 민감한 변수값이 있을 수 있음
*.tfvars.json
```

→ 다음 단계: [2단계: 실무 기본기](/docs/02-basics)
