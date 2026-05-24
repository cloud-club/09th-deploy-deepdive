# Daily Quote Agent 스터디 과제 정리

## 1. 프로젝트 개요

`daily-quote-agent`는 매일 사용할 수 있는 한국어 명언 생성 AI Agent를 만드는 프로젝트다.

초기 목표는 단순히 LLM으로 명언을 생성하는 것이었지만, 학습 목적상 AWS 기반 운영 흐름까지 경험할 수 있도록 다음 요소를 포함했다.

- FastAPI 기반 HTTP API
- Amazon Bedrock Runtime 기반 LLM 호출
- DynamoDB 기반 명언 생성 이력 저장
- 중복 명언 방지 로직
- Docker 컨테이너화
- ECR 이미지 push
- 이후 ECS EC2유형, EventBridge Scheduler, CloudWatch, GitHub Actions로 확장 예정

## 2. 사용 기술 스택

| 영역 | 기술 |
| --- | --- |
| Language | Python |
| API Server | FastAPI |
| LLM | Amazon Bedrock Runtime |
| Model | Anthropic Claude 3 Haiku |
| Database | Amazon DynamoDB |
| Container | Docker |
| Image Registry | Amazon ECR |
| 예정 배포 환경 | Amazon ECS EC2유형 |
| 예정 스케줄러 | EventBridge Scheduler |
| 예정 로그 관리 | CloudWatch Logs |
| 예정 CI/CD | GitHub Actions |

## 3. 현재 아키텍처

```text
Client / curl
  -> FastAPI
  -> QuoteAgent
  -> Amazon Bedrock Runtime
  -> Claude 3 Haiku
  -> QuoteHistoryStore
  -> DynamoDB
```

현재 API는 명언을 생성한 뒤 DynamoDB에 저장하고, 사용자 이메일 기준으로 최근 생성 이력을 조회할 수 있다.

## 4. 설계 원칙

이번 프로젝트를 진행하면서 Agent 개발에서 중요한 기준을 하나 정했다.

```text
확실해야 하는 것은 코드와 DB로 처리한다.
창의적인 것은 AI에게 맡긴다.
```

예를 들어 다음 기능은 코드와 DB가 담당한다.

- 중복 명언 검사
- 사용자별 발송/생성 이력 저장
- DynamoDB 저장 여부
- API 응답 형식
- 재시도 횟수

반면 다음 기능은 AI 모델이 담당한다.

- 명언 생성
- 짧은 해설 생성
- 오늘의 성찰 질문 생성
- 문장 톤 조정

## 5. 구현 완료 내역

### 5.1 프로젝트 초기화

- 프로젝트 이름을 `daily-quote-agent`로 정함
- Python/FastAPI 프로젝트 구조 생성
- 기본 테스트 환경 구성

주요 파일:

```text
app/
  config.py
  history_store.py
  main.py
  models.py
  quote_agent.py
tests/
  test_main.py
Dockerfile
requirements.txt
requirements-dev.txt
.env.example
```

### 5.2 FastAPI 서버 구현

구현한 endpoint:

```text
GET /health
GET /quote/today?user_email=...
GET /quote/recent?user_email=...&limit=5
```

`/health`는 서버 상태 확인용이다.

`/quote/today`는 사용자 이메일 기준으로 최근 명언 이력을 조회하고, Bedrock으로 새 명언을 생성한 뒤 DynamoDB에 저장한다.

`/quote/recent`는 DynamoDB에 저장된 최근 명언 이력을 조회한다.

### 5.3 Amazon Bedrock Runtime 연동

처음에는 OpenAI API를 고려했지만, AWS SAA 학습 및 회사 환경과 맞추기 위해 Amazon Bedrock Runtime으로 변경했다.

현재 기본 모델:

```text
anthropic.claude-3-haiku-20240307-v1:0
```

Bedrock 호출 방식:

```text
boto3.client("bedrock-runtime").converse(...)
```

### 5.4 DynamoDB 테이블 생성

명언 중복 방지를 위해 DynamoDB 테이블을 생성했다.

테이블 이름:

```text
daily-quote-agent-delivery-history
```

키 구조:

```text
Partition Key: user_email
Sort Key: sent_at
```

생성 명령어:

```bash
aws dynamodb create-table   --table-name daily-quote-agent-delivery-history   --attribute-definitions     AttributeName=user_email,AttributeType=S     AttributeName=sent_at,AttributeType=S   --key-schema     AttributeName=user_email,KeyType=HASH     AttributeName=sent_at,KeyType=RANGE   --billing-mode PROVISIONED   --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5   --region ap-northeast-2   --no-cli-pager
```

### 5.5 중복 명언 방지 로직

현재 중복 방지 흐름:

```text
1. user_email 기준으로 최근 명언 이력 조회
2. 최근 명언의 quote_hash 목록 생성
3. Bedrock 프롬프트에 최근 명언 목록 전달
4. 새 명언 생성
5. 생성된 명언의 hash 계산
6. 기존 hash와 중복되면 재생성
7. 중복이 아니면 DynamoDB에 저장
```

기본 재시도 횟수:

```text
QUOTE_GENERATION_ATTEMPTS=3
```

### 5.6 Docker화

Dockerfile을 작성하고 로컬 컨테이너 실행을 검증했다.

이미지 빌드:

```bash
docker build -t daily-quote-agent:local .
```

컨테이너 실행:

```bash
docker run -d   --name daily-quote-agent-local   -p 8001:8000   -e AWS_REGION=ap-northeast-2   -e AWS_DEFAULT_REGION=ap-northeast-2   -e BEDROCK_MODEL_ID=anthropic.claude-3-haiku-20240307-v1:0   -e DYNAMODB_TABLE_NAME=daily-quote-agent-delivery-history   -v /Users/yrji/.aws:/root/.aws:ro   daily-quote-agent:local
```

컨테이너에서 검증한 항목:

- `/health` 호출 성공
- Bedrock 명언 생성 성공
- DynamoDB 저장 성공
- DynamoDB 최근 이력 조회 성공

### 5.7 ECR 이미지 Push

## 6. 현재 호출 방법

로컬 FastAPI 서버 기준:

```bash
curl 'http://127.0.0.1:8000/health'
```

```bash
curl 'http://127.0.0.1:8000/quote/today?user_email=yrji@example.com'
```

```bash
curl 'http://127.0.0.1:8000/quote/recent?user_email=yrji@example.com&limit=5'
```

Docker 컨테이너 기준:

```bash
curl 'http://127.0.0.1:8001/health'
```

```bash
curl 'http://127.0.0.1:8001/quote/today?user_email=docker-test@example.com'
```

```bash
curl 'http://127.0.0.1:8001/quote/recent?user_email=docker-test@example.com&limit=5'
```

## 7. 테스트

테스트 실행:

```bash
pytest
```

현재 테스트 항목:

- `/health` 응답 확인
- `/quote/today`에서 중복 명언이면 재생성 후 저장하는지 확인
- `/quote/recent`에서 최근 이력을 반환하는지 확인

마지막 확인 결과:

```text
3 passed
```

## 8. 현재 완료 체크리스트

- [x] 프로젝트명 `daily-quote-agent`로 변경
- [x] FastAPI 서버 생성
- [x] `/health` API 생성
- [x] `/quote/today` API 생성
- [x] `/quote/recent` API 생성
- [x] OpenAI API 제거
- [x] Amazon Bedrock Runtime 연동
- [x] DynamoDB 테이블 생성
- [x] DynamoDB 연동
- [x] 생성한 명언을 발송 이력 테이블에 저장
- [x] 최근 명언 조회
- [x] 중복 명언 방지 로직 추가
- [x] 기본 테스트 작성
- [x] 로컬 Bedrock 호출 성공
- [x] 로컬 DynamoDB 저장/조회 성공
- [x] Docker 이미지 빌드 성공
- [x] Docker 컨테이너 실행 성공
- [x] 컨테이너 안에서 Bedrock 호출 성공
- [x] 컨테이너 안에서 DynamoDB 저장/조회 성공
- [x] 기존 ECR repository 사용
- [x] Docker image ECR tag 생성 성공
- [x] Docker image ECR push 성공

## 9. 앞으로 할 일

### 9.1 ECS EC2유형 배포

- ECS Cluster 생성
- Task Definition 작성
- 컨테이너 이미지로 ECR 이미지 사용
- Task Role에 Bedrock/DynamoDB 권한 부여
- FastAPI 앱을 EC2유형 Service로 실행
- 보안 그룹과 네트워크 설정
- 필요 시 ALB 연결

사용할 이미지:

```text
104871657422.dkr.ecr.ap-northeast-2.amazonaws.com/se24145/dev:daily-quote-0.1
```

### 9.2 EventBridge Scheduler

- 매일 정해진 시간에 Agent 실행
- 현재는 API 호출 방식 또는 ECS task 실행 방식 중 선택 필요
- 스케줄 실행 결과를 CloudWatch에서 확인

### 9.3 CloudWatch Logs

- ECS task log 확인
- API 요청 로그 확인
- Bedrock/DynamoDB 에러 로그 확인

### 9.4 GitHub Actions CI/CD

- push 시 pytest 실행
- Docker image build
- ECR login
- Docker image push
- ECS service update

### 9.5 SES 이메일 발송

SES는 CI/CD와 배포 흐름이 정리된 뒤 고도화 기능으로 추가한다.

- SES 발신 이메일 인증
- SES 수신 이메일 인증 또는 sandbox 해제
- `/quote/send-email` API 추가
- 명언 생성 후 이메일 전송
- 전송 성공 후 DynamoDB에 발송 이력 저장

---
## Week4 할 일
- LangGraph 로 Agent가 해야 할 여러 단계를 관리
- LangChain 적용시켜보기
