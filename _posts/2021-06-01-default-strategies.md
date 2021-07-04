---
layout : post
title : "DispatcherServlet 의 Default 전략"
image : assets/img/default-strategies.png
---

# 도입

---

스프링은 상당 부분 우리가 신경쓰지 않아도 되도록 많은 부분을 기본적으로 설정해주고 있다.

특히 MVC 를 구현할 때, 우리는 다음과 같은 컨트롤러를 생성하고, 

```java
@Controller
public class GreetingController {
	@GetMapping("/greeting")
	public String greeting(){
		return "greeting";
	}
}
```

`src/main/resources/templates` 에 greetings.html 파일을 작성하면 

`localhost:8080/greeting` 으로 접속하여 html 화면을 볼 수 있다.



어떻게 이런것이 가능한 것일까? 

아래는 스프링의 MVC가 동작하는 흐름이다.

![SpringMVCArchitecture](../assets/img/SpringMVC.png)

이 그림에서 보더라도 `HandlerMapping`, `HandlerAdapter`, `ViewResolver` 라는 전략을 사용하는 것을 볼 수 있다.

`DispatcherServlet` 은 아래와 같은 전략들을 사용하며, 초기에 설정한다.

```java
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
```

우리가 설정해 주지 않았지만 이미 기본적으로 스프링에서 정의해놓은 default 설정들이 있다.

그 디폴트 설정은 `DispacherServlet.properties` 에 있으며, 다음과 같다.

```properties
org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter


org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

`HandlerMapping`, `HandlerAdapter`, `HandlerExceptionResolver` 는 여러 개의 디폴트 설정이 있으며, 상황에 맞는 전략을 사용한다.

이제 각 전략별로 디폴트 전략들이 어떤 것들인지 알아보자.

# MultipartResolver

---

> DispatcherServlet이 Multipart Request 를 받았을 때, 
> MultipartHttpServletRequest 인터페이스를 호출할 수 있도록 Wrapped 된 HttpServletRequest 를 만들어주는 전략
 
> MultipartHttpServletRequest 의 인터페이스를 통해 MultipartFile 을 가져올 수 있다.

스프링은 `MultipartResolver` 는 따로 디폴트 설정을 하지 않는다. 

`MultiplartResolver` 을 사용하려면 `MultiplartResolver` 인터페이스의 구현체를 빈으로 만들고, 그 빈의 id를 `multipartResolver` 로 지정하면 된다.

스프링은 구현체인 `CommonsMultipartResolver` 와 `StandardServletMultipartResolver` 을 내장하고 있으며 보통 `CommonsMultipartResolver`가 자주 사용된다.

MultipartFile을 가져올 수 있는 방법은 다음과 같이 2가지 방법이 있다.

```java
 public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) {
   MultipartHttpServletRequest multipartRequest = (MultipartHttpServletRequest) request;
   MultipartFile multipartFile = multipartRequest.getFile("image");
   ...
 }
```

```java
public String handleRequest(@RequestParam("file") MultipartFile multipartFile) {
    
  }
```


# LocaleResolver

---

> 사용자에게 적절한 언어로 서비스를 제공하기 위해, 지역 정보를 저장/반환하는 전략

> request, cookie, session 등에 기반하여 지역 정보를 저장/반환한다.

디폴트 전략은 `AcceptHeaderLocaleResolver` 로 HTTP header의 `Accept-locale` 헤더로부터 Locale을 지정한다.

# HandlerMapping

---

> 사용자의 요청을 처리할 Handler를 선정하는 전략

`BeanNameUrlHandlerMapping` , `RequestMappingHandlerMapping` , `RouterFunctionMapping` 가 디폴트 전략이다. 

`BeanNameUrlHandlerMapping` 은 빈 이름이 request의 url 과 일치하는 컨트롤러를 선정한다.

`RequestMappingHandlerMapping` 는 `@RequestMapping`에 명시되어 있는 url이 request url과 일치하는 컨트롤러를 선정한다.

`RouterFunctionMapping` 은 MVC가 아닌 웹플럭스의 `RouterFunction`을 지원하는 핸들러 매핑 전략이다. 


# HandlerAdapter

---

> 특정 핸들러(컨트롤러) 객체의 메서드를 호출하며, 호출하기 전/후 처리를 하는 전략

> HandlerAdapter는 컨트롤러의 어댑터로 DispatcherServlet 은 컨트롤러의 메서드를 직접 호출하는 대신 HandlerAdapter의 메서드를 호출한다.
 
디폴트 전략에는 `HttpRequestHandlerAdapter`, `SimpleControllerHandlerAdapter`, `RequestMappingHandlerAdapter`, `HandlerFunctionAdapter` 가 있다.

`HttpRequestHandlerAdapter` 는 `HttpRequestHandler` 인터페이스의 구현체를 지원하는 어댑터이다.

`SimpleControllerHandlerAdapter`는 `Controller` 인터페이스의 구현체를 지원하는 어댑터이다.

`RequestMappingHandlerAdapter`는 `@RequestMapping` 어노테이션이 붙은 컨트롤러를 지원하는 어댑터이다.

> RequestMappingHandlerAdapter가 어떤 파라미터를 어떻게 지원하는지는{% link _posts/2021-05-30-method-parameter-binding.md %} 을 참고하기 바랍니다.

`HandlerFunctionAdapter`는 스프링 웹플럭스의 `HandlerFunctions`를 지원한다.

> 핸들러 어댑터에 대해서는 이전에 작성한 {% link _posts/2021-03-03-handler-adapter.md %} 를 참고하기 바랍니다.

# HandlerExceptionResolver

---

> 컨트롤러 작업 중 발생한 예외를 어떻게 처리할지 결정하는 전략

디폴트 전략은 `ExceptionHandlerExceptionResolver`, `ResponseStatusExceptionResolver` , `DefaultHandlerExceptionResolver`가 있다.

`ExceptionHandlerExceptionResolver` 는 `@ExceptionHandler` 어노테이션이 붙은 메서드에서 예외를 처리하는 전략이다.

`ResponseStatusExceptionResolver` 는 `@ResponseStatus`를 특정 예외 클래스에 붙여, 해당 예외 발생시 지정한 status code와 이유를 반환할 수 있다.

`DefaultHandlerExceptionResolver` 는 마지막으로 실행되는 예외 처리 전략이다. 발생한 예외에 따라 특정 status code를 반환하게 된다. 

실제 어떤 예외에 어떤 status code가 반환되는지는 [Class DefaultHandlerExceptionResolver](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/mvc/support/DefaultHandlerExceptionResolver.html) 를 참고하면 된다.

> 스프링에서의 예외처리에 대한 글을 작성할 계획이 있다. ExceptionHandelerExceptionResolver와 ControllerAdvice, Valid를 사용할 예정이다.

# RequestToViewNameTranslator

---

> 컨트롤러에서 view 이름을 명시적으로 반환하지 않았을 때 view 이름을 생성하는 전략

디폴트 전략은 `DefaultRequestToViewNameTranslator` 이다. 

이 전략은 request의 url에 따라 빈 이름을 생성해준다.

기본으로는 다음과 같이 생성해주며, prefix 및, suffix를 설정할 수도 있다.
```
http://localhost:8080/gamecast/display.html » display
http://localhost:8080/gamecast/displayShoppingCart.html » displayShoppingCart
http://localhost:8080/gamecast/admin/index.html » admin/index
```

# ViewResolver

---

> String으로 주어진 뷰 이름으로부터 사용할 뷰 객체를 결정하는 전략

디폴트 전략은 `InternalResourceViewResolver`이다. 

이 전략은 컨트롤러가 지정한 뷰 이름을으로부터 사용될 뷰를 선택하며, prefix 및, suffix를 설정할 수 있다.

# FlashMapManager

---

> FlashMap 객체를 저장, 반환하기 위해 사용하는 전략

디폴트 전략은 `SessionFlashMapManager` 로 Http Session을 이용해 FlashMap 객체를 저장, 반환을 한다.
