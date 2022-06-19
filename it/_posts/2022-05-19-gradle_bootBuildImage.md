---
layout: post
date: 2022-05-19 09:10:00
title: Gradle-Spring bootBuildImage
description: 배포방법에 있어서 하나의 참고사항만 되었으면 좋겠다. 
tags: [spring, gradle]
comments: true
---

* toc
{:toc .large-only}

 
 프로젝트를 진행 할때면 **CI/CD** 방법론에 대한 고민이 항상 새롭게 찾아온다. 환경에 따라서도 다르겠지만, 이전보다 더 나은 방법은 없을까? 고민하게 되는것 같다.
오늘은 gradle 에서의 **빌드 및 배포** 방법중 하나를 소개해볼까 한다.  

# Cloud Native Buildpack (CNB)   

> [buildpakcs 사이트 참조](https://buildpacks.io/)   
> 통용되는 개념인지는 모르겠으나 Buildpacks 라는 사이트에서 공식적으로 지원한다고 말해준다.

OCI 호환 스펙으로 컨테이너 이미지를 빠르게 빌드 및 배포해주는 기술이다. 
docker를 사용해서 `컨테이너 빌드 -> 이미지 빌드 -> 저장소 등록` 과정을 거치려면 준비과정도 필요하고 다소 번거로울 수 있다.
또한, docker로 빌드 구성을 할 때, docker는 어플리케이션의 속성을 알지 못하므로 필요한 이미지 레이어 설정도 수동으로 해주어야한다.

`CNB`는 이런 과정들을 간략하게 축소하였다. 컨테이너를 생성함과 동시에 이미지를 만들고 저장소에 이미지도 push 해준다. 그리고 사용하는 어플리케이션 특성에 맞게 레이어를 생성하기 때문에
처음 빌드된 이후로는 빠른 빌드속도를 보여준다.



# bootBuildImage for Spring   

> [Spring 공식 플러그인 문서](https://docs.spring.io/spring-boot/docs/current/gradle-plugin/reference/htmlsingle/)   
> Springboot 2.3 버전 이상에서만 사용할 수 있다.

springboot를 위해 제공된 `CNB`인 `bootBuildImage`를 사용해보려고 한다.
bootJar로 생성된 jar파일을 OCI 컨테이너로 포함시켜 이미지로 만들고 개인저장소에도 올라가는지 본다.


## Gradle

### Plugin

예시에서는 spring의 2가지 plugin을 사용했다.

- **spring-boot-gradle-plugin** : 사용할 springboot 버전에 따라 포함된 task 를 사용할 수 있게 해주고, spring 종속성에 대해서 명시된 버전으로 강제할 수 있게 해준다. 
  여기에 bootBuildImage Task가 포함된다.
- **dependency-management-plugin** : 종속성을 통해 모듈을 가져올때, 명시된 spring 종속성에 맞게 모듈을 구성해준다. 
  spring 종속성 내부의 jar파일들과 충돌할 수 있거나 사용될 수 없는 모듈에 대한 오류를 방지할 수 있다. 

~~~groovy
//file: 'build.gradle'
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:2.5.3")
        classpath("io.spring.gradle:dependency-management-plugin:1.0.11.RELEASE")
    }
}

apply plugin: 'java'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'
~~~

이렇게 플러그인까지 등록하면 IDE를 사용한다면 task 목록에 `bootBuildImage`가 있을것이고
터미널로 확인하려면 `gradlew tasks`를 해보면 확인할 수 있다.   


### bootBuildImage 설정

bootBuildImage task에서 사용할 docker와 최종적으로 이미지를 push할 repository에 대한 설정을 해줄 수 있다.

- **imageName** : 생성될 이미지명을 정한다.
- **publish** : 이미지빌드가 끝나면 repository에 push를 자동으로 진행할지 여부이다.
- **publish** : 이미지 빌드에 사용할 docker에 대한 설정이다.
  - *host* : remote docker를 사용한다면 사용할 docker의 url를 입력한다.
  - *publishRegistry* : 이미지를 push할 repository에 대한 설정이다.
    - *username* : repository에 로그인할 ID
    - *password* : repository ID의 password
    - *url* : repository URL. 오직 https만 허용이 된다.

~~~groovy
//file: 'build.gradle'
buildscript {
    ext {
        namespace = "my"
        docker_host = "tcp://127.0.0.1::2375"
        docker_registryIp = "127.0.0.1"
        docker_username = "admin"
        docker_password = "wlwndgo"
    }
}

bootBuildImage {
    //곧바로 저장소에 push할 것이기 때문에  {저장소 IP}/{네임스페이스}/{프로젝트명}:{태그명} 형식으로 만들었다.
    imageName = "${docker_registryIp}/${kube_namespace}/${project.name}:${getVersion()}"
    publish = true
    docker {
        host = docker_host
        publishRegistry {
            username = docker_username
            password = docker_password
            url = "https://${docker_registryIp}/"
        }
    }
}
~~~


### gradle task 실행

task 실행은 너무 간단하다.
~~~
gradlew bootBuildImage
~~~

한 번의 설정으로 이렇게 간단하게 이미지를 publish 까지 할 수 있다.


## 단점

`CNB`에서 지원하는 spring boot 이미지화가 완전하지 않은지 이미지빌드된 시간에 오류가 있다.
나는 아무리해도 42년전에 빌드됬다고 한다..이미지빌드 시간을 정할 수 있는지 모르겠다.
다행인건, 저장소에 올라간 시간은 정상적으로 나온다.

docker 커맨드 삽입이 안되어서 bootJar만 컨테이너화 된다.
컨테이너에 포함할 동작이 있으면 따로 Dockerfile로 빌드해야 할 듯 하다.

차차 옵션이 추가될 예정이라는데, 아직은 기능이 부족해보인다.