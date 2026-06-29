---
title: "15분 만에 첫 인프라 배포하기"
weight: 2
---

이 실습은 AWS S3 버킷 1개를 Terraform으로 생성하고 삭제합니다. 클라우드 콘솔을 열 필요 없습니다.

## 사전 준비

### Terraform 설치

```bash
# macOS (Homebrew)
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Ubuntu/Debian
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# 버전 확인
terraform version
```

### AWS CLI 설정

```bash
# AWS CLI 설치 후
aws configure
# AWS Access Key ID: [발급한 키]
# AWS Secret Access Key: [발급한 시크릿]
# Default region name: ap-northeast-2
# Default output format: json
```

{{< callout type="info" >}}
**AWS 계정이 없다면**: AWS Free Tier로 무료 가입 후 진행하세요. 이 실습에서 사용하는 S3 버킷은 Free Tier 범위 내입니다.
{{< /callout >}}

---

## 실습 디렉토리 구성

```bash
mkdir terraform-first-lab && cd terraform-first-lab
```

총 2개의 파일만 만들면 됩니다.

---

## Provider 설정

```hcl
# provider.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-2"  # 서울 리전
}
```

**Provider란?** Terraform이 AWS API와 통신하기 위한 플러그인입니다. AWS, GCP, Azure, Kubernetes 등 각각의 Provider가 있습니다.

---

## 첫 번째 리소스: S3 버킷 생성

```hcl
# main.tf
resource "aws_s3_bucket" "my_first_bucket" {
  bucket = "my-terraform-first-bucket-20240101"  # 전 세계 고유한 이름 필요

  tags = {
    Name        = "my-first-bucket"
    Environment = "dev"
    ManagedBy   = "terraform"
  }
}

output "bucket_name" {
  value = aws_s3_bucket.my_first_bucket.bucket
}

output "bucket_arn" {
  value = aws_s3_bucket.my_first_bucket.arn
}
```

{{< callout type="warning" >}}
**S3 버킷 이름은 전 세계에서 유일해야 합니다.** `my-terraform-first-bucket-20240101`에서 날짜 부분을 자신의 날짜나 랜덤 문자열로 바꾸세요.
{{< /callout >}}

---

## terraform init

```bash
$ terraform init

Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.31.0...
- Installed hashicorp/aws v5.31.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

**init이 하는 일:**
- `.terraform/` 디렉토리 생성
- 필요한 Provider 플러그인 다운로드
- Backend 초기화 (지금은 로컬)

---

## terraform plan

```bash
$ terraform plan

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.my_first_bucket will be created
  + resource "aws_s3_bucket" "my_first_bucket" {
      + bucket                      = "my-terraform-first-bucket-20240101"
      + tags                        = {
          + "Environment" = "dev"
          + "ManagedBy"   = "terraform"
          + "Name"        = "my-first-bucket"
        }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + bucket_arn  = (known after apply)
  + bucket_name = "my-terraform-first-bucket-20240101"
```

`+` 기호는 **새로 생성**됨을 의미합니다. apply 전에 반드시 확인하세요.

---

## terraform apply

```bash
$ terraform apply

# plan 결과 다시 보여줌...

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

aws_s3_bucket.my_first_bucket: Creating...
aws_s3_bucket.my_first_bucket: Creation complete after 2s [id=my-terraform-first-bucket-20240101]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

bucket_arn = "arn:aws:s3:::my-terraform-first-bucket-20240101"
bucket_name = "my-terraform-first-bucket-20240101"
```

AWS 콘솔 → S3에서 버킷이 생성된 것을 확인할 수 있습니다.

---

## terraform destroy

실습이 끝났으면 리소스를 삭제합니다. 비용이 발생하지 않도록 **반드시 정리**합니다.

```bash
$ terraform destroy

aws_s3_bucket.my_first_bucket: Refreshing state...

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # aws_s3_bucket.my_first_bucket will be destroyed
  - resource "aws_s3_bucket" "my_first_bucket" {
      ...
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Enter a value: yes

aws_s3_bucket.my_first_bucket: Destroying...
aws_s3_bucket.my_first_bucket: Destruction complete after 1s

Destroy complete! Resources: 1 destroyed.
```

---

## 이 실습에서 배운 것

- `provider.tf`: Terraform이 어떤 클라우드와 통신할지 선언
- `resource` 블록: 만들고 싶은 인프라를 선언
- `output` 블록: 생성 후 확인하고 싶은 값 출력
- `init` → `plan` → `apply` → `destroy` 기본 흐름 체득

→ 다음: [Terraform 실행 흐름 완전 이해](workflow)
