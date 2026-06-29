---
title: 가이드
weight: 1
---

## Terraform 실무 온보딩 로드맵

이 가이드는 "문법 설명" 중심이 아닌, **현업에서 바로 쓰는 흐름 중심**으로 설계되었습니다.

```mermaid
flowchart LR
    A[1단계\n빠른 입문] --> B[2단계\n실무 기본기]
    B --> C[3단계\n재사용 코드]
    C --> D[4단계\n팀 협업]
    D --> E[5단계\nCI/CD 자동화]
    E --> F[6단계\n고급 설계]
    F --> G[7단계\n보안·거버넌스]
    G --> H[8단계\n운영 심화]

    style A fill:#10b981,color:#fff
    style B fill:#3b82f6,color:#fff
    style C fill:#8b5cf6,color:#fff
    style D fill:#f59e0b,color:#fff
    style E fill:#ef4444,color:#fff
    style F fill:#06b6d4,color:#fff
    style G fill:#ec4899,color:#fff
    style H fill:#6b7280,color:#fff
```

## 학습 단계

| 단계 | 주제 | 핵심 메시지 | 결과물 |
|------|------|------------|--------|
| 1단계 | [빠른 입문](01-intro) | 클릭 대신 코드로 인프라 관리 | 첫 리소스 생성 |
| 2단계 | [실무 기본기](02-basics) | plan/apply/state 이해가 사고 방지 | 단일 환경 배포 |
| 3단계 | [재사용 가능한 코드](03-reusable) | 변수·모듈·환경 분리 구조 | 팀 표준 템플릿 |
| 4단계 | [팀 협업](04-team) | Remote State·Import·Drift 대응 | 팀 운영 기본기 |
| 5단계 | [CI/CD 자동화](05-cicd) | 로컬 apply → 파이프라인 apply | 운영 자동화 체계 |
| 6단계 | [고급 설계](06-advanced) | 멀티 계정·State 분할·모듈 고도화 | 확장성 있는 구조 |
| 7단계 | [보안·거버넌스](07-security) | 정책 통제·보안 스캔·컴플라이언스 | 운영 통제 플랫폼 |
| 8단계 | [운영 심화](08-ops) | 실패 대응·State 복구·트러블슈팅 | 운영 안정성 확보 |
