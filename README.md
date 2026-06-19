# AWS CI/CD 실습 가이드

이 저장소는 AWS의 개발자 도구(CodePipeline, CodeBuild, CodeDeploy) 및 S3, EC2를 활용하여 애플리케이션의 CI/CD 파이프라인을 단계별로 구축하는 실습 가이드를 제공합니다.

---

## 📂 실습 목차
- [Lab 10. AWS CodePipeline Basics (S3 정적 웹 호스팅 배포)](#-lab-10-aws-codepipeline-basics)
- [Lab 11. 소스 위치로 GitHub 저장소 활용하기 (Git 트리거 기반 배포)](#-lab-11-github-소스-연동)
- [Lab 12. AWS CodeBuild로 코드 빌드하기 (Angular 프로젝트 빌드 및 배포)](#-lab-12-aws-codebuild-코드-빌드)
- [Lab 13. AWS CodeDeploy로 EC2 인스턴스에 배포하기 (Nginx 연동 배포)](#-lab-13-aws-codedeploy-ec2-배포)

---

## 🛠️ Lab 10. AWS CodePipeline Basics
S3 버킷의 버전 관리와 정적 웹 호스팅 기능을 사용하여 수동 업로드한 파일이 자동으로 호스팅 서버로 배포되는 기본 파이프라인을 구축합니다.

### 1) S3 버킷 설정
* **Source 버킷** (`user**-source-website`):
  * 버킷 버전 관리(Bucket Versioning)를 **활성화(Enabled)**로 설정하여 생성합니다.
* **Prod 버킷** (`user**-prod-website`):
  * "모든 퍼블릭 액세스 차단" 설정을 **해제**합니다.
  * 객체 소유권(Object Ownership)을 **"ACL 활성화됨(ACLs enabled)"**으로 체크합니다.
  * 속성(Properties) -> **정적 웹 사이트 호스팅(Static website hosting)**을 활성화하고 인덱스 문서(`index.html`)와 오류 문서 (`error.html`)를 설정합니다.

### 2) 소스 파일 준비 및 업로드
* 실습용 파일(`index.html`, `error.html`)을 압축하여 `test-website.zip`을 만듭니다.
> [!IMPORTANT]
> **압축 시 주의사항**: `test-website` 폴더 자체를 우클릭해서 압축하면 배포 후 파일 경로가 틀어집니다. 반드시 폴더 내부로 이동하여 두 파일만 선택해 압축해야 최상위 경로에 `index.html`이 위치하게 됩니다.
* 생성된 zip 파일을 **Source 버킷**에 업로드합니다.

### 3) 파이프라인 생성
* 파이프라인 이름: `user**-website-pipeline`
* **Source 단계**: 소스 공급자 = `Amazon S3`, 버킷 = `user**-source-website`, 객체 키 = `test-website.zip`
* **Build/Test 단계**: 건너뜁니다.
* **Deploy 단계**: 배포 공급자 = `Amazon S3`, 버킷 = `user**-prod-website`, 추가 구성 -> **"배포하기 전에 파일 압축 풀기(Extract file before deploying)"** 체크, 표준 ACL = `public-read`.
* 파이프라인이 정상적으로 동작한 후, Prod 버킷의 정적 호스팅 엔드포인트 URL로 웹사이트가 정상 접근되는지 확인합니다.

### 4) 웹사이트 업데이트 및 파이프라인 흐름 제어
* **단계 전환 비활성화(Disable Transition)** 기능을 사용해 파이프라인의 Source와 Deploy 단계 간 흐름을 끊습니다.
* 소스 코드(`index.html`)를 수정(Version 2.0)하여 새로 압축한 뒤 Source 버킷에 업로드합니다.
* 파이프라인의 전환을 다시 **활성화(Enable Transition)**하면 보류 중이던 업데이트가 정상적으로 진행되며 배포되는 것을 실습합니다.

---

## 🛠️ Lab 11. GitHub 소스 연동
S3 소스 대신 GitHub 원격 저장소를 파이프라인과 연동하고, Git push 명령을 통해 파이프라인이 자동 트리거되도록 구성합니다. 또한 원격/로컬 갈등 상황의 해결 방법을 학습합니다.

### 1) GitHub 연동 설정
* GitHub 프라이빗 리포지토리(`test-website-repo`)를 생성합니다.
* 개인 계정 설정에서 `repo` 권한을 가진 Personal Access Token을 발급받아 보관합니다.
* CodePipeline 소스 공급자를 **"GitHub (GitHub 앱을 통해)"**로 선택하고 커넥터를 설정하여 리포지토리와 `main` 브랜치를 연결합니다.

### 2) 파이프라인 배포 및 동작 확인
* 기존 S3 배포(Prod 버킷)의 객체를 모두 삭제합니다.
* 파이프라인의 Deploy 단계는 이전과 동일하게 S3 Prod 버킷으로 설정하고 파일 압축 풀기 및 `public-read` ACL 설정을 지정합니다.
* 로컬 웹 리소스를 생성된 GitHub 리포지토리에 push합니다:
  ```bash
  git init
  git add .
  git commit -m "first commit"
  git remote add origin https://github.com/<계정명>/test-website-repo.git
  git branch -M main
  git push -u origin main
  ```
* Push 완료 후 파이프라인이 자동으로 트리거되어 Prod 버킷에 정상 배포됨을 확인합니다.

### 3) 갈등(Conflict) 상황 해결 실습
* **원격 수정**: GitHub 웹 브라우저 편집기에서 `index.html`을 수정(Version 3.0)하여 직접 커밋합니다. (파이프라인 자동 배포 완료)
* **로컬 수정**: 로컬 디렉터리의 `index.html`도 다른 내용(Version 2.2)으로 수정한 후 저장합니다.
* **충돌 해결**: 로컬 터미널에서 `git pull origin main`을 실행하면 Conflict가 발생합니다. VS Code 등의 에디터에서 갈등 코드를 병합한 후, 다시 commit 및 push하여 최종 배포 버전을 정렬합니다.

---

## 🛠️ Lab 12. AWS CodeBuild 코드 빌드
프레임워크(Angular) 애플리케이션을 빌드 환경(CodeBuild)에서 컴파일/빌드하고, 빌드 결과물(Artifact)을 S3 정적 호스팅 웹 버킷에 업로드합니다.

### 1) GitHub 리포지토리 및 S3 설정
* Angular 실습 코드를 담을 GitHub 리포지토리(`test-angular-repo`)를 생성하고 로컬 파일을 push합니다.
* 배포용 S3 버킷(`user**-angular-website`)을 생성하고 정적 호스팅 설정을 활성화합니다. (SPA이므로 인덱스와 오류 문서 모두 `index.html`로 설정) 퍼블릭 액세스 허용 및 ACL 활성화도 수행합니다.

### 2) 파이프라인 및 CodeBuild 설정
* 파이프라인 이름: `user**-AngularPipeline` (Source: GitHub `test-angular-repo`)
* **Build 단계**: 빌드 공급자로 **AWS CodeBuild**를 선택하고 "프로젝트 생성"을 클릭합니다.
  * 프로젝트 이름: `user**-AngularBuild`
  * 환경 이미지: 관리형 이미지, Ubuntu, Standard 런타임, 이미지 `aws/codebuild/standard:7.0` (최신 이미지)
  * 빌드 스펙: **"buildspec 파일 사용(Use a buildspec file)"** 선택
* **Deploy 단계**: 배포 공급자 = `Amazon S3`, 버킷 = `user**-angular-website`, 파일 압축 풀기 체크, 표준 ACL = `public-read`.

### 3) buildspec.yml 파일 작성
처음 빌드를 진행하면 리포지토리에 buildspec 파일이 없어 빌드가 실패합니다. 다음 명세를 Angular 프로젝트 루트에 `buildspec.yml` 이름으로 작성하여 업로드합니다.

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - npm install
  build:
    commands:
      - npm run build -- --prod
artifacts:
  files:
    - '**/*'
  base-directory: dist/my-angular-project
```
* 작성 후 push하면 자동으로 빌드와 배포가 성공하여 Angular App이 가동됨을 확인합니다.

---

## 🛠️ Lab 13. AWS CodeDeploy EC2 배포
빌드된 아티팩트를 Nginx가 구동 중인 AWS EC2 가상 서버에 CodeDeploy Agent를 통해 다운로드하고 설치(배포)하는 완전한 CD 파이프라인을 구축합니다.

### 1) EC2 인스턴스 구성 및 역할 바인딩
* **EC2용 IAM 역할 생성**: `AmazonEC2RoleforAWSCodeDeploy` 정책이 첨부된 IAM 역할(`user**-WebServerRole`)을 생성합니다.
* **EC2 인스턴스 기동**: Amazon Linux 2023 (`t3.micro`)로 기동하며 퍼블릭 IP를 자동 할당하고 SSH/HTTP/HTTPS 포트를 보안 그룹에서 오픈합니다.
* **역할 연결**: EC2 인스턴스의 작업 -> 보안 -> IAM 역할 수정 메뉴에서 생성한 `user**-WebServerRole` 역할을 연결합니다.
* **의존 도구 및 CodeDeploy Agent 설치**: EC2 인스턴스 연결 후 아래 스크립트를 순서대로 실행합니다.
  ```bash
  sudo yum update -y
  sudo yum install -y wget ruby
  # CodeDeploy Agent 다운로드 및 설치 (서울 리전 기준)
  wget https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install
  chmod +x install
  sudo ./install auto
  # 설치 완료 및 가동 상태 확인 (active (running) 확인)
  sudo systemctl status codedeploy-agent.service
  ```
* **Nginx 설치 및 경로 설정**:
  ```bash
  sudo yum install -y nginx
  sudo service nginx start
  sudo systemctl enable nginx.service
  # 배포 디렉토리 생성
  sudo mkdir -p /var/www/my-angular-project
  ```
  `/etc/nginx/nginx.conf` 파일을 열어 server 블록 내 `root` 경로를 `/var/www/my-angular-project`로 변경하고 Nginx를 재구동합니다:
  ```bash
  sudo service nginx restart
  ```
  브라우저에서 EC2 퍼블릭 DNS로 접속해 403 Forbidden 에러 혹은 Welcome to Nginx 화면이 나오는지 확인합니다.

### 2) CodeDeploy 구성
* **CodeDeploy용 IAM 역할 생성**: `AWSCodeDeployRole` 정책을 가진 역할(`user**-CodeDeployEC2ServiceRole`)을 생성합니다.
* **애플리케이션 생성**: 이름 `user**-AngularApp`, 컴퓨팅 플랫폼 = **EC2/On-premises**.
* **배포 그룹 생성**: 이름 `User**TaggedEC2Instances`
  * 서비스 역할 = 생성한 `user**-CodeDeployEC2ServiceRole`
  * 배포 유형 = 현재 위치(In-place)
  * 환경 구성 = Amazon EC2 인스턴스 체크, 태그 설정 (Key: `Name`, Value: `user**-AngularProject` 등 EC2 인스턴스명 입력)
  * 로드 밸런서 = **"로드 밸런싱 활성화" 체크 해제**

### 3) 파이프라인 Deploy 단계 수정
* `user**-AngularPipeline` 편집으로 이동하여 Deploy 단계의 기존 S3 배포 액션을 제거합니다.
* 작업 추가: 작업 공급자 = **AWS CodeDeploy**, 입력 아티팩트 = `BuildArtifact`, 애플리케이션 및 배포 그룹을 선택하고 저장합니다.

### 4) AppSpec 및 빌드 설정 업데이트 (실패 해결)
EC2 배포를 정상적으로 완료하기 위해 배포 라이프사이클을 통제하는 `appspec.yml` 파일과 업데이트된 빌드 및 권한 설정이 필요합니다.

* **appspec.yml 작성** (Angular 프로젝트 루트에 생성):
  ```yaml
  version: 0.0
  os: linux
  files:
    - source: /
      destination: /var/www/my-angular-project
  hooks:
    ApplicationStart:
      - location: deploy-scripts/application-start-hook.sh
        timeout: 300
        runas: root
  ```
* **배포 스크립트 작성** (`deploy-scripts/application-start-hook.sh` 생성):
  ```bash
  #!/bin/bash
  # Nginx 웹 서버를 재시작하여 새 빌드 적용
  service nginx restart
  ```
* **buildspec.yml 업데이트**:
  CodeDeploy가 배포를 시작하기 위해서는 압축된 아티팩트 내에 `appspec.yml`과 스크립트 파일이 함께 패키징되어 있어야 합니다. 기존 buildspec.yml을 아래와 같이 수정합니다:
  ```yaml
  version: 0.2

  phases:
    install:
      runtime-versions:
        nodejs: 18
      commands:
        - npm install
    build:
      commands:
        - npm run build -- --prod
  artifacts:
    files:
      - 'dist/my-angular-project/**/*'
      - 'appspec.yml'
      - 'deploy-scripts/**/*'
  ```
* **파이프라인 IAM 역할 권한 부여**:
  배포 시 권한 부족 에러가 발생할 수 있습니다. IAM에서 파이프라인의 서비스 역할(`AWSCodePipelineServiceRole-...`)을 검색하고, 다음 권한 정책을 첨부합니다:
  * `AWSCodeDeployDeployerAccess`
  * `AWSCodeDeployFullAccess`

모든 설정을 완료하고 리포지토리에 push하면 Source -> Build -> Deploy 단계를 거쳐 EC2 인스턴스의 Nginx 웹 사이트가 정상적으로 무중단 배포 및 자동 갱신됩니다.
