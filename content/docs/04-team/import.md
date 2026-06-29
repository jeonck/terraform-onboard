---
title: "기존 수동 인프라를 Terraform으로 편입하는 방법"
weight: 2
---

## Import가 필요한 상황

현실에서는 Terraform을 처음 도입할 때 이미 수동으로 만들어진 리소스가 있습니다. 이 리소스를 Terraform 관리하에 두지 않으면, plan을 실행할 때마다 "이 리소스를 새로 만들겠다"는 결과가 나오거나, 실수로 삭제될 수 있습니다.

- 콘솔에서 직접 생성한 VPC, EC2, RDS
- 다른 팀이 클릭으로 만든 보안 그룹
- 예전에 스크립트로 생성한 S3 버킷
- 레거시 인프라 마이그레이션

---

## terraform import 명령어

기본 사용법:

```bash
terraform import <resource_type>.<resource_name> <resource_id>
```

예시: 기존 EC2 인스턴스 가져오기

```bash
# 1. Terraform 코드에 리소스 블록을 먼저 작성
# main.tf
resource "aws_instance" "web" {
  # 내용은 나중에 채움
}

# 2. import 실행
terraform import aws_instance.web i-0123456789abcdef0

# 3. state에 등록됐는지 확인
terraform state show aws_instance.web
```

---

## Import 시 주의사항

{{< callout type="warning" >}}
**import는 state에만 등록합니다. 코드는 직접 작성해야 합니다.**

import를 실행해도 `.tf` 파일이 자동으로 생성되지 않습니다. state에 리소스가 등록된 후, `terraform state show`로 현재 설정을 확인하고 코드를 직접 작성해야 합니다.
{{< /callout >}}

---

## 실전 시나리오: 기존 EC2를 Terraform으로 가져오기

**Step 1: AWS 콘솔에서 리소스 ID 확인**

```
Instance ID: i-0123456789abcdef0
AMI: ami-0c55b159cbfafe1f0
Instance Type: t3.micro
VPC: vpc-0abc123
Subnet: subnet-0def456
Security Groups: sg-0ghi789
```

**Step 2: 코드에 빈 리소스 블록 작성**

```hcl
# main.tf
resource "aws_instance" "legacy_web" {
}
```

**Step 3: import 실행**

```bash
terraform import aws_instance.legacy_web i-0123456789abcdef0
```

**Step 4: state에서 현재 설정 확인**

```bash
terraform state show aws_instance.legacy_web
```

출력 예시:
```hcl
resource "aws_instance" "legacy_web" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  vpc_security_group_ids = ["sg-0ghi789"]
  subnet_id              = "subnet-0def456"
  tags = {
    Name = "web-server"
  }
  # ... 기타 속성
}
```

**Step 5: 코드 완성**

```hcl
# main.tf - state show 결과를 바탕으로 코드 완성
resource "aws_instance" "legacy_web" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = data.aws_subnet.existing.id

  tags = {
    Name        = "web-server"
    Environment = "prod"
    ManagedBy   = "terraform"
  }
}
```

**Step 6: plan으로 drift 없는지 확인**

```bash
terraform plan
# "No changes. Your infrastructure matches the configuration." 이 나와야 함
```

---

## Import 후 코드 정리

import 직후의 코드는 "현재 상태"를 그대로 반영하는 경우가 많습니다. 이후에는 변수·locals·출력값을 적용해 팀 표준에 맞게 정리합니다.

```bash
# import된 리소스 목록 확인
terraform state list

# 특정 리소스 상세 확인
terraform state show aws_instance.legacy_web

# 여러 리소스를 한꺼번에 import할 때
for id in i-001 i-002 i-003; do
  terraform import "aws_instance.web[$id]" "$id"
done
```

---

## 자동 import 방식 (Terraform 1.5+)

Terraform 1.5부터는 `import` 블록을 코드에 직접 작성할 수 있습니다.

```hcl
# main.tf
import {
  to = aws_instance.legacy_web
  id = "i-0123456789abcdef0"
}

resource "aws_instance" "legacy_web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
  # ...
}
```

```bash
terraform plan    # import 계획 확인
terraform apply   # import 실행
```

{{< callout type="info" >}}
**generate 기능 (Terraform 1.5+)**

```bash
terraform plan -generate-config-out=generated.tf
```

이 명령을 사용하면 import 블록에 대한 리소스 코드를 자동 생성해줍니다. 생성된 코드를 검토하고 정리한 뒤 적용하세요.
{{< /callout >}}
