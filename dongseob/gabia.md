# 가비아 클라우드와 함께 만든 AIOps 배포 플랫폼 회고

> Cloud Club 9기 Deploy Deep Dive 스터디에서 가비아 클라우드의 크레딧 지원을 받아, 실제 클라우드 인프라 위에 AIOps 배포 플랫폼을 구축했다. 이 글은 단순한 기술 구축기가 아니라, 클라우드를 직접 만져보며 배운 것들과 그 과정에서 느낀 Cloud Club, 가비아 클라우드의 가치를 정리한 회고다.

<img width="5016" height="3839" alt="SA_Feedback_week5" src="https://github.com/user-attachments/assets/3eb44144-19b6-4630-a497-15d7d4bb0692" />


## 클라우드는 결국 직접 써봐야 알게 된다

클라우드 공부를 하다 보면 자주 듣는 말이 있다.

VPC를 나누고, Public Subnet과 Private Subnet을 분리하고, Bastion을 두고, ALB 뒤에 서버를 연결하고, 모니터링을 붙이고, CI/CD로 배포한다.

글로 읽으면 익숙하다.
강의로 들으면 이해한 것 같다.
그런데 막상 직접 만들기 시작하면 완전히 다른 이야기가 된다.

어떤 subnet에 어떤 VM을 둘지, ALB가 어떤 source IP로 들어오는지, 보안그룹을 어디까지 열어야 하는지, 도메인을 어떻게 연결해야 하는지, 컨테이너 이미지는 어디에 올리고 서버에서는 어디서 pull해야 하는지까지 전부 직접 결정해야 한다.

이번 Cloud Club 9기 Deploy Deep Dive 스터디에서 나는 이 과정을 가비아 클라우드 위에서 직접 경험했다.
그리고 이 경험이 가능했던 가장 큰 이유는 가비아 클라우드의 전폭적인 크레딧 지원이었다.

개인 비용으로는 쉽게 시도하기 어려운 VM 여러 대, ALB, Container Registry, 데이터 디스크, 도메인 연결, 모니터링 서버까지 마음껏 구성해볼 수 있었다.
덕분에 “아키텍처를 그려봤다”가 아니라 “아키텍처를 실제로 띄워봤다”에 가까운 경험을 할 수 있었다.

## Cloud Club에서 했던 것

Cloud Club 9기 Deploy Deep Dive 스터디에서 내가 진행한 프로젝트는 AIOps 배포 플랫폼이다.

단순히 프론트엔드와 백엔드를 서버에 올리는 데서 끝내지 않고, 배포와 운영에 필요한 흐름을 최대한 실제 서비스처럼 구성해보고 싶었다.

구축한 핵심 흐름은 다음과 같다.

```text
개발자
  -> GitLab Repository
  -> GitLab CI/CD
  -> 가비아 클라우드 Container Registry
  -> FE / BE Blue-Green 배포
  -> Prometheus / Grafana / Loki 모니터링
  -> AI Agent 장애 분석
  -> Discord 알림
```

처음에는 막연히 “배포 플랫폼을 만들어보자” 정도였는데, 실제로 구현하다 보니 자연스럽게 운영에 필요한 질문들이 생겼다.

- 서버는 어디에 둘 것인가?
- 외부 사용자는 어떤 경로로 들어올 것인가?
- 백엔드는 어떻게 무중단에 가깝게 교체할 것인가?
- DB는 어떻게 분리할 것인가?
- 로그와 메트릭은 어디로 모을 것인가?
- 장애가 나면 누가, 어떻게 알 수 있을 것인가?
- AI Agent는 장애 상황에서 어떤 정보를 보고 판단할 것인가?

이 질문에 하나씩 답하면서 프로젝트가 단순 배포 실습에서 운영형 아키텍처 실습으로 커졌다.

## 왜 가비아 클라우드였나

이번 프로젝트에서 사용한 클라우드는 가비아 클라우드다.

가비아 클라우드는 국내 환경에서 VM, VPC, Subnet, Public IP, ALB, Container Registry, DNS를 직접 구성해볼 수 있는 클라우드였다.
특히 이번 프로젝트처럼 “내가 직접 네트워크를 나누고, 서버를 만들고, 보안그룹을 설계하고, 컨테이너를 배포해보고 싶다”는 목적에는 잘 맞았다.

관리형 서비스 몇 개를 클릭해서 빠르게 배포하는 경험도 좋지만, 인프라를 공부하는 입장에서는 직접 VM을 만들고 트래픽을 열고 닫아보는 경험이 훨씬 크게 남는다.
가비아 클라우드에서는 이 과정을 한 프로젝트 안에서 이어서 해볼 수 있었다.

이번에 내가 실제로 사용해본 가비아 클라우드 기능은 다음과 같다.

- VPC 및 Subnet 구성
- VPC Router와 외부망 연결
- VM 생성 및 Ubuntu 22.04 LTS 서버 구축
- Public IP 할당
- Managed Load Balancer 구성
- Security Group 기반 접근 제어
- Container Registry 생성 및 이미지 push/pull
- 데이터 디스크 사용
- 가비아에서 구매한 도메인 DNS 연결

특히 좋았던 점은 도메인까지 가비아에서 구매해서 실제 서비스처럼 연결해볼 수 있었다는 것이다.
서버 IP로만 확인하는 것과 직접 구매한 도메인으로 접속하는 것은 체감이 꽤 다르다.

당시 DNS는 아래처럼 구성했다.

```text
dogsub.cloud      -> FE ALB
www.dogsub.cloud  -> FE ALB
api.dogsub.cloud  -> BE ALB
```

지금은 서버를 내려둔 상태지만, 구축 당시에는 실제 도메인으로 FE 페이지와 BE API가 모두 동작하는 것을 확인했다.

<img width="1552" height="987" alt="FE_BE_DB-연결완료1" src="https://github.com/user-attachments/assets/182e3e01-c713-464f-af90-075403596ff6" />
<img width="1552" height="987" alt="FE_BE_DB-연결완료2" src="https://github.com/user-attachments/assets/bb927e38-7ebe-40db-9ad6-cec0c04ce0a6" />



## 가비아 클라우드 지원이 특히 좋았던 이유

이런 프로젝트는 머릿속으로만 설계하면 그럴듯하게 끝난다.
문제는 직접 해보려면 리소스가 꽤 필요하다는 점이다.

프론트엔드 VM 하나, 백엔드 Blue/Green VM 두 개, DB VM, Monitoring VM, Agent VM, GitLab Runner VM, Bastion까지 만들다 보면 개인 실습으로는 부담이 생긴다.
게다가 ALB, Public IP, Container Registry, 데이터 디스크까지 붙이면 비용을 의식하지 않을 수 없다.

이번 Cloud Club 9기에서는 가비아 클라우드가 크레딧을 지원해주었고, 덕분에 비용 걱정보다 설계와 구현에 집중할 수 있었다.

나에게 이 지원은 단순히 “무료로 서버를 썼다”는 의미가 아니었다.

실패해도 다시 만들 수 있었고, 설계가 틀리면 지우고 다시 구성할 수 있었다.
라우팅을 바꿔보고, 보안그룹을 좁혀보고, ALB 연결 서버를 바꿔보고, 컨테이너를 다시 pull해서 실행해보는 과정이 가능했다.

이게 정말 중요했다.
클라우드 공부는 한 번에 성공하는 것보다, 틀리고 고치면서 몸에 익히는 과정에서 훨씬 많이 배운다.

그리고 좋은 소식도 있었다.
가비아 클라우드는 이번 Cloud Club 9기뿐 아니라 다음 Cloud Club 10기에도 지원을 이어갈 예정이라고 한다.
클라우드를 제대로 배워보고 싶은 사람이라면, Cloud Club은 꽤 좋은 기회가 될 것 같다.
스터디 안에서 방향을 잡고, 실제 클라우드 리소스로 직접 구현해볼 수 있기 때문이다.

## 처음 구상한 아키텍처

처음 그리고 싶었던 구조는 표준적인 Public/Private Subnet 구조였다.

```text
Public Subnet
  -> Bastion
  -> NAT 역할
  -> 외부 진입점 ALB

Private Ops Subnet
  -> FE
  -> BE Blue
  -> BE Green
  -> GitLab Runner
  -> Monitoring
  -> AI Agent

Private DB Subnet
  -> MySQL Primary / Replica
```

사용자 트래픽은 ALB를 통해서만 들어오고, 서버 관리는 Bastion을 통해서만 접근하고, Private Subnet의 서버들은 직접 외부에 노출하지 않는 구조를 만들고 싶었다.

처음에는 이 구조를 논리적으로만 설명해야 하는 부분도 있었다.
당시에는 하나의 VPC에 하나의 라우팅 테이블만 사용할 수 있는 제약이 있어, subnet별로 route table을 다르게 붙이는 표준 Public/Private 구조를 완전히 그대로 가져가기 어려웠다.

그래서 초반에는 Security Group과 Bastion을 중심으로 private-like 접근 통제를 구현했다.

그런데 이후 가비아 클라우드에 다수의 라우팅 테이블을 생성할 수 있는 업데이트가 들어오면서 상황이 바뀌었다.
이제 subnet별 라우팅 정책을 다르게 가져갈 수 있게 되었고, 내가 처음 그리고 싶었던 Public/Private Subnet 구조를 더 실제에 가깝게 구성할 수 있게 되었다.

이 부분이 개인적으로 꽤 인상 깊었다.
클라우드 서비스가 업데이트되면서 사용자가 구성할 수 있는 아키텍처의 폭이 넓어지는 것을 직접 체감했기 때문이다.

## 실제로 만든 흐름

최종적으로는 아래와 같은 흐름을 만들었다.

사용자가 서비스에 접속하면 먼저 FE ALB로 들어온다.

```text
User
  -> dogsub.cloud
  -> DNS
  -> FE ALB
  -> FE VM:80
  -> nginx
  -> Vite로 빌드된 React 페이지
```

사용자가 페이지에서 기능을 누르면 브라우저 안의 React 앱이 BE API를 호출한다.

```text
User Browser
  -> api.dogsub.cloud
  -> DNS
  -> BE ALB
  -> Active BE:8080
  -> MySQL Primary:3306
```

백엔드는 Blue/Green 구조로 두었다.

```text
BE ALB
  -> BE Blue:8080
  -> BE Green:8080
```

새 버전을 배포할 때는 현재 active가 아닌 standby 서버에 먼저 올린다.
health check가 성공하면 ALB의 연결 서버 weight를 바꿔 트래픽을 전환한다.
이전 active 서버는 바로 지우지 않고 rollback 대상으로 남긴다.

이 과정을 통해 “무중단 배포”라는 말을 단순히 개념으로만 이해하는 것이 아니라, 실제로 어떤 서버에 먼저 배포하고, 어디에서 health check를 하고, 어디에서 트래픽을 바꾸는지까지 구체적으로 이해할 수 있었다.

<img width="5016" height="3839" alt="SA_Feedback_week5" src="https://github.com/user-attachments/assets/21c3052e-69d6-48db-acf4-f1a9214d6d18" />


## Container Registry까지 써본 경험

이번 프로젝트에서는 이미지 저장소도 가비아 클라우드 Container Registry를 사용했다.

로컬에서 이미지를 빌드하고, 가비아 클라우드 Container Registry의 public URI로 push했다.
이후 FE/BE VM에서는 private URI로 이미지를 pull해서 컨테이너를 실행했다.

```text
Local PC
  -> docker buildx build
  -> Gabia Cloud Container Registry push

FE / BE VM
  -> docker pull
  -> docker run
```

이 과정을 해보면서 “Docker 이미지를 만든다”와 “운영 서버가 이미지를 받아 실행한다” 사이에 Registry가 어떤 역할을 하는지 명확히 이해할 수 있었다.

CI/CD까지 연결하면 최종적으로는 아래 흐름이 된다.

```text
GitLab CI/CD
  -> Docker image build
  -> Gabia Cloud Container Registry push
  -> Runner가 FE/BE VM에 SSH 접속
  -> VM에서 image pull
  -> docker run
```

## 모니터링을 붙이면서 프로젝트가 진짜 운영처럼 보이기 시작했다

처음에는 FE, BE, DB가 연결되는 것만으로도 충분히 뿌듯했다.
브라우저에서 페이지가 뜨고, 게시글을 만들면 DB에 저장되고, API health check가 정상으로 나오는 것만으로도 꽤 큰 단계였다.

그런데 모니터링을 붙이자 프로젝트의 느낌이 달라졌다.

Prometheus가 BE Blue/Green의 `/actuator/prometheus`를 scrape하고, Grafana에서 request rate, error rate, JVM heap, process CPU를 볼 수 있게 되었다.

```text
Prometheus
  -> BE Blue /actuator/prometheus
  -> BE Green /actuator/prometheus

Grafana
  -> BE Overview
  -> JVM Detail
```

<img width="1552" height="987" alt="Prometheus-구축" src="https://github.com/user-attachments/assets/4783aac3-b7d2-4bd1-acae-50e03ee975ea" />
<img width="1552" height="987" alt="Grafana-BEOverview" src="https://github.com/user-attachments/assets/3ef8bb86-58cb-4ea2-b532-d34cac139653" />
<img width="1552" height="987" alt="Grafana-JVMDetail" src="https://github.com/user-attachments/assets/6d99949a-f45e-416e-a880-3a25b92ee34f" />


로그는 Vector를 통해 Loki로 보냈다.
FE nginx access log와 BE application log가 Grafana에서 함께 보이기 시작하니, 단순 배포 실습이 아니라 운영 플랫폼처럼 느껴졌다.

<img width="1552" height="987" alt="Grafana-AllApplicationLogs" src="https://github.com/user-attachments/assets/8898439c-fab4-4b96-9d4b-d2459172a328" />

Alert rule도 만들었다.

- Backend 5xx Error Detected
- Backend Instance Down
- JVM Heap High
- Loki Error Log Detected

<img width="1552" height="987" alt="Alert" src="https://github.com/user-attachments/assets/3c0d6736-66a4-460c-9540-3acfbdd0d8d0" />

## AI Agent와 Discord 알림

이번 프로젝트 이름에 AIOps를 붙인 이유는 단순히 모니터링까지만 하고 싶지 않았기 때문이다.

Prometheus Alertmanager에서 발생한 알림을 AI Agent로 전달하고, Agent가 Prometheus와 Loki에서 추가 정보를 조회한 뒤 Gemini를 통해 장애 상황을 요약하도록 구성했다.
분석 결과는 Discord로 전송했다.

```text
Prometheus Alert Rule
  -> Alertmanager
  -> AI Agent
  -> Prometheus / Loki context query
  -> Gemini analysis
  -> Discord Alert
```

추가로 Qwen 기반 제한적 초동 조치 Agent도 구상했다.
다만 운영 서버에서 자동 조치는 위험할 수 있기 때문에 command whitelist를 두고, 허용된 명령만 제한적으로 실행하는 방향으로 설계했다.

이 부분은 앞으로 더 고도화해보고 싶은 영역이다.
알림을 보내는 것을 넘어, 장애 상황을 요약하고, 사람이 확인해야 할 다음 행동을 제안하는 흐름까지 가면 훨씬 재미있는 AIOps 플랫폼이 될 수 있을 것 같다.

## Cloud Club이 좋았던 점

혼자였다면 여기까지 만들지 못했을 가능성이 높다.

Cloud Club 스터디에서는 매주 진행 상황을 공유하고, 아키텍처를 피드백받고, 막히는 부분을 정리하면서 프로젝트를 계속 앞으로 밀고 갈 수 있었다.
클라우드나 DevOps는 공부 범위가 넓어서 혼자 하면 중간에 방향을 잃기 쉬운데, 스터디라는 구조가 있으니 끝까지 이어갈 수 있었다.

특히 Deploy Deep Dive라는 이름처럼, 단순히 “배포해봤다”가 아니라 배포를 둘러싼 네트워크, 보안, CI/CD, 모니터링, 장애 대응까지 이어서 생각해볼 수 있었다.

Cloud Club을 한 문장으로 표현하면, 클라우드를 글로만 공부하던 사람이 실제 인프라를 만지며 성장할 수 있는 곳이었다.

그리고 이 경험을 가능하게 해준 것이 가비아 클라우드의 지원이었다.
Cloud Club과 가비아 클라우드의 조합은, 클라우드를 제대로 실습해보고 싶은 사람에게 꽤 매력적인 환경이라고 느꼈다.

## 이런 사람에게 추천하고 싶다

Cloud Club과 가비아 클라우드 실습 환경은 이런 사람에게 특히 좋을 것 같다.

- 클라우드를 공부했지만 아직 직접 VPC를 구성해본 적은 없는 사람
- 배포가 `docker run`에서 끝나는 것이 아쉽다고 느끼는 사람
- ALB, Bastion, Security Group, DNS를 실제로 연결해보고 싶은 사람
- GitLab CI/CD나 GitHub Actions를 실제 서버 배포와 이어보고 싶은 사람
- Prometheus, Grafana, Loki 같은 모니터링 스택을 직접 붙여보고 싶은 사람
- 비용 부담 때문에 클라우드 실습을 망설였던 사람

나도 처음에는 이런 것들을 각각 따로 알고 있었다.
하지만 이번 프로젝트를 통해 하나의 흐름으로 연결해볼 수 있었다.

```text
도메인
  -> ALB
  -> FE / BE
  -> DB
  -> CI/CD
  -> Registry
  -> Monitoring
  -> Alert
  -> AI Agent
```

이 연결 경험이 가장 큰 수확이었다.

## 마무리

이번 AIOps 배포 플랫폼 프로젝트는 가비아 클라우드의 지원이 있었기에 가능했다.

Cloud Club 9기 활동에서 제공받은 크레딧 덕분에 실제 VM을 만들고, VPC와 Subnet을 나누고, ALB와 Container Registry를 사용하고, 도메인까지 연결해보며 클라우드 인프라를 몸으로 익힐 수 있었다.

가비아 클라우드는 국내 환경에서 클라우드 인프라를 직접 실습해보고 싶은 사람에게 좋은 선택지가 될 수 있다고 느꼈다.
특히 VM 기반으로 네트워크와 보안그룹을 직접 설계해보고 싶은 사람이라면 얻어갈 것이 많다.

그리고 Cloud Club은 그런 실습을 혼자가 아니라 함께 이어갈 수 있는 환경이었다.
다음 Cloud Club 10기에도 가비아 클라우드의 지원이 이어진다고 하니, 클라우드와 배포를 제대로 경험해보고 싶은 사람이라면 관심을 가져봐도 좋을 것 같다.

나에게 이번 프로젝트는 단순히 “서버를 띄워봤다”가 아니라, 클라우드 위에서 서비스가 운영되는 전체 흐름을 처음부터 끝까지 연결해본 경험이었다.
앞으로는 GitLab CI/CD의 Blue/Green 전환 자동화, ALB 전환 API 연동, Agent의 조치 범위 고도화까지 이어가며 더 실제 운영에 가까운 AIOps 플랫폼으로 발전시켜보고 싶다.
