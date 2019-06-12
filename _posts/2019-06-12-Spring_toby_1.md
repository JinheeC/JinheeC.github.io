---
title: "토비의 스프링 3.1: 들어가며, 01. 오브젝트와 의존관계"
categories: 
- Spring
excerpt: |
  스프링이란 무엇인가?
  스프링은 어플리케이션 개발을 빠르고 효율적으로 할 수 있도록 해주는 프레임워크다. 빠르고 효율적일 수 있는 이유는 스프링이 개발에 필요한 모든 틀을 제공해 주고 개발자가 어플리케이션에 필요한 코드만 작성해주면 되기 때문이다. 
  스프링의 특징을 하나하나 살펴보자
feature_text: |
  Spring, SpringBoot 의 기초에 대해 공부해 보자
feature_image: "https://picsum.photos/2560/600?random"
---
해당 글은 예전 스프링 스터디 당시 스프링 기본서인 토비의 스프링 3.1 을 보고 작성한 글 입니다. 책 내용과 더불어 이해하기 쉽도록 첨언하였습니다.

* table of contents
{:toc .toc}

## 들어가며
스프링에 대해 이해하는 첫 시작인 만큼 가볍게 스프링이 무엇인지 훑어보면서 지나가보자.
### 스프링이란 무엇인가?
스프링은 어플리케이션 개발을 빠르고 효율적으로 할 수 있도록 해주는 `프레임워크`다. 빠르고 효율적일 수 있는 이유는 스프링이 개발에 필요한 모든 틀을 제공해 주고 개발자가 어플리케이션에 필요한 코드만 작성해주면 되기 때문이다. 

스프링의 특징을 하나하나 살펴보자
* **스프링 컨테이너**
	스프링은 스프링 컨테이너 (= application context) 라고 부르는 `런타임 엔진`을 제공한다. 스프링 컨테이너는 어플리케이션에 작성된 객체들을 생성하고 관리하는 엔진이라고 볼 수 있다. 모든 오브젝트를 관리하는 것은 아니고 뒤 챕터에서 자세히 배우겠지만 `@Bean, @Component, @Controller, @Service, @Repository` 과 같은 코드를 클래스나 method 레벨에 코드를 작성해서 스프링 컨테이너한테 이 객체를 관리해 달라고 알려야 한다. (뒤에 자세히 나오겠지만 이 작업을 "Bean 으로 등록한다" 라고 말한다. 그리고 스프링 컨테이너는 Bean Object 를 관리한다.)
* **IoC/DI, 서비스 추상화, AOP**
	- IoC (Inversion of Control): 한글로 말하면 `제어의 역전`이라고 말할 수 있다. 이 특징은 사실 스프링 뿐 아니라 모든 프레임 워크에도 적용되는 프레임워크의 특징이다. 제어란 정말 말 그대로 코드의 제어권이다. 프레임워크를 사용하게 되면 개발을 위한 틀이 제공되고 해당 어플리케이션에 필요한 코드(비즈니스로직)만 개발자가 작성하는데, 이때 프레임워크가 개발자의 코드(객체)를 가져다 사용하는 형태가 된다. 즉 제어권이 개발자가 아닌 프레임워크에게 역전된다. 라이브러리와 비교하면 이해가 쉽다. 라이브러리의 경우 개발자가 라이브러리를 가져다 사용하는 형태이다. 그런데 프레임워크는 그 반대로 제어가 역전되어 개발자의 코드를 라이브러리 처럼 프레임워크가 사용하는 형태가 된다.
	- DI (Dependency Injection): `의존관계 주입` 이라는 것은 스프링 IoC 특징을 좀 더 좁힌 설명이다. 그러나 스프링만의 특징은 아니다. 의존관계라는 것은 객체와 객체의 의존관계를 말한다. 객체지향 코드에서는 관계를 느슨하게 코드를 작성하라고 한다. 관계가 느슨하지 않다는 것은 하나의 클래스 A 안에서 다른 클래스 B의 객체를 만드는 상황을 말한다. 다른 클래스 B의 정의와 구현이 달라지면 관계가 없던 클래스 A의 내부의 코드도 바뀌게 된다. 이를 위해 이런 의존성을 클래스 A 밖에서 주입해야 한다. 클래스 A 에서는 B 객체의 코드를 사용해야 하므로 B 객체의 인터페이스를 대신 사용해서 코드를 작성하고 런타임 시 해당 객체를 받도록 코드를 작성해야 한다. 이럴 경우 클래스 재사용성을 높이고 테스트 시 독립성을 높일 수 있다. 이렇게 외부에서 의존성, 의존관계를 주입해주는일도 스프링 컨테이너가 해주기 때문에 의존관계 주입(DI) 을 스프링의 특징이라고 하는 것이다.
	- AOP (Aspect Oriented Programming): `관점지향 프로그래밍` OOP(Object Oriented Programming) 를 살짝 다른 관점에서 보는 시각을 말한다. OOP 를 기반으로 하되 모든 코드에 동일하게 짜여지는 코드를 모듈화 해서 재사용할 수 있게 만드는 개념이다. 예를 들면 모든 클래스에서 method 호출때 마다 찍혀야 하는 로깅과 같은 부분은 AOP 관점으로 하나의 모듈을 만들어 모든 클래스에서 재사용 할 수 있는 것이다. `@Before (이전), @After (이후), @AfterReturning (정상적인 리턴 후), @AfterThrowing (예외 발생 후), @Around (메소드 실행 전과 후)` 등 과 같은 시점에 대한 어드바이스와 몇가지 설정만 하면 여러군데에서 등장해야 하는 코드를 한번에 처리할 수 있다.


이런 특징을 가지는 스프링은 여러 API 를 제공한다. UI, 웹 프레젠테이션, 비즈니스 서비스, 기반 서비스, 도메인, 데이터 액세스 등 ... 여러 방면에서 사용될 만한 API 가 이미 제공되고 있기 떄문에 쉽게 어플리케이션을 개발할 수 있다는 장점이 있다.

## 1장. 오브젝트와 의존관계
### 스프링의 철학
* 자바 엔터프라이즈 기술의 혼란
* 잃어버린 객체지향 가치 회복
* 객체지향의 혜택을 누릴 수 있도록 기본으로 돌아가자

=> **스프링을 이해하려면 오브젝트에 관심을 가져야 한다.**
그래서 이 책의 앞부분에서는 <U>객체지향</U>, <U>디자인 패턴</U>, <U>리팩토링</U>, <U>단위테스트</U>에 대한 설명을 먼저 다룰 것.

### DAO 리팩토링 과정
아래의 User 를 데이터베이스에 저장하고 조회할 DAO를 만들기로 한다.
``` java
public class User {
    String id;
    String name;
    String password;
	// ... getters, setters 생략
}
```
로컬의 데이터베이스에는 아래와 같은 테이블이 존재한다.
![Alt text](./1538383407575.png)


#### 첫번째 시도: 하나의 UserDao 클래스로 jdbc 연결부터 저장, 조회까지
처음 아래와 같이 만들어 보았다.
``` java
public class UserDao {
    public void add (User user) throws ClassNotFoundException, SQLException {
        // 중복
        // 드라이버 연결
        Class.forName("com.mysql.jdbc.Driver");
        Connection connection = DriverManager.getConnection("jdbc:mysql://localhost/springbook", "root", "root");

        // 실행할 쿼리 설정
        PreparedStatement preparedStatement = connection.prepareStatement("insert into users(id, name, password) values(?,?,?)");
        preparedStatement.setString(1, user.getId());
        preparedStatement.setString(2, user.getName());
        preparedStatement.setString(3, user.getPassword());

        // 쿼리 실행
        preparedStatement.executeUpdate();

        // 쿼리, 연결 종료
        preparedStatement.close();
        connection.close();
    }

    public User get(String id) throws ClassNotFoundException, SQLException {
        // 중복
        // 드라이버 연결
        Class.forName("com.mysql.jdbc.Driver");
        Connection connection = DriverManager.getConnection("jdbc:mysql://localhost/springbook", "root", "root");

        // 실행할 쿼리 설정
        PreparedStatement preparedStatement = connection.prepareStatement("select * from users where id = ?");
        preparedStatement.setString(1, id);

        // 쿼리 실행
        ResultSet resultSet = preparedStatement.executeQuery();
        resultSet.next();
        // 결과 세팅
        User user = new User();
        user.setId(resultSet.getString("id"));
        user.setName(resultSet.getString("name"));
        user.setPassword(resultSet.getString("password"));

        // 쿼리, 연결 종료
        resultSet.close();
        preparedStatement.close();
        connection.close();

        // 결과값 리턴
        return user;
    }
}
```
위의 코드에 여러가지 **문제**가보인다.
* `중복` 코드
* 한 클래스, 한 메서드 안의 `여러개의 관심사`

우선은 위의 문제점은 뒤로하고 잘 동작하는지 테스트를 해보자

``` java
	@Test
    public void main1() throws SQLException, ClassNotFoundException {
        UserDao userDao = new UserDao();

        User user = new User();
        user.setId("hello");
        user.setName("홍길동");
        user.setPassword("password");

        userDao.add(user);

        System.out.println(user.getId() + "등록 성공");

        User user2 = userDao.get(user.getId());
        System.out.println(user2.getName());
        System.out.println(user2.getPassword());

        System.out.println(user2.getId() + "조회 성공");
    }
```
돌려보면 아래와 같이 잘 동작하는 것을 확인할 수 있다. (위의 코드를 2번 이상 돌리면 데이터가 이미 있기때문에 id 중복으로 에러가난다)
```
hello등록 성공
홍길동
password
hello조회 성공
```

이제 동작을 확인했으니 다시 코드를 고쳐보자. 위에서 언급한 부분이 왜 문제가 될까
* **`중복` 코드**
	* 코드는 변한다.
	* 요구사항이 변한다.
	* 중복된 코드 부분이 고쳐야 하는 부분이라면 중복 코드를 모두 찾아가서 고쳐야 한다.
		* 잠재적 버그
* **한 클래스, 한 메서드 안의 `여러개의 관심사`**
	* 변화하는 요구사항은 <U>한번에 하나의 관심사</U> 이다.
		* ex. 데이터베이스를 오라클에서 MySql 로 바꿔라
		* ex. 로그의 날짜 포맷을 6자리에서 8자리로 바꿔라
	* 작업을 한곳에 집중시켜야 한다
	* 즉, 같은 관심끼리 모으는 것
	* `관심사의 분리` (SoC: Seperation of Concerns) 가 필요하다

**=> `분리와 확장`에 용이하도록 작성되어야 한다.**

#### 두번째 시도: 중복코드를 제거하자. 메서드 추출, 상속을 통한 확장

위의 UserDao 코드에서 중복된 부분인
``` java
Class.forName("com.mysql.jdbc.Driver");
        Connection connection = DriverManager.getConnection("jdbc:mysql://localhost/springbook", "root", "root");
```
을 아래와 같이 분리해서 getConnection() 메서드 호출로 중복을 제거하였다.
``` java
...
Connection connection = getConnection();
}
private Connection getConnection() throws ClassNotFoundException, SQLException {
        Class.forName("com.mysql.jdbc.Driver");
        return DriverManager.getConnection("jdbc:mysql://localhost/springbook", "root", "root");
}
```
데이터베이스의 데이터를 지워주고 다시 위의 테스트 코드를 돌리니 정상 동작하는 것을 확인할 수 있다.

```
hello등록 성공
홍길동
password
hello조회 성공
```

이제 요구사항이 변화한다고 생각해보자.
* 다른 데이터베이스 커낵션 방법으로 위 코드를 재사용해야 하는 경우가 생겼다.

**문제점**
위 코드에서는 user 를 읽고 저장하는 부분과 데이터베이스 연결 부분이 같이 코딩되어있으므로 고칠방법이 없다.
**=> 확장에 용이하도록 변경하고 싶다!**

상속하여 하위 클래스를 여러개 두어 여러개의 구현체를 두는 abstract 클래스를 만들자
위의 getConnection() 메서드를 아래와 같이 abstract 로 바꿔주고 하위 클래스를 만들어보았다.
``` java
abstract Connection getConnection() throws ClassNotFoundException, SQLException;
```

``` java
public class NUserDao extends UserDao {
    @Override
    Connection getConnection() throws ClassNotFoundException, SQLException {
        // for N company
        return null;
    }
}
```

``` java
public class DUserDao extends UserDao {
    @Override
    Connection getConnection() throws ClassNotFoundException, SQLException {
        // for D Company
        return null;
    }
}
```

> Abstract 클래스로 템플릿을 만들어 두고 필요에 맞게 구현하도록 하는 방법을 `템플릿 메서드 패턴` 이라고 한다.
> 동시에 하위클래스 입장에서는 구체적인 오브젝트 생성 방법이 결정되므로 `팩토리 메서드 패턴` 이라고 할 수 있다.

하지만 이 방법은 또하나의 문제가 있다. 바로 `상속` 이다.
상속은 아래와 같은 한계를 만들어 낸다.
1.  **추후 다른 목적의 상속을 사용할 수 없다.** 자바는 다중상속을 허용하지 않기 때문
2.  상속관계의 상하위 클래스 관계는 밀접하다. 즉 **두 가지 다른 관심사에 대해 결합이 강해지게 만든다.**

#### 세번째 시도: 클래스의 분리
그렇다면 어떤 구조로 작성해야 할까. 위와 같은 상속의 한계를 느끼게 되는 이유는 사실 한가지다. **"변화"** 하기 때문이다. 변화할 것이기 때문에 다중 상속의 한계를 느끼게 될 것이며 변화를 해야하는데 두 관심사에 대해 결합이 강해서 코드를 고치는게 쉽지 않아진 것이다. 즉, 변화에 잘 대응할 수 있는 코드를 작성해야 한다. 

`변화`에 대해서 좀 더 생각해보자
* 변화의 성격이 다르다는 것은 **변화의 이유와 시기, 주기** 등이 다르다는 뜻이다.
	* 예를 들어 (1)db 커낵션을 맺는 코드와 (2)유저 데이터를 읽고 쓰는 부분의 코드가 있다고 생각하면
	* db 커낵션 방법을 바꿔라 라는 요구사항이 왔을때 (1)번 코드만 변경된다. 
	* 유저 데이터를 다르게 써야 한다면 (2)번 코드만 변경된다. 
	* db 커낵션  방법도 고치고 유저데이터를 다르게 써야 한다 는 요구사항은 일반적으로 동시에 오는 경우가 없다.
	* <U>"관심사"가 다르기 때문이다.</U>
	* 결과적으로 변화의 이유도 달랐고 시기, 주기도 다르게 된다.
코드를 고치는 것이 쉬우려면 하나의 모듈은 한 관심사에만 집중되어야 한다. **관심사 별로 클래스를 분리해 보자!**
  
///

> 
> 
>  **여기 정리 안함!!!!!!!!!!!**
>  
>   


### 원칙과 패턴
지금까지의 코드에 어떤 객체지향적 특징이 있는 지 확인해보면 아래와 같다.
#### OCP(Open-Closed Principle, 개방폐쇄 원칙)
> **클래스나 모듈은 확장에는 열려있어야 하고 변경에는 닿여있어야 한다.**

라는 원칙이지만 사실 그대로 읽으면 와닿지 않는다. 조금 더 풀어서 얘기하자면 
**=>** 클래스나 <U>모듈이 **확장**이 가능하고</U> 이러한 <U>변화로 인해 **영향**을 받지 않는다</U>  라는 의미이다.

> 이런 객체지향 설계 원칙을 정리하면 SOLID 원칙으로 정리할 수 있는데 자세한 내용은 [SOLID 원칙](https://jinheec.github.io/design%20pattern/2018/05/30/DesignPatternAndSpring/#solid-%EC%9B%90%EC%B9%99)을 참고한다.

#### 높은 응집도와 낮은 결합도
위에서 설명했던 OCP 원리는 높은 응집도와 낮은 결합도의 결과라고 볼 수 있다. 자세한 설명은 [해당 글](https://jinheec.github.io/design%20pattern/2018/05/30/DesignPatternAndSpring/#%EB%82%98%EC%81%9C-%EC%84%A4%EA%B3%84-%EB%82%AE%EC%9D%80-%EC%9D%91%EC%A7%91%EB%8F%84)을 참고하고 간단하게 설명하자면 아래와 같다.
 * **높은 응집도**란
	 * 변화가 일어 날 때 모듈의 한 부분만 변화가 일어난다.
	 * 하나의 모듈은 하나의 관심사를 갖는다.  
 * **낮은 결합도**란 
	 * 책임과 관심사가 다른 모듈과는 연관이 느슨하다. 
	 * 꼭 필요한 최소한의 간접적인 연결이 있다.
	 * 서로 독립적이고 알필요가 없다.

##### 예시
위 코드에서 예시를 들자면 **낮은 응집도**는 아래와 같다.
``` java
public class UserDao {
    public void add (User user) throws ClassNotFoundException, SQLException {
        // 중복
        // 드라이버 연결
        Class.forName("com.mysql.jdbc.Driver");
        Connection connection = DriverManager.getConnection("jdbc:mysql://localhost/springbook", "root", "root");

        // 실행할 쿼리 설정
        PreparedStatement preparedStatement = connection.prepareStatement("insert into users(id, name, password) values(?,?,?)");
        preparedStatement.setString(1, user.getId());
        preparedStatement.setString(2, user.getName());
        preparedStatement.setString(3, user.getPassword());
...
	}

	public User get(String id) throws ClassNotFoundException, SQLException {
        // 중복
        // 드라이버 연결
        Class.forName("com.mysql.jdbc.Driver");
        Connection connection = DriverManager.getConnection("jdbc:mysql://localhost/springbook", "root", "root");

        // 실행할 쿼리 설정
        PreparedStatement preparedStatement = connection.prepareStatement("select * from users where id = ?");
        preparedStatement.setString(1, id);

        // 쿼리 실행
        ResultSet resultSet = preparedStatement.executeQuery();
        resultSet.next();
...
        // 결과값 리턴
        return user;
	}    
}
```
위 코드는 응집도가 매우 낮은 코드라고 할 수 있다. (1) 드라이버를 `com.mysql.jdbc.Driver` 에서 다른 드라이버로 변경하고 싶을 경우 한 부분만 변화가 일어나는것이 아니라 **add() 와 get()** 매서드 두 부분에서 변화가 일어나야하고, 하나의 모듈 단위가 되는 add, get 이라는 매서드는 하나의 관심사를 가지지 않았기 때문이다. 


그렇다면 **높은 결합도**의 예시를 보자.

``` java
public class UserDao {
}
```



## 2장. 테스트
스프링이 제공하는 가장 <U>중요한 가치</U>
* **객체지향**
	* IoC/DI 를 이용해 `객체지향`을 개발자가 쉽게 적용할 수 있도록 도와준다.
* **테스트**
	* 만들어진 코드를 확신할 수 있도록 테스트를 도와준다.

=> 테스트를 하지 않는것은 **스프링 가치의 절반**을 포기하는 것.

테스트란 무엇이고 장점, 활용방법, 스프링과의 관계에 대해 다룬다.
### UserDaoTest 다시 보기
1장에서 본 main 으로 작성된 테스트 코드
``` java
	public static void main(String[] args) {
		ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");

        UserDao userDao = context.getBean("userDao", UserDao.class);

        User user = new User();
        user.setId("hello");
        user.setName("홍길동");
        user.setPassword("password");

        userDao.add(user);

        System.out.println(user.getId() + "등록 성공");

        User user2 = userDao.get(user.getId());
        System.out.println(user2.getName());
        System.out.println(user2.getPassword());

        System.out.println(user2.getId() + "조회 성공");
    }
```
####  특징
1.  main 을 이용한다
2.  결과를 출력해 눈으로 확인한다
3.  테스트에 사용할 객체를 테스트 안에서 직접 만들어준다.

**main 으로 동작한다는 것**, 테스트 통과를 **눈으로 확인한다는 점** 을 제외하면 코드의 동작을 확신하게 해줬던 작업.

#### 보통의 DAO 테스트 방법 : 웹을 통해 직접 동작 확인
웹 화면에 직접 입력. 버튼 누르기, 결과 확인
=> 테스트 실패 원인을 찾기 힘듦, 번거로움, 정확하지 않음

어떻게 하면 효율적으로 테스트를 할 것인가?

#### 작은 단위의 테스트
위에서의 1차원적인 방법으로 느낀것. 
* 테스트 대상이 명확하다면 그 대상에만 집중하는 것이 오류파악이 쉽다.    

=> 가능한 **작은** 단위로 쪼개서 테스트 해야한다!
=> 관심이 다르다면 **분리**하고 집중 접근.

이런걸 **`단위테스트(Unit Test)`**(= 개발자 테스트, 프로그래머 테스트) 라고 한다. 
:단위란 충분히 하나의 관심에 집중해서 효율적으로 테스트할 만한 **최소의** 범위

> 최종에는 웹 ~ 사용자 UI ~ DB 까지의 **통합테스트를 수행해야할 필요는 있다**! 
> * 단위테스트가 잘 되더라도 묶어놓으면 안되는 경우도 종종 있다.
> * 이 통합테스트는 단위테스트가 완료 되면 작성한다. 
> 	* 동작을 개발자 스스로 더 빨리 확인받을 수 있다.
>  * 어디에서 오류가 났는지 찾기 힘들기 때문.

#### 자동수행 테스트 코드

위에서 말했던 웹에서 UI 를 통한 테스트는 반복잡업을 수동으로 진행한다.
이 작업은 작업에 착오가 있을 수도 있고 시간이 오래걸린다.

main 으로 테스트 했던 코드는 **자동 수행 코드** 
=> 짧고 정확하고 자동화된 테스트
=> 자주 반복할 수 있다.

#### 지속적인 개선과 점진적인 개발을 위한 테스트
* 처음 초난감 DAO 를 리팩토링 했을때 일등공신은 **테스트 코드**
	* 빠르게 확신할 수 있었고
	* 오류를 빨리 찾을 수 있었다
	* 테스트 코드가 없을때 보다 전체적으로 리팩토링 작업이 더 빨랐을 것.
	* 기능이 추가될 때 테스트 코드가 추가되면서 점진적인 개발이 가능했다.

그럼에도 만족스럽지 못한 부분이 있다.

#### UserDaoTest의 문제점
1. 수동 **결과 확인** 작업이 번거롭다
	* 콘솔에 출력된 값이 맞는 값인지 사람이 확인한다.
	* 완전한 자동화 테스트는 아니다
	* 이런 테스트가 많아지면 문제가 생긴다
2. 실행작업이 번거롭다
	* 하나의 main 일 경우는 상관없지만
	* 여러개의 DAO 가 있을 경우 여러개의 main 을 실행하는 것은 번거로움

### UserDaoTest 개선
#### 결과 확인의 자동화
``` java
	public static void main(String[] args) {
        ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");

        UserDao userDao = context.getBean("userDao", UserDao.class);

        User user = new User();
        user.setId("hello");
        user.setName("홍길동");
        user.setPassword("password");

        userDao.add(user);

        System.out.println(user.getId() + "등록 성공");

        User user2 = userDao.get(user.getId());
        // 아래부분을 수정하자
        System.out.println(user2.getName());
        System.out.println(user2.getPassword());
        System.out.println(user2.getId() + "조회 성공");
    }
```

``` java
	public static void main(String[] args) {
        ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");

        UserDao userDao = context.getBean("userDao", UserDao.class);

        User user = new User();
        user.setId("hello");
        user.setName("홍길동");
        user.setPassword("password");

        userDao.add(user);

        System.out.println(user.getId() + "등록 성공");

        User user2 = userDao.get(user.getId());

        if (!user.getName().equals(user2.getName())) {
            System.out.println("테스트 실패 (name)");
        } else if (!user.getPassword().equals(user2.getPassword())) {
            System.out.println("테스트 실패 (password)");
        } else {
            System.out.println("조회 테스트 성공");
        }
    }
```

맞는 값인지 **눈으로 확인하던 부분을 코드**로 실패, 성공을 확인하도록 변경하였다.
그러나 여전히 main() 메소드로는 한계가 존재
2번째 단점이었던 "main 개수가 많아지면 어떻게 할 것인가" 를 해결해야 한다.
=> 자바에 자바 테스팅 **프레임 워크**인 JUnit 이 존재한다.
>   (**`프레임 워크`**이기 때문에 main도 다른 오브젝트 생성도 필요 없다. 개발자의 코드를 프레임워크가 가져다가 쓴다)

대신 **JUnit 에게 `개발자의 코드를 사용`하라고 알려주기 위한 조건**이 있다.
1. 메소드는 **public** 이어야 한다.
2. 메소드에 **@Test** 어노테이션이 붙여있어야 한다.

적용해서 재구성하면 아래와 같다.
``` java
	@Test
    public void addAndGet() throws SQLException, ClassNotFoundException {
        ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");

        UserDao userDao = context.getBean("userDao", UserDao.class);

        User user = new User();
        user.setId("hello");
        user.setName("홍길동");
        user.setPassword("password");

        userDao.add(user);

        System.out.println(user.getId() + "등록 성공");

        User user2 = userDao.get(user.getId());

        if (!user.getName().equals(user2.getName())) {
            System.out.println("테스트 실패 (name)");
        } else if (!user.getPassword().equals(user2.getPassword())) {
            System.out.println("테스트 실패 (password)");
        } else {
            System.out.println("조회 테스트 성공");
        }
    }
```

if/ else 문으로 개선했던 테스트 성공/실패 코드가 JUnit 으로 바꾸니 좀 아쉽다.
JUnit 에는 assertThat() 이라는 **테스트 검증 메소드**가 존재하므로 System,out.println() 을 대체하자

``` java
	@Test
    public void test3() throws SQLException, ClassNotFoundException {
		ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");

        UserDao userDao = context.getBean("userDao", UserDao.class);

        User user = new User();
        user.setId("hello");
        user.setName("홍길동");
        user.setPassword("password");

        userDao.add(user);

        System.out.println(user.getId() + "등록 성공");

        User user2 = userDao.get(user.getId());

        assertThat(user2.getName(), is(user.getName()));
        assertThat(user2.getPassword(), is(user.getPassword()));
    }
```
검증이 틀릴경우 **AssertionError** 를 던진다.
