---
title: "7단계: 보안·거버넌스"
weight: 7
---

Terraform을 단순 배포 도구가 아닌 **운영 통제 플랫폼**으로 이해하는 단계입니다. 정책 통제, 보안 스캔, 컴플라이언스, 비용 관리를 배포 흐름에 내재화합니다.

```mermaid
graph TD
    A[코드 작성] --> B[정책 검사\nSentinel / OPA]
    B --> C{정책 통과?}
    C -->|실패| D[차단 & 피드백]
    C -->|통과| E[보안 스캔\ntfsec / checkov]
    E --> F{보안 이슈?}
    F -->|Critical| D
    F -->|통과| G[비용 추정\ninfracost]
    G --> H[PR 리뷰 & 승인]
    H --> I[Plan 실행]
    I --> J[Apply 실행]
    J --> K[컴플라이언스 감사 로그]

    style D fill:#ef4444,color:#fff
    style J fill:#10b981,color:#fff
    style K fill:#3b82f6,color:#fff
```

## 이 단계에서 배우는 것

| 주제 | 핵심 내용 |
|------|----------|
| [정책 기반 통제](policy-control) | Sentinel, OPA로 리소스 생성 규칙 강제화 |
| [보안 스캐닝](security-scan) | tfsec, checkov로 IaC 취약점 자동 탐지 |
| [컴플라이언스 대응](compliance) | 표준 태깅, 감사 로그, 변경 추적 |
| [비용 통제](cost-control) | infracost로 배포 전 비용 예측 및 통제 |

## 이 단계의 산출물

- Terraform을 단순 배포 도구가 아닌 운영 통제 플랫폼으로 이해
- 보안/비용/정책을 배포 흐름에 내재화
- 컴플라이언스 요건을 자동으로 검증하는 파이프라인 구축
