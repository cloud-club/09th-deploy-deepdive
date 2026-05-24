# Week 3 구현 결과

## 1. 프로젝트 구성

```text
src/
├── app.ts
├── config.ts
├── jobs/
│   ├── DDos.job/job.ts
│   ├── brute-force.job/job.ts
│   ├── mail-notification.job/job.ts
│   ├── server-error.job/job.ts
│   └── web-error.job/job.ts
└── utils/
    ├── elastic-query.client.ts
    └── elastic-user.client.ts
```

## 2. 구현 결과

### 2.1 Node.js + TypeScript 실행 환경 구성

Node.js + TypeScript 기반 프로젝트를 구성했다.

`package.json`에는 개발 실행 스크립트가 정의되어 있다.

```bash
npm run dev
```

실행 시 `tsx src/app.ts`를 통해 TypeScript 진입 파일을 실행한다.

### 2.2 환경 변수 로딩 구현

`src/app.ts`에서 `dotenv`를 사용해 `.env` 파일을 로딩하도록 구현했다.

```ts
import dotenv from 'dotenv';
dotenv.config();
```

현재 코드에서 사용하는 환경 변수는 다음과 같다.

| 환경 변수 | 용도 |
| --- | --- |
| `ELASTIC_USERNAME` | Elasticsearch Basic Auth 사용자명 |
| `ELASTIC_PASSWORD` | Elasticsearch Basic Auth 비밀번호 |

### 2.3 주기 실행 구조 구현

`src/app.ts`에서 `setInterval`을 사용해 잡을 반복 실행하는 구조를 구현했다.

```ts
setInterval(async () => {
    await bruteForceJob();
    await DDosJob();
    await serverErrorJob();
    await webErrorJob();
    await mailNotification();
}, jobsPollingMinutes);
```

폴링 주기는 `src/config.ts`에서 관리한다.

```ts
export const config = {
    jobsPollingMinutes: 1
};
```

### 2.4 Elasticsearch 로그 조회 클라이언트 구현

`src/utils/elastic-query.client.ts`에 Elasticsearch ES|QL API 호출 클라이언트를 구현했다.

구현된 동작은 다음과 같다.

- `ELASTIC_USERNAME`, `ELASTIC_PASSWORD` 환경 변수 조회
- 인증 정보 누락 시 에러 발생
- `https://api.gdgoc.net/_query` 엔드포인트 호출
- `iis-*` 인덱스 대상 최근 n분 로그 조회
- 응답 성공 시 `response.data.values` 반환
- 호출 실패 시 에러 로그 출력 후 빈 배열 반환

사용 쿼리는 다음과 같다.

```esql
FROM iis-*
| WHERE @timestamp > NOW() - ${minutes}m
| DROP *.keyword
| SORT @timestamp DESC
```

### 2.5 Elasticsearch 사용자 조회 클라이언트 구현

`src/utils/elastic-user.client.ts`에 Elasticsearch 사용자 조회 클라이언트를 구현했다.

구현된 동작은 다음과 같다.

- `ELASTIC_USERNAME`, `ELASTIC_PASSWORD` 환경 변수 조회
- 인증 정보 누락 시 에러 발생
- `https://api.gdgoc.net/_security/user` 엔드포인트 호출
- 응답 성공 시 사용자 정보 반환
- 호출 실패 시 에러 로그 출력 후 빈 배열 반환

### 3.6 탐지 및 알림 잡 파일 생성

탐지와 알림 처리를 위한 잡 파일을 생성하고 `src/app.ts` 실행 흐름에 연결했다.

| 파일 | 역할 |
| --- | --- |
| `src/jobs/brute-force.job/job.ts` | 무차별 대입 공격 탐지 잡 |
| `src/jobs/DDos.job/job.ts` | DDoS 의심 트래픽 탐지 잡 |
| `src/jobs/web-error.job/job.ts` | 4xx 웹 서비스 에러 탐지 잡 |
| `src/jobs/server-error.job/job.ts` | 5xx 서버 에러 탐지 잡 |
| `src/jobs/mail-notification.job/job.ts` | 이메일 알림 잡 |

### 2.7 에러 로그 형식 구현

Elasticsearch API 호출 실패 시 다음 형식으로 에러 로그를 출력하도록 구현했다.

```text
[ISO timestamp] [filename] [ERROR] - message
```
