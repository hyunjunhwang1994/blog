---
title: (작성중)CICD 01 - Jenkins CI CD 구축 (gitlab) 
date: 2025-06-17 18:30:00
categories: CICD
tags:
  - CICD
  - GitLab
---

# Jenkins SSH 배포 완전 구축 가이드

> SSH 계정 방식으로 원격 서버 배포하는 Jenkins CI/CD 시스템 구축

## 📋 목차
1. [Jenkins 설치](#1-jenkins-설치)
2. [Jenkins 초기 설정](#2-jenkins-초기-설정)
3. [SSH 도구 설치](#3-ssh-도구-설치)
4. [대상 서버 준비](#4-대상-서버-준비)
5. [Jenkins 계정 정보 등록](#5-jenkins-계정-정보-등록)
6. [GitLab 연동 설정](#6-gitlab-연동-설정)
7. [Pipeline Job 생성](#7-pipeline-job-생성)
8. [SSH 배포 Jenkinsfile](#8-ssh-배포-jenkinsfile)
9. [배포 테스트](#9-배포-테스트)
10. [문제 해결](#10-문제-해결)

---

## 🏗️ 전체 아키텍처

```
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│   Jenkins 서버   │  SSH+PW  │   개발 서버      │  SSH+PW  │   운영 서버      │
│   192.168.99.5  │ ────────→│ 192.168.99.100  │ ────────→│ 192.168.99.200  │
│                 │          │                 │          │                 │
│  ┌─────────┐    │          │  기존 계정      │          │  기존 계정      │
│  │Jenkins  │    │          │  사용자/비밀번호  │          │  사용자/비밀번호  │
│  │+ sshpass│    │          │  + Docker       │          │  + Docker       │
│  └─────────┘    │          │  + sudo 권한    │          │  + sudo 권한    │
└─────────────────┘          └─────────────────┘          └─────────────────┘
```

---

## 1. Jenkins 설치

### 1-1. Docker로 Jenkins 설치
```bash
# Jenkins 컨테이너 실행
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --restart unless-stopped \
  jenkins/jenkins:lts

# 초기 비밀번호 확인
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### 1-2. 설치 확인
- 브라우저에서 `http://192.168.99.5:8080` 접속
- 초기 비밀번호 입력 준비

---

## 2. Jenkins 초기 설정

### 2-1. 플러그인 설치
1. **Unlock Jenkins** 페이지에서 초기 비밀번호 입력
2. **Install suggested plugins** 선택
3. 플러그인 설치 완료까지 대기 (5-10분)

### 2-2. 관리자 계정 생성
```
Username: admin
Password: [안전한 비밀번호]
Full name: Jenkins Admin
E-mail: admin@company.com
```

### 2-3. Jenkins URL 설정
- Jenkins URL: `http://192.168.99.5:8080/`
- **Save and Finish** 클릭

---

## 3. SSH 도구 설치

### 3-1. Jenkins 컨테이너에 SSH 도구 설치
```bash
# Jenkins 컨테이너에 root로 접속
docker exec -u root -it jenkins bash

# 시스템 업데이트
apt-get update

# SSH 관련 도구 설치
apt-get install -y sshpass openssh-client

# Docker 설치 (빌드용)
apt-get install -y docker.io

# jenkins 사용자를 docker 그룹에 추가
usermod -aG docker jenkins

# Docker socket 권한 설정
chmod 666 /var/run/docker.sock

# 설치 확인
sshpass -V
ssh -V
docker --version

# 컨테이너에서 나가기
exit
```

### 3-2. Jenkins 재시작 및 권한 확인
```bash
# Jenkins 컨테이너 재시작
docker restart jenkins

# 1-2분 후 권한 테스트
docker exec jenkins docker ps
```

---

## 4. 대상 서버 준비

### 4-1. 개발서버 (192.168.99.100) 설정

#### Docker 설치
```bash
# 개발서버에 SSH 접속
ssh username@192.168.99.100

# Docker 설치
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 현재 사용자를 docker 그룹에 추가
sudo usermod -aG docker $USER

# 로그아웃 후 다시 로그인 또는
newgrp docker

# Docker 권한 확인
docker ps
```

#### SSH 서비스 확인
```bash
# SSH 서비스 상태 확인
sudo systemctl status ssh

# SSH 서비스 시작 (필요시)
sudo systemctl start ssh
sudo systemctl enable ssh

# 방화벽 설정 (필요시)
sudo ufw allow 22
sudo ufw allow 4000  # 개발서버 포트
```

#### sudo 권한 확인
```bash
# 현재 사용자 sudo 권한 테스트
sudo docker ps

# sudo 그룹 확인
groups $USER

# sudo 그룹에 추가 (필요시)
sudo usermod -aG sudo $USER
```

### 4-2. 운영서버 (192.168.99.200) 설정

**개발서버와 동일한 설정 반복**:
```bash
# 운영서버에 SSH 접속
ssh username@192.168.99.200

# Docker 설치, SSH 설정, sudo 권한 확인
# (위의 개발서버 설정과 동일)

# 방화벽 설정
sudo ufw allow 22
sudo ufw allow 80   # 운영서버 포트
```

---

## 5. Jenkins 계정 정보 등록

### 5-1. 개발서버 계정 등록
1. **Jenkins** → **Manage Jenkins** → **Credentials**
2. **System** → **Global credentials** → **Add Credentials**
3. 설정:
   ```
   Kind: Username with password
   Username: [개발서버 사용자명]
   Password: [개발서버 비밀번호]
   ID: develop-server-account
   Description: Development Server SSH Account
   ```
4. **Create** 클릭

### 5-2. 운영서버 계정 등록
1. **Add Credentials** 다시 클릭
2. 설정:
   ```
   Kind: Username with password
   Username: [운영서버 사용자명]
   Password: [운영서버 비밀번호]
   ID: prod-server-account
   Description: Production Server SSH Account
   ```
3. **Create** 클릭

### 5-3. SSH 연결 테스트
```bash
# Jenkins 컨테이너에서 테스트
docker exec -it jenkins bash

# 개발서버 연결 테스트
sshpass -p "개발서버비밀번호" ssh -o StrictHostKeyChecking=no 사용자명@192.168.99.100 "echo 'Dev server connection successful'"

# 운영서버 연결 테스트
sshpass -p "운영서버비밀번호" ssh -o StrictHostKeyChecking=no 사용자명@192.168.99.200 "echo 'Prod server connection successful'"

# 컨테이너에서 나가기
exit
```

---

## 6. GitLab 연동 설정

### 6-1. GitLab Personal Access Token 생성
1. **GitLab 접속** → 우상단 아바타 → **Edit profile**
2. **Access Tokens** 메뉴 클릭
3. **Add new token** 설정:
   ```
   Token name: jenkins-ssh-access
   Expiration date: 1년 후
   Scopes: ✅ read_repository
   ```
4. **Create personal access token** 클릭
5. **생성된 토큰 복사** (glpat-xxxxx 형태)

### 6-2. Jenkins GitLab Credential 등록
1. **Jenkins** → **Manage Jenkins** → **Credentials**
2. **System** → **Global credentials** → **Add Credentials**
3. 설정:
   ```
   Kind: Username with password
   Username: [GitLab 사용자명]
   Password: [위에서 생성한 토큰]
   ID: gitlab-ssh-credentials
   Description: GitLab SSH Access Token
   ```
4. **Create** 클릭

---

## 7. Pipeline Job 생성

### 7-1. New Item 생성
1. **Jenkins 메인** → **New Item** 클릭
2. **Item name**: `ott-backend-ssh-pipeline`
3. **Pipeline** 선택 → **OK** 클릭

### 7-2. Pipeline 설정
**Pipeline** 섹션에서:
```
Definition: Pipeline script from SCM
SCM: Git
Repository URL: http://192.168.99.8/your-group/your-project
Credentials: gitlab-ssh-credentials
Branch Specifier: */develop
Script Path: Jenkinsfile
```

### 7-3. 저장
**Save** 클릭

---

## 8. SSH 배포 Jenkinsfile

### 8-1. 프로젝트에 Jenkinsfile 생성
프로젝트 루트 디렉토리에 `Jenkinsfile` 파일 생성:

```groovy
pipeline {
   agent any

   environment {
      IMAGE_NAME = "ott-backend:${new Date().format('yyyyMMdd')}"
      IMAGE_FILE = "${IMAGE_NAME}.tar.gz"

      // 서버 정보
      DEV_SERVER_IP = "192.168.99.45"
      PROD_SERVER_IP = "192.168.99.999"

      // 포트 정보
      DEV_PORT = "4000"
      PROD_PORT = "80"
   }

   stages {
      stage('Build') {
         steps {
            echo "🔨 Building Docker image: ${IMAGE_NAME}"
            sh 'docker build -t ${IMAGE_NAME} .'
            sh 'docker save ${IMAGE_NAME} | gzip > ${IMAGE_FILE}'
            echo "✅ Build completed"
         }
      }

      stage('Deploy to Dev') {
         when {
            expression { env.GIT_BRANCH == 'origin/develop' }
         }
         steps {
            script {
               echo "🚀 Deploying to Development Server: ${DEV_SERVER_IP}"

               // 개발서버 계정 정보 사용
               withCredentials([usernamePassword(credentialsId: 'develop-server-account',
                       usernameVariable: 'DEV_USER',
                       passwordVariable: 'DEV_PASS')]) {

                  // 개발서버로 이미지 파일 전송
                  sh '''
                            echo "📦 Transferring image to development server..."
                            sshpass -p "$DEV_PASS" scp -o StrictHostKeyChecking=no \\
                                ${IMAGE_FILE} $DEV_USER@${DEV_SERVER_IP}:/tmp/
                        '''

                  // 개발서버에서 배포 실행
                  sh '''
                            echo "🚀 Executing deployment on development server..."
                            sshpass -p "$DEV_PASS" ssh -o StrictHostKeyChecking=no \\
                                $DEV_USER@${DEV_SERVER_IP} "

                                echo '📦 Loading Docker image...'
                                cd /tmp
                                gunzip -c ${IMAGE_FILE} | sudo docker load

                                echo '🛑 Stopping existing container...'
                                sudo docker stop ott-backend-dev || true
                                sudo docker rm ott-backend-dev || true

                                echo '▶️  Starting new container...'
                                sudo docker run -d \\
                                    --name ott-backend-dev \\
                                    -p ${DEV_PORT}:3000 \\
                                    --restart unless-stopped \\
                                    -e NODE_ENV=development \\
                                    ${IMAGE_NAME}

                                echo '🧹 Cleaning up...'
                                rm -f /tmp/${IMAGE_FILE}
                                sudo docker image prune -f

                                echo '✅ Development deployment completed!'
                            "
                        '''
               }

               echo "🌐 Development server: http://${DEV_SERVER_IP}:${DEV_PORT}"
            }
         }
      }

      stage('Deploy to Prod') {
         when {
            expression { env.GIT_BRANCH == 'origin/main' }
         }
         steps {
            // 운영 배포 승인
            script {
               def userInput = input(
                       message: '🚨 Deploy to Production Server?',
                       parameters: [
                               choice(
                                       name: 'DEPLOY_CONFIRM',
                                       choices: ['No', 'Yes'],
                                       description: 'Are you sure you want to deploy to production?'
                               )
                       ]
               )

               if (userInput == 'No') {
                  error('Production deployment cancelled by user')
               }

               echo "🚀 Deploying to Production Server: ${PROD_SERVER_IP}"

               // 운영서버 계정 정보 사용
               withCredentials([usernamePassword(credentialsId: 'prod-server-account',
                       usernameVariable: 'PROD_USER',
                       passwordVariable: 'PROD_PASS')]) {

                  // 운영서버로 이미지 파일 전송
                  sh '''
                            echo "📦 Transferring image to production server..."
                            sshpass -p "$PROD_PASS" scp -o StrictHostKeyChecking=no \\
                                ${IMAGE_FILE} $PROD_USER@${PROD_SERVER_IP}:/tmp/
                        '''

                  // 운영서버에서 배포 실행
                  sh '''
                            echo "🚀 Executing deployment on production server..."
                            sshpass -p "$PROD_PASS" ssh -o StrictHostKeyChecking=no \\
                                $PROD_USER@${PROD_SERVER_IP} "

                                echo '📦 Loading Docker image...'
                                cd /tmp
                                gunzip -c ${IMAGE_FILE} | sudo docker load

                                echo '🛑 Stopping existing container...'
                                sudo docker stop ott-backend-prod || true
                                sudo docker rm ott-backend-prod || true

                                echo '▶️  Starting new container...'
                                sudo docker run -d \\
                                    --name ott-backend-prod \\
                                    -p ${PROD_PORT}:3000 \\
                                    --restart unless-stopped \\
                                    -e NODE_ENV=production \\
                                    ${IMAGE_NAME}

                                echo '⏳ Waiting for container to start...'
                                sleep 10

                                echo '🏥 Health check...'
                                curl -f http://localhost || echo 'Health check failed - manual verification required'

                                echo '🧹 Cleaning up...'
                                rm -f /tmp/${IMAGE_FILE}
                                sudo docker image prune -f

                                echo '✅ Production deployment completed!'
                            "
                        '''
               }

               echo "🌐 Production server: http://${PROD_SERVER_IP}"
            }
         }
      }

      stage('Cleanup') {
         steps {
            echo "🧹 Cleaning up Jenkins workspace..."
            sh 'rm -f ${IMAGE_FILE}'
            sh 'docker image prune -f'
         }
      }
   }

   post {
      success {
         echo '🎉 Pipeline completed successfully!'
         script {
            if (env.GIT_BRANCH == 'origin/develop') {
               echo "🌐 Development: http://${DEV_SERVER_IP}:${DEV_PORT}"
            } else if (env.GIT_BRANCH == 'origin/main') {
               echo "🌐 Production: http://${PROD_SERVER_IP}"
            }
         }
      }
      failure {
         echo '❌ Pipeline failed! Check logs for details.'
      }
      always {
         echo '📊 Pipeline finished.'
      }
   }
}
```

### 8-2. Dockerfile 확인
프로젝트에 `Dockerfile`이 있는지 확인 (이전에 생성한 것 사용)

### 8-3. GitLab에 파일 업로드
```bash
git add Jenkinsfile
git commit -m "Add Jenkins SSH deployment pipeline"
git push origin develop
```

---

## 9. 배포 테스트

### 9-1. develop 브랜치 배포 테스트
```bash
# 코드 수정
echo "console.log('SSH deployment test');" >> src/main.ts

# 커밋 및 푸시
git add .
git commit -m "Test SSH deployment to dev server"
git push origin develop
```

**Jenkins에서 확인**:
1. **ott-backend-ssh-pipeline** → **Build Now** 클릭
2. **Build History**에서 진행 상황 확인
3. **Console Output**에서 상세 로그 확인
4. `✅ Development deployment completed successfully!` 메시지 확인
5. `http://192.168.99.100:4000` 접속 확인

### 9-2. main 브랜치 배포 테스트
```bash
# main 브랜치로 전환 및 merge
git checkout main
git merge develop
git push origin main
```

**Jenkins에서 확인**:
1. **Build Now** 클릭
2. **Production Approval** 단계에서 대기
3. `✅ Yes` 선택 및 `DEPLOY` 입력
4. 운영서버 배포 진행 확인
5. `http://192.168.99.200` 접속 확인

---

## 10. 문제 해결

### 10-1. SSH 연결 문제

#### 에러: `Connection refused`
```bash
# 대상 서버에서 SSH 서비스 확인
sudo systemctl status ssh
sudo systemctl start ssh

# 방화벽 확인
sudo ufw status
sudo ufw allow 22
```

#### 에러: `Permission denied`
```bash
# Jenkins에서 연결 테스트
docker exec -it jenkins bash
sshpass -p "비밀번호" ssh -o StrictHostKeyChecking=no 사용자명@192.168.99.100 "whoami"
```

### 10-2. Docker 권한 문제

#### 에러: `Permission denied while trying to connect to the Docker daemon`
```bash
# 대상 서버에서
sudo usermod -aG docker $USER
sudo systemctl restart docker
# 로그아웃 후 다시 로그인
```

#### 에러: `sudo: docker: command not found`
```bash
# 대상 서버에 Docker 재설치
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### 10-3. 포트 충돌 문제

#### 에러: `port is already allocated`
```bash
# 대상 서버에서 포트 사용 확인
sudo netstat -tlnp | grep :4000
sudo lsof -i :4000

# 사용 중인 컨테이너 중지
docker stop $(docker ps -q --filter "publish=4000")
```

### 10-4. sudo 권한 문제

#### 에러: `sudo: command not found` 또는 권한 없음
```bash
# 대상 서버에서 sudo 권한 확인
groups $USER

# sudo 그룹에 추가
sudo usermod -aG sudo $USER

# 비밀번호 없이 sudo 실행 (선택사항)
sudo visudo
# 추가: your-username ALL=(ALL) NOPASSWD:ALL
```

### 10-5. 이미지 전송 실패

#### 에러: `No space left on device`
```bash
# 대상 서버에서 용량 확인
df -h
sudo docker system prune -a

# /tmp 디렉토리 정리
sudo rm -f /tmp/*.tar.gz
```

---

## ✅ 완료 체크리스트

### Jenkins 서버 설정
- [ ] Jenkins 설치 및 접속 확인
- [ ] sshpass, openssh-client 설치 완료
- [ ] Docker 권한 설정 완료
- [ ] GitLab Personal Access Token 생성
- [ ] Jenkins GitLab Credential 등록

### 대상 서버 설정
- [ ] 개발서버 Docker 설치 완료
- [ ] 운영서버 Docker 설치 완료
- [ ] 각 서버 SSH 서비스 활성화
- [ ] 각 서버 방화벽 포트 허용
- [ ] 각 서버 sudo 권한 확인

### Jenkins 배포 설정
- [ ] 개발서버 SSH 계정 Jenkins 등록
- [ ] 운영서버 SSH 계정 Jenkins 등록
- [ ] SSH 연결 테스트 성공
- [ ] Pipeline Job 생성 완료
- [ ] Jenkinsfile 작성 및 업로드

### 배포 테스트
- [ ] develop 브랜치 자동 배포 성공
- [ ] 개발서버 접속 확인 (포트 4000)
- [ ] main 브랜치 수동 승인 배포 성공
- [ ] 운영서버 접속 확인 (포트 80)

**🎉 축하합니다! SSH 기반 Jenkins CI/CD 파이프라인이 성공적으로 구축되었습니다!**

---

## 📚 다음 단계 (고도화)

1. **다중 서비스 관리**: 여러 서비스의 파이프라인 추가
2. **보안 강화**: SSH 키 방식으로 업그레이드
3. **모니터링**: 배포 상태 알림 시스템 구축
4. **롤백 기능**: 이전 버전으로 되돌리기 기능
5. **환경별 설정**: 환경변수 및 설정 파일 관리









---










🎯 핵심 포인트
주요 변경사항

✅ sshpass 설치 포함
✅ SSH 연결 테스트 방법 포함
✅ 대상 서버 준비 상세 가이드
✅ 계정 정보 등록 단계별 설명
✅ 문제 해결 섹션 대폭 강화

실제 동작 방식
bash# 1. Jenkins에서 이미지 빌드
docker build -t ott-backend:20250617-23 .
docker save ott-backend:20250617-23 | gzip > image.tar.gz

# 2. SSH로 파일 전송
sshpass -p "비밀번호" scp image.tar.gz user@192.168.99.100:/tmp/

# 3. SSH로 원격 배포 실행
sshpass -p "비밀번호" ssh user@192.168.99.100 "
gunzip -c /tmp/image.tar.gz | sudo docker load
sudo docker run -d --name ott-backend-dev -p 4000:3000 ott-backend:20250617-23
"
🚀 설정 순서 요약

Jenkins 설치 (5분)
sshpass 설치 (5분)
대상 서버 Docker 설치 (10분)
계정 정보 등록 (5분)
SSH 연결 테스트 (5분)
Jenkinsfile 작성 (10분)

총 40분으로 완전한 SSH 원격 배포 시스템 완성!
이제 Docker Hub 비용 없이도 안전하고 효율적인 원격 배포가 가능합니다! 🎉

위는 백엔드 용으로 만들었으며 추후 프론트용으로 생성시 Jenkinsfile 변경과, :8080에서 파이프라인을 추가해주면된다.


pipeline {
agent any

    environment {
        IMAGE_NAME = "ott-frontend:${new Date().format('yyyyMMdd')}"
        IMAGE_FILE = "${IMAGE_NAME}.tar.gz"

        // 서버 정보
        DEV_SERVER_IP = "192.168.99.45"
        PROD_SERVER_IP = "192.168.99.999"
    }

    stages {
        stage('Build') {
            steps {
                echo "🔨 Building Docker image: ${IMAGE_NAME}"
                sh 'docker build -t ${IMAGE_NAME} .'
                sh 'docker save ${IMAGE_NAME} | gzip > ${IMAGE_FILE}'
                echo "✅ Build completed"
            }
        }

        stage('Deploy to Dev') {
            when {
                expression { env.GIT_BRANCH == 'origin/develop' }
            }
            steps {
                script {
                    echo "🚀 Deploying to Development Server: ${DEV_SERVER_IP}"

                    // 개발서버 계정 정보 사용
                    withCredentials([usernamePassword(credentialsId: 'develop-server-account',
                                                    usernameVariable: 'DEV_USER',
                                                    passwordVariable: 'DEV_PASS')]) {

                        // 개발서버로 이미지 파일 전송
                        sh '''
                            echo "📦 Transferring image to development server..."
                            sshpass -p "$DEV_PASS" scp -o StrictHostKeyChecking=no \\
                                ${IMAGE_FILE} $DEV_USER@${DEV_SERVER_IP}:/tmp/
                        '''

                        // 개발서버에서 배포 실행
                        sh '''
                            echo "🚀 Executing deployment on development server..."
                            sshpass -p "$DEV_PASS" ssh -o StrictHostKeyChecking=no \\
                                $DEV_USER@${DEV_SERVER_IP} "

                                echo '📦 Loading Docker image...'
                                cd /tmp
                                gunzip -c ${IMAGE_FILE} | sudo docker load

                                echo '🛑 Stopping existing container...'
                                sudo docker stop ott-frontend-dev || true
                                sudo docker rm ott-frontend-dev || true

                                echo '▶️  Starting new container...'
                                sudo docker run -d \\
                                    --name ott-frontend-dev \\
                                    -p 3000:80 \\
                                    --restart unless-stopped \\
                                    ${IMAGE_NAME}

                                echo '🧹 Cleaning up...'
                                rm -f /tmp/${IMAGE_FILE}
                                sudo docker image prune -f

                                echo '✅ Development deployment completed!'
                            "
                        '''
                    }

                    echo "🌐 Development server: http://${DEV_SERVER_IP}:3000"
                }
            }
        }

        stage('Deploy to Prod') {
            when {
                expression { env.GIT_BRANCH == 'origin/main' }
            }
            steps {
                // 운영 배포 승인
                script {
                    def userInput = input(
                        message: '🚨 Deploy to Production Server?',
                        parameters: [
                            choice(
                                name: 'DEPLOY_CONFIRM',
                                choices: ['No', 'Yes'],
                                description: 'Are you sure you want to deploy to production?'
                            )
                        ]
                    )

                    if (userInput == 'No') {
                        error('Production deployment cancelled by user')
                    }

                    echo "🚀 Deploying to Production Server: ${PROD_SERVER_IP}"

                    // 운영서버 계정 정보 사용
                    withCredentials([usernamePassword(credentialsId: 'prod-server-account',
                                                    usernameVariable: 'PROD_USER',
                                                    passwordVariable: 'PROD_PASS')]) {

                        // 운영서버로 이미지 파일 전송
                        sh '''
                            echo "📦 Transferring image to production server..."
                            sshpass -p "$PROD_PASS" scp -o StrictHostKeyChecking=no \\
                                ${IMAGE_FILE} $PROD_USER@${PROD_SERVER_IP}:/tmp/
                        '''

                        // 운영서버에서 배포 실행
                        sh '''
                            echo "🚀 Executing deployment on production server..."
                            sshpass -p "$PROD_PASS" ssh -o StrictHostKeyChecking=no \\
                                $PROD_USER@${PROD_SERVER_IP} "

                                echo '📦 Loading Docker image...'
                                cd /tmp
                                gunzip -c ${IMAGE_FILE} | sudo docker load

                                echo '🛑 Stopping existing container...'
                                sudo docker stop ott-frontend-prod || true
                                sudo docker rm ott-frontend-prod || true

                                echo '▶️  Starting new container...'
                                sudo docker run -d \\
                                    --name ott-frontend-prod \\
                                    -p 80:80 \\
                                    --restart unless-stopped \\
                                    ${IMAGE_NAME}

                                echo '⏳ Waiting for container to start...'
                                sleep 10

                                echo '🏥 Health check...'
                                curl -f http://localhost || echo 'Health check failed - manual verification required'

                                echo '🧹 Cleaning up...'
                                rm -f /tmp/${IMAGE_FILE}
                                sudo docker image prune -f

                                echo '✅ Production deployment completed!'
                            "
                        '''
                    }

                    echo "🌐 Production server: http://${PROD_SERVER_IP}"
                }
            }
        }

        stage('Cleanup') {
            steps {
                echo "🧹 Cleaning up Jenkins workspace..."
                sh 'rm -f ${IMAGE_FILE}'
                sh 'docker image prune -f'
            }
        }
    }

    post {
        success {
            echo '🎉 Pipeline completed successfully!'
            script {
                if (env.GIT_BRANCH == 'origin/develop') {
                    echo "🌐 Development: http://${DEV_SERVER_IP}:3000"
                } else if (env.GIT_BRANCH == 'origin/main') {
                    echo "🌐 Production: http://${PROD_SERVER_IP}"
                }
            }
        }
        failure {
            echo '❌ Pipeline failed! Check logs for details.'
        }
        always {
            echo '📊 Pipeline finished.'
        }
    }
}