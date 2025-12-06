## Jenkins + Docker + GitHub SSH 설정 가이드

#이 문서는 Docker 환경에서 Jenkins가 GitHub SSH 인증과 Docker 데몬 제어를 수행할 수 있도록 설정하는 절차를 정리한 가이드입니다.

## 1. Jenkins 컨테이너에서 GitHub SSH 설정

### 1.1 컨테이너 접속
docker exec -it jenkins bash

### 1.2 SSH 디렉터리 생성
mkdir -p ~/.ssh
chmod 700 ~/.ssh

### 1.3 SSH Key 생성
ssh-keygen -t ed25519 -C "your_email@example.com
"
기본 저장 경로 사용 (/var/jenkins_home/.ssh/id_ed25519)

### 1.4 GitHub에 공개키 등록
cat ~/.ssh/id_ed25519.pub
출력 내용을 GitHub SSH Keys에 등록

### 1.5 known_hosts 등록
ssh-keyscan github.com >> ~/.ssh/known_hosts
chmod 644 ~/.ssh/known_hosts

### 1.6 키 권한 설정
chmod 600 ~/.ssh/id_ed25519
chmod 600 ~/.ssh/id_ed25519.pub

### 1.7 GitHub 연결 테스트
ssh -T git@github.com

## 2. Jenkins Pipeline에서 SSH Credentials 사용

### 2.1 Jenkins Credentials 추가
Kind: SSH Username with private key
Username: git
Private key: id_ed25519 내용
ID: GITHUB_SSH_ID

### 2.2 Pipeline 예시
pipeline {
agent any
stages {
stage('Checkout') {
steps {
sshagent(credentials: ['GITHUB_SSH_ID']) {
sh 'git clone git@github.com
:YOUR_REPO.git'
}
}
}
}
}

## 3. Jenkins 컨테이너에서 Docker.sock 접근 설정

### 3.1 호스트에서 docker 그룹 GID 확인
getent group docker
예: docker:x:113:ubuntu
113이 docker 그룹 GID

### 3.2 docker-compose.yml 설정
jenkins:
build:
context: .
dockerfile: Dockerfile.jenkins
container_name: jenkins
ports:
- "8080:8080"
environment:
TZ: Asia/Seoul
volumes:
- /var/run/docker.sock:/var/run/docker.sock
- ./jenkins_home:/var/jenkins_home
group_add:
- "113"

## 4. Jenkins Dockerfile 예시
FROM jenkins/jenkins:lts-jdk17
USER root
RUN apt-get update && apt-get install -y docker.io docker-compose && rm -rf /var/lib/apt/lists/*
USER jenkins

## 5. 설정 확인
id jenkins
ls -ln /var/run/docker.sock
docker ps
