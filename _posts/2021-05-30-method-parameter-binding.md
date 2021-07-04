---
layout : post
title : "@MVC는 어떻게 HttpRequest에서 원하는 형태로 데이터를 전달받아 사용할 수 있을까?"
---

# 도입
---

어노테이션을 활용한 컨트롤러를 작성하다보면 우리가 만든 Entity, Dto 나 int, bool, String 타입등으로 파라미터를 받아본 적이 있을 것이다.

사용자가 `HttpRequest`를 보낼 때 텍스트로 데이터가 되어있을 텐데 해당 타입으로 바뀌어서 사용할 수 있게 되는걸까?

스프링은 파라미터의 타입과 이름, 애노테이션 정보를 참고해서 그에 맞는 파라미터 값을 제공해준다.

즉, 스프링이 중간에서 해당 타입으로 바꿔주는 것인데, 이를 어떻게 하는지 알아보자.

# 메소드 파라미터 종류
--- 

@RequestMapping의 메소드에 사용할 수 있는  파라미터의 종류들을 메소드 파라미터라고 한다. 

> 좀 더 정확하게는 AnnotationMethodHandlerAdapter(3.0) 또는 RequestMappingHandlerAdapter(3.1 이상) 가 허용하는 메소드 파라미터의 종류이다.

먼저 메소드 파라미터의 종류에는 어떤 것들이 있는지 알아보고,

어떻게 파라미터에 데이터가 담기는지 알아보자.


## HttpServletRequest, HttpServletResponse

서블릿의 `HttpServletRequest`, `HttpServletResponse`이며, 모든 요청 / 반환 정보를 다 포함하고 있다.

```java
@RequestMapping("/example")
public String example(HttpServletRequest request, HttpServletResponse responese){
	...
}
```

`HttpServletRequest`, `HttpServletResponse`를 그대로 전달 받게 된다.

## HttpSession

`HttpSession`은 session 정보를 포함하고 있다.

```java
@RequestMapping("/example")
public String example(HttpSession session){
	...
}
```

`HttpServletRequest`에 있는 `HttpSession`을 전달받게 된다.

서버의 상태에 따라 멀티 쓰레드 환경에서 안정성이 보장되지 않으므로, 안전하게 사용하려면 핸들러 어댑터의 synchronizeOnSession 프로퍼티를 true롤 설정해줘야 한다.

## @PathVariable

@RequestMapping의 URL에 {}에 들어가는 패스 변수이다.

```java
@RequestMapping("/user/{userId}/order/{orderId}")
public String getMyOrder(@PathVariable("userId") long userId, @PathVariable("orderId") long orderId){
	...
}
```

{}에 들어가는 스트링을 `PropertyEditor`, `Converter`, `Formatter`에 의해 명시되어 있는 타입으로 변환하여 전달받게 된다.

## @RequestParam

Http 요청 파라미터를 메소드 파라미터에 1:1로 넣어주는 애노테이션이다.

```java
@RequestMapping("/example")
public String example(@RequestParm("name") String name){
	...
}
```

파라미터 목록 중 명시되어 있는 이름을 key로 갖는 파라미터를 찾는다.

찾은 파라미터의 value 스트링을 `PropertyEditor`, `Converter`, `Formatter`에 의해 명시되어 있는 타입으로 변환하여 전달받게 된다.

이를 사용하면 해당 파라미터가 반드시 있어야 하며, 없으면 `400 - Bad Request`를 받게 된다.

존재하지 않도록 설정하거나 디폴트 값을 주려면 required, defaultValue를 설정하면 된다.

## @CookieValue

Http 요청과 함께 전달된 쿠키 값을 메소드 파라미터에 넣을 때 사용한다.

```java
@RequestMapping("/example")
public String example(@CookieValue("auth") String auth){
	...
}
```

요청 헤더 목록에서 지정한 이름과 같은 해당 key를 가지는 찾는다.

찾은 파라미터의 value 전달받는다.

해당 파라미터가 반드시 있어야하며, 없으면 `400 - Bad Request`를 받게된다.

존재하지 않도록 설정하거나 디폴트 값을 주려면 required, defaultValue를 사용하면 된다.

## @RequestHeader

요청 헤더정보를 메소드 파라미터에 넣어주는 애노테이션이다.

```java
@RequestMapping("/example")
public String example(@RequestHeader("Host") String host){
	...
}
```

요청 헤더에서 지정한 이름과 같은 key를 가지는 파라미터를 찾는다.

찾은 파라미터의 value를 전달받는다.

마찬가지로 헤더 값이 반드시 존재해야하며

존재하지 않도록 설정하거나 디폴트 값을 주려면 required, defaultValue를 사용하면 된다.

## Model, ModelMap

```java
@RequestMapping("/example")
public String example(ModelMap model){
	model.addAttribute(new User(1, "Spring"));
}
```

모두 모델정보를 담는 데 사용할 수 있는 오브젝트가 전달된다.

addAttribute() 메소드로 오브젝트를 넣을 수 있으며, 

스프링은 이에 담긴 모든 오브젝트를 자동 이름 생성 방식을 적용해서 모두 모델로 추가해준다.

## @ModelAttribute

클라이언트로부터 컨트롤러가 받는 요청정보 중에서, 

하나 이상의 값을 가진 오브젝트 형태로 만들 수 있는 구조적인 정보를 @ModelAttribute 모델이라고 부른다.

```java
@RequestMapping("/example")
public String example(@ModelAttribute User user){
	...
}
```

요청 파라미터를 메소드 파라미터에서 1:1로 받는 @RequestParam과는 다르게,

@ModelAtribute는 도메인 오브젝트나 DTO의 프로퍼티에 요청 파라미터를 바인딩해준다.

바인딩되는 과정(+검증) 은 다음과 같다.

1. 파라미터 타입의 오브젝트를 디폴트 생성자를 이용해 만든다. 만약, @SessionAttributes에 의해 세션에 저장된 모델 오브젝트가 있다면, 새로운 오브젝트를 생성하는 대신 세션에 저장되어 있는 오브젝트를 가져온다.
2. 준비된 모델 오브젝트의 프로퍼티에 웹 파라미터를 바인딩해준다. 
    - 이 때 프로퍼티가 스트링 타입이 아니라면 `PropertyEditor`, `Converter`, `Formatter`를 통해 변환을 한다.
    - 이 때 실패하면 `BindingResult` 오브젝트에 바인딩 오류를 저장한다.
3. 모델 값을 검증한다. 

또한 @ModelAttribute는 모델 오브젝트를 자동으로 모델 맵에 추가해준다.

> @ModelAttribute는 생략이 가능하다.

## Errors, BindingResult

@ModelAttribute가 붙은 파라미터를 처리할 때는 @RequestParam과 달리 Validation 작업이 추가적으로 진행된다. 변환이 불가능하면, `Errors`와 `BindingResult`에 해당 오류가 저장된다.

```java
@RequestMapping("/example")
public String example(@ModelAttribute User user, BindingResult bindingResult){
	...
}
```

@RequestParm이 바인딩이 불가능하면 바로 작업이 중단되고 `400 - Bad Request` 가 전달되는데, 

왜 모델 바인딩에는 중단되지도 않고 결과가 `Errors`와 `BindingResult`에 저장되는 것일까?

1. 그 이유는 @RequestParam과는 다르게 모델 오브젝트의 프로퍼티와 타입이 일치하는지 뿐만 아니라, 다른 여러 검증 작업 (필수정보의 입력 여부, 길이 제한, 포맷, 값의 허용범위 등) 을 수행하기 때문이다. 즉, 바로 중지가 된다면, 다른 검증 에러들이 담기지 않게되기 때문이다.
2. 또한 웹사이트에서 회원가입을 하는 것을 예로 들 수 있다. 특정 항목이 잘못되었다고 `400 - Bad Request` 가 나면 사용자 입장에서 황당할 수 있다. 따라서 컨트롤러에게 에러 처리에 대해 맡기는 것이다.

주의할 점은 다음과 같다. 

1. 모델 어트리뷰트를 사용할 때는 `Errors`나 `Binding` 파라미터를 함께 사용하지 않으면 스프링이 바인딩에 문제가 없도록 애플리케이션이 보장해준다고 생각한다. 이 때는 문제가 생기면 `BindingException` 예외가 던져진다.
이 때는 따로 `400 - Bad Request` 로 변환되지도 않으니, 적절하게 예외처리를 해줘야한다.
2. 또 사용시 반드시 주의할 점은, @ModelAttribute 파라미터 바로 뒤에 둬야한다는 점이다. 그 이유는 자신의 앞에 있는 @ModelAttribute 파라미터의 검증 작업에서 발생한 오류만을 전달해주기 때문이다.

## SessionStatus

세션 내에 모델 오브젝트가 저장할 필요가 없을 때, 오브젝트를 세션에서 제거해주는 작업을 `SessionStatus`를 통해 할 수 있다.

```java
@RequestMapping("/example")
public String example(SessionStatus sessionStatus){
	sessionStatus.setComplete();
}
```

## RequestBody

Http Request의 본문 부분이 그대로 전달된다.

```java
@RequestMapping("/example")
public String example(@RequestBody String body){
	...
}
```

이는 XML 또는 JSON 기반의 메시지를 사용하는 요청의 경우에 매우 유용하다.

`MessageConverter`에 의해 해당 타입으로 변환이 된다.

`AnnotationMethodHandlerAdapter`에는 `HttpMessageConverter` 타입의 메시지 변환기가 여러 개 들어 있다.

- `StringHttpMessageConverter` 는 모든 종류의 미디어 타입을 String 타입으로 변환해준다.
- `MarshallingHttpMessageConverter` 는 미디어 타입이 XML일 때, 오브젝트로 변환해준다.
- `MappingJacksonHttpMessageConverter` 는 미디어 타입이 Json일 때, 모델 오브젝트로 변환해준다.

@RequestBody가 붙은 파라미터가 있으면, 스프링은 다음과 같은 작업을 수행한다.

1. HTTP 요청의 미디어 타입과 파라미터 타입을 먼저 확인한다.
2. 메시지 변환기 중에서, 해당 미디어 타입과 파라미터 타입을 처리할 수 있는 것이 있다면, HTTP 요청의 Body 부분을 통째로 변환해서 지정된 메소드 파라미터로 전달해준다.


# 프로퍼티 바인딩
---

HttpRequest 정보는 모두 문자열이기 때문에, @ModelAttribute, @RequestParam @PathVariable 이 붙은 파라미터 타입이 String이 아니라면 적절한 변환을 해야한다.

이를 위해 `PropertyEditor`, `Converter`, `Formatter` 등 총 3가지 변환 API를 사용할 수 있다.

프로퍼티 바인딩에는 다음과 같은 순서대로 먼저 적용된다.

1. @InitBinder에 직접 등록한 `PropertyEditor`와 `ConversionService`
2. 모든 컨트롤러에 적용되는 커스텀한 `PropertyEditor`
3. 모든 컨트롤러에 적용되는 커스텀한 `Converter`, `Formatter`
4. Default `Converter`
5. Default `PropertyEditor` 순으로 적용된다.

## PropertyEditor

디폴트 에디터가 20개 가량 있으며, 목록은 다음과 같다.

```jsx
1. File  2. URL 3. Charset 4. Class 5. Class[] 6. Currency 7. InputStream
8. InputSource 9. Locale  10. Pattern 11. Properties 12. Resource[]
13. TimeZone 14. UUID 15. byte[] 16. char[] 17. char 18. boolean
19. double 20. String[]
```

위에서 본 20가지 타입외에 다른 타입으로 변환하고 싶다면 `PropertyEditorSupport` 를 extends하여 직접 구현해주어야 한다.

`PropertyEditorSupport` 에서 기본적으로 구현해줘야 하는 목록은 다음과 같이 `getAsText()`  와 `setAsText` 가 있다.

```java
class PropertyEditorSupport implements PropertyEditor{
	public String getAsText(); // 저장되어 있는 해당 타입 오브젝트를 String 변환해준다.
  void setAsText(String text); // text를 가지고 해당 타입의 객체를 생성 후, 저장한다.
}
```

구현한 PropertyEditor를 다음 중 하나의 방식을 선택해서 등록할 수 있다.

1. Controller 내 @InitBinder 메소드에서 등록해준다. ( 작성된 컨트롤러에서만 PropertyEditor가 작동한다 )

```java
@InitBinder
public void initBinder(WebDataBinder dataBinder){
	// 아래는 타입을 기준으로 적용되는 프로퍼티 에디터 등록이다.
	dataBinder.registerCustomEditor(MyClass.class, new MyCustomPropertyEditor());
	// 또는 다음과 같이 특정 이름으로 적용되는 프로퍼티 에디터 등록이다.
	dataBinder.registerCustomEditor(int.class, "age", new MyCustomAgePropertyEditor());
}
```
 2. `WebBindingInitializer` 인터페이스를 구현하고 AnnotationMethodHandlerAdapter의 webBindingInitializer 프로퍼티에 DI 해준다. ( 모든 컨트롤러에 적용된다 )

커스텀 에디터는 @RequestParam, @CookieValue, @RequestHeader, @PathVariable, @ModelAttirbute 파라미터에 바인딩 할 때 모두 적용될 수 있다.

프로퍼티 에디터는 해당 타입의 객체를 저장한다. 

따라서 쓰레드 세이프 하지 않으므로 싱글톤 빈으로는 적합하지 않으며, 프로토타입 빈으로 선언할 수는 있다. 

하지만, 이는 변환할때마다 매번 객체가 생성되는 것이므로, 이때는 `Converter`를 사용할 수 있다. 

## Converter

스프링 3.0에서 추가된 새로운 타입 변환 API로서, 변환 과정에서 메서드가 딱 한번만 호출된다.

즉, 인스턴스 변수를 저장해놓지 않는다는 의미므로, 싱글톤 빈으로 사용가능하다.

커스텀 컨버터를 만들기 위해서는 아래 인터페이스를 구현해야한다. 

단방향이므로, 스트링과 타입간 양방향 변환을 구현하려면 2개의 컨버터를 만들어야 한다.

```java
public interface Converter<S, T>{
	T convert(S source)
}
```

커스텀 컨버터를 등록하는 방법은 `ConversionService` 타입의 오브젝트를 통해 `WebDataBinder` 에 설정해주어야 한다. `WebDataBinder` 에 설정하는 방법은 다음과 같다.

1. 스프링 부트를 사용시 빈으로 만들어주기. `WebConversionService` 가 자동으로 등록해준다
2. Web Config에 등록한다.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new MyCustomConverter());
    }
}
```

## Formatter

스프링이 기본적으로 지원하는 타입 변환 API이다.

모든 타입간 변환을 지원하는 `Converter`와 다르게, String과 타입간 변환을 지원한다. 

현재 지역정보도 함께 제공되는 타입 변환 API이다. 이를 이용해 다국어 지원도 할 수 있다.

`Formatter`는 이렇게 컨트롤러 내의 바인딩과 타입 변환에 사용되도록 특화되었다고 볼 수 있다.

또한 상태 저장을 하지 않으므로 싱글톤 빈으로 만들 수 있다.

`Formatter` 인터페이스를 구현할 때, 다음과 같은 2가지 메소드를 구현해야한다.

```java
String print(T object, Locale locale);
T parse(String text, Locale locale) throw ParseException;
```

다음과 같은 방법들을 통해  등록할 수 있다.

1. SpringBoot를 사용시 빈으로 등록한다.
2. 아래와 같이 Web Config에 등록한다.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry formatterRegistry) {
        formatterRegistry.addFormatter(new MyFormatter());
    }
}
```

## 바인딩 기술의 적용 우선순위와 활용 전략

### 사용자 정의 타입의 바인딩을 위한 일괄 적용 →  Converter

User의 UserType 처럼 모델(User)에서 자주 사용되는 타입이라면 `Converter`로 만드는 것이 편하다.

프로퍼티 에디터로 만들면 매번 오브젝트가 새로 만들어진다.

### 필드와 메소드 파라미터, 애노테이션 등의 메타정보를 활용하는 조건부 변환 기능 → ConditionalGenericConverter

바인딩이 일어나는 필드와 메소드 파라미터 등의 조건에 따라 변환을 할지 말지 결정하거나,

이런 조건을 변환 로직에 참고할 필요가 있으면 `ConditionalGenericConverter`를 사용하는 것이 좋다.

가장 유연하고 강력한 기능을 가졌지만, 복잡하고 구현이 까다롭다.

## 에노테이션 정보를 활용한 HTTP 요청과 모델 필드 바인딩 → AnnotationFormatterFactory와  Formatter

@NumberFormat이나 @DateTimeFormat처럼 필드에 부여하는 애노테이션 정보를 이용할 변환 기능을 지원할 때, `AnnotationFormatterFactory`를 이용해 애노테이션에 따른 포맷터를 생성해주는 팩토리를 구현하면 좋다.

## 특정 필드에만 적용되는 변환 기능 : PropertyEditor

특정 모델의 필드에 제한해서 변환 기능을 적용해야할 때 좋다. 

프로퍼티 에디터가 지정된 이름을 가진 필드에 제한적으로 적용할 수 있기 때문이다.

조건에 따라 변환 기능 적용 여부는 `ConditionalGenericConverter`나 `AnnotationFormatterFactory`를 이용할 수도 있지만 구현 과정이 복잡하다.

따라서 단순 필드 조건만 판별할꺼변 프로퍼티 에디터가 낫다.

또한 이럴때는 `WebBindingInitializer`를 통해 모든 컨트롤러에 적용하는 것보다는 @InitBinder를 통해 특정 클래스에만 적용하는 것이 좋다.

## 글을 마치며

이번에는 HttpRequest로부터 어떻게 @Controller의 메소드의 파라미터로 바인딩 되는지 알아보았다.

Spring은 @Controller에 지원하는 파라미터들이 있으며, 

이 파라미터들을 바인딩할 때는 `PropertyEditor`, `Converter`, `Formatter` 등을 변환하여 사용한다.

생각보다 글이 너무 길어져 많이 정리하지 못한 내용이 있어 아쉽다. 


다음에는 Validation과 ControllerAdvice를 함께 다뤄봐야겠다.

> 참조 : 토비의 스프링 3.1
