# 이번주 한일

1. 개발 루프 SKILLS로 만들기
2. 쿠버네티스 환경 정리
3. AI 개발 파이프라인 Flow 정리

# AI 개발 파이프라인 Flow

## Skill 구성 파이프라인

```mermaid
flowchart LR
    A[중앙 Skill 저장소] --> B[Skill 구성 프롬프트]
    B --> C[프로젝트 구조 파악]
    C --> D[개발환경 요약<br/>Frontend / Backend / DB / Test]
    D --> E{애매한 부분 있음?}
    E -- 예 --> F[사용자에게 질문]
    F --> C
    E -- 아니오 --> G[프로젝트용 Dev Skill 작성]
    G --> H[AI 도구가 Skill 로드]
```

## 실제 개발 작업 루프

```mermaid
flowchart LR
    A[개발 요청] --> B[Skill 로드]
    B --> C[작업 유형 판별]
    C --> D[요구사항 정리]
    D --> E{명세가 애매함?}
    E -- 예 --> F[사용자에게 질문]
    F --> D
    E -- 아니오 --> G[PLAN 작성]
    G --> H[구현]
    H --> I[검증]
    I --> J{통과?}
    J -- 아니오 --> K[수정 후 재검증]
    K --> I
    J -- 예 --> L[문서 / Lessons 정리]
    L --> M[최종 보고]
```

## 전체 흐름 요약

```mermaid
flowchart LR
    A[Skill 저장소] --> B[프로젝트 분석]
    B --> C[프로젝트용 Dev Skill 작성]
    C --> D[Skill 기반 개발 수행]
    D --> E[구현]
    E --> F[검증]
    F --> G[문서 / Lessons 정리]
    G --> H[최종 보고]
```

# 쿠버네티스 환경 정리

## 아키텍처

- Excalidraw: [kubernetes-icon-flow-horizontal.excalidraw](./kubernetes-icon-flow-horizontal.excalidraw)
- PNG: [application-dependency-flow.png](./application-dependency-flow.png)
- 목적: 쿠버네티스 리소스 단위가 아니라 애플리케이션이 어떤 컴포넌트에 의존하는지 중심으로 정리

![Application Dependency Flow](./application-dependency-flow.png)


## 주요 구성

| 구성 | 역할 |
|---|---|
| Envoy Gateway | 외부 HTTP/S 트래픽 진입점 |
| myeverything | 프론트엔드와 백엔드 API를 포함한 핵심 애플리케이션 |
| PostgreSQL | 애플리케이션 영속 데이터 저장소 |
| Gemini API wrapper | AI 요청을 외부 AI provider로 중계 |
| Zot Registry | 컨테이너 이미지 저장소 |
| Kubernetes | 워크로드 실행과 스케줄링 담당 |

## 의존성 요약

- 외부 사용자는 `Envoy Gateway`를 통해 `myeverything`에 접근한다.
- `myeverything`의 백엔드 API는 `PostgreSQL`에 데이터를 읽고 쓴다.
- AI 기능은 `Gemini API wrapper`를 거쳐 외부 AI provider에 요청한다.
- 배포 이미지는 `Zot Registry`에서 가져온다.
- 쿠버네티스는 애플리케이션 워크로드를 실행하고 스케줄링한다.
