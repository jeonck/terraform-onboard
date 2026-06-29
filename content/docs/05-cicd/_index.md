---
title: "5단계: CI/CD 자동화"
weight: 5
next: /docs/05-cicd/github-actions
---

## 배포 자동화: Terraform을 CI/CD에 붙여서 운영 흐름 만들기

실무에서는 **로컬 apply보다 자동화 파이프라인에서 안전하게 실행하는 구조**가 중요합니다. 사람이 직접 apply하는 방식은 실수와 감사 추적 어려움을 낳습니다.

```mermaid
flowchart LR
    A[코드 작성] --> B[PR 생성]
    B --> C[fmt / validate]
    C --> D[tflint]
    D --> E[보안 스캔]
    E --> F[terraform plan]
    F --> G[PR 리뷰]
    G --> H{승인?}
    H -->|Yes| I[merge]
    H -->|No| A
    I --> J[terraform apply]
    J --> K[알림 발송]

    style A fill:#e0f2fe
    style H fill:#fef3c7
    style J fill:#dcfce7
    style K fill:#f0fdf4
```

## 이 단계의 학습 목표

- Terraform을 GitHub Actions에 연결할 수 있다
- PR마다 plan을 자동으로 실행하고 리뷰할 수 있다
- 승인 기반 apply 흐름을 구성할 수 있다
- 비밀정보를 안전하게 파이프라인에 주입할 수 있다

## 핵심 주제

| 주제 | 내용 |
|------|------|
| [GitHub Actions 기초](github-actions) | PR 시 plan, merge 후 apply 워크플로우 |
| [파이프라인 표준 단계](pipeline-stages) | fmt → validate → lint → plan → scan → apply |
| [승인 기반 배포](approval-deploy) | GitHub Environments 활용 approval gate |
| [비밀정보 관리](secret-management) | OIDC, Vault, Secrets Manager 연계 |

## 이 단계의 산출물

- Terraform CI/CD 파이프라인 구축 완료
- PR마다 plan 결과가 자동으로 코멘트로 등록됨
- prod 환경은 승인 없이 apply 불가
- 비밀정보가 코드에 노출되지 않는 구조
