---
layout: post
date: 2022-05-17 00:00:00
title: Docker로 jenkins 설치하기
categories: [linux]
tags: [docker, linux, jenkins, CI/CD]
comments: true
---

 회사 쿠버네티스에는 다른 CI/CD가 있어서 개인적으로 CI/CD를 구성해보려고 한다. CI/CD 도구는 래퍼런스도 많고 널리 쓰이는 jenkins를 선택했다.
다들 약어로 '씨아이씨디'라고 말하지만 초보개발자를 위해 간단하게 CI/CD에 대해서도 알아보자

# CI/CD
- CI (Continuous Integration) : **지속적 통합**이라고 하며, 새로운 코드변경이 있으면 빌드 및 테스트 과정이 자동으로 진행하고 공유된 저장소에 통합하는 것을 의미한다.  
    쉽게 얘기하면 `빌드/테스트 자동화`
- CD (Continuous Deployment / Continuous Deployment): **지속적인 서비스 제공/지속적인 배포**이라고 하며, 이전의 CI 과정을 포함하여 제품을 배포 까지 이어지는 것이다.
    제품에 이슈가 있거나 프로그램 버그가 있을때, 빠르게 복구하고 재반영하는 과정을 신속하고 정확하게 할 수 있도록 한다.
  

간단하게 `CI`는 `커밋->빌드->테스트` , `CD`는 `커밋->빌드->테스트->배포` 라고 볼 수 있다.
나는 길게 얘기하는걸 잘못해서 이정도만 하겠다.
사실, 더 길게 써봤자 실제로 커밋->빌드->테스트->배포 과정을 수동으로 겪어보지 않으면 필요성을 못느낄지도 모른다.


## 설치
docker로 jenkins를 구동할 예정이기 때문에 docker가 기본적으로 구성된 linux 환경에서 진행한다.

나는 컨테이너의 볼륨을 설정할 위치에 **docker-compose** 구성을 할 생각이다.

#### docker-compose.yml
볼륨위치는 `/home/jenkins/` 로 정했다. 해당 위치에 `docker-compose.yml` 을 생성하자
~~~
version: '3.7'

services:
  jenkins:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: jenkins
    restart: always
    user: root
    ports:
      - 8080:8080
      - 50000:50000
    expose:
      - 8080
      - 50000
    volumes:
      - /home/jenkins:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      TZ: "Asia/Seoul"
~~~
services.jenkins 설정만 보자
- container_name: 실행할 컨테이너명
- dockerfile: 컨테이너를 실행할때 Dockerfile 을 실행시킨다.
- user: 컨테이너를 실행할 계정
- ports: 컨테이너 외부(호스트)에서 접근할 수 있는 포트 포워딩 설정 `{호스트 포트}:{컨테이너 내부 포트}`
- expose: 외부에 공개할 포트를 지정한다. 위의 ports 옵션을 사용한다면 굳이 하지 않아도 된다.
- volumes: 컨테이너 외부(호스트)와 컨테이너 내부의 데이터를 바인딩. `{호스트 위치}/{컨테이너 내부 위치}`

젠킨스는 컨테이너 내부에 `/var/jenkins_home`에 설치되므로 아까 컨테이너 외부에 생성한 `/home/jenkins` 디렉토리와 바인딩했다.
추가로, `/var/run/docker.sock` 를 바인딩한 이유를 말해보고자 한다.

그냥 jenkins만 설치하면 컨테이너 내부에선 docker가 없기 때문에 jenkins로 docker 명령어를 실행할 수 없다.
그래서 호스트의 docker를 볼륨 마운트로 연결한 것이다. 이를 `Docker out of docker(DooD)` 라고 하더라..
`Docker in docker` 방식도 있지만 `DooD` 방식을 권장하기 때문에 이대로 진행했다.  


### docker_install.sh
Dockerfile을 생성하기 앞서, 쉘파일을 하나 만든다.
쉘파일을 만드는 이유는 앞으로 컨테이너 내부에 설치가 필요한 패키지 들이 있을지도 모르는데 매번 Dockerfile에
복잡하게 명령어를 작성하지 않기 위해서다.

`docker_install.sh` 파일명을 보면 알겠지만 docker를 설치하기 위한 쉘이다.
아까 docker 볼륨을 마운트했지만 이는 docker 데몬만 연결한 것 뿐이고, 컨테이너 내부에 docker-cli 도 있어야
컨테이너의 Agent가 docker를 실행할 수 있다.

~~~
#!/bin/sh

apt-get update && \
apt-get -y install apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     zip \
     unzip \
     software-properties-common && \
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg > /tmp/dkey; apt-key add /tmp/dkey && \
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID") \
   $(lsb_release -cs) \
   stable" && \
apt-get update && \
apt-get -y install docker-ce

~~~


### Dockerfile
이제 Dockerfile를 생성해보자. 마찬가지로 `docker-compose.yml`과 같은 위치로 `/home/jenkins/`에 생성한다.
~~~
FROM jenkins/jenkins:lts

#root 계정으로 로그인
USER root

#Docker in Docker 를 위해 내부에 docker 설치
COPY docker_install.sh /docker_install.sh

RUN chmod +x /docker_install.sh

RUN /docker_install.sh


#docker 와jenkins 명어러를 수행하기 위한 계정 등록
RUN usermod -aG docker jenkins $USER
USER jenkins
~~~
마지막 줄에서 jenkins와 docker를 실행할 user를 등록해주지 않으면 권한 문제로 실행이 안된다.


## 실행
이제 설치는 끝났으니 컨테이너를 실행해보자
~~~
docker-compose up -d
~~~
![](/assets/post/img_3.png)  
자, 완료 되었다고 한다. 아까 포트포워딩한 8080 을 브라우저로 열어보겠다.


jenkins를 설치하면 Unlock을 해달라고 한다.
![](/assets/post/img_4.png)  
unlock 비밀번호는 jenkins 설치할때 로그에서 알려주는데 스샷을 찍어놓은게 없다..
뭐 위치는 다 똑같다. `/var/jenkins_home/secrets/initialAdminPassword` 파일을 열어보면 비밀번호가 적혀있다.
아까 호스트에 바운드한 `/home/jenkins` 위치에도 똑같이 jenkins 파일들이 설치되므로 `/home/jenkins/secrets/initialAdminPassword` 파일이 있다.  

아까 페이지에서 비밀번호를 입력하면 jenkins 가 정상적으로 접속되고, admin 계정을 생성해서 사용하도록 하자.

### Git Credential 설정
CI/CD는 기본적으로 git 저장소를 기반으로 많이 쓰므로 기본설정으로 git 인증 설정을 해놓으면 편하다.
![](/assets/post/img_5.png)  
![](/assets/post/img_6.png)  
![](/assets/post/img_7.png)  
위와 같이 이동한 후에 `adding some credentials` 을 한다.

![](/assets/post/img_8.png)  
**kind** 항목에는 **Username with password** 를 선택하고 
**Scope** 에는 **Global** 로 한다.
**Username**  에는 **Git 계정**을 넣어주고
**Password** 에 **git accessToken** 을 넣어준다.
accessToken발급은 사용하는 git에서 찾아보시길!

이제 git 인증서는 등록 되었고, jenkins item을 생성하면
Git 저장소 연동 항목에서 credentials를 적용하면 잘 된다.
![](/assets/post/img_10.png)  

credentials이 적용되지 않으면 아래와 같이 오류가 발생한다.
![](/assets/post/img_9.png)  


## 마무리
jenkins 설치와 git 인증서 등록까지 해보았다. 오늘도 한가지는 배워간다.