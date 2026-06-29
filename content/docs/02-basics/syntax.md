---
title: "count와 for_each를 언제 어떻게 써야 하는가"
weight: 2
---

반복문은 Terraform에서 가장 많이 쓰는 기능 중 하나입니다. 하지만 잘못 쓰면 나중에 State가 꼬이거나, 의도치 않은 리소스 재생성이 발생합니다.

## Terraform의 기본 데이터 타입

```hcl
# 문자열 (string)
variable "region" {
  type    = string
  default = "ap-northeast-2"
}

# 숫자 (number)
variable "port" {
  type    = number
  default = 8080
}

# 불리언 (bool)
variable "enable_https" {
  type    = bool
  default = true
}

# 리스트 (list) - 순서 있음, 중복 허용
variable "availability_zones" {
  type    = list(string)
  default = ["ap-northeast-2a", "ap-northeast-2b", "ap-northeast-2c"]
}

# 맵 (map) - 키-값 쌍
variable "instance_types" {
  type = map(string)
  default = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.medium"
  }
}

# 객체 (object) - 여러 타입 혼합
variable "database_config" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
}
```

---

## count: 숫자 기반 반복

```hcl
# 3개의 서브넷을 만들 때
resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}

# 참조 시: 인덱스로 접근
output "subnet_ids" {
  value = aws_subnet.public[*].id  # 전체 목록
}

# 특정 인덱스 접근
resource "aws_route_table_association" "public" {
  count          = 3
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

### count의 한계

{{< callout type="warning" >}}
**count의 치명적 문제**: 중간 항목을 삭제하면 이후 모든 인덱스가 바뀝니다.

```hcl
# Before: 3개
variable "subnets" {
  default = ["subnet-a", "subnet-b", "subnet-c"]
}
# State: subnet[0]=a, subnet[1]=b, subnet[2]=c

# subnet-b를 제거하면:
variable "subnets" {
  default = ["subnet-a", "subnet-c"]
}
# Terraform이 인식하는 것:
# subnet[0]=a (변경 없음)
# subnet[1]=c (b 위치였는데 c가 됨 → destroy + create!)
# subnet[2] 삭제
```

이 경우 subnet-b와 subnet-c가 불필요하게 재생성됩니다.
{{< /callout >}}

---

## for_each: 키 기반 반복 (권장)

```hcl
# 맵으로 서브넷 정의
variable "subnets" {
  type = map(object({
    cidr_block        = string
    availability_zone = string
    public            = bool
  }))
  default = {
    "public-2a" = {
      cidr_block        = "10.0.1.0/24"
      availability_zone = "ap-northeast-2a"
      public            = true
    }
    "public-2b" = {
      cidr_block        = "10.0.2.0/24"
      availability_zone = "ap-northeast-2b"
      public            = true
    }
    "private-2a" = {
      cidr_block        = "10.0.10.0/24"
      availability_zone = "ap-northeast-2a"
      public            = false
    }
  }
}

resource "aws_subnet" "this" {
  for_each = var.subnets

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.availability_zone

  tags = {
    Name   = each.key
    Public = each.value.public
  }
}

# 참조 시: 키로 접근
output "subnet_ids" {
  value = { for k, v in aws_subnet.this : k => v.id }
}
```

**for_each의 장점**: State가 키 기반으로 관리되므로 중간 항목을 삭제해도 다른 항목에 영향 없음.

---

## 코드 비교: count vs for_each

**나쁜 예** (count 사용):
```hcl
resource "aws_iam_user" "team" {
  count = length(var.team_members)
  name  = var.team_members[count.index]
}
# team_members = ["alice", "bob", "carol"]
# alice를 제거하면? bob → [0], carol → [1]
# bob과 carol이 재생성됨!
```

**좋은 예** (for_each 사용):
```hcl
resource "aws_iam_user" "team" {
  for_each = toset(var.team_members)
  name     = each.value
}
# alice를 제거해도 bob, carol에 영향 없음
```

---

## 조건식 (Conditional Expression)

```hcl
# 형식: condition ? true_value : false_value
locals {
  instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"
  min_size      = var.environment == "prod" ? 3 : 1
  max_size      = var.environment == "prod" ? 10 : 3
}

# 리소스를 조건부로 생성
resource "aws_cloudwatch_metric_alarm" "cpu" {
  count = var.environment == "prod" ? 1 : 0  # prod에서만 생성

  alarm_name = "high-cpu-utilization"
  # ...
}
```

---

## 동적 블록 (Dynamic Block)

반복적으로 나타나는 내부 블록을 동적으로 생성합니다.

**나쁜 예** (하드코딩):
```hcl
resource "aws_security_group" "web" {
  name = "web-sg"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
}
```

**좋은 예** (dynamic block):
```hcl
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    { port = 80,   protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { port = 443,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { port = 8080, protocol = "tcp", cidr_blocks = ["10.0.0.0/8"] },
  ]
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

---

## 언제 무엇을 쓸까

| 상황 | 권장 방법 |
|------|----------|
| 같은 리소스 N개 생성, 순서만 다름 | `count` |
| 고유한 식별자(이름/키)가 있는 리소스 반복 | `for_each` |
| 중간 항목을 자주 추가/삭제할 가능성 | `for_each` |
| 단순 on/off 조건 (0 or 1) | `count = condition ? 1 : 0` |
| 리소스 내 블록이 반복 | `dynamic` |

→ 다음: [실무자가 꼭 이해해야 하는 실행 흐름](execution-flow)
