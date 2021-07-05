---
layout: post
title:  "DispatcherServlet은 어떤 타입의 오브젝트라도 컨트롤러로 사용할 수 있다고?"
image : https://images.velog.io/images/hyunmin/post/3c9bada1-ca18-40f1-a124-1952a2db73c7/velog.001.jpeg
---

# 도입

---

Spring의 `Controller` 클래스를 작성하다보면 어떤 형식이던지 잘 작동하는 것을 보고 신기하였다.

대부분이 사용하는 `@RestController` 를 컨틀롤러 클래스에 붙이는 방법도 있고

`Controller` 인터페이스를 확장하여 사용하는 방법도 있었다.

어떻게 이렇게 여러가지 타입의 Controller를 사용할 수 있을까?

여기에는 여러 객체지향이 숨어있는데 한 번 알아보자.

먼저 어댑터 패턴에 대해 알아보자.

# 어댑터 패턴

---

> 클래스의 인터페이스를 객체 사용자가 기대하는 인터페이스 형태로 변환시킨다.
> 이를 통해, 서로 일치하지 않는 인터페이스를 갖는 클래스들을 함께 동작시킨다.

그 비결은 Adapter 클래스를 활용함에 있다.

사용하려는 객체가 다음과 같은 인터페이스를 갖는다 하자
```java 
class AdapteeA{
	Result request(int parm1, int parm2){
		...
	}
}
```

기본적으로 Adapter는 인터페이스와 그 구현 클래스로 구성한다.
Adapter 의 인터페이스는 다음과 같이 구성한다 하자.
```java
interface Adapter{
	boolean support(Object adaptee);
	Result handle(Object adaptee, Parameter parameter);
}
```

support 메서드는 Adapter가 Adaptee(사용할 객체)를 지원하는지 여부를 반환한다.

handle 메서드는 실제 Client가 호출하는 메서드로, 
Adapter는 Adaptee의 메서드를 호출한 후 결과를 반환한다.
```java
class ConcreteAdapterA implements Adapter{
	@Override
	boolean support(Object adaptee){
		return adaptee instanceof AdapteeA;
	}

	@Override
	Result handle(Object adaptee, Parameter parameter){
		return ((AdapteeA) adaptee).request(parameter.getA(), parameter.getB());
	}
}
```

클라이언트는 Adapter의 연산을 호출, Adapter는 연달아 지원하는 Adaptee의 연산을 호출한다.
이를 통해 Adapter는 클라이언트(객체 사용 객체)와 해당 Adaptee의 중간에서 연결시켜줄 수 있다.
```java
class Client{
	...
	void methodA(Adapter adpater, Object adaptee){
		... // parameter 생성
		if(adpater.support(adaptee)){
			Result result = adapter.handle(adaptee, parameter);
			... // do task
		}
	}
}
```

여기서 다른 인터페이스를 가진 AdapteeB가 있다고 해보자
```java
class AdapteeB{
	boolean request(boolean parm1, String parm2, int parm3){
		...
	}
}
```

AdapterB를 클라이언트에서 사용하려 한다면 어떻게 해야할까?

바로 이 AtapeeB를 사용하는 Adapter를 구현한 후, 클라이언트에서 사용하면 된다.
다음과 같이 ConcreteAdapterB를 만들자
```java
class ConcreteAdapterB{
	@Override
	boolean support(Object adaptee){
		return adaptee instanceof AdapteeB; // AdapterB인지 확인한다.
	}

	@Override
	Result handle(Object adaptee, Parameter parameter){
		boolean apateeResult = ((AdapteeB) adaptee).request(parameter.getC(), parameter.getD(), parameter.getE());
		return new Result(adapteeResult);
	}
}
```

이와 같이 AdapterB를 만든다면, 클라이언트는 바꿔야할 부분이 있는가?
AdapterB와 AdapteeB만 파라미터로 받으면, 바꿀 필요가 없어진다.

이처럼 어댑터 패턴을 활용하면, 클라이언트는 수정 없이 확장이 가능해진다.
단순히 대응하는 어댑터를 동일한 인터페이스로 사용하면 된다.

즉 OCP(개방-폐쇄 원칙)을 지킬 수 있게되어, 확장성이 무한해진다.


# DispatcherServlet은?

---

바로 DispacterServlet이 사용하는 방법이 Adapter Pattern이다.

다음과 같은 구성으로 되어있다.
![어댑터를 이용한 임의의 컨트롤러 호출 방식](https://images.velog.io/images/hyunmin/post/3c9bada1-ca18-40f1-a124-1952a2db73c7/velog.001.jpeg)

## HandlerAdapter

그렇다면 스프링에서 정의해놓은 HandlerAdapter를 보도록 하자.
```java
public interface HandlerAdapter{
	boolean supports(Object handler);
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
	long getLastModified(HttpServletRequest request, Object handler);
}
```

supports와 handle의 경우 위 어댑터 패턴에서의 형태와 비슷하다.

getLastModified는 변경되지 않은 리소스의 불필요한 전송을 방지하기 위해 사용된다.
지원하지 않을 경우 단순하게 -1을 반환하면 된다.

## DispatcherServlet의 HandlerAdapter 초기화

DispatcherServlet은 아래와 같이 HandlerAdapter를 저장하고 있다.
```java
@Nullable
private List<HandlerAdapter> handlerAdapters;
```

어떻게 HandlerAdapter를 가져오는지 보자

HandlerAdapter를 구현한 Bean들을 IoC 컨테이너를 통해 가져온다.
```java
Map<String, HandlerAdapter> matchingBeans =
		BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);

this.handlerAdapters = new ArrayList<>(matchingBeans.values());
```

그 후 HandlerAdapter들을 order(우선순위)를 기준으로 정렬한다.
```java
AnnotationAwareOrderComparator.sort(this.handlerAdapters);
```

HandlerAdapter를 우선순위에 따라 정렬하는 이유는 다음과 같다.

아래는 DispatcherServlet이 Handler(Adaptee)에 대응하는 HandlerAdapter를 찾는 과정이다.
어떤 HandlerAdapter를 적용할지를 우선순위에 따라 먼저 찾는다.
```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
   if (this.handlerAdapters != null) {
    	Iterator var2 = this.handlerAdapters.iterator();
    	while(var2.hasNext()) {
           HandlerAdapter adapter = (HandlerAdapter)var2.next();
           if (adapter.supports(handler)) {
              return adapter;
           }
        }
   }
	... // throw Exception
}
```

## DispatcherServlet의 HandlerAdapter 사용

그렇다면 실제로 이 HandlerAdapter를 어떻게 DispatcherServlet이 호출하는지 보자.

HandlerMapper를 통해 Handler(Adaptee)를 찾는다.
```java
mappedHandler = getHandler(processedRequest);
```

찾은 Handler에 대응되는 HandlerAdapter를 가져온다.
```java
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```

HandlerAdapter의 handle 메서드를 호출하여, 어댑터가 Handler를 호출하도록 한다.
HandlerAdapter는 작업이 끝난 후, ModelAndView 객체를 반환한다.
```java
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

반환받은 ModelAndView를 가지고 DispatcherServlet은 남은 작업을 마저 한다.

## HandlerAdapter 구현 예

이미 구현되어 있는  HandlerAdapter 중 하나인 SimpleHandlerAdapter를 살펴보자

SimpleHandlerAdapter는 Controller에 대응된다.
handle 메서드가 호출되면, 컨트롤러 타입의 어댑티를 실행한다.
```java
public class SimpleControllerHandlerAdapter implements HandlerAdapter {
	@Override
	public boolean supports(Object handler) {
		return (handler instanceof Controller); 
	}

	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
		return ((Controller) handler).handleRequest(request, response);
	}

	@Override
	public long getLastModified(HttpServletRequest request, Object handler) {
		...
	}
}
```


# 커스텀 핸들러 어댑터 만들어보기

---

DispatcherServlet이 HandlerAdapter를 통해 어떤 종류의 클래스라도 호출할 수 있음을 알았다.

정말 그런지 직접 아무 클래스나 만든 후, 
그에 대응하는 HandlerAdapter를 만들어보자.

그리고 이를 DispatcherServlet이 이용하게 하자.

## SpringBoot 어플리케이션 생성

아래와 같은 과정으로 프로젝트를 설정하자.
![](https://images.velog.io/images/hyunmin/post/4a09994b-6d08-4b23-9324-429784dfd545/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-03-06%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%2011.09.21.png)![](https://images.velog.io/images/hyunmin/post/646c03c2-3ace-4038-bcf6-00512f2458f1/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-03-06%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%2011.10.22.png)![](https://images.velog.io/images/hyunmin/post/db44697f-229e-41bd-a9a7-9bf34255065c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-03-06%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%2011.10.39.png)![](https://images.velog.io/images/hyunmin/post/a8593b51-272b-45d5-b2e2-f7d22aac891b/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-03-06%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%2011.11.01.png)![](https://images.velog.io/images/hyunmin/post/3dd5f7fd-fb37-435e-b654-2e4ef1d42092/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-03-06%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%2011.11.11.png)

## Custom Handler
Handler를 아래와 같이 작성하자.
```java
@Controller("/hello") // 기본 전략인 HandlerAnnotationHandlerMapping을 이용하기 위하여 적용한다. /hello에 매핑된다.
@Slf4j // log를 쉽게 사용하기 위해
public class HelloController {
    public String hello(String name){
        log.info("hello controller");
        return "hello " + name;
    }
}
```

만약 localhost:8080/hello?name=me를 실행하게 된다면 
로그에는 “hello controller”가 찍히며, 클라이언트는 “hello me”를 받을 것이다.

하지만 아래와 같은 오류를 보게 된다.
```
{
    "timestamp": "2021-03-15T05:10:25.742+00:00",
    "status": 500,
    "error": "Internal Server Error",
    "trace": "javax.servlet.ServletException: No adapter for handler 	[com.example.myhandleradapter.controller.HelloController@d0613b7]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports 
...
```

Handler에 대응되는 HandlerAdapter를 찾을 수 없다는 오류 메시지를 볼 수 있다.
HandlerAdapter를 작성하자.

## Custom HandlerAdapter
우리가 만들 HandlerAdapter의 기능은 다음과 같다.

1. Dispatcher가 handle 메서드를 호출하면 받은 메시지를 로그를 남긴다.

2. 그 후  ModelAndView에 보낼 메시지를 적용하여 보낸다.
```java
@Component
@Slf4j
public class MyHandlerAdapter implements HandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        return handler instanceof HelloController;
    }

    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String received  = ((HelloController)handler).hello(request.getParameter("name"));
        log.info("received : " + received);
        return new ModelAndView("index").addObject("received", received);
    }

    @Override
    public long getLastModified(HttpServletRequest request, Object handler) {
        return -1;
    }
}
```

@Component 어노테이션을 붙임으로, IoC 컨테이너가 이 클래스의 객체를 빈으로 등록하도록 한다.
그러면 DispatcherServlet이 IoC Container를 통해 MyHandlerAdapter의 객체를 가져올 수 있게 된다.

이후 다시 실행해보자

아래와 같은 결과를 클라이언트에서 볼 수 있다.
![](https://images.velog.io/images/hyunmin/post/bdcebe38-40f1-4837-9659-af1d48137ad9/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-03-15%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%202.14.44.png)


서버의 로그에는 아래와 같이 찍히게 된다.
![](https://images.velog.io/images/hyunmin/post/c2f4e346-3aab-4ec0-9298-4dbb4e07210b/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-03-15%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%202.18.21.png)

# 글을 마치며

---

DispatcherServlet이 어떻게 모든 타입의 객체를 Controller를 사용할 수 있는지 알아보았다.

물론 이렇게 커스텀으로 Handler Adapter를 작성할 수도 있지만 
어노테이션을 지원하는 HandlerAdapter를 이용하면 더욱 편하게 컨트롤러를 작성할 수 있게 된다.

이후에 작성할 글은 다음과 같다.
Spring의 MVC가 어노테이션으로 어떻게 작동하는지 알아보자.

# 참조

---

* 토비의 스프링(이일민 저)
* GoF Design Pattern
