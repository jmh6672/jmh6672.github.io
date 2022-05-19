---
layout: post
date: 2021-12-16 10:50:00
title: Gradle 멀티모듈 프로젝트
tags: [spring, gradle]
comments: true
---

* toc
{:toc .large-only}

**모놀리식(Monolithic)** 아키텍처로 운영되고 있던 웹사이트가 있었다.
서비스 관리자 웹이었는데, 기능확장이 필요하게 되었을때 수정도 어렵고, 패치할때 재시작 해야해서 일시적인 서비스 중단이 일어나기도 했다.
 문제는 서버에 로그 하나만 추가해보려고 재시작해야 하는거... 약간의 기능을 추가 한다고 전체 서비스를 내릴 필요는 없기 때문에 필요한 기능만큼 API 모듈을 만들기로 했다. 이렇게 **MSA(Micro Servie Architecture)**가 시작된다.
 
 처음엔 쓸때없이 설계에 대한 고민을 많이 했다. 기존의 모놀리식 프로젝트에서 필요한 기능을 분리하고 싶었는데 기존 프로젝트는 생각이상으로 개판이었다. 오직 단일 프로젝트를 위한 구조였기 때문에 시간이 부족한 나는 그냥 전체를 새로 만들기로했다.
 
결과는 완전히 망했다. 기존 프로젝트에서 필요한 기능을 복붙해오고 늘어나는 모듈에 똑같은 소스를 붙여넣는 짓거리를 하고 있었다. 모듈 간에 연동 및 통신도 필요해지니까 모듈간 상관관계가 개판이 되어갔다.

그래서 이 프로젝트를 통합하기로 했다. 모듈의 상하관계를 나눌 수 있는 멀티모듈 gradle로!
   

### 왜 해야하나?
 왜 이걸 하는지 모르면 하기 싫지 않은가? 새로운건 항상 귀찮거든...이유도 없는데 왜 해?
 그래서 왜 멀티모듈이 좋냐고?
 
 1. 소스 관리하기가 용이하다.
 - 각 모듈별로 소스관리를 하면 소스가 여기저기 엎어지는 경우는 거의 없을 것이다. 다수가 참여하는 프로젝트의 경우는 소스 복구와 버전업/다운 에서 이런 장점이 더 잘 드러난다.
    
 2. 모듈간 관계성 정의
 - 빌드 및 종속성 추가, 모듈간 연관 관계등을 정의하기가 좋다.
 
 3. 재사용과 소스 공유
 - 한 번 만들어놓은 기능을 반복해서 새로운 모듈에 만들필요가 없다. 필요한 기능의 모듈을 의존성으로 추가하기만 하면 된다.
    
 4. 목적의 세분화와 의존성 최소화
 - 서비스를 운영하다보면 이것도 하고싶고 저것도 하고싶다. 그렇다고 뭔가 시도해보기는 부담이 많을 수 있다. 하지만, 멀티모듈은 추가 개발의 부담을 줄여준다. 모듈의 추가나 수정이 서비스에 미치는 영향도가 적고 오류의 원인 추적도 쉽다. 특정 목적에 맞게 모듈을 세분화 하고 다른 모듈에 대한 의존성을 적게 가져가기 때문이다.


## 프로젝트 구성을 해보자

### 루트 프로젝트 생성
우선 gradle 프로젝트를 생성한다.   
![](/assets/post/img.png)   
위 이미지에 보이는 프로젝트는 멀티모듈의 루트 모듈이 된다.
위 구조에서 보이는 `gradle/wrapper` , `gradlew` , `settings.gradle` 은 루트 모듈에만 존재하게 될텐데, 이는 모든 모듈에서 똑같은 build 환경을 사용하기 위함이다.


### 서브모듈 추가
이제 서브모듈을 추가해본다.   
![](/assets/post/img_1.png)   
서브모듈에는 소스파일과 `build.gradle` 만 추가하고 `settings.gradle` 에 서브모듈을 include 한다.
~~~ java
rootProject.name = 'multi_module'
include 'sub_module_A'
include 'sub_module_B'
~~~
여기까지는 모듈 간의 의존성을 부여할 수 있도록 하기 위한 준비과정이다.

이제 외존성 정보를 공유하도록 해본다. `루트 모듈의 build.gradle` 파일을 수정한다.
~~~ java
subprojects {
    apply plugin: 'java'

    group 'org.example'

    repositories {
        mavenCentral()
    }

    dependencies {
        testImplementation 'org.junit.jupiter:junit-jupiter-api:5.7.0'
        testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.7.0'
    }

    test {
        useJUnitPlatform()
    }
}
~~~
`subprojects` 는 서브모듈에 공통으로 적용되는 의존성을 지정할 수 있다. 플러그인, 저장소, 공통 그룹, 테스트 환경까지 동일하게 조성가능하다.


### 모듈간 의존성 주입
이제 `sub_module_A`와 `sub_module_B`를 각각 개발한다고 해보자, 서로 고통으로 필요한 로직이 있을 수 있지만 서로 다른 모듈이기 때문에 동일한 로직을 복붙해서 개발하게 되면 나중에 수정도 어렵고 코드를 복붙하면 실수도 있을 수 있다. 그래서 `common_module`을 추가해서 공통로직을 처리하도록 하자.   
![](/assets/post/img_2.png)   
위와 같이 모듈을 추가하고 `settings.gradle`에 모듈을 include 하였다.

`sub_common 모듈의 build.gradle` 에서 컴파일시 jar 파일을 생성하도록 설정한다.
~~~ java
jar { enabled = true }
~~~

그리고 `루트 모듈의 build.gradle` 에서 각 서브모듈에 공통모듈을 의존성으로 추가해준다.
~~~ java
project(':sub_module_A') {
    dependencies {
        implementation project(':sub_common')
    }
}
project(':sub_module_B') {
    dependencies {
        implementation project(':sub_common')
    }
}
~~~
모듈간 의존성 추가는 위와 같이 지정해주며, common 모듈이 필요치 않는 모듈에서는 의존성을 제거하는게 좋다. 다른 케이스로 `sub_module_A`의 로직을 `sub_module_B` 에서 필요할때 위와 같은 방법으로 의존성을 추가해서 사용한다.

gradle 멀티모듈 구성방법은 여기까지다. 

## 마치며
내가 작성한 예시는 매우 간단한 예제이지만 실제로 적용할때는 아키텍처의 분리 및 부분 통합이 가능하기 때문에 복잡한 구조로 설계가 가능하다. 개발을 진행하다보면 설계에 대한 의문점이 생길때가 많은데, 멀티 모듈은 설계구조를 바꾸기도 용이한 면이 있어서 능동적인 개발을 하기에도 참 좋은 구조라고 생각이 된다.
