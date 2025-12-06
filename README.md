### JENKINS 설정
✅ 1. Jenkins 컨테이너 내부 들어가기
docker exec -it jenkins bash

안에서 다음처럼 나와야 정상:

whoami
# jenkins

echo $HOME
# /var/jenkins_home

✅ 2. ~/.ssh 디렉터리 준비
mkdir -p ~/.ssh
chmod 700 ~/.ssh

✅ 3. SSH 키 생성 (ed25519 사용 권장)
ssh-keygen -t ed25519 -C "your_email@example.com"


중간에 물어보는 건 그냥 엔터:

Enter file in which to save the key (/var/jenkins_home/.ssh/id_ed25519): [Enter]
Enter passphrase (empty for no passphrase): [Enter]
Enter same passphrase again: [Enter]


생성된 파일:

/var/jenkins_home/.ssh/id_ed25519 (개인키)

/var/jenkins_home/.ssh/id_ed25519.pub (공개키)

✅ 4. GitHub에 SSH 공개키 등록하기

컨테이너 안에서:

cat ~/.ssh/id_ed25519.pub


출력된 전체 문자열을 복사해서:

GitHub → Settings → SSH and GPG keys → New SSH Key

에 그대로 붙여넣기.

✅ 5. known_hosts 등록 (매번 yes 안 뜨게)

컨테이너 안에서:

ssh-keyscan github.com >> ~/.ssh/known_hosts
chmod 644 ~/.ssh/known_hosts

✅ 6. 권한 확인
chmod 600 ~/.ssh/id_ed25519
chmod 600 ~/.ssh/id_ed25519.pub

✅ 7. GitHub 연결 테스트
ssh -T git@github.com -v


정상 연결 시:

Hi YOUR_GITHUB_USERNAME! You've successfully authenticated, but GitHub does not provide shell access.

✅ 8. Jenkins Pipeline에서 Git SSH 사용하기
Jenkins Credentials 등록

Jenkins Dashboard → Credentials

Kind: SSH Username with private key

Username: git

Private key: (컨테이너 내부 id_ed25519 내용 그대로 붙여넣기)

ID: GITHUB_SSH_ID 같은 이름으로

Pipeline 사용 예시
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

### JENKINS Container Docker.sock 설정
1단계. 호스트에서 docker 그룹 GID 확인

호스트에서 한 번만 실행:

getent group docker

예시 출력:

docker:x:998:

2단계. docker-compose.yml 파일 수정
  # -------------------------------------------------------
  # Jenkins (+도커 커스텀 이미지 사용)
  # -------------------------------------------------------
  jenkins:
    build:
      context: .                # 현재 디렉터리
      dockerfile: Dockerfile.jenkins
    container_name: jenkins
    ports:
      - "8080:8080"
    environment:
      TZ: Asia/Seoul
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock     # Host Docker Engine 제어
      - ./jenkins_home:/var/jenkins_home              # Jenkins 데이터
    group_add:
      - "999"   # ← 호스트(서버) getent group docker 명령어로 확인한 docker GID로 바꿔 넣기
    networks:
      - nyd-net




