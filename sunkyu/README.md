# Web Log Monitoring SaaS

교내 학생 사이드 프로젝트를 대상으로 Web Log 기반 이상 징후 탐지와 알림을 제공하는 모니터링 SaaS 프로젝트이다.

## 프로젝트 소개

이 프로젝트는 자체 모니터링 환경을 갖추기 어려운 학생 개발자와 프로젝트 팀을 대상으로 한다. 학생이 운영하는 웹 서비스의 로그를 수집하고, 비정상 요청이나 서비스 오류 징후를 탐지해 운영자가 빠르게 확인할 수 있도록 돕는 것이 목표이다.

현재는 Elasticsearch 기반 로그 조회와 주기 실행형 탐지 서비스의 기반을 구성하는 단계이다.

## 현재 상태

| 항목 | 내용 |
| --- | --- |
| 현재 단계 | Week 3 구현 결과 정리 완료 |
| 구현 기준 프로젝트 | `/Users/luke/Projects/25-26-Web-Log-Monitoring-Study` |
| 주요 구현 범위 | Node.js + TypeScript 실행 기반, Elasticsearch 클라이언트, 주기 실행 구조, 탐지/알림 잡 파일 |
| 상세 설계 문서 | [Week 2 README](./week2/README.md) |
| 구현 결과 문서 | [Week 3 README](./week3/README.md) |

## Task Dashboard

### 완료된 Task

| Task | 결과 |
| --- | --- |
| Week 2 아키텍처 및 마일스톤 문서 작성 | [week2/README.md](./week2/README.md)에 정리 |
| Week 3 구현 결과 문서 작성 | [week3/README.md](./week3/README.md)에 정리 |
| Node.js + TypeScript 실행 환경 구성 | `npm run dev` 기반 실행 구조 구성 |
| 환경 변수 로딩 구조 구현 | `dotenv` 기반 `.env` 로딩 |
| 주기 실행 구조 구현 | `setInterval` 기반 잡 실행 흐름 구성 |
| Elasticsearch 로그 조회 클라이언트 구현 | ES\|QL API 호출 기반 로그 조회 |
| Elasticsearch 사용자 조회 클라이언트 구현 | 사용자 정보 조회 API 호출 구조 구성 |
| 탐지 및 알림 잡 파일 생성 | 탐지/알림 잡 파일을 실행 흐름에 연결 |

### 진행 중 Task

| Task | 상태 |
| --- | --- |
| 탐지 잡 내부 로직 구현 | 잡 파일 구조 생성 후 로직 구현 단계 |
| 이메일 알림 잡 구현 | 알림 잡 파일 생성 후 발송 로직 구현 단계 |
| 로그 응답 데이터 파싱 및 정규화 | Elasticsearch 응답을 탐지 입력 데이터로 변환하는 구조 정리 단계 |

### 다음 Task

| Task | 목적 |
| --- | --- |
| 예외 IP / 예외 Domain 처리 | 오탐 및 보호 대상 차단 방지 |
| 알림 메일 본문 생성 | 탐지 사유와 대상 정보를 운영자가 이해할 수 있도록 정리 |
| 중복 알림 방지 | 같은 이상 징후에 대한 과도한 반복 알림 방지 |
| 탐지 결과 저장 구조 설계 | 탐지 이력과 알림 이력 추적 기반 마련 |
| Kibana 대시보드 및 모니터링 시나리오 고도화 | 로그 조회와 운영 확인 흐름 개선 |

## 주차별 상세

| 주차 | 문서 | 내용 |
| --- | --- | --- |
| Week 2 | [week2/README.md](./week2/README.md) | 서비스 구조, 아키텍처, Reverse Proxy 제공 방식, 주차별 개발 계획 |
| Week 3 | [week3/README.md](./week3/README.md) | 현재 구현된 실행 기반, Elasticsearch 클라이언트, 잡 파일 구성 결과 |
