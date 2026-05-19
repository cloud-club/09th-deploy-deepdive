# Architecture

## 구성
- 프런트엔드 1개  
  - (선택) 값싼 도메인 구매/연결 고민 중
- CRUD 기반 백엔드 2개 (Blue/Green)

## AI 구성 (총 3개)
1) **Code Rabbit**
   - CI/CD 과정에서 코드 리뷰
   - 문제점 확인 및 수정 방향 가이드

2) **Gemini 또는 Codex + Vector**
   - 서버 내부 모니터링/로그 기반 경고 및 알람

3) **Qwen 또는 Ollama**
   - 에러 발생 시 제한된 범위의 초동 조치(가벼운 자동 대응)

---

# 주차 별 계획

## Week 2
- 대략적인 아키텍처 구상
- 주차 별 계획(마일스톤) 확정

## Week 3
- GitLab 구성
- 프런트/백엔드 프로젝트 생성
- 가비아 클라우드에 서버 인스턴스 구축

## Week 4
- CI/CD 전체 구축  
  - CD는 Blue/Green 방식으로 구성 예정
- 프런트-백엔드 API 연동 및 도메인 연결
- 모니터링 구축

## Week 5
- Code Rabbit 도입/연동
- 모니터링 영역에 AI(경고/알람) 추가

## Week 6
- 에러 시 제한된 초동 조치 AI 추가(Qwen 또는 Ollama)

## Week 7
- 전체 QA 진행
- 시간 남을 시 Kubernetes(k8s) 추가 검토
