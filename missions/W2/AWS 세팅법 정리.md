# Mac 환경에서 Docker 이미지를 AWS ECR 및 EC2에 배포하기

본 문서는 Mac 환경에서 Jupyter Notebook 기반 Docker 이미지를 빌드한 뒤 AWS ECR에 업로드하고, EC2 인스턴스에서 실행하는 과정을 순서대로 정리한 문서이다.

---

## 1. EC2 인스턴스 생성 및 초기 설정

AWS 콘솔에서 EC2 인스턴스를 생성한 뒤 아래 항목을 설정한다.

1. **퍼블릭 IP 자동 할당**

   * `활성화(Enable)` 선택
   * 비활성화하면 외부에서 EC2에 접속할 수 없다.

2. **보안 그룹(Inbound Rules)**

   * SSH / 포트 `22` / 소스 `0.0.0.0/0`
   * 사용자 지정 TCP / 포트 `8888` / 소스 `0.0.0.0/0`

3. **사용자 데이터(User data)**

   * 인스턴스 생성 화면 하단의 **고급 세부 정보 → 사용자 데이터(User data)** 에 아래 스크립트를 입력한다.
   * 인스턴스가 시작될 때 Docker가 자동으로 설치된다.

```bash
#!/bin/bash
apt-get update -y
apt-get install -y docker.io
usermod -aG docker ubuntu
systemctl start docker
systemctl enable docker
```

---

## 2. 로컬(Mac)에서 Docker 이미지 빌드 및 ECR 업로드

터미널을 열고 `Dockerfile`이 있는 프로젝트 디렉터리로 이동한 뒤 아래 순서대로 진행한다.

### 2.1 MFA 임시 세션 토큰 발급

MFA ARN과 현재 OTP를 입력하여 임시 인증 정보를 발급받는다.

```bash
aws sts get-session-token --serial-number "본인_MFA_ARN" --token-code 123456
```

### 2.2 환경 변수 설정

위 명령어의 출력값을 같은 터미널에 등록한다.

```bash
export AWS_ACCESS_KEY_ID="출력된_AccessKeyId"
export AWS_SECRET_ACCESS_KEY="출력된_SecretAccessKey"
export AWS_SESSION_TOKEN="출력된_SessionToken"
```

### 2.3 ECR 로그인

```bash
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin [본인계정ID].dkr.ecr.ap-northeast-2.amazonaws.com
```

### 2.4 Docker 이미지 빌드 및 업로드

Mac은 ARM 아키텍처를 사용하므로 EC2에서 실행할 수 있도록 `linux/amd64` 플랫폼으로 빌드한다.

```bash
# 이미지 빌드
docker build --platform linux/amd64 -t [본인계정ID].dkr.ecr.ap-northeast-2.amazonaws.com/부트캠프_리포지토리명:latest .

# ECR 업로드
docker push [본인계정ID].dkr.ecr.ap-northeast-2.amazonaws.com/부트캠프_리포지토리명:latest
```

---

## 3. EC2에서 이미지 실행

### 3.1 SSH 접속

`.pem` 파일이 있는 디렉터리에서 아래 명령어를 실행한다.

```bash
cd ~/Downloads
chmod 400 본인키이름.pem
ssh -i "본인키이름.pem" ubuntu@[EC2_퍼블릭_IP]
```

### 3.2 EC2에서 ECR 로그인

EC2에서도 ECR에 접근할 수 있도록 **2.1~2.3 과정(MFA 토큰 발급 → 환경 변수 설정 → ECR 로그인)** 을 한 번 수행한다.

### 3.3 이미지 다운로드 및 컨테이너 실행

```bash
# 이미지 다운로드
docker pull [본인계정ID].dkr.ecr.ap-northeast-2.amazonaws.com/부트캠프_리포지토리명:latest

# 컨테이너 실행
docker run -d -p 8888:8888 --name my-jupyter [본인계정ID].dkr.ecr.ap-northeast-2.amazonaws.com/부트캠프_리포지토리명:latest
```

---

## 4. 실행 확인

웹 브라우저에서 아래 주소로 접속한다.

```
http://[EC2_퍼블릭_IP]:8888
```

JupyterLab이 정상적으로 실행되고, 작업했던 `.ipynb` 파일이 보이면 배포가 완료된 것이다.

---

## 부록. 수정 사항 반영(재배포)

코드를 수정한 후에는 아래 순서로 다시 배포한다.

1. 로컬에서 Docker 이미지를 다시 빌드한다.

```bash
docker build --platform linux/amd64 ...
```

2. 새 이미지를 ECR에 업로드한다.

```bash
docker push ...
```

3. EC2에서 기존 컨테이너를 삭제한다.

```bash
docker rm -f my-jupyter
```

4. 최신 이미지를 다시 내려받아 실행한다.

```bash
docker pull ...
docker run ...
```

위 과정을 수행하면 수정된 내용이 서버에 반영된다.
