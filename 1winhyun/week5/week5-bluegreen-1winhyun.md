# Blue/Green 무중단 배포 — CloudFront + ALB + ASG 위에 적용하기

```
            (배포 전)                         (배포 후)
CloudFront → ALB → [Blue ASG: v1]   →   CloudFront → ALB → [Green ASG: v2]
                                                              (Blue는 잠시 대기 후 종료)
```

> 핵심: **CloudFront는 건드릴 필요가 없다.** CloudFront의 오리진은 ALB의 DNS 이름이고, Blue/Green 전환은 그 ALB 뒤(타깃)에서 일어나므로 오리진 설정은 그대로다. (정적 자산 배포는 캐시 무효화만 별도로 신경 쓰면 된다.)

---

## 1. ASG + ALB에서 Blue/Green이 동작하는 메커니즘

ALB는 **대상 그룹(Target Group)** 에 등록된 인스턴스로만 트래픽을 보낸다. Blue/Green 전환은 결국 "대상 그룹에 어떤 인스턴스가 등록돼 있는가"를 바꾸는 일이다. 두 가지 구현 방식이 있다.

| 방식 | 트래픽 전환 단위 | 대상 그룹 | 전환 방법 |
|---|---|---|---|
| **A. AWS CodeDeploy** | 인스턴스 등록/해제 | **1개**로 충분 | Green 인스턴스를 대상 그룹에 등록 → Blue 인스턴스 해제 (자동) |
| **B. 수동/IaC + ALB 가중치** | 가중치(%) | **2개**(blue-tg / green-tg) | ALB 리스너의 **weighted forward**로 0→100% 점진 이동 |

- **방식 A (CodeDeploy)**: 운영에서 가장 표준적. CodeDeploy가 Green ASG를 **자동 복제 생성**하고, 앱 배포 → 대상 그룹 등록 → Blue 종료까지 오케스트레이션한다. 본 문서의 메인.
  - 단, EC2/On-Premises 플랫폼의 CodeDeploy Blue/Green은 트래픽을 **한 번에(all-at-once) 전환**한다. (Canary/Linear 같은 점진 트래픽 전환은 **ECS·Lambda 전용**이고 EC2에는 없다.)
- **방식 B (가중치)**: EC2에서도 진짜 Canary(예: 10% → 50% → 100%)를 하고 싶을 때. 대상 그룹 2개를 만들고 ALB 리스너 규칙에서 `forward`의 가중치를 조정한다. 직접 제어가 필요하면 선택.

> 정리: **"안전한 표준 자동화" → CodeDeploy(A)**, **"EC2에서 직접 Canary 비율 제어" → ALB 가중치(B)**.

---

## 2. 사전 준비물 (둘 다 공통)

1. **무상태(stateless) 앱**: 세션은 ElastiCache(Redis)에, 업로드 파일은 S3 등 외부에. (Blue/Green 전환 시 인스턴스가 통째로 교체되므로 로컬 상태는 사라진다.)
2. **헬스 체크 엔드포인트**: 앱 기동 + 의존성(예: DB 연결) 확인까지 포함한 `/actuator/health` 같은 경로. ALB 대상 그룹 헬스 체크가 이것을 본다.
3. **DB 스키마 하위 호환(Expand/Contract)**: 전환 순간 Blue(v1)와 Green(v2)이 **같은 RDS를 동시에** 바라본다. 두 버전이 함께 돌아도 깨지지 않게 마이그레이션을 설계해야 한다(→ 8장).
4. **CodeDeploy를 쓸 경우**:
   - EC2에 **CodeDeploy 에이전트** 설치(런치 템플릿 user data).
   - **IAM 역할 2개**: ① CodeDeploy 서비스 역할, ② EC2 인스턴스 프로파일(배포 번들 S3 읽기 등).

### 3.1 CodeDeploy 에이전트 설치 (Launch Template user data, Amazon Linux 2 기준)

```bash
#!/bin/bash
yum update -y
yum install -y ruby wget
cd /home/ec2-user
# 리전에 맞게 버킷 주소를 바꾼다 (예: ap-northeast-2)
REGION=ap-northeast-2
wget https://aws-codedeploy-${REGION}.s3.${REGION}.amazonaws.com/latest/install
chmod +x ./install
./install auto
systemctl enable codedeploy-agent
systemctl start codedeploy-agent
```

---

## 3. 방식 A — AWS CodeDeploy Blue/Green 구축

### 4.1 IAM 역할

```hcl
# CodeDeploy 서비스 역할 (CodeDeploy가 ASG/ELB를 조작할 수 있게)
resource "aws_iam_role" "codedeploy" {
  name = "myapp-codedeploy-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "codedeploy.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "codedeploy" {
  role       = aws_iam_role.codedeploy.name
  # 블루/그린은 ASG/ELB 제어가 필요하므로 AWSCodeDeployRole 사용
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole"
}

# EC2 인스턴스 프로파일: 배포 번들을 S3에서 받아오기 위한 읽기 권한 등
resource "aws_iam_role" "ec2" {
  name = "myapp-ec2-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ec2_s3" {
  role       = aws_iam_role.ec2.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

resource "aws_iam_instance_profile" "ec2" {
  name = "myapp-ec2-profile"
  role = aws_iam_role.ec2.name
}
```

### 3.2 CodeDeploy 애플리케이션 + 배포 그룹 (Terraform)

```hcl
resource "aws_codedeploy_app" "app" {
  name             = "myapp"
  compute_platform = "Server"   # EC2/On-Premises
}

resource "aws_codedeploy_deployment_group" "bg" {
  app_name              = aws_codedeploy_app.app.name
  deployment_group_name = "myapp-bluegreen"
  service_role_arn      = aws_iam_role.codedeploy.arn

  # EC2 블루/그린은 한 번에 전환(all-at-once). Canary/Linear는 EC2에 없음.
  deployment_config_name = "CodeDeployDefault.AllAtOnce"

  deployment_style {
    deployment_option = "WITH_TRAFFIC_CONTROL"   # ELB로 트래픽 제어
    deployment_type   = "BLUE_GREEN"
  }

  blue_green_deployment_config {
    # Green이 준비되면 자동으로 트래픽 전환 (수동 승인 원하면 STOP_DEPLOYMENT)
    deployment_ready_option {
      action_on_timeout = "CONTINUE_DEPLOYMENT"
    }
    # 기존 ASG를 복제해 Green 플릿을 자동 생성
    green_fleet_provisioning_option {
      action = "COPY_AUTO_SCALING_GROUP"
    }
    # 배포 성공 후 Blue를 종료하되, 롤백 대비로 잠시(5분) 살려둔다
    terminate_blue_instances_on_deployment_success {
      action                           = "TERMINATE"
      termination_wait_time_in_minutes = 5
    }
  }

  # 복제 대상이 되는 기존(Blue) ASG
  autoscaling_groups = [aws_autoscaling_group.app.name]

  # 트래픽을 제어할 ALB 대상 그룹 (1개면 충분)
  load_balancer_info {
    target_group_info {
      name = aws_lb_target_group.app.name
    }
  }

  # 실패하거나 알람이 울리면 자동 롤백
  auto_rollback_configuration {
    enabled = true
    events  = ["DEPLOYMENT_FAILURE", "DEPLOYMENT_STOP_ON_ALARM"]
  }

  alarm_configuration {
    enabled = true
    alarms  = [aws_cloudwatch_metric_alarm.alb_5xx.alarm_name]
  }
}
```

> 동작 순서: ① CodeDeploy가 Blue ASG를 복제해 **Green ASG 생성** → ② Green 인스턴스에 새 버전 설치(appspec 훅 실행) → ③ Green이 대상 그룹 헬스 체크 통과 → ④ **대상 그룹에 Green 등록 + Blue 해제(트래픽 전환)** → ⑤ 5분 대기 후 Blue ASG 종료. 실패/알람 시 ④에서 자동 롤백.

### 3.3 appspec.yml (배포 번들 루트에 위치)

CodeDeploy 에이전트가 이 파일을 읽고 **생명주기 훅**을 순서대로 실행한다.

```yaml
version: 0.0
os: linux

files:
  - source: /
    destination: /home/ec2-user/app   # 번들 내용을 인스턴스의 이 경로로 복사

# 파일 권한(선택)
permissions:
  - object: /home/ec2-user/app
    owner: ec2-user
    group: ec2-user
    mode: 755

hooks:
  # 기존 프로세스 정지 (Green은 새 인스턴스라 보통 no-op이지만 안전하게 둠)
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 60
      runas: ec2-user

  # 설치 후 환경 준비 (설정 주입 등)
  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 120
      runas: ec2-user

  # 앱 기동
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 120
      runas: ec2-user

  # 트래픽 받기 전 자체 검증 (실패하면 배포 실패 → 롤백)
  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 120
      runas: ec2-user
```

> Blue/Green에서 위 훅들은 **Green(신규) 인스턴스에서** 실행된다. `ValidateService`까지 통과해야 트래픽 전환(AllowTraffic) 단계로 넘어간다.

### 3.4 배포 스크립트 (`scripts/`)

`scripts/stop_server.sh`
```bash
#!/bin/bash
set -e
# 실행 중인 서비스가 있으면 정지 (없어도 에러 안 나게)
if systemctl is-active --quiet myapp; then
  sudo systemctl stop myapp
fi
```

`scripts/after_install.sh`
```bash
#!/bin/bash
set -e
APP_DIR=/home/ec2-user/app
# 예: systemd 유닛 배치 (최초 1회만 의미 있음)
sudo cp "${APP_DIR}/deploy/myapp.service" /etc/systemd/system/myapp.service
sudo systemctl daemon-reload
# 빌드 산출물 권한 정리
chmod +x "${APP_DIR}/app.jar" || true
```

`scripts/start_server.sh`
```bash
#!/bin/bash
set -e
sudo systemctl enable myapp
sudo systemctl restart myapp
```

`scripts/validate_service.sh`
```bash
#!/bin/bash
set -e
# 앱이 실제로 health를 200으로 응답할 때까지 최대 ~60초 폴링
for i in $(seq 1 12); do
  code=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/actuator/health || true)
  if [ "$code" = "200" ]; then
    echo "health OK"
    exit 0
  fi
  echo "waiting health... ($i) got=$code"
  sleep 5
done
echo "health check failed"
exit 1   # 0이 아니면 CodeDeploy가 배포 실패로 처리 → 자동 롤백
```

참고용 `deploy/myapp.service` (systemd 유닛)
```ini
[Unit]
Description=myapp
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user/app
ExecStart=/usr/bin/java -jar /home/ec2-user/app/app.jar
SuccessExitStatus=143
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3.5 배포 번들 구조

```
bundle/
├── appspec.yml
├── app.jar
├── deploy/
│   └── myapp.service
└── scripts/
    ├── stop_server.sh
    ├── after_install.sh
    ├── start_server.sh
    └── validate_service.sh
```

---

## 4. CI/CD 연결 (GitHub Actions 예시)

빌드 → 번들 zip → S3 업로드 → CodeDeploy 배포 트리거.

`.github/workflows/deploy.yml`
```yaml
name: deploy

on:
  push:
    branches: [ main ]

permissions:
  id-token: write   # OIDC로 AWS 자격증명 받기
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "17"

      - name: Build jar
        run: |
          ./gradlew clean bootJar
          cp build/libs/*.jar app.jar

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/github-actions-deploy
          aws-region: ap-northeast-2

      - name: Package & upload bundle to S3
        run: |
          zip -r bundle.zip appspec.yml app.jar deploy scripts
          aws s3 cp bundle.zip s3://my-deploy-bucket/myapp/${{ github.sha }}.zip

      - name: Create CodeDeploy deployment
        run: |
          aws deploy create-deployment \
            --application-name myapp \
            --deployment-group-name myapp-bluegreen \
            --s3-location bucket=my-deploy-bucket,key=myapp/${{ github.sha }}.zip,bundleType=zip \
            --description "deploy ${{ github.sha }}"
```

> CodePipeline을 쓴다면 Source(GitHub) → Build(CodeBuild) → Deploy(CodeDeploy) 스테이지로 같은 흐름을 GUI/IaC로 구성할 수 있다. 핵심은 동일: **번들을 만들어 CodeDeploy 배포를 트리거**.

---

## 5. Terraform 없이 구축하기 (AWS CLI / 콘솔)

앞의 3장에서 IAM 역할·CodeDeploy 앱·배포 그룹을 Terraform(`.hcl`)으로 보여줬지만, **Terraform은 필수가 아니다.** 같은 리소스를 **AWS 콘솔(클릭)** 이나 **AWS CLI(명령어)** 로 똑같이 만들 수 있다. 여기서는 CLI 기준으로 정리한다. (appspec.yml·배포 스크립트·GitHub Actions는 IaC 도구와 무관하게 그대로 필요하다.)

> 선택 기준: **처음 학습·일회성 구축이면 콘솔**, **명령어로 재현하고 싶으면 CLI**, **버전관리되는 운영 인프라면 IaC(Terraform 등)**. 결과물은 모두 동일하다.

### 5.1 IAM 역할 (CLI)

```bash
# (1) CodeDeploy 서비스 역할
aws iam create-role \
  --role-name myapp-codedeploy-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"codedeploy.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }'

aws iam attach-role-policy \
  --role-name myapp-codedeploy-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole

# (2) EC2 인스턴스 프로파일 (배포 번들 S3 읽기 등)
aws iam create-role \
  --role-name myapp-ec2-role \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }'

aws iam attach-role-policy \
  --role-name myapp-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam create-instance-profile --instance-profile-name myapp-ec2-profile
aws iam add-role-to-instance-profile \
  --instance-profile-name myapp-ec2-profile \
  --role-name myapp-ec2-role
```

> 만든 인스턴스 프로파일(`myapp-ec2-profile`)을 **Launch Template에 연결**해야 EC2가 S3에서 번들을 받을 수 있다.

### 5.2 CodeDeploy 애플리케이션 (CLI)

```bash
aws deploy create-application \
  --application-name myapp \
  --compute-platform Server   # EC2/On-Premises
```

### 5.3 배포 그룹 — Blue/Green (CLI)

3.2의 Terraform과 동일한 설정을 CLI로 만든 것이다.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws deploy create-deployment-group \
  --application-name myapp \
  --deployment-group-name myapp-bluegreen \
  --service-role-arn arn:aws:iam::${ACCOUNT_ID}:role/myapp-codedeploy-role \
  --deployment-config-name CodeDeployDefault.AllAtOnce \
  --auto-scaling-groups myapp-asg \
  --deployment-style deploymentType=BLUE_GREEN,deploymentOption=WITH_TRAFFIC_CONTROL \
  --blue-green-deployment-configuration '{
    "deploymentReadyOption": {"actionOnTimeout":"CONTINUE_DEPLOYMENT"},
    "greenFleetProvisioningOption": {"action":"COPY_AUTO_SCALING_GROUP"},
    "terminateBlueInstancesOnDeploymentSuccess": {"action":"TERMINATE","terminationWaitTimeInMinutes":5}
  }' \
  --load-balancer-info '{
    "targetGroupInfoList":[{"name":"myapp-tg"}]
  }' \
  --auto-rollback-configuration enabled=true,events=DEPLOYMENT_FAILURE,DEPLOYMENT_STOP_ON_ALARM
```

- `--auto-scaling-groups` : 복제 대상이 되는 기존(Blue) ASG 이름.
- `--load-balancer-info` : 트래픽을 제어할 ALB 대상 그룹 (1개면 충분).
- 나머지 옵션의 의미는 3.2 Terraform 설명과 동일하다.

### 5.4 콘솔(GUI)로 할 경우 — 클릭 순서 요약

1. **IAM** → 역할 생성 → 신뢰 주체 `CodeDeploy` → `AWSCodeDeployRole` 정책 연결. EC2용 역할도 동일하게 생성.
2. **CodeDeploy** → Applications → *Create application* → Compute platform: **EC2/On-Premises**.
3. 해당 앱에서 *Create deployment group* →
   - Deployment type: **Blue/green**
   - Environment configuration: **Automatically copy Auto Scaling group** → 기존 ASG 선택
   - Load balancer: **Enable** → 대상 그룹 선택
   - Deployment settings: `CodeDeployDefault.AllAtOnce`
   - Rollbacks: **Roll back when a deployment fails / alarm** 체크
4. 배포는 동일하게 *Create deployment* (또는 4장 GitHub Actions의 `aws deploy create-deployment`).

> 결론: **콘솔이든 CLI든 Terraform이든 만들어지는 CodeDeploy 리소스는 같다.** 배포를 실제로 굴리는 4장(번들·appspec·GitHub Actions)은 어느 방식을 택하든 그대로 쓰인다.

---

## 6. 롤백

- **자동 롤백**: `auto_rollback_configuration`(콘솔/CLI에서는 `--auto-rollback-configuration`)에 걸어둔 이벤트(`DEPLOYMENT_FAILURE`, 알람 발생)로 자동 롤백된다.
- **전환 후 롤백 여유**: 트래픽 전환 후에도 Blue를 `termination_wait_time_in_minutes`만큼 살려두므로, 그 사이 문제 발견 시 빠르게 되돌릴 수 있다.
- **수동 롤백**: 직전 성공 리비전을 다시 배포(`aws deploy create-deployment`)하면 된다.
- **공통**: **연결 드레이닝(deregistration delay)** 으로 전환 중 처리 중이던 요청이 끊기지 않게 한다(대상 그룹 속성, 기본 300초 — 앱에 맞게 조정).

---
---

## 8. 전체 흐름 요약

1. 코드 push → GitHub Actions가 jar 빌드 → 번들 zip → S3 업로드 → CodeDeploy 배포 생성.
2. CodeDeploy가 Blue ASG를 복제해 **Green ASG** 생성.
3. Green 인스턴스에서 appspec 훅 실행(설치 → 기동 → `ValidateService`).
4. Green이 ALB 대상 그룹 헬스 통과 → **트래픽 Blue→Green 전환(all-at-once)**.
5. 5분 대기 후 Blue 종료. 실패·알람 시 **자동 롤백**.
6. (스키마 변경이 있었다면) v1 소멸 후 Contract 마이그레이션으로 마무리.

---
