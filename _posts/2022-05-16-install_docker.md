---
layout: post
date: 2022-05-17 10:50:00
title: Docker 설치 및 Repository 연결
description: Linux 환경에 docker 설치하기
categories: [linux]
tags: [docker, linux]
comments: true
---

회사 고용서버의 kubernetes 환경에 이미지를 올릴 공간이 없어서 다른서버에 docker를 설치해서 remote docker 연결 하려고 한다.
개인 PC에 하려고 했지만 PC 성능이 좋지 못하다.. 내 윈도우에는 설치하기 싫어! 조만간 윈도우 docker 지원도 중지한다고 하던데, 암튼 쾌적한 환경을 원한다.


## yum 으로 설치
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


## Remote Docker 사용
docker가 설치된 리눅스 서버에서 docker 명령어를 사용하면 잘될것이다.
하지만, 다른 서버에서 원격으로 docker를 사용하려고 하면 안된다.

