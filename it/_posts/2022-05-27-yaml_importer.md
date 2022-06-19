---
layout: post
date: 2022-05-27 00:00:00
title: Spring 프로퍼티 importer
tags: [Mac, ruby, rbenv, brew ]
comments: true
---

* toc
{:toc .large-only}
멀티모듈 방식의 Spring Boot 프로젝트를 만드는 도중 여러모듈에서 적용되는 공통되는 환경변수 값의 적용에 문제가 있었다.

가령, `module-A` 와 `module-common` 이 있을 때, `module-A` 는 `module-common` 을 참조한다. 이럴 때 `module-common` 의 환경변수 값을 `module-A` 에서도 참조하고 싶지만 뜻대로 되지 않을 것이다.

오늘은 이러한 경우에 환경변수를 주입하는 방법을 알아보려고 한다.



## EnvironmentPostProcessor

Spring 환경변수를 주입하는 `EnvironmentPostProcessor` 클래스를 하나 만들어보자.

 **.yaml** 파일로 환경변수를 주입하는 예제를 만들어본다.

class를 생성하고 2가지 인터페이스를 정의하도록 했다.

- `EnvironmentPostProcessor` : 환경변수 등록을 위한 인터페이스
- `Ordered` : bean 입력순서를 정하기 위한 인터페이스

```java
@Slf4j
public class CustomYmlEnvironmentPostProcessor implements EnvironmentPostProcessor, Ordered {
    
  	@Override
    public int getOrder() {
      	//Bean 주입 순서를 가장 앞순서로 한다.
        return Ordered.LOWEST_PRECEDENCE;
    }
  
  	@Override
    public void postProcessEnvironment(ConfigurableEnvironment environment,SpringApplication application) {
      	//.yml 같은 파일은 리소스에서 가져오므로 리소스 로더를 준비한다.
        ResourceLoader resourceLoader = Optional
                .ofNullable(application.getResourceLoader())
                .orElseGet(DefaultResourceLoader::new);

        ResourcePatternResolver resourcePatternResolver = new PathMatchingResourcePatternResolver(resourceLoader);

        Resource[] resources = new Resource[]{};
        try{
          	//클래스패스의 .yml 파일을 모두 불러온다.
            resources = resourcePatternResolver.getResources("classpath*:*.yml");
        }catch(IOException e){
            log.error("{}", e.getMessage(), e);
        }
      	//불러온 yml 파일에서 모든 프로퍼티를 가져와서 환경변수로 등록한다.
        for(Resource resource : resources) {
            loadYaml(resource, environment)
                    .forEach(propertySource -> {
                        log.info("[Yaml Third Properties Load] file: {}", resource.getFilename());
                        environment.getPropertySources().addLast(propertySource);
                    });
        }
    }
  
  	//yml 파일에서 프로퍼티를 가져오기 위한 함수
  	private List<PropertySource<?>> loadYaml(Resource path, ConfigurableEnvironment environment) {
        if (!path.exists()) {
            throw new IllegalArgumentException("Resource " + path + " does not exist");
        }

        try {
            List<PropertySource<?>> defaultPropertySource = new ArrayList<>();

            List<PropertySource<?>> propertySourceList = new YamlPropertySourceLoader().load(path.getFilename(), path).stream().filter(
                    propertySource -> {
                        Binder binder = new Binder(
                                ConfigurationPropertySources.from(propertySource),
                                new PropertySourcesPlaceholdersResolver(environment)
                        );
                      	//프로파일 등록
                        String[] profiles = binder.bind(SPRING_PROFILES, Bindable.of(String[].class))
                                .orElse(new String[0]);

                        if (profiles.length == 0) {
                            defaultPropertySource.add(propertySource);
                            return false;
                        }
                        return environment.acceptsProfiles(Profiles.of(profiles));
                    }).collect(Collectors.toList());

            propertySourceList = sortProfilePriority(environment, propertySourceList);
            propertySourceList.addAll(defaultPropertySource);

            return propertySourceList;
        }
        catch (IOException ex) {
            throw new IllegalStateException("Failed to load yaml configuration from " + path, ex);
        }
    }
  
  	//프로파일에 따라 입력순서를 정하기 위한 함수
  	private List<PropertySource<?>> sortProfilePriority(ConfigurableEnvironment environment, List<PropertySource<?>> propertySourceList) {
        String[] activeProfiles = environment.getActiveProfiles();

        return IntStream.range(0, activeProfiles.length)
                .map(i -> activeProfiles.length - i - 1)
                .mapToObj(i -> activeProfiles[i])
                .filter(Objects::nonNull)
                .map(profile -> propertySourceList.stream().filter(propertySource -> matchProfiles(profile, propertySource)).findAny().orElse(null))
                .filter(Objects::nonNull)
                .collect(Collectors.toList());
    }
}
```



## spring.factories

spring boot 는 Auto-configuration 수행을 위해서 boot가 실행될 때 클래스패스에 있는 `spring.factories` 파일을 찾아서 `spring.factories` 파일 내의 bean 생성 클래스를 실행한다.

이러한 원리를 이용해서 spring 환경변수 등록 클래스인 `EnvironmentPostProcessor` 를 정의해서 bean 등록해주면 spring 환경변수를 원하는 시점에 등록할 수 있다.

아까 생성했던 `CustomYmlEnvironmentPostProcessor` class를 `spring.factories` 에서 등록하자.

```properties
# file: ".spring.factories"

org.springframework.boot.env.EnvironmentPostProcessor=org.test.importer.CustomYmlEnvironmentPostProcessor
```





이러면 끝이다. 내가 생략한 설명이 많은데 요는 이거다.

Spring boot의 bean 생성 주기를 이해하고 기본 bean 생성을 위한 인터페이스 정의를 알고있다면, boot 설정이 정말 쉬워진다.

이 글에서는 spring boot 환경변수 등록을 위한 예제를 작성했지만, spring boot의 기본 bean 주입 순서에 대한 이해가 필요할때 `spring.factories` 에 등록된 클래스를 따라가보면 좋을 것 같다.

