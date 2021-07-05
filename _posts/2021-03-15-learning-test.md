---
layout : post
title : "학습테스트로 프레임워크, 라이브러리, API를 학습해보자"
image : assets/img/testing.jpg
---

# 학습 테스트란? 

---

>자신이 만들지 않은 프레임워크나 라이브러리 등에 대해 작성하는 테스트다.

이를 통해 여러가지 목적을 달성할 수 있다.

1. 기능을 손쉽게 확인해보기.
2. 개발 중 만들어놓은 학습 테스트 코드를 참고.
3. 프레임워크나 라이브러리의 버전이 바뀔 때, 그에 대한 검증 가능.
4. 새로운 기술을 배우는 것이 즐거워진다.

# 토비의 스프링의 예

---

## Junit

>Junit은 Unit Test 도구다.

Junit은 테스트 메서드 수행시마다 새로운 테스트 오브젝트를 생성한다.
실제로 그런지 확인해보자

다음과 같이 프로젝트를 생성해주자.

![](https://images.velog.io/images/hyunmin/post/21d2a2f6-cca8-4b91-9a79-f9d1f69b12f1/image.png)프로젝트를 `spring boot initializar`를 통해 스프링 부트를 생성하면 

아래와 같은 dependency가 자동으로 추가되어 있다.
`org.springframework.boot:spring-boot-starter-test`

이는 아래와 같은 라이브러리를 사용할 수 있게 해준다.
- JUnit 5
- Spring Test & Spring Boot Test:
- AssertJ
- Hamcrest
- Mockito
- JSONassert
- JsonPath

junit5 라이브러리가 추가되어 있으므로 
이를 이용해 테스트 코드 작성 및 실행이 가능하다.
![](https://images.velog.io/images/hyunmin/post/4f508e39-45c4-4957-b2ad-0bf0ce97a737/image.png) 못보던 Spring Native가 생겼다. 
언젠가 기회가 된다면 이것도 포스트해보아야겠다!
![](https://images.velog.io/images/hyunmin/post/c9865963-d239-4255-a01b-b4e3bcfa043b/image.png)![](https://images.velog.io/images/hyunmin/post/ea44cde5-edd8-4d98-8d57-2763d15ed5f1/image.png)



Test 코드를 작성해보자.

다음 클래스를 찾자.
`src/test/java/com.example.learningwithtest/LearningwithtestApplicationTests`

그리고 아래와 같이 코드를 작성하자.
```java
package com.example.learningwithtest;

import org.junit.jupiter.api.Test;
import java.util.HashSet;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class LearningwithtestApplicationTests {
    static Set<Object> testObjects = new HashSet<>();

    @Test
    public void test1(){
        assertCreateNewTestObject();
    }

    @Test
    public void test2(){
        assertCreateNewTestObject();
    }

    @Test
    public void test3(){
        assertCreateNewTestObject();
    }

    private void assertCreateNewTestObject(){
        assertFalse(testObjects.contains(this));
        testObjects.add(this);
        assertTrue(testObjects.contains(this));
    }
}
```
테스트 객체를 저장하기 위하여 HashSet 을 이용하였다.
static으로 선언함으로 모든 객체들이 공유할 수 있도록 하였다.
```java
static Set<Object> testObjects = new HashSet<>();
```

코드의 핵심인 `assertCreateNewTestObject()` 메서드를 살펴보자.


현재 객체(테스트 객체)가 set에 없는지 확인한다.
매번 새로운 객체가 생성되는게 아니라면 분명 어디서는 실패해야 한다.
```java 
assertFalse(testObjects.contains(this));
```

실패하지 않았다면 다음 테스트 오브젝트에서 확인할 수 있도록,
set에 넣고 잘 들어갔는지 테스트 한다.
```java
testObjects.add(this);
assertTrue(testObjects.contains(this));
```

이제 테스트를 실행해보자.
![](https://images.velog.io/images/hyunmin/post/d6083c54-41b3-49b4-bb8d-68ff30eb8db4/image.png)성공이다.

## Context

이번에는 스프링을 테스트해보자.
컨텍스트는 무조건 한개만 생성되는지 확인해보자.

프로젝트는 그대로 사용하고, 아래와 같이 코드를 변경하자.
```java
package com.example.learningwithtest;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.ApplicationContext;

import java.util.HashSet;
import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
class LearningwithtestApplicationTests {
    @Autowired
    ApplicationContext context;

    static Set<ApplicationContext> contexts = new HashSet<>();

    @Test
    public void test1(){
        assertCreateOnlyOneContext();
    }

    @Test
    public void test2(){
        assertCreateOnlyOneContext();
    }

    @Test
    public void test3(){
        assertCreateOnlyOneContext();
    }

    private void assertCreateOnlyOneContext(){
        if(contexts.isEmpty()){
            contexts.add(this.context);
        }
		  assertTrue(contexts.size() == 1);
        assertTrue(contexts.contains(this.context));
    }
}
```

`@SpringBootTest` 애너테이션을 새롭게 클래스에다가 적용시킨 것을 볼 수 있다.

이 애너테이션도 스프링 이니셜라이저를 통해 
자동으로 추가되어 있어 바로 사용가능하다.

`@SpringBootTest`는 통합 테스트를 위해 제공되는 어노테이션으로,
어플리케이션 컨텍스트를 가져올 수 있게 해준다.

코드의 핵심 `assertCreateOnlyOneContext()` 메서드를 살펴보자

`contexts`이 비어있으면 추가해준다.
```java
if(contexts.isEmpty()){
    contexts.add(this.context);
}
```

비어있을 때만 추가하므로 반드시 한개만 `contexts`에 있음이 자명하다.
그래도 아래 테스트 코드로 확인
```java
assertTrue(contexts.size() == 1);
```

현재 테스트 오브젝트가 가지고 있는 컨텍스트를 
`contexts`가 가지고 있는지 확인한다.

만약 가지고 있지 않아면 테스트는 실패할 것이다.
```java
assertTrue(contexts.contains(this.context));
```

테스트를 돌려보자.
![](https://images.velog.io/images/hyunmin/post/d4365a0d-ff17-496d-b5f6-caa15348f564/image.png)성공 하였음을 볼 수 있다.
따라서 우리는 테스트 오브젝트들이 하나의 컨텍스트를 공유함을 알 수 있다.



# 심심이 API

---

학교에서 하는 팀프로젝트에서 여러가지 프로젝트 주제 아이디어가 나왔었다.
이 중에 심심이 API를 사용하는 아이디어가 있었다.

학습테스트를 통해 심심이 API이 어떤건지 학습해보자.

##  API 발급하기

사이트는 [SimSimi Workshop](https://workshop.simsimi.com/) 로 들어가면 된다.

회원가입하는 방법은 간단하다.

1. 로그인 -> 계정 만들기에서 이메일과 사용할 비밀번호를 입력한다.
2. 이메일에서 확인 메일을 확인한다. 내용에 있는 링크를 눌러 인증한다.
3. 로그인한다.
4. 개인 정보를 입력한다.

다 입력하고 나면 다음 화면처럼 Demo로 100회 사용할 수 있는 API 키를 받게 된다.
사용하기 전에는 상단에 프로젝트 활성화를 꼭 해줘야한다.
![](https://images.velog.io/images/hyunmin/post/02cf6680-9e77-41a1-b03f-65e47ca8acca/image.png)

그러면 이제 간단하게 살펴본 후 테스트 해보자.

## 일상 대화 API

이미 심심이 API에는 1억 건 이상의 일상대화 전용 대화세트들이 있으며,
일상 대화 API를 이용하면 이 대화세트들에서 데이터를 얻을 수 있게 된다.

일단 기본적으로 대화세트는 다음과 같이 이루어져 있다.

> 각 대화세트(talkset)는 질문문장(qtext)과 답변문장(atext)의 쌍으로 이루어집니다.

전체적인 흐름은 다음 그림과 같다.
![](https://images.velog.io/images/hyunmin/post/e6e56079-39ba-4012-b6a1-a46b46694acd/image.png)

## 가르치기 API

가르치기 API를 이용하면 `질문-대답`세트를 저장할 수 있다.

가르치기 API를 사용해 만든 대화세트들은 커스텀 저장소에 저장된다.

일상대화 API를 통해 결과를 가져올 때, 심심이 API는 다음과 같은 과정으로 결과를 가져온다.

1. 먼저 지정한 커스텀 대화세트 저장소에서 응답을 선택한다.
![](https://images.velog.io/images/hyunmin/post/0aa81d28-4450-434c-b0f1-f551343a238a/image.png)

2. 적절한 응답을 가져오지 못하면, 심심이 대화세트 저장소에서 응답을 가져온다.
![](https://images.velog.io/images/hyunmin/post/c769a80a-db00-4103-8a1a-99038799b711/image.png)

그렇다면 이제 테스트를 해보자.

## 학습 테스트 코드 작성
테스트 하고 싶은 내용은 총 3가지이다.

1. 일상대화 API를 통해 제대로 가져오는지.
2. 가르치기 API를 통해 데이터를 생성하는지.
3. 가르치기 API를 통해 데이터를 생성하기 전 후에, 일상대화 API로 가져오는 결과가 다른지.

차근차근 수행해보자.

## 일상대화 API 테스트 코드

먼저 요청 예시를 살펴보자.
```
curl -X POST https://wsapi.simsimi.com/190410/talk \
     -H "Content-Type: application/json" \
     -H "x-api-key: PASTE_YOUR_PROJECT_KEY_HERE" \
     -d '{
            "utext": "안녕", 
            "lang": "ko" 
     }'   
```

`header`에 추가할 내용은 두가지이다.
1. content type이 json 형태인 것
2. x-api-key에 api key를 줄 것

`body`에 추가할 내용은 두가지이다.
1. 심심이에게 보낼 내용
2. 언어

심심이 API에 접근할 때 `RestTemplate`을 사용하자.
`RestTemplate`는 쉽게 Rest 형식으로 메시지를 전송할 수 있다.

`RestTemplate`는 `spring-web` 의존성에 포함되어 있다.

아까 스프링 프로젝트를 만들때 추가하지 않았으므로
`build.gradle`에 아래처럼 dependency를 추가해주자.
`implementation group: ‘org.springframework’, name: ‘spring-web’, version: ‘5.3.4’`

이제 테스트 코드를 작성해보자.
```java
@SpringBootTest
class LearningwithtestApplicationTests {
    private final Logger logger = LoggerFactory.getLogger(this.getClass()); // log를 찍기 위해 사용

    static final String baseUri = "https://wsapi.simsimi.com/190410/"; // 요청할 base uri
    static final String talkAPIKey = "my talk api key"; // 대시보드에 있는 API 키를 가져오자.

    static JsonObject talkBody1; // post시 보내는 body 내용

    @BeforeAll
    public static void setUp(){
        talkBody1 = new JsonObject();
        talkBody1.addProperty("utext", "안녕 velog야");
        talkBody1.addProperty("lang", "ko");
    }

    @Test
    public void testTalkAPI(){
        JsonObject resultJson = postTalk(talkBody1);
        assertTrue("200".equals(resultJson.get("status").toString()));
        logger.info(resultJson.get("atext").toString());
    }

    private JsonObject postTalk(JsonObject body){
        return postAPI("talk", talkAPIKey, body);
    }

    private JsonObject postAPI(String api, String key, JsonObject body){
        RestTemplate restTemplate = new RestTemplate();

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.add("x-api-key", key);

        HttpEntity<String> request = new HttpEntity<String>(body.toString(), headers);
        URI uri = URI.create(baseUri + api);

        ResponseEntity<String> response = restTemplate.postForEntity(uri, request, String.class);
        return JsonParser.parseString(response.getBody()).getAsJsonObject();
    }
}
```

차근차근 살펴보자.

먼저 `talkBody1`부터 살펴보자.

`TalkBody1`는 gson의 `JsonObject` 타입이다.
아래 dependency를 `build.gradle`에 추가해주자.
`implementation group: 'com.google.code.gson', name: 'gson', version: '2.8.6'`

아래 코드는 `talkBody1`을 초기화 하는 코드이다.
보낼 메시지와 언어를 지정해준다.
```java
@BeforeAll
public static void setUp(){
    talkBody1 = new JsonObject();
    talkBody1.addProperty("utext", "안녕 velog야");
    talkBody1.addProperty("lang", "ko");
}
```

`postTalk()` 메서드와 그 메서드가 호출하는 `postAPI()`메서드를 살펴보자.

`postAPI()` 메서드는 `api`(여기서는 talk를 사용한다), `key`(x-api-key), `body`를 파라미터로 받는다.
이를 이용해 `header`를 만들고, 심심이 api에게 `body`와 함께 요청을 보낸다.

응답 받은 결과는 `JsonObject`형식으로 반환한다.
```java
private JsonObject postTalk(JsonObject body){
    return postAPI("talk", talkAPIKey, body);
}

private JsonObject postAPI(String api, String key, JsonObject body){
    RestTemplate restTemplate = new RestTemplate();

    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.APPLICATION_JSON);
    headers.add("x-api-key", key);

    HttpEntity<String> request = new HttpEntity<String>(body.toString(), headers);
    URI uri = URI.create(baseUri + api);

    ResponseEntity<String> response = restTemplate.postForEntity(uri, request, String.class);
    return JsonParser.parseString(response.getBody()).getAsJsonObject();
}
```

테스트 코드를 살펴보자.

`talkBody1`을 body로 하여 심심이 API로부터 결과를 받는다.

심심이 API는 잘 보내졌으면 `200` status code와 함께
 `atext(응답 메시지)`를 포함한 json 응답을 보낸다.

어떤 내용이 올 지 모르니, 정확한 응답 메시지가 뭔지 테스트 할 수는 없다.
하지만 status code를 통해 잘 보냈는 지 확인 할 수 있다.

만약 제대로 잘 보내졌다면 테스트는 통과할 것이다.
```java
@Test
public void testTalkAPI(){
    JsonObject resultJson = postTalk(talkBody1);
    assertTrue("200".equals(resultJson.get("status").toString()));
    logger.info(resultJson.get("atext").toString());
}
```


제대로 작동하는지 테스트해보자.
![](https://images.velog.io/images/hyunmin/post/d85bdf01-290c-4c96-8d46-3645c36e6447/image.png)성공!


## 가르치기 API 

이제 남은 2가지를 테스트해보자.

1. 가르치기 API를 통해 데이터를 생성하는지.
2. 가르치기 API를 통해 데이터를 생성하기 전 후에, 일상대화 API로 가져오는 결과가 다른지.

1번의 경우 가르치기 API를 호출 한 후에 status code가 200인지 살펴보면 된다.
2번의 경우 가르치기 API를 하기 전후로, 일상대화 API를 호출해서 응답 메시지가 서로 다른지 살펴보자.

Demo로 발급받은 API key는 Talk(일상대화) API이므로 Teach API를 발급받도록 하자.

빨간 테두리 쳐있는 부분을 클릭한다.
![](https://images.velog.io/images/hyunmin/post/5b407960-d898-40d5-8618-c5da7169739e/image.png)

여기서 Teach로 새 프로젝트를 만들기한다.
![](https://images.velog.io/images/hyunmin/post/a15b17db-7eac-445c-bacd-2821ff6caa60/image.png)

만든 후 다시 창에 들어가 Teach-Demo를 선택하여 들어간다.

새롭게 만든 Teach-Demo이다. 
새롭게 만들면 API 키가 없을 것이다.
![](https://images.velog.io/images/hyunmin/post/e8939386-a8b0-40f6-ad48-b01eb33d68b4/image.png)

Teach같은 경우에는 Demo로 제공하지 않기 때문에
별도로 요금제를 구독해야 한다.

하지만 테스트를 해보기 위해 제일 싼 $3 요금제를 구독한 후에 테스트를 진행하였다.


static 변수로 `talkBody2`와 `teachBody`를 추가한 후에
`setUp()`함수에 아래 코드를 추가하자.
```java
talkBody2 = new JsonObject();
talkBody2.addProperty("utext", "안녕 velog");
talkBody2.addProperty("lang", "ko");
talkBody2.addProperty("teach_key", "toVelog!");

teachBody = new JsonObject();
teachBody.addProperty("qtext", "안녕 velog야"); // teach api에서는 qtext(질문)을 포함해 총 4가지를 body에 포함시켜야 한다.
teachBody.addProperty("atext", "안녕, 반가워요~~");
teachBody.addProperty("lang", "ko");
teachBody.addProperty("teach_key", "toVelog!");
```

테스트 코드이다.
```java
@Test
public void testTeachAPI(){
    JsonObject resultTalkBeforeTeachJson1 = postTalk(talkBody1);
    JsonObject resultTalkBeforeTeachJson2 = postTalk(talkBody2);

    JsonObject resultTeachJson = postTeach(teachBody);
    assertTrue("200".equals(resultTeachJson.get("status").toString()));
    logger.info(resultTeachJson.get("request").toString());

    JsonObject resultTalkAfterTeachJson1 = postTalk(talkBody1);
    JsonObject resultTalkAfterTeachJson2 = postTalk(talkBody2);

    assertFalse(resultTalkBeforeTeachJson2.get("atext")
            .equals(resultTalkAfterTeachJson2.get("atext")));

    assertFalse(resultTalkAfterTeachJson1.get("atext")
            .equals(resultTalkAfterTeachJson2.get("atext")));
}

private JsonObject postTeach(JsonObject body){
    return postAPI("teach", teachAPIKey, body);
}
```

하나하나 살펴보자.

만약 우리가 Teach API를 통해 성공적으로 POST했다면 `200` status code를 받을 것이다.
```java
assertTrue("200".equals(resultTeachJson.get("status").toString())
```

가르치기 전 후로, `teach_key`를 통해 얻은 응답 메시지는 다를 것이다.
```java
assertFalse(resultTalkBeforeTeachJson2.get("atext")
        .equals(resultTalkAfterTeachJson2.get("atext")));
```

추가적으로 하나만 더 테스트해보자.

가르친 후라도, `teach_key`를 통해 커스텀 디비에 우선 접근 여부를 정할 수 있으니
`teach_key`를 이용한 것과 이용하지 않은 것은 결과가 다를 것이다.
```java
assertFalse(resultTalkAfterTeachJson1.get("atext")
        .equals(resultTalkAfterTeachJson2.get("atext")));
```

이제 테스트를 돌려보자.
![](https://images.velog.io/images/hyunmin/post/de83d5b8-3145-4482-bea1-1214a9af97e8/image.png)성공이다.

# 글을 마치며

---

이번에는 학습테스트에 대해 공부해보았다.

토비의 스프링에서 보고 좋겠다 싶어 정리할 겸 공부해보았다.

마침 테스팅 수업도 듣고 있고, 테스트가 중요하다고 생각하고 있기 때문에
앞으로 이어질 포스트에 가능하면 테스트 코드 작성을 통해 학습을 하려 한다.

# 참조

---

* 토비의 스프링 3.1

* [문서 | SimSimi Workshop](https://workshop.simsimi.com/document)
