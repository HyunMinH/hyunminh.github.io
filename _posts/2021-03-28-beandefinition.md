---
layout : post
title : "BeanDefinition과 어노테이션 (feat. IoC 컨테이너)"
image : https://images.velog.io/images/hyunmin/post/1767f0c2-837c-4129-8715-9f58690ba9a6/velog.002.jpeg
---

# 도입

---

이전 글에서 `HandlerAdapter`에 대해 알아보았다.

그리고 이번에는 Spring의 MVC가 어노테이션을 이용해 어떻게 동작 하는지 알아보기로 했었다.
하지만 그전에 알아볼 것이 있다.

왜 IoC Container가 빈을 등록하는 역할이 있는지, 
그 IoC Container가 어노테이션으로 작성된 클래스를 어떻게 빈으로 등록하는 지 알아보자.

# 의존성

---

> 함께 변경될 수 있는 가능성을 의미.

위 내용을 쉽게 풀어쓰면 아래와 같다.

> 만약 A 모듈이 B 모듈에게 메시지를 전송하는 관계이면,
B 모듈이 변경될 때, A도 같이 바뀔 수 있다.

하지만 의존성이 실제로 전이될지 여부는 
변경의 방향과 캡슐화의 정도에 따라 달라진다.

다시 말하면 의존하는 객체에 대해 많이 알면 알 수록,
특정 컨텍스트(특정 환경, 시나리오)에 묶일 수 있다는 것이다.

의존도가 높으면 높을수록 유연하지 못한 코드가 된다.

그런 관점에서 생성-사용 분리 원칙의 필요성을 이해해보자.

# 생성 및 사용 역할이 둘 다 있는 객체

---

코드를 보자.
Class A는 B 객체를 생성한 후, 사용하고 있다.
```java
public Class A{
	private B b;

	public A(int parm1, int parm2){
		b = new B(parm1, parm2);
		...	
	}

	...
	public Result func1(){
		...
		Result = b.func2();
		...
	}
}
```

ClassA는 B의 생성자를 직접 호출한다.
즉, 생성자에 필요한 파라미터들을 알고 있어야 한다.

여기서 변경의 관점에서 살펴보자.

만약 B의 파라미터가 2개가 아닌 3개가 된다면?

* A의 생성자의 파라미터 목록을 수정해줘야 한다.
* A를 생성하는 객체또한 그에 맞게 파라미터를 넘겨줘야한다.
* B의 생성자 호출하는 코드를 바꿔줘야한다.

B가 변경됨으로 인해 A도 같이 변경되게 된 것이다.
A와 B가 강하게 결합되어 있으므로 특정 컨텍스트에 묶인다.

다시 말해, 
의존 객체를 생성하는 방식은 유연한 설계가 되지 않는다.



# 생성-사용 분리

---

의존객체를 생성하는 역할을 다른 객체에 부여하는게 유연성 측면에서 좋다.
이를 생성-사용 분리라고 한다.

> 소프트웨어 시스템은 시작 단계와 실행 단계를 분리해야 한다.
> 시작 단계 : 응용 프로그램 객체를 제작하고 의존성을 서로 “연결”하는 단계.
> 실행 단계 : 시작 단계 이후에 이어지는 단계.

우리는 객체를 생성하는 역할(시작단계)을 다른 객체에게 넘겨줄 것이다.
바로 객체 생성 역할을 따로 하는 Factory를 이용할 것이다.

# Factory

---

> 도메인 모델과 관련 없는 인공적인 객체로서
> 생성과 사용을 분리하기 위해 객체 생성에 특화된 객체.

모든 책임을 도메인 객체에게 할당하면 낮은 응집도, 높은 결합도, 재사용성 저하가 생길 수 있다.
따라서 이런 상황이 발생한다면 인공적인 객체를 도입을 생각해보아야 한다.

우리는 생성-사용 분리를 위해 Factory를 사용하는 것이다.

# BeanFactory

---

> Spring에서 제공하는 BeanFactory는
> bean(스프링이 관리하는 객체)을 생성, 설정 및 관리한다.  

 BeanFactory는 객체 생성 뿐만 아니라, 필요한 의존성을 주입시켜준다.

위 코드 예로 보면, A가  B 객체를 필요로 하다면,
BeanFactory는 해당하는 B 객체를 찾아 A의 생성자에 넘겨줄 수 있다.

> IoC는 제어 역전 원칙이며, 
> 스프링에서는 컨테이너가 코드 대신 오브젝트에 대한 제어권을 갖고 있어 IoC 컨테이너라고 부름.

> ApplicationContext는 DI(의존성 주입)를 위한 BeanFactory에 
> 여러 가지 컨테이너 기능을 추가한 것임.

이제 Bean을 어떻게 등록하는지 보자

# Spring Application의 빈 등록 과정과 BeanDefinition

---

XML 또는 관련 애너테이션을 통해 빈 설정을 해본 적이 있을 것이다.
실제 어떻게 IoC 컨테이너가 이 정보를 인식하고 빈으로 등록하는지 보자.

![](https://images.velog.io/images/hyunmin/post/1767f0c2-837c-4129-8715-9f58690ba9a6/velog.002.jpeg)

1. 각 메타정보 리소스에 대응되는 `BeanDefinitionReader`가 리소스를 `BeanDefinition` 으로 변환해준다.
2. IoC 컨테이너는 POJO 클래스와 BeanDefinition을 이용해 빈을 생성하게 된다.


## BeanDefinition의 몇 가지 정보

실제 BeanDefinition은 인터페이스로, 여러 정보를 컨테이너에게 제공한다.

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
    String SCOPE_SINGLETON = "singleton";
    String SCOPE_PROTOTYPE = "prototype";
    int ROLE_APPLICATION = 0;
    int ROLE_SUPPORT = 1;
    int ROLE_INFRASTRUCTURE = 2;

    void setParentName(@Nullable String var1);

    @Nullable
    String getParentName();

    void setBeanClassName(@Nullable String var1);

    @Nullable
    String getBeanClassName();

    void setScope(@Nullable String var1);

    @Nullable
    String getScope();

    void setLazyInit(boolean var1);

    boolean isLazyInit();

    void setDependsOn(@Nullable String... var1);

    @Nullable
    String[] getDependsOn();

    void setAutowireCandidate(boolean var1);

    boolean isAutowireCandidate();

    void setPrimary(boolean var1);

    boolean isPrimary();
	
	// ... 
  // 이 외에도 여러 옵션들이 있음.
}
```

많은 옵션들이 있지만, 
IoC Conatainer의 `BeanDefinition`이 등록될 때 꼭 필요한 것은 `beanClassName`이다.

실제 IoC Container는 `BeanDefinition`을 재활용 할 수 있기 때문에, 
`Bean의 아이디(또는 이름)`는 `BeanDefinition`에 없다.

Bean을 등록할 때  `Bean의 아이디(또는 이름)`는 `BeanDefinition`이 등록될 때 IoC Container가 같이 부여한다.

## 어노테이션과 BeanScanning

앞서, `BeanDefinitionReader`가 리소스를 `BeanDefinition`으로 만들어준다고 했다.

하지만, 모든 Bean을 XML 문서와 같이 일일이 선언하는 것이 귀찮을 수도 있다.

Bean으로 사용될 클래스에 특정 애노테이션을 붙이면, 이런 클래스를 자동으로 찾아서 `BeanDefinition`으로 만들어주는 클래스가 있다.

바로 BeanScanner이다.
> 물론 어노테이션이 붙은 클래스를 명시적으로 BeanDefinition으로 바꿔주는 `AnnotatedBeanDefinitionReader` 도 있다.

빈 스캐닝시, 등록할 클래스를 찾게되는 원리는 다음과 같다.

1. 스캐닝될 `basePackage`들을 지정한다.  `basePackeges`   과 그 서브 패키지들이 자동 검색 대상이 된다.
2. `@Component`과 `@Component`를 메타 애노테이션으로 가진 애노테이션이 붙은 클래스를 모두 찾아 `BeanDefinition`으로 변환 후 Bean으로 등록한다.
또한, `@Configuration`이 붙은 클래스들을 찾아, `@Bean`이 붙은 메서드를 통해서도 Bean으로 등록해준다.

> `@Component`를 메타 애노테이션으로 가진 애노테이션들의 예들은 다음과 같다.
> `@Repositoy` , `@Service`, `@Controller` …
> 이같은 애노테이션들을 스테레오타입 애노테이션이라고 하며, 
> 종류에 따라 부가기능을 부여해준다.

## @ComponentScan

마지막으로 SpringBoot가 어떻게 `basePackage`를 지정하는지 살펴보자.

Spring Initializer로 스프링 부트 프로젝트를 생성했을 때,
`basePackage`를 지정하지 않았지만, 코드에 관련 애노테이션들을 붙이면 빈으로 동작하는 걸 확인할 수 있다.

이는 이미 `basePackage`가 지정되었기 때문이다.

`@SpringBootApplication`을 보면 `@ComponentScan`이 붙어있는 걸 볼 수 있다.

`@ComponentScan`을 통해 scanning할 `basePackage`를 지정할 수 있으며,
spring boot는 기본적으로 `@SpringBootApplication`이 붙은 클래스의 패키지를 `basePackage`로 지정한다.

# 글을 마치며

---

빈이 어떻게 등록되는지 알 수 있는 좋은 주제였다.

이 어노테이션을 통해 등록되는 빈에 대하여 또 여러가지 주제들이 있기 때문에,
이 글에 더 수정하던지, 새로운 글을 통해 언젠가 해보려 한다.

이제 @MVC로 가보자.

> 참조
> 토비의 스프링
> 오브젝트
> docs.spring.io
> [java - Difference between register() and @ComponentScan - Stack Overflow](https://stackoverflow.com/questions/62512172/difference-between-register-and-componentscan)
