---
layout: post
date: 2022-05-16 10:50:00
title: Docker 설치 및 Remote Docker 사용하기
categories: [linux]
tags: [docker, linux]
comments: true
---

회사 공용서버의 kubernetes 환경에 이미지를 올릴 공간이 없어서 다른서버에 docker를 설치해서 remote docker 연결 하려고 한다.
개인 PC에 하려고 했지만 PC 성능이 좋지 못하다.. 내 윈도우에는 설치하기 싫어! 조만간 윈도우 docker 지원도 중지한다고 하던데, 암튼 쾌적한 환경을 원한다.


## Docker 설치
먼저 yum을 업데이트하고 패키지를 설치하자
~~~
yum -y update
yum install -y yum-utils
~~~

yum 환경에 docker 패키지가 있는 저장소를 등록해준다
~~~
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum-config-manager --enable docker-ce-nightly
~~~

이제 docker 패키지를 설치!
~~~
yum -y install docker-ce docker-ce-cli containerd.io
~~~

설치가 끝났으면 docker를 실행해보자
~~~
systemctl start docker

# 서버 부팅시 docker 자동실행 등록
systemctl enable docker

# 실행되었는지 확인
systemctl status docker
~~~


## Docker Compose 설치
docker로는 컨테이너를 하나씩 밖에 실행하지 못한다. 실행시 명령어들도 매번 입력하기 힘들것이다.
그래서 docker-compose 를 사용하면 docker 컨테이너 실행을 편리하게 관리할 수 있다.
~~~
# docker-compose 설치
$ sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# docker-compose 실행 권한 부여
$ sudo chmod +x /usr/local/bin/docker-compose

# 실행 확인
$ docker-compose --version
~~~


## Remote Docker 사용
docker가 설치된 리눅스 서버에서 docker 명령어를 사용하면 잘될것이다.
하지만, 다른 서버에서 원격으로 docker를 사용하려고 하면 안된다.
왜냐하면, 기본 설정된 systemctl 설정으로 docker 를 실행하면 외부 포트를 열어주지 앟는다.
그래서 system 데몬으로 docker의 포트를 열어줘야 한다.

service에 등록된 docker의 데몬설정을 바꿔보자
~~~
vi /lib/systemd/system/docker.service
~~~

`docker.service` 의 `ExecStart` 항목을 수정해준다.
`-H 0.0.0.0:2375` 옵션으로 2375 포트로 데몬을 접근할 수 있게 해준다.
~~~
ExecStart=/usr/bin/dockerd -H fd:// -H 0.0.0.0:2375 --containerd=/run/containerd/containerd.sock
~~~

이제 설정은 끝났고 데몬을 로드하고 재시작해준다
~~~
systemctl daemon-reload
systemctl restart docker
~~~

이제 외부에서도 docker 명령어가 잘될듯?