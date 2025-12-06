Jenkins + Docker + GitHub SSH 설정 가이드

이 문서는 Docker 환경에서 Jenkins가 GitHub SSH로 코드 체크아웃하고
Docker 데몬(/var/run/docker.sock)에 정상 접근하도록 설정하는 절차를 정리한 가이드입니다.

1. Jenkins 컨테이너에서 GitHub SSH 설정
1) 컨테이너 접속
docker exec -it jenkins bash

2) SSH 디렉터리 생성
mkdir -p ~/.ssh
chmod 700 ~/.ssh

3) SSH 키 생성
ssh-keygen -t ed25519 -C "your_email@example.com"


경로는 기본값 사용
→ /var/jenkins_home/.ssh/id_ed25519

4) GitHub에 공개키 등록
cat ~/.ssh/id_ed25519.pub


출력 내용 → GitHub > Settings > SSH and GPG Keys → New SSH Key

5) known_hosts 등록
ssh-keyscan github.com >> ~/.ssh/known_hosts
chmod 644 ~/.ssh/known_hosts

6) 권한 설정
chmod 600 ~/.ssh/id_ed25519
chmod 600 ~/.ssh/id_ed25519.pub

7) 연결 테스트
ssh -T git@github.com

2. Jenkins Pipeline에서 GitHub SSH 사용
Jenkins Credentials 추가

Kind: SSH Username with private key

Username: git

Private key: id_ed25519 내용 복사

ID: GITHUB_SSH_ID

Pipeline 예시
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                sshagent(credentials: ['GITHUB_SSH_ID']) {
                    sh 'git clone git@github.com:NamYounDong/aws-nyd-server-infra.git'
                }
            }
        }
    }
}

3. Jenkins 컨테이너에서 Docker.sock 권한 설정

Jenkins가 Docker 명령을 실행하려면
컨테이너 내부 jenkins 유저가 호스트 docker 그룹(GID) 을 가져야 합니다.

1) 호스트에서 docker 그룹 GID 확인
getent group docker


예)

docker:x:113:ubuntu


→ 113이 GID

2) docker-compose.yml 수정
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
    - "113"   # 호스트 docker GID

4. Jenkins Dockerfile 예시
FROM jenkins/jenkins:lts-jdk17

USER root

RUN apt-get update && \
    apt-get install -y docker.io docker-compose && \
    rm -rf /var/lib/apt/lists/*

USER jenkins
