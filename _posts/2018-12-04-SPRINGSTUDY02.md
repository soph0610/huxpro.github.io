---
layout:     post
title:      "SpringStudy #2 : 오브젝트와 의존관계 -2"
date:       2018-12-04 18:00:00
author:     "Soph"
header-img: "img/posts/02.jpg"
catalog: true
tags:
    - Spring
    - toby's Spring
---

## 오브젝트 팩토리
  이전 코드에서 오브젝트 생성에 대한 책임을 떠넘기고 떠넘겨 UserDaoTest가 담당하게 되었다. 그러나 UserDaoTest는 말 그대로 동작을 테스트하기 위한 코드였기 때문에 오브젝트 생성을 그쪽에 넘기는 것은 UsetDaoTest에게 두가지 일을 떠넘긴 것이 된다. 그래서 이번에는 오브젝트 생성만을 담당하는 팩토리 클래스를 만들어 본다.
  
```java
public class DaoFactory {
	public UserDao userDao() {
		UserDao userDao = new UserDao(connectionMaker());
		return userDao;
	}
	public ConnectionMaker connectionMaker() {
		return new SimpleConnectionMaker();
	}
}

```

위와 같이 DaoFactory는 오브젝트의 생성만을 담당한다. 

```java
public class UserDaoTest {
	
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		
		UserDao dao = new DaoFactory().userDao();
		
		User user = new User();
		user.setId("soph");
		user.setName("ara");
		user.setPassword("iok");
		
		dao.add(user);
		
		System.out.println(String.format("%s 등록 성공", user.getId()));
		
		User user2 = dao.get(user.getId());
		System.out.println(user2.getName());
		System.out.println(user2.getPassword());
		
		System.out.println(String.format("%s 조회 성공", user2.getId()));
		
	}
}
```

이제 UserDaoTest에서는 어떤 연결인지 알필요 없이 DaoFactory에서 생산한 UserDao를 사용만 하면 된다 !
즉 오브젝트 생성의 제어권이 서비스 로직에서 전혀 필요없게 된 것이다.
순수 자바로만 짜여진 이 코드로도 기능이 잘 분리되어 있고 IoC를 구현하고 있지만, 이제부터는 스프링에서 제공해주는 기능을 사용해보도록 한다.

먼저 스프링의 기능을 이용하기 위해서 라이브러리를 추가해주거나, maven을 사용할 수 있는데 본인은 maven을 사용하였다. 기존 프로젝트는 자바 프로젝트였기 때문에, 우선 Eclipse에서 프로젝트를 maven project로 바꿔주었다.

## Maven 설정

<img src="https://soph0610.github.io/img/posts/springstudy02_01.png"/>

위 캡처와 같이 프로젝트 이름 위에서 마우스 우클릭 후 , Configure를 누르면 오른쪽에 Convert to Maven이 나온다. 클릭하면 Maven Project로 바뀌면서 pom.xml이 자동생성된다. 프로젝트 패키지를 war로 선택했을 경우, 프로젝트에 web.xml 파일이 없기 때문에 pom.xml에 build 에러가 난다. 이런 경우 빈 web.xml 파일을 생성하거나, first child depth로 아래 태그를 넣어주면 해결된다.

```xml
<properties>
  	<failOnMissingWebXml>false</failOnMissingWebXml>
</properties>
```

그리고 토비의 스프링 프로젝트 진행을 위해 필요한 dependency는 아래와 같다.

```xml
<!-- mariaDB Client -->
<dependency>
    <groupId>org.mariadb.jdbc</groupId>
    <artifactId>mariadb-java-client</artifactId>
    <version>2.3.0</version>
</dependency>
 <!-- https://mvnrepository.com/artifact/org.springframework/spring-context -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>3.0.7.RELEASE</version>
</dependency>
<!-- https://mvnrepository.com/artifact/cglib/cglib -->
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.2.0</version>
</dependency>
<!-- https://mvnrepository.com/artifact/commons-logging/commons-logging -->
<dependency>
    <groupId>commons-logging</groupId>
    <artifactId>commons-logging</artifactId>
    <version>1.1.1</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.springframework/org.springframework.asm -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-asm</artifactId>
    <version>3.0.7.RELEASE</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.springframework/org.springframework.beans -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>3.0.7.RELEASE</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.springframework/org.springframework.core -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>3.0.7.RELEASE</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.springframework/org.springframework.expression -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-expression</artifactId>
    <version>3.0.7.RELEASE</version>
</dependency>
```

준비가 되었다면, 이제 DaoFactory를 스프링 빈팩토리가 관계를 짝지어줄 수 있도록 빈 설정 객체로 바꾸어주자.

## 스프링 어노테이션을 사용하여 빈(bean) 객체 만들기

- 스프링 빈 : 스프링 컨테이너가 생성과 관계설정, 사용 등을 제어해주는 제어의 역전이 적용된 오브젝트
- 빈 팩토리 : 빈의 생성과 관계설정와 같은 제어를 담당하는 IoC 오브젝트
- 빈 : 스프링이 만들고 관계를 짝지어주는 오브젝트

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DaoFactory {
	
	@Bean
	public UserDao userDao() {
		UserDao userDao = new UserDao(connectionMaker());
		return userDao;
	}
	
	@Bean
	public ConnectionMaker connectionMaker() {
		return new SimpleConnectionMaker();
	}

}
```

@Configuration은 스프링 빈 팩토리에서 Bean을 탐색할 수 있도록 Bean 설정 객체라는 것을 명명해주는 어노테이션이다. DaoFactory를 설정 객체로 명시한 다음, @Bean 어노테이션으로 오브젝트를 생성해주는 메서드를 빈으로 등록한다. 설정 객체는 스프링 컨텍스트 패키지의 ApplicationContext 인터페이스로 등록하여 사용할 수 있다.

```java
public class UserDaoTest {
	
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		
        //빈 팩토리가 찾을 설정 객체 등록
		ApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
        //설정 객체에서 사용할 빈 등록 및 리턴 타입 명시
		UserDao dao = context.getBean("userDao", UserDao.class);
		
		User user = new User();
		user.setId("soph");
		user.setName("ara");
		user.setPassword("iok");
		
		dao.add(user);
		
		System.out.println(String.format("%s 등록 성공", user.getId()));
		
		User user2 = dao.get(user.getId());
		System.out.println(user2.getName());
		System.out.println(user2.getPassword());
		
		System.out.println(String.format("%s 조회 성공", user2.getId()));
		
	}
}
```

코드를 실행시키면, 순수 자바 어플리케이션이었던 이전과 다른 로그를 확인 할 수 있다.

```
12월 04, 2018 4:13:16 오후 org.springframework.context.support.AbstractApplicationContext prepareRefresh
정보: Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@2dda6444: startup date [Tue Dec 04 16:13:16 KST 2018]; root of context hierarchy
12월 04, 2018 4:13:16 오후 org.springframework.beans.factory.support.DefaultListableBeanFactory preInstantiateSingletons
정보: Pre-instantiating singletons in org.springframework.beans.factory.support.DefaultListableBeanFactory@67f89fa3: defining beans [org.springframework.context.annotation.internalConfigurationAnnotationProcessor,org.springframework.context.annotation.internalAutowiredAnnotationProcessor,org.springframework.context.annotation.internalRequiredAnnotationProcessor,org.springframework.context.annotation.internalCommonAnnotationProcessor,daoFactory,userDao,connectionMaker]; root of factory hierarchy
soph 등록 성공
ara
iok
soph 조회 성공
```

#스프링 빈의 스코프

  위와 같이 스프링 빈을 이용해 오브젝트를 관리하게 되면, 기본적으로 싱글톤 객체를 돌려받게 된다. 스프링이 한 번 초기화해주고 이후 수정이 필요 없는 단순한 읽기 전용 객체라면 상관없겠지만, 매번 새로 생성해서 사용해야 하는 객체의 경우 싱글톤 빈으로 사용하게 되면 멀티 스레드 환경에서 오류를 발생시킬 수 있다. 따라서 스프링의 싱글톤 빈으로 사용되는 클래스를 만들 때는 개별적으로 바뀌는 정보는 로컬 변수로 정의하거나, 파라미터로 주고받으면서 사용하는 것이 좋다.
  
  이와 같이 빈이 쓰이는 범위를 빈 스코프라고 하는데, 싱글톤 외의 스코프도 지정할 수 있다.
  
  - 싱글톤(singleton) : 컨테이너 내의 한 개의 오브젝트 존재, 강제로 제거하지 않는 한 스프링 컨테이너가 존재하는 동안 계속 유지됨
  - 프로토타입(prototype) : 컨테이너에 빈을 요청할 때마다 매번 새로운 오브젝트 생성
  - 요청(request) : HTTP 요청이 생길때마다 새로운 오브젝트 생성
  - 세션(session) : 웹의 세션과 유사





