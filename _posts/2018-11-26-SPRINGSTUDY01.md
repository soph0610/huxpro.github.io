---
layout:     post
title:      "SpringStudy #1 : 오브젝트와 의존관계 -1"
date:       2018-11-22 18:00:00
author:     "Soph"
header-img: "img/posts/02.jpg"
catalog: true
tags:
    - Spring
    - toby's Spring
---

## 시작하기 전에
  토비의 스프링 책을 보고, 공부한 것을 정리하고자 하는 시작하는 기록. 꾸준히 해서 책을 완독할 수 있기를 ! 
  
## 스프링이란 무엇일까 ?
- 스프링은 EJB라는 자바 개발 표준 기술이 불필요하게 복잡해지면서 객체지향언어라는 자바의 장점을 잘 살리지 못하자, 객체지향언어에 맞는 개발을 지향하기 위해서 나타난 프레임워크이다. 따라서 스프링은 POJO(Plain Old Java Object) 프로그래밍을 지향한다.
- 자바 엔터프라이즈 기술의 혼란속에서 잃어버렸던 객체지향 기술의 진정한 가치를 회복시키고 객체지향 프로그래밍이 제공하는 폭넓은 혜택을 누리자.

## 오브젝트와 의존관계
일단 스프링을 사용하지 않고 DAO(Data Access Object)를 만들어본다. 처음에는 UserDao 클래스 안에서 모든 걸 해결하는 코드를 짜고, 기능을 점점 분리해가며 클래스 간의 의존관계를 최소화하는 작업을 진행한다.

- 상위 클래스를 작성하고 추상 메소드를 하위 클래스에서 구현하는 식으로 디자인할 수도 있지만, 자바에서는 다중 상속을 금지하고 있어 여러 기능을 상속받고 싶을 경우 문제가 될 수 있다. 또한 상속 관계도 결국 어떤 클래스를 구현해야 하는지 하위 클래스에서 지정해주어야 하므로 클래스 간의 의존관계를 극복할 수는 없다.

- 의존 관계를 해결하려고 하는 것은 결국, 클래스 간 서로 영향을 주지 않은 채로 각각 필요한 시점에 독립적으로 변경할 수 있게 하기 위해서이다.

- 클래스 의존 관계 해결은 아래와 같은 원칙과 패턴을 사용했다.
1. 개방폐쇄원칙 (OCP, Open-Closed Principle)
: 클래스나 모듈은 확장에는 열려있어야 하고, 변경에는 닫혀있어야 한다. 예제 코드에서는 DB 연결은 인터페이스를 통해 제공되어 확장이 가능하고, DB 연결 외에 데이터 조작은 DAO 클래스 안에서만 해결하도록 변화에 닫혀있다. 

2. 높은 응집도과 낮은 결합도
-높은 응집도  : 하나의 공통 관심사는 한 클래스에 모은다. 즉 기능을 분리시켜, 변화가 일어나는 모습을 한눈에 알아볼 수 있게 하는 것이다.
-낮은 결합도 : 결합도란 하나의 오브젝트가 변경이 일어날 때 관계를 맺고 있는 다른 오브젝트에게 변화를 요구하는 정도로, 결합도가 낮다는 것은 하나의 변경이 다른 모듈과 객체에 영향을 주지 않는다는 것이다. 

```java
//사용자 정보 클래스
public class User {
	String id;
	String name;
	String password;
	
	public String getId() {
		return id;
	}
	public void setId(String id) {
		this.id = id;
	}
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	public String getPassword() {
		return password;
	}
	public void setPassword(String password) {
		this.password = password;
	}
}
```

```java
//사용자 정보 삽입, 조회 클래스
public class UserDao {
	
	private ConnectionMaker connectionMaker;
	
    //생성자에서 인터페이스로 오브젝트를 받기 때문에, 클래스 구현에 대해 전혀 알 필요가 없다.
    //클래스 구현, 오브젝트 생성은 클라이언트에게 떠맡긴 상태
	public UserDao(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker;
	}
	
	public void add(User user) throws ClassNotFoundException, SQLException {
		Connection c = connectionMaker.makeConnection();
		PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());
		
		ps.executeUpdate();
		
		ps.close();
		c.close();
	}
	
	public User get(String id) throws ClassNotFoundException, SQLException {
		Connection c = connectionMaker.makeConnection();
		PreparedStatement ps = c.prepareStatement("select * from users where id = ?");
		ps.setString(1, id);
		
		ResultSet rs = ps.executeQuery();
		rs.next();
		User user = new User();
		user.setId(rs.getString("id"));
		user.setName(rs.getString("name"));
		user.setPassword(rs.getString("password"));
		
		rs.close();
		ps.close();
		c.close();
		
		return user;
	}
}
```

```java
//데이터베이스 연결을 명시한 인터페이스
public interface ConnectionMaker {
	public Connection makeConnection() throws ClassNotFoundException, SQLException;
}
```

```java
//ConnectionMaker를 구현해 데이터베이스 연결 코드를 작성한 클래스
public class SimpleConnectionMaker implements ConnectionMaker {
	public Connection makeConnection() throws ClassNotFoundException, SQLException {
		Class.forName("org.mariadb.jdbc.Driver");
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook", "spring", "book");
		return c;
	}
}

```

```java
//객체를 생성해 DAO에 주입하는 클라이언트 클래스
public class UserDaoTest {
	
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		
        //Mysql 연결을 위해 SimpleConnectionMaker 객체를 생성하여 주입 
		UserDao dao = new UserDao(new SimpleConnectionMaker());
		
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

  위 코드를 통해, 데이터베이스에서 데이터를 수정, 조회하는 UserDao는 데이터베이스 연결에 대해 구체적인 구현을 전혀 몰라도 아무 문제가 없게 되었다. UserDao는 데이터 조작이라는 자신의 역할에 충실할 수 있게 된 것이다. 또한 데이터베이스 연결 코드에 수정사항이 생기더라도 UserDao까지 함께 테스트할 필요 없이 ConnectionMaker를 구현한 클래스에서 연결이 잘 되는지만 테스트 해보면 되는 것이다. 이렇듯 응집도를 높이고 결합도를 낮추는 것은 변경사항으로 발생할 수 있는 비용과 위험부담을 줄이는데 큰 도움이 된다.
  
  
- - -

추가로 알아볼 것 : SOLID 객체지향 설계 원칙
  