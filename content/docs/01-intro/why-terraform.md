---
title: "Terraform을 왜 배우는가: 콘솔 운영에서 코드 운영으로"
weight: 1
---

## 수작업 인프라 운영의 문제

AWS 콘솔에서 EC2를 만들고, Security Group을 설정하고, S3 버킷을 생성해본 적 있으신가요? 처음엔 편리해 보입니다. 그런데 몇 가지 문제가 생기기 시작합니다.

```mermaid
flowchart LR
    subgraph 수작업 운영["❌ 수작업 운영 (콘솔 클릭)"]
        direction TB
        A1[콘솔 접속] --> B1[리소스 생성]
        B1 --> C1[누가 만들었지?]
        C1 --> D1[왜 만들었지?]
        D1 --> E1[같은 환경 복제 불가]
        E1 --> F1[변경 이력 없음]
    end

    subgraph iac["✅ IaC 운영 (Terraform)"]
        direction TB
        A2[코드 작성] --> B2[terraform plan]
        B2 --> C2[변경 사항 확인]
        C2 --> D2[terraform apply]
        D2 --> E2[State 자동 기록]
        E2 --> F2[Git에 이력 남음]
    end

    수작업 운영 -- "전환" --> iac

    style 수작업 운영 fill:#fee2e2
    style iac fill:#dcfce7
```

### 실제 현장에서 자주 겪는 상황

1. **"이 리소스 누가 만들었어요?"** — 콘솔에서 만든 리소스는 생성자 기록이 없음
2. **"dev 환경이랑 똑같이 prod 만들어 주세요"** — 클릭으로 만든 환경은 완전히 동일하게 재현 불가
3. **"지난주에 뭔가 바꿨는데 뭘 바꿨는지 모르겠어요"** — 콘솔 변경은 이력이 남지 않음
4. **"신규 입사자가 실수로 운영 리소스 삭제했어요"** — 접근 통제와 변경 리뷰 없음

---

## 클릭 운영 vs 코드 운영

| 항목 | 콘솔 클릭 운영 | Terraform 코드 운영 |
|------|--------------|-------------------|
| 변경 이력 | 없음 (CloudTrail은 있지만 복잡) | Git 커밋 이력으로 명확 |
| 환경 재현 | 수동으로 하나씩 반복 | `terraform apply` 한 번 |
| 팀 리뷰 | 불가능 | Pull Request로 코드 리뷰 |
| 실수 복구 | 어려움 (무엇을 바꿨는지 모름) | 이전 커밋으로 rollback |
| 문서화 | 별도 작성 필요 | 코드 자체가 문서 |
| 여러 환경 관리 | 각각 수동 작업 | 변수만 바꿔서 재사용 |
| 변경 영향 파악 | 직접 해봐야 앎 | `plan`으로 사전 확인 |

{{< callout type="info" >}}
**핵심 개념**: Terraform은 "내가 원하는 인프라 상태"를 코드로 선언합니다. 그러면 Terraform이 현재 상태와 비교해서 필요한 변경만 자동으로 적용합니다.
{{< /callout >}}

---

## Terraform이 해결하는 실무 문제

### 1. 신규 환경 빠른 복제

신규 프로젝트가 시작될 때마다 네트워크, 서버, 데이터베이스를 콘솔에서 하나씩 클릭해 만들 필요가 없습니다.

```hcl
# 이 코드 하나로 전체 환경이 생성됩니다
module "new_service" {
  source      = "./modules/service"
  environment = "staging"
  region      = "ap-northeast-2"
  service_name = "payment-api"
}
```

### 2. 운영 환경 표준화

모든 팀이 같은 모듈을 쓰면, 보안 설정이나 네이밍 규칙이 자동으로 통일됩니다.

### 3. 인프라 변경 실수 감소

`terraform plan`은 apply 전에 무엇이 생성/변경/삭제되는지 미리 보여줍니다.

```
Plan: 2 to add, 1 to change, 0 to destroy.
```

이 한 줄을 보고 예상과 다르면 취소하면 됩니다. 콘솔에서 클릭하면 이미 늦습니다.

### 4. 팀 협업 시 가시성 확보

코드 리뷰 = 인프라 리뷰입니다. 시니어 엔지니어가 변경 사항을 Pull Request에서 검토하고 승인하는 구조가 만들어집니다.

---

## 현업에서 바로 느끼는 차이

### 사례 1: 재해 복구 시간

- **Before**: 운영 환경 장애 → 수동으로 새 환경 구축 → 4시간 소요
- **After**: `terraform apply` → 새 환경 생성 완료 → 15분 소요

### 사례 2: 신규 개발자 온보딩

- **Before**: 로컬 환경 설정 문서 보고 콘솔에서 하나씩 → 반나절
- **After**: `terraform apply -var="env=dev"` → 10분 완료

### 사례 3: 비용 절감

- **Before**: 사용하지 않는 dev 리소스가 운영 중인지 파악 불가
- **After**: 코드를 보면 어떤 환경에 무엇이 있는지 한눈에 확인, `terraform destroy`로 정리

{{< callout type="warning" >}}
**주의**: Terraform은 만능이 아닙니다. 배포 파이프라인(Kubernetes 앱 배포), 구성 관리(Ansible, Chef), 서비스 메시(Istio) 설정은 Terraform의 영역이 아닙니다. **인프라 프로비저닝**이 Terraform의 핵심 역할입니다.
{{< /callout >}}

---

## 다음 단계

이제 이유를 알았으니, 직접 만들어봅시다. → [15분 만에 첫 인프라 배포하기](quick-start)
