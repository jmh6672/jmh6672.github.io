---
layout: post
date: 2021-12-18 10:50:00
title: Spring MongoDB Aggregation
tags: [spring, mongodb, gradle]
comments: true
---

* toc
{:toc .large-only}

Spring에서 공식적으로 제공하는 mongoDB 라이브러리는 JPA 기반이다.
mongoDB도 타 RDB 처럼 document 라는 쿼리문을 작성해서 직접 날릴수도 있겠지만 mybatis 처럼 좋은 매퍼를 제공해주지 않는다.
좋은게 좋은거라고 java로 편하게 쓸 수 있을거란 기대로 시작했는데..ㅎㅎ 쉬운건 없다.
한 번 보자

## Spring 설정

#### build.gradle
프로젝트에서 mongodb를 사용하기 위한 **dependencies**를 추가해준다. 나는 gradle을 사용해서 추가해주었다.
~~~ groovy
//file: 'build.gradle'
dependencies{
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    //for mongoDB
	implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
}
~~~

#### application.yml
**.properties** 파일도 되지만 나는 **yaml** 파일을 선호한다. 해당 옵션은 아래 예시와 같다.
프로퍼티값만 써주면 커넥션 관련 **bean**을 따로 작성하거나 할 필요없이 자동 생성된다.
~~~ yml
#file: 'application.yml'
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017/service
~~~



## MongoDB 설명

### Operation
operation 명령어 앞에는 꼭 `$`를 붙여야 한다. `$`가 붙지않는 경우는 operation의 필드나 설정값 등이다.
spring 에서는 `AggregationOperation` 인터페이스의 구현체를 사용한다. 대부분의 구현체가 static 으로 선언되어 있어서 사용하기 편하다. 

match operation을 예시로 들면 아래와 같이 선언할 수 있다.
~~~ java
AggregationOperation operation1 = Aggregation.match(Criteria.where("name").is("test"));
MatchOperation operation2 = new MatchOperation(Criteria.where("email").is(email));
~~~


#### $match
조회 조건을 넣어주는 연산이다.
Criteria 클래스를 파라미터로 사용하며 Criteria에서 조회 조건을 추가해줄 수 있다.
~~~ java
MatchOperation match = Aggregation.match(
		Criteria.where("_id").is(new ObjectId(_id))
	);
~~~

#### $lookup
RDBMS 의 join과 유사한 역할을 하지만, 조건을 추가하거나 document의 일부 필드만 가져오고 싶으면 추가로 연산이 필요하다. mongoDB에서는 연산을 복잡하게 하는것을 권장하지 않으며 실제로 성능도 안좋아진다. 이 때문인지 spring에서도 많은 기능을 지원하지 않는 모양이다. mongoDB에선 lookup 연산에 **pipeline**을 넣을 수 있도록 되어있지만 spring 에서 지원하진 않는다.
- **from** : 조회할 collection
- **localField** : 현재 collection의 비교값이 되는 필드
- **foreignField** : 조회할 collection의 비교값이 되는 필드
- **as** : 조회된 데이터가 보여질 필드명을 정한다.

~~~ java
LookupOperation lookup = LookupOperation.newLookup()
                .from("account")	//account 컬렉션에서 조회
                .localField("tntId")
                .foreignField("_id")//lookup할 조건이 되는 account 컬렉션의 필드명
                .as("account");		//lookup 해 온 데이터를 account 필드명로 보여준다.
~~~

#### $unwind
**Array 필드**를 분리해서 각각의 **element**로 만들어 **document**로 넣어주는 연산이다. 
아래와 같이 파라미터가 정의되어 있다.
- **field** : element로 분리할 array 필드명
- **arrayIndex** : 분리한 element를 document로 합칠때 나타낼 필드명이다.
- **preserveNullAndEmptyArrays** : 필드의 값이 없을때 document에 포함할지 여부이다. **true** 이면 빈값이라도 document가 나오지만 **false** 이면 document 가 나오지 않는다.
**기본값은 false** 이다.
~~~ java
// method 정의
Aggregation.unwind(String field, String arrayIndex, boolean preserveNullAndEmptyArrays);
// 사용 예시
UnwindOperation unwind = Aggregation.unwind('item','item',true);
~~~

### Aggregation
직역하면 **집계** 인데, `operation` 이라고 하는 **연산**의 모음이라고 할 수 있다. 이 `operation`을 순서대로 수행한 결과를 나타내어 준다. 수행 순서가 있으므로 `pipeline` 이라고도 한다.

아래와 같이 **MongoTemplate.aggregate**를 실행하여 결과를 받는다.
~~~ java
// method 정의
Aggregation.newAggregation(AggregationOperation... operations);
// 쿼리 수행 예시
return mongoTemplate.aggregate(
	Aggregation.newAggregation(match,lookup,unwind),
	"COLLECTION",
	HashMap.class
);
~~~

https://docs.mongodb.com/manual/reference/operator/aggregation/   
MongoDB 공식 문서를 보면 상당히 많은 연산 문법이 있다.
복잡하지만 차근차근 공부해보려고 한다.