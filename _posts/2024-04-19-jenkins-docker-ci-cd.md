---
title: (DevOps) Jenkins와 Docker를 활용한 CI/CD Pipeline 구축(NCP 환경)
date: 2024-04-19 16:55:00 +0800
description: Jenkins와 Docker로 CI/CD 환경을 구축해보자. GitHub의 Webhook을 통한 CD 처리를 구축해보자
excerpt: Jenkins와 Docker로 CI/CD 환경을 구축해보자. GitHub의 Webhook을 통한 CD 처리를 구축해보자
categories:
  - DevOps
  - Docker
tags:
  - Jenkins
  - NaverCloudPlatform
  - NCP
  - Docker
  - Dockerhub
  - GitHub
  - Webhook
---

프로젝트 진행을 위해 NCP(Naver Cloud Platform) 환경에서 Jenkins Pipeline을 통한 CI/CD 구축 과정을 기록하였다.

### 구성

Jenkins와 Docker를 활용한 Pipeline 구성은 아래 이미지와 같다.

![](../assets/img/2024-04-19-jenkins-docker-ci-cd/Jenkins Docker Pipeline 구축.png)

### **NCP 클라우드 서버 생성**

위와 같은 환경 구축을 위해 NCP에서 서버 2EA 를 생성해야한다.
- **Spring Boot Server - Ubuntu 18.0.4**
	- [Standard] 4vCPU, 16GB Mem [g2]
- **Jenkins Server - Ubuntu 18.0.4**
	- [Standard] 2vCPU, 8GB Mem [g2]

NCP 서버 생성 관련 가이드는 쉬운 가이드<https://www.ncloud.com/guideCenter/guide> 를 참고하여 생성하도록 하자. 이번 구축은 NCP Classic, SSH는 Putty를 사용하여 진행하였다.

간략한 서버 생성 절차는 아래와 같다.
1. **ACG 생성( ncloud-default-acg)**
2. **서버 생성**
3. **공인 IP 신청 및 서버와 매핑**
4. **서버 접속(Putty)**
5. **서버 password 변경(passwd root)**

서버 생성이 완료되고 Putty를 통하여 원격접속하면 아래와 같은 화면을 확인할 수 있다.
![](../assets/img/2024-04-19-jenkins-docker-ci-cd/server 접속.png)

### **Jenkins 서버에 Docker 설치**

```
$ apt update
	: apt 패키지 관리자를 사용하여 패키지 목록을 최신 상태로 업데이트

$ apt install docker.io -y
	: apt 패키지 관리자를 사용하여 Docker를 설치 
	: -y 옵션은 사용자에게 확인 메시지를 요구하지 않고 설치를 진행

$ systemctl status docker
	: Docker 서비스의 현재 상태를 확인합니다.
	: 서비스가 정상적으로 실행 중인지 여부를 확인하는 데 사용

$ usermod -aG docker root
	: 현재 사용자를 docker 그룹에 추가하여 Docker 명령어를 사용하도록 추가

$ id root
	: root 사용자의 그룹 정보 확인

$ sudo curl https://raw.githubusercontent.com/docker/docker-ce/master/components/cli/contrib/completion/bash/docker \ -o /etc/bash_completion.d/docker.sh
	: Docker CLI의 자동 완성 기능을 사용하기 위해 Docker 커뮤니티 에디션의 bash 컴플리션 스크립트를 다운로드
```


### **Jenkins 서버 Docker 컨테이너에 Jenkins 설치**

#### **Docker 컨테이너에 Jenkins 설치**

Docker 컨테이너에 Jenkins를 설치하여 실행하도록 명령어를 작성한다.

```
docker run -d --name [컨테이너 명] -p 8080:8080 -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock --user root jenkins/jenkins:latest
```

위 명령어로 docker의 jenkins 컨테이너를 실행한다.
- **-v** : 호스트의 파일 시스템 디렉토리나 파일을 컨테이너 내부로 마운트
- **/var/run/docker.sock:/var/run/docker.sock**: 호스트의 Docker 소켓을 컨테이너 내부의 Docker 소켓에 연결하여 Docker 명령어를 컨테이너 내부에서 실행가능 하도록 설정

정상적으로 실행되었다면 docker ps 명령을 통해 컨테이너의 상태를 확인한다.

![](../assets/img/2024-04-19-jenkins-docker-ci-cd/docker 컨테이너 실행여부.png)

#### **Jenkins 관련 설정**

Jenkins에 설정한 공인 IP를 통해 접속한다.

```
http://[공인 IP]:8080
```

![](../assets/img/2024-04-19-jenkins-docker-ci-cd/Jenkins 초기 접속.png)

위와 같은 이미지가 나오는데 초기 비밀번호는 아래 명령어로 확인 가능하다.

```
$ docker logs [컨테이너 명]
```

비밀번호를 입력하면 **Customize Jenkins** 화면이 나오는데 Install suggested plugins을 선택하여 기본적인 plugin을 설치한다.

![](../assets/img/2024-04-19-jenkins-docker-ci-cd/Jenkins Customize Jenkins.png)

plugin 설치가 완료되면 **Create First Admin User**를 등록할 수 있다. 추가할 사용자가 있다면 등록하고 그렇지 않다면 **Skip and contiunue as admin**을 통해 넘어간다.

마지막으로 Jenkins 접속할 Url 설정을 끝으로 기본 설정은 마무리된다.

### **Docker Jenkins 컨테이너 내부에 Docker 설치**

이제 Docker Jenkins 컨테이너로 이동하여 내부에 Docker를 설치한다.
Jenkins 서버에서 아래 명령어로 Docker 내부의 Jenkins 컨테이너로 이동한다.

```
$ docker exec -it [컨테이너 명] /bin/bash
```

![](../assets/img/2024-04-19-jenkins-docker-ci-cd/컨테이너 접속.png)

정상적으로 컨테이너 내부로 접속하면 컨테이너 ID로 경로가 변경된다.

```
$ apt update
	: 패키지 목록을 업데이트

$ apt -y install apt-transport-https ca-certificates curl gnupg2 jq software-properties-common lsb-release
	: apt-transport-https ca-certificates curl - https 통신과 관련됨
	: gnupg2 - 암호화 및 디지털 서명을 위한 도구
	: software-properties-common - 소프트웨어 저장소관리, 제3자 저장소를 apt 소스 리스트에 추가할 때 사용

$ curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey
	: Docker의 공식 GPG 키를 다운로드하여 추가
	
$ add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") $(lsb_release -cs) stable"
	: Docker의 API 저장소를 추가

$ apt update
	: 패키지 목록을 다시 업데이트

$ app install docker.io -y
	: Docker 설치
```

이제 Jenkins 컨테이너 내부에서도 Docker 에 대한 명령어를 실행할 수 있다.

### **Jenkins 에서 Credentials 설정**

#### **관련 plugin 설치**

Jenkins에 접속하여 Docker와 Docker pipeline, SSH Agent를 설치한다.

![](../assets/img/2024-04-19-jenkins-docker-ci-cd/플러그인 설치.png)

Dashbaord > Manage Jenkins > Plugins > Avaliable plugins  에서 docker, docker pipeline 플러그인을 설치하고 재시작

```
$ docker restart [컨테이너 명]
```

#### **Credential 추가**
Jenkins와 Docker Hub 연결을 위해 Credential을 추가한다.

Dashbaord > Manage Jenkins > Creadentials의 하단 Store scoped to Jenkins에서 (global) 선택하고 우측 상단에 + Add Credentials  버튼을 누른다.

New creadentials 화면에서 아래 정보를 입력한다.

>- Kind : Username with password 선택
>- Scope : Global (Jenkins, nodes, items, all child items, etc) 선택
>- Username : 연결할 Docker Hub ID 입력
>- Password : 연결할 Docker Hub Password 입력
>- ID : Pipelin Script에서 식별자로 사용할 credentials ID

입력 완료 후 Create 버튼을 선택한다.

#### **SSH Credentials 추가**

SSH credentials를 등록하기 위해서는 먼저 젠킨스 서버에서 SSH키를 생성한다.
SSH키를 생성하면 비밀번호를 입력하지 않아도 원격 시스템에 액세스 할 수 있으며,  암호기반보다 보안이 강화

Jenkins 서버로 Putty를 통해 접속한다.

```
$ ssh-keygen
```

순차적으로 Enter 키를 입력하면 SSH 키가 생성된다.
생성 여부 확인을 위해 아래 명령어를 입력한다.

```
$ cat /root/.ssh/id_rsa.pub
```

정상적으로 생성되었다면 마지막에 서버명으로 끝나는 데이터가 생성된다.

```
ssh-rsa [임의의 문자열] root@jenkins-server
```

이제 해당 문자열을 Spring Boot Server에 인증키로 추가 해야한다.
해당 작업은 우선 Jenkins 설정을 마무리하고 다음 항목에서 작업하도록 하자.

다음으로 Jenkins 서버에 생성한 SSH 키를 확인한다.

```
$ cat /root/.ssh/id_rsa
```

정상적으로 실행된다면 아래 내용과 같이 데이터가 표시된다.
해당 데이터를 모두 복사한다.
```
-----BEGIN RSA PRIVATE KEY-----

[임의의 문자열]

-----END RSA PRIVATE KEY-----
```

다시 Dashbaord > Manage Jenkins > Creadentials의 하단 Store scoped to Jenkins에서 (global) 선택하고 우측 상단에 + Add Credentials  버튼을 누른다.

New creadentials 화면에서 아래 정보를 입력한다.

>- Kind : SSH Username with private key 선택 **(SSH 연결 시 사용할 종류)**
>- Scope : Global (Jenkins, nodes, items, all child items, etc) 선택
>- ID : ssh 입력 **(Pipelin Script에서 식별자로 사용할 credentials ID)**
>- Username : root 입력 **(서버 사용자 계정)**
   > Private Key의 Enter directly 선택
>- Key : 복사한 id_rsa 데이터를 입력

입력 완료 후 Create 버튼을 선택한다.
다음으로 Spring Boot Server 작업을 진행한다.

#### **Spring Boot Server 생성 및 Docker 설치**

위에서 진행 Jenkins Server 생성한 것과 마찬가지로 Spring Boot Server를 NCP에서 생성한다.
생성된 서버에 원격 접속하여 Docker까지 설치를 진행한다.
아래 명령어는 Spring Boot Server에서 진행한다.

```
$ cd /root/.ssh/
	: SSH 폴더 경로로 이동

$ vi authorzied_keys
	: authorzied_keys 파일을 생성한다.
	: 파일에 작성할 내용은 Jenkins 서버의 id_rsa.pub의 내용이 들어가면 된다. 
```

### **Github Credentials 추가**

Github webhook을 사용하기 위해 Github의 credential 을 추가한다.

Github의 creadential 추가를 위해 사전에 Github의 Token을 발급받아야 하므로 Github에 접속한다.

로그인 > 우측 상단 개인 프로필 > Settings > Developer settings > Personal access tokens > Tokens(classic) 에서 Token을 발급한다.

Scope 중 아래 항목은 webhook을 사용하기 위해 check 한다.

![](../assets/img/2024-04-19-jenkins-docker-ci-cd/Github Webhook 설정.png)

발급된 Token을 별도로 저장해두고 다시 Jenkins로 이동한다.

Dashboard > Manage Jenkins > System 에 들어가 GitHub를 찾은 후 **Add GitHub Server** 버튼을 눌러 GitHub Server를 추가한다.

GitHub Server에서 Name은 GitHub 사용자 명을 입력하고 Credentials에 + Add 버튼을 선택하고 Jenkins를 선택한다.

Add Credentials 에서 아래와 같이 입력한다.

> - Domain : Global credentials (unrestricted) 선택
> - Kind : Secret text 선택
> - Scope :  Global (Jenkins, nodes, items, all child items, etc) 선택
> - Secret : GitHub Token
> - ID : github-access-token 입력 **(Pipelin Script에서 식별자로 사용할 credentials ID)**

작성 완료 후 Add 버튼을 선택

Credentials 에서 방금 Add한 github-access-token를 선택하고 우측 하단 Test connection을 누르면 connection이 정상적으로 되었을 경우 아래와 같이 표시된다.

![](../assets/img/2024-04-19-jenkins-docker-ci-cd/Test Connection.png)

### **GitHub Webhooks 설정**

1. Jenkins에서 CI/CD를 수행할 GitHub의 Repository로 이동
2. 상단 Tab에서 Settings 선택
3. 우측 메뉴에서 Webhooks 선택
4. 해당 내용을 설정한다.
	- Payload URL : 'http://[Jenkins 서버 IP]:[Jenkins 서버 Port]/github-webhook/' 입력
	- Content type : application/json 선택
	- Which events would you like to trigger this webhook? 에서 Let me select individual events 라디오 버튼 선택.
	- 해당 체크 박스 중 Pull Requests, Pushes 선택
	- 하단 Active는 활성화
5. 작성 완료 후 Update webhook 버튼 선택

### **Jenkins Pipeline 구성**

이제 CI/CD를 적용할 프로젝트에 Pipeline을 구성한다.

1. Jenkins에 접속하여 좌측 상단 **New Item** 버튼 선택
2. Enter an item name에 명칭 입력
3. Select an item type에서 Pipeline 선택
4. OK 선택
5. General - GitHub project 를 체크하고 Project url에 GitHub Url을 입력
6. General - Build Triggers - GitHub hook trigger for GITScm polling 체크
7. Pipeline Script 작성

Pipeline을 통해 Script 파일을 작성하면 차례대로 이벤트를 진행시킬 수 있다.
Script의 옵션은 2가지 인데
- Pipeline Script : Jenkins 웹 내에서 Script를 작성하여 관리
- Pipeline Script from SCM : 프로젝트 내에서 Jenkinsfile에 Script를 작성하여 관리

```
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: '[브랜치명]', : [GitHub Credential ID], url: '[본인 Github Repository]'
            }
        }

        stage('Build Gradle') {
            steps {
                echo 'Build Gradle'
                dir('.') {
                    sh 'chmod +x gradlew'
                    sh './gradlew clean build'
                }
            }
        }

        stage('Docker Image Build') {

            steps {
                echo 'Build Docker'
                script {
                    app = docker.build("[Docker Hub ID]/[이미지 이름]")
                }
            }

            post {
                failure {
                    error 'This pipeline stops here...'
                }
            }
        }

        stage('Push Docker') {
            steps {
                echo 'Push Docker'
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }

            post {
                failure {
                    error 'This pipeline stops here...'
                }
            }
        }

        stage('Docker Run') {
            steps {
                echo 'Pull Docker Image & Docker Image Run'
                sshagent(credentials: ['ssh']) {
                    script {
                        sh "ssh -o StrictHostKeyChecking=no root@[Spring Boot Server IP주소] 'docker pull [Docker Hub ID]/[이미지 이름]:[태그]'"
                        sh "ssh -o StrictHostKeyChecking=no root@[Spring Boot Server IP주소]'docker rm -f [컨테이너 이름]'"
                        sh """
                        ssh -o StrictHostKeyChecking=no root@[Spring Boot Server IP주소] 
                        'docker run -d --name [컨테이너 이름] \
                        -p 8080:8080 \
                        [Docker Hub ID]/[이미지 이름]:[태그]'"""
                        def containerId = sh(script: "ssh -o StrictHostKeyChecking=no root@[Spring Boot Server IP주소]'docker ps -q --filter name=[컨테이너 이름]'", returnStdout: true).trim()
                        echo "Container ID: ${containerId}"              
                        sh "ssh -o StrictHostKeyChecking=no root@[Spring Boot Server IP주소] 'docker exec ${containerId} env'"
                    }
                }
            }
        }
    }
}
```

위와 같이 Pipeline Script를 작성하여 저장한다.
Docker Image 를 선언하는 것 중 [태그]는 입력하지 않을 경우 latest로 처리된다.
추가로 Docker에 환경변수를 추가할 경우 run 명령에 -e 로 환경변수를 추가하고 General에 This project is parameterized 항목을 체크하면 된다.

### **프로젝트 Dockerfile 추가**

Dockerfile은 새로운 Docker Image를 만들기 위한 명령을 포함하고 있는 파일이다. 이 파일에 작성된 명령어를 통해 새로운 환경의 Image를 만들 수 있다.
Spring Project에 최상단에 Dockerfile을 생성하면 해당 Dockerfile을 통하여 build 할 수 있다.


```Dockerfile
FROM azul/zulu-openjdk:17.0.10-jre
  
ARG JAR_FILE=/build/libs/session-storage-1.0.jar
  
COPY ${JAR_FILE} /session-storage.jar  
  
ENTRYPOINT ["java","-jar","-Dspring.profiles.active=develop", "/session-storage.jar"]
```

- FROM Docker Base Image
- ARG : 컨테이너 내에서 사용할 수 있는 변수를 지정
- COPY : 복사 명령
- ENTRYPOINT : 컨테이너가 시작되었을 때 스크립트 실행

### **CI / CD 동작 확인**

![](../assets/img/2024-04-19-jenkins-docker-ci-cd/Success.png)

GitHub Push 에 따라 순차적으로 위 Pipeline이 동작하고 Spring Boot Server에서 해당 jar가 실행되는 것까지 확인하여 구성을 완료했다.

### **마무리**

오늘은 Jenkins와 Docker를 이용한 CI / CD  Pipeline을 구축하였다. 현재는 단일 서버를 가지고 테스트를 하였는데 추후에 분산 서버를 활용하여 무중단 배포에 대해서도 알아보는 시간을 갖도록 할 예정이다. 그리고 Jenkins를 활용하지 않고 GitHub Action을 통한 CI / CD에 대해서도 추가로 알아보도록 하겠다.


