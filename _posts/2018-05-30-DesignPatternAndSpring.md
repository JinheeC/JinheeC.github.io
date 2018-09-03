---
title: "Design Pattern과 Spring Framework"
categories: 
- Design Pattern
excerpt: |
  디자인 패턴과 Spring Framework 의 동작 원리는 굉장히 밀접한 연관이 있다.
  Spring Framework 의 주요 개념인 IOC, AOP, DI 를 이해하기 위해서는 디자인 패턴을 먼저 이해할 필요가 있다. 먼저 디자인 패턴을 설명하려고 한다.
feature_text: |
  Design Pattern과 Spring Framework
---

# Design Pattern과 Spring Framework

* table of contents
{:toc .toc}

디자인 패턴과 Spring Framework 의 동작 원리는 굉장히 밀접한 연관이 있다.
Spring Framework 의 주요 개념인 IOC, AOP, DI 를 이해하기 위해서는 디자인 패턴을 먼저 이해할 필요가 있다. 먼저 디자인 패턴을 설명하려고 한다.

## 디자인 패턴이란
디자인 패턴이란 사실 건축학에서 먼저 언급된 방법이다. 자주 발생하는 건축 설계 문제에 대한 해답을 문서화하고 일반화, 패턴화 하는 작업에서 "디자인 패턴" 이라는 방법이 나왔고 이후 소프트웨어 설계에 해당 개념이 도입되었다. 그리고 오히려 현재는 건축학 보다는 컴퓨터공학에서 쓰이는 단어가 된 것 같다.
소프트웨어 설계에서 사용되는 디자인 패턴이라는 단어도 건축에서 말하는 개념과 같다. 소프트웨어 설계시 자주 발생되는 문제를 해결하는 해답을 "패턴"이라는 답안으로 정리해 놓은 것이다.

그렇다면 소프트웨어 설계는 어떻게 설계가 되어야하고 무엇을 위반하여 자주 어떤 문제가 발생하는 걸까. 그 기준은 무엇이 될까?

### 좋은 설계란 
객체지향 소프트웨어에서 좋은 설계란 많이들 언급되는 "낮은 유지보수 비용, 확장성, 재사용성"이 될 것이다. 결국은 이런 요소를 위한 것인데 이 세가지 요소가 결합된 잘 설계된 소프트웨어를 구현하기 위해서 로버트 마틴이 쉽게 몇가지 규칙을 정리해서 이름을 붙였는데 그게 **SOLID**  원칙이다. 이 다섯가지 원칙만 지켜도 좋은 설계를 가지는 소프트웨어를 구현할 수 있다.

#### SOLID 원칙
SOLID 라는 이름은 SRP, OCP, LSP, ISP, DIP 의 앞자를 따 부르는 말이다. 순서는 중요하지 않다. 각각은 아래와 같은 의미를 지닌다. 명칭은 어려워 보이지만 내용은 그렇게 어렵지 않다.

##### **S**RP (단일 책임의 원칙 Single responsibility principle)
쉽게 말해서 하나의 일만 하라는 뜻이다. 한 클래스는 하나의 일만 한다. 즉 자동차라는 클래스는 자동차에 대한 동작과 변수가 있어야 하지 자동차라는 클래스인데 자전거에 대한 코드가 있어서는 안된다.

##### **O**CP (개방-폐쇄 원칙 Open/closed principle)
소프트웨어 요소는 확장에는 열려 있으나 변경에는 닫혀 있어야 한다. 쉽게 말하면 인터페이스(추상)를 사용하라는 뜻이다. 왜냐하면 하나의 변경이 외부에 영향을 미치면 안되기 때문이다. 
``` java
House myHouse = new House();
Tree tree = new Tree();

myHouse.buildBy(tree);
```
위의 간단한 코드를 보면 좀더 쉽게 이해할 수 있다. 만약 집을 만드는 작업을 구현 하려고 할 때 "집"이라는 객체를 만드려면 재료가 필요하다. 이때 "나무"라는 클래스가 있어서 "나무"라는 객체로 집을 만들었다고 치자. 이때 벽돌로 만들어야 한다고 요구사항이 바뀌면 아래와 같이 코드를 고쳐야 한다.
``` java
House myHouse = new House();
//Tree tree = new Tree();  // 삭제됨
Brick brick = new Brick();  // 추가됨

myHouse.buildBy(brick);   // 변경됨
```
이렇게 코드를 꽤나 여러군데 고쳐야 한다면 이것은 변경에 닫혀있다고 할 수 없다. 그렇다면 만약 인터페이스를 구현했었다면 어땠을까?
Material 이라는 인터페이스가 있고 Material 을 구현한 Tree라는 구현체가 있다면 아래와 같이 사용할 수 있을 것이다.
``` java
House myHouse = new House();
Material material = new Tree();

myHouse.buildBy(material);
```
이때 위의 상황과 같이 Brick 으로 바꿔야 했다면, 이때 Material 을 구현하는 또 다른 구현체 Brick을 만들고 아래와 같이 수정하면 된다.
``` java
House myHouse = new House();
Material material = new Brick(); // 변경됨

myHouse.buildBy(material);
```
보는 것 과 같이 변경이 매우 적은 것을 확인 할 수 있다. (뒤에서 설명을 자세히 하겠지만 특히나 Spring과 같은 Framework 에서는 DI-의존성주입- 이 일어나기 때문에 더 변경이 적을 것이다.) 이런 상황을 확장에는 열려있고(Brick과 같은 재료를 계속 추가 가능하다) 변경에는 닫혀있다고 할 수 있다. 

##### **L**SP (리스코프 치환 원칙 Liskov substitution principle)
프로그램의 객체는 프로그램의 정확성을 깨뜨리지 않으면서 하위 타입의 인스턴스로 바꿀 수 있어야 한다. 리스코프 치환이 뭔지는 자세히 알 필요 없다. 이름이 어려워 보이지만 단순히 SuperClass 타입과 SubClass 타입을 바꿀 수 있어야 한다는 뜻이다.
``` java
1 SuperClass object1 = new SuperClass();
2 object1.print();
3 
4 SuperClass object2 = new SubClass();
5 object2.print();
```
위 코드에서 object1과 object2 가 서로 바뀌어도 문제가 없어야 한다는 뜻이다. 미치는 영향이 같아야 한다는 뜻인데 쉽게말하면 수퍼클래스에서 상속받은 서브클래스의 메소드는 수퍼클래스에서 사용되던 목적이 같은 작업이 이루어져야 한다는 것이다. 좀 애매하게 들릴 것 같은데 그 반대 상황인 위반했을 경우를 보면 좀 더 이해가 갈 수 있다. 이 치환의 법칙을 위반하게 되면 객체별로 호출시 처리해줘야 할 것들이 좀 달라지는데 그럴 경우 아래와 같이 처리를 해줘야 한다.
``` java
if(object2 instanceOf SubClass){
	...
} else {
	...
}
```
좀더 상속관계가 복잡해지면 위와같은 instance 를 비교해 따로 처리해주는 if 문이 기하급수적으로 증가할 것이다. 상속받은 클래스 별로 행동이 다르기 때문이다.

##### **I**SP    (인터페이스 분리 원칙 Interface segregation principle)
특정 클라이언트를 위한 인터페이스 여러 개가 범용 인터페이스 하나보다 낫다. 사용자 입장에서 사용할 때 분리된 인터페이스가 좋다는 것이다. 사용하지 않는 메소드까지 의존하면 안된다. 예를 들면 스캔기능만 필요한데 복합기 인터페이스를 받아서 스캔기능을 사용하는 설계가 좋지 않다는 것이다.

##### **D**IP (의존관계 역전 원칙 Dependency inversion principle)
인터페이스(추상화)를 통해 접근하라는 뜻이다. 직접 클래스에서 또다른 클래스를 접근하면 결합도가 높아진다. "역전"이라고 표현한 것은 아래와 같이 방향이 달라서다.
![Alt text](https://monosnap.com/image/AJeoTKpFP10Xpj0SVFL4cXQgtaw8dw)

### 그렇다면 나쁜 설계란?
나쁜 설계란 어떤 결과를 야기시키는 것일까. 나쁜 설계가 어떤 설계인지 보기 전에 우선 그 설계의 미래 또는 결과를 먼저 보자면 아래와 같다.

* 변경요구사항이 있어서 변경을 했는데 깨지는, 수정할 사항이 너무 많게 되어버렸다.
* 비슷한 로직을 여러군데에서 사용하게 되었는데 현재 있는 기존의 코드를 사용할 수 없었다. 그 코드가 의존하는게 많기 때문이다.

이런 결과는 왜 생기는 것이고 우리 코드의 어떤 점 때문에 일어났을까.

#### 나쁜 설계의 원인
좋은 설계는 high cohesion loose coupling(높은 응집도 루즈한 커플링)을 가지고 있어야 하는데 그 반대의 성질이 나쁜 설계의 원인이라고 볼 수 있다. 
##### 나쁜 설계: 낮은 응집도
사전에서 찾아보면 응집도라는 것은 "똘똘 뭉치는 정도" 또는 "밀접한 정도"를 뜻한다고 한다. 즉 한 클래스 안의 메소드들이 하는일, 그리고 그 메소드의 안에서 실제로 이루어지는 작업이 얼마나 서로 연관되어있는가를 뜻한다. 다시말하면 하나의 클래스 하나의 메소드는 하나의 일(SoC: Separation of Concerns, 관심사의 분리)을 해야한다는 뜻이다. 그런데 관심사의 분리가 안된채로 하나의 모듈 단위가 여러가지 일을 하게 되면 응집도가 떨어지게되고 낮응 응집도를 가진 프로그램이 된다. 결국 이런 특징이 나쁜 설계의 원인이 되는 것이다.

##### 나쁜 설계: 타이트한 커플링
커플링이라는 것은 결합도를 말하는데 즉 하나의 모듈과 다른 하나의 모듈이 얼마나 결합도가 있는지, 얼마나 의존을 하는지를 말한다. 결합도가 높은(=타이트한 커플링) 예를 들자면, 하나의 클래스의 메소드가 다른 클래스의 메소드를 호출하고 그 메소드가 또다른 클래스의 메소드를 호출하는 식으로 서로가 서로의 의존도가 매우 클 경우 결합도(커플링)가 높다고 말한다. 이럴 경우 하나의 메소드의 수정이 일어나면 연달아 그 메소드를 사용하고 의존하는 다른메소드가 깨지므로 좋지 않은 설계라 할 수 있다. 그리고 이 성질 역시 나쁜 설계의 원인이 된다. 이경우 추상화를 사용하여 결합도를 낮춰야 한다.

그렇다면 좋은 설계를 위한 여러 디자인 패턴을 익혀보자.

## Abstract Factory Pattern
여러 클래스의 객체를 생성하기 위한 인터페이스를 가지는 패턴을 말한다. 실제 객체들을 생성하는 팩토리클래스들이 상속받을 인터페이스가 있는 구조를 말하는데 아래와 같이 같은 객체를 생성하지만 그 객체의 세부 타입이 다를 때 사용하면 좋다. 

![Alt text](https://monosnap.com/image/4PhBz7jA2sgfhPLt99xFV3q8ZoQbuW)

> NOTE: Inversion of Control (제어의 역전)
>  
> 원래 POJO 자바에선 객체를 사용하려면 1. 생성하고 2. 객체를 호출한다. 
> 
> 하지만 스프링에서는 그 객체를 사용하고 싶은 클래스가 생성자로 받아서 사용하기 때문에 의존관계가 주입되는 형태가 된다.
> 그래서 역전된 의존관계라 해서 제어가 역전되었다 라고하는 것이다. 이건 사실 스프링만의 특징은 아니다. 주로 spring 이 이 개념을 사용하고 있기때문에 spring의 특징으로 여겨진다. 그렇다면 왜 생성자를 통해 받아서 사용해야 하고 이런 패턴이 왜 등장하게 되었을까.
> 
> 일반적으로 객체를 사용하는 방식은 new 객체()로 객체의 생성자, 객체의 이름 등이 바뀌게 되면 수정해줘야 할 곳이 많이 생길 수 있다. 그래서 IoC 라는 개념이 사용되었다.

## Factory Method Pattern
이 패턴은 객체 생성관련 작업을 subClass에 위임하여 근본적으로는 비즈니스 로직과 factory 성 로직을 분리하기 위함이다. 보기 좋은 코드를 위한 목적도있지만 factory 성 코드는 생성할 객체가 추가되거나 수정되면 같이 수정될 수 있는 부분이지만 비즈니스는 다른 빈도로 변경되거나 변경되지 않을 수도 있기 때문에 분리해 주는 것이다. 
![Alt text](https://monosnap.com/image/KuT4orHZ2kL4AgQyIOsqDNpTsv5XMq)

> NOTE: Abstract Factory Pattern 과 Factory Method Pattern 의 차이
> 
> Factory Method 는 Single method 다.  그래서 하위 클래스에서 해당 메소드를 오버라이드 하게 된다. 사용할때 factory 인터페이스를 인스턴스 변수로 받아서 인스턴스 안의 팩토리 메소드를 사용한다.
> Abstract Factory 는 오브젝트다. 그래서 그 오브젝트 안에 여러개의 팩토리성 매소드들이 많이 있다. 사용할때  Factory Method Pattern을 가진 해당 클래스 안에 있는 비즈니스 로직에서 해당 메소드를 호출해 객체를 만든다.

## Template Method Pattern
사용될 알고리즘의 구조(템플릿)을 미리 상위에서 정의하고 하위에서 구체적인 구현을 하는 패턴이다. 그냥 말그대로 뼈대(틀)를 만드는 것에 목표를 두는 패턴이고 특징이 있다면 클래스 속 한 메서드가 다른 메서드들을 전부 호출한다는 특징이 있다. 마치 프레임워크 처럼 말이다. 
![Alt text](https://monosnap.com/image/xXnbmgT9FTjxN3tpoxSy51oW0ZkqUl)

> NOTE: 프레임워크와 라이브러리의 차이
> 
> 제어의 주체가 다르다. 라이브러리는 개발자의 코드에서 사용되는 것이고 프레임워크는 프레임 워크가 개발자의 코드를 사용하는 것이다. 

## Proxy Pattern
프록시 패턴은 실제 객체에 접근하지 않고 프록시로 다른 객체를 중간에 두고 접근하는 방식이다. 호출하는 입장에서는 프록시인것을 알 필요가 없다. 이 패턴을 사용하는 목적은 객체의 용량이 굉장히 커서 최대한 미리 조건문으로 생성할 필요가 없는 상황을 거르고(예외인 상황 미리 캐치 등...) 필요할 때 만드려는 목적, 원래 객체가 하는 일이 추가적인 동작을 미리 해주려는 목적, 네트워크 연결 과 같은 동작을 프록시로 미리 거르고(=미리 해주고) 마치 네트워크가 필요없는 로컬 호출과 같이 구현하기 위한 목적이다. 비즈니스 로직과 그 이외의 일을 분리하기에도 좋은 패턴이다. 그런 면에서 데코레이션 패턴과 비슷하다고 볼 수 있다. 

![Alt text](https://monosnap.com/image/Y1vVyguQwZBRWxLeUJPlTPPJwfgqtf)

아래는 Proxy Pattern을 구현하지 않고서 Spring 의 AOP 개념을 사용한 방법. aspect class도 스프링 Bean 이어야 하고 사용되는 pointcut 도 bean 이어야 한다. 
``` java
//Aspect Class - 공통 로직 (cross cutting concern) 메소드를 제공하는 클래스.

@Aspect  // 이게 aspect 클래스라는 것임을 명시.
@EnableAspectJAutoProxy  // 이 Bean 중에 aspect 가 있고 그 중에 pointcut이 있으니까 설정대로 자동으로 프록시 만들어서 읽도록 해라 라는 의미. 설정 중 하나.
public class LoggerAspect {
    
    @AfterThrowing(pointcut= "within(service.impl.UserServiceImpl)")  // 예외가 발생해서 throw이후에 실행하라. pointcut 은 대상이되는 메서드 
    public void errorLog(){
        System.out.println("errorLog(): 오류가 발생하였습니다.");
    }
    
    @AfterReturning(pointcut= "within(service.impl.*)")
    public void infoLog(){
        System.out.println("infoLog(): 정상적으로 종료되었습니다.");
    }
}
```
> NOTE: Spring의 AOP 개념이란?
>  
> AOP 는 Aspect Oriented Programming 의 약자이다. 관점지향 프로그래밍이라고 불리는데 잘 와닫는 이름은 아니지만 프로그램에서 반복되는 공통 비즈니스로직를 핵심 비즈니스와 분리해서 프레임 워크가 공통 로직을 처리해주는 일을 말한다. 이 개념은 프레임워크가 개발자의 코드를 사용해서 공통로직을 주입해주는 개념인데 이 프레임워크의 로직은 내부적으로 Proxy Pattern을 사용한다. 그래서 개발자의 코드의 전 후에 알맞는 공통로직이 처리되는 것이다. 그래서 위의 코드와 같이 Proxy Pattern 을 상속구조를 POJO로 직접 만들지 않고도 사용할 수 있는 **Aspect** 어노테이션이 있는것이다.  
>  
> 아래는 관련 AOP 용어다.
>  
> Joinpoint: 공통 관심사항이 적용될 그 **대상**이 될 수 있는 메소드를 말한다. 
>  
> Pointcut: Joinpoint 중에 실제 대상을 말한다.
> 
> Advice: 어느 시점에 공통관심사항을 적용할것인지에 대한 설명이다. (before, after-running, after-throwing, after, around) 
>  
> Weaving: Proxy Pattern 이 만들어지는 것. (스프링은 실행시 위빙)
>  
> Aspect: 여러 객체에 공통적용되는 공통 관심 사항
>  

## Strategy Pattern
같은 목적의 알고리즘인데 구현로직이 조금씩 상황에 따라 다를 경우 여러개의 하위클래스를 둬서 그때그때 다른 전략을 사용할 수 있게하는 패턴이다. 
