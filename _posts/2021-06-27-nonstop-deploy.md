---
layout : post
title : "기존 프로젝트를 무중단 배포로 바꿔보자! (via jenkins)"
image : assets/img/nonstop-deploy/architecture2.png
---

# 도입

---

이전 학기에 학교 프로젝트의 서버 인프라를 구성할 때, ec2 서버에 CI/CD를 구축한 적이 있다.
깃허브에 코드를 올리면 자동으로 배포가 되고, 서버를 재실행해준다는 점은 상당한 편리함을 제공해주었다.

아래는 기존에 구축한 아키텍처다.

![](../assets/img/nonstop-deploy/architecture1.png)

하지만 서버를 재실행하기 위해서 약 10초 가량 서버 재시동 시간이 있었고, 클라이언트를 개발하는 학우들에게는 갑자기 서버가 실행이 안된다는 불편함이 있었다고 한다.

따라서 이를 해결하기 위해 무중단 배포를 도입해보려 한다.

새롭게 구축하려는 아키텍처는 다음과 같다.

![](../assets/img/nonstop-deploy/architecture2.png)

기존 아키텍처와 다른점은 다음과 같다.

1. nginx가 프론트 서버 역할을 맡으며, 스프링에게 사용자의 요청을 전달한다. 
   
2. 젠킨스가 새롭게 서버 파일을 배포하면 스프링 서버 중 하나를 재시동한 후, nginx가 이 스프링을 가리키게 한다.

3. ec2를 2개 두어, 하나는 젠킨스용, 하나는 서버용으로 구축한다.

# ec2 인스턴스 swap memory 설정

---

> 프리티어의 마이크로 ec2 2대를 사용할 예정이다. 마이크로 ec2는 메모리가 1GB밖에 안되므로 이 과정을 진행한다. 만약 메모리가 여유있다면 이 과정은 뛰어넘어도 된다.

아래 명령어로 디스크 2GB를 스왑파일에 할당한다.
```shell
sudo dd if=/dev/zero of=/swapfile bs=128M count=16
```

스왑파일의 읽기, 쓰기 권한을 바꾼다.
```shell
sudo chmod 600 /swapfile
```

스왑 영역을 설정한다.
```shell
sudo mkswap /swapfile
```

스왑 공간에 스왑파일을 추가하여, 스왑파일을 사용할 수 있도록 한다.
```shell
sudo swapon /swapfile
```

![](../assets/img/nonstop-deploy/swap1.png)

스왑공간이 설정되었는지 확인한다.
```shell
sudo swapon -s
```

![](../assets/img/nonstop-deploy/swap-s.png)


`/etc/fstab` 파일을 vim으로 열어 다음 줄을 맨 마지막에 추가한다. 이를 통해 우분투가 재시동될 때 자동으로 스왑메모리를 설정한다.
```shell
/swapfile swap swap defaults 0 0
```

![](../assets/img/nonstop-deploy/swap2.png)

다음 메모리 확인 명령어로 확인한다.
```shell
free -m
```
![](../assets/img/nonstop-deploy/swap-free.png)

> ec2 인스턴스 2개 모두 위 과정을 수행하면 된다.

# 스프링 프로필 세팅

---

> 무중단 배포를 하기 위해서는 nginx가 두 개의 스프링을 구분할 수 있어야한다. 이를 위해 프로필 세팅을 해보자.

스프링 프로젝트 안에 WebRestController 클래스를 생성하고 아래와 같이 작성한다.

```java
@RestController
@RequiredArgsConstructor
public class WebRestController {
    private final Environment env;

    @GetMapping("/profile")
    public String getProfile(){
        return Arrays.stream(env.getActiveProfiles()).findFirst().orElse("");
    }
}
```

이 코드는 현재 동작중인 프로파일의 이름을 반환하게 된다.

어떻게 동작하는지 한번 확인해보자. 먼저 `resource` 디렉터리 안에 있는 `application.yaml`에 아래를 추가한다.

```yaml
spring:
  config:
    activate:
      on-profile: local
```

그리고 스프링을 재시동한다. 웹 브라우저로 `localhost:8080/profile` 로 접속해보자. local을 반환함을  확인할 수 있을 것이다.

![](../assets/img/nonstop-deploy/profile-localhost.png)

이제 이를 ec2 인스턴스에 적용해보자.

ec2 인스턴스에 접속한 뒤 다음 명령어로 디렉터리 2개를 만들어주자.

```shell
sudo mkdir ~/app
sudo mkdir ~/app/config
```

config 폴더에 `prod-application.yaml` 파일을 생성한 후 디비 설정과 같은 여러 프로퍼티를 설정한 후 다음을 맨 마지막에 추가해주자.

```yaml
---
spring:
  config:
    activate:
      on-profile: set1
server:
  port: 8081

---
spring:
  config:
    activate:
      on-profile: set2
server:
  port: 8082
```

yaml 파일은 한 문서 내에서 여러 개의 문서를 적을 수 있으며, 각각을 `---`로 구분할 수 있다. 

프로파일 설정을 `set1`으로 주면 8081 포트로 동작하고 `set2`로 주면 8082 포트로 동작하게 된다.
한번 확인해보자.

먼저 기존 스프링 프로젝트를 `~/` 폴더에 가져오자. 

![](../assets/img/nonstop-deploy/git-clone.png)

아래 명령어로 실행 파일을 만들자.

```shell
cd ~/Server
gradle build
```

실행파일을 `~/app` 디렉터리에 옮기자.

```shell
mv ~/Server/build/libs/bookreservationserver-0.0.1-SNAPSHOT.jar ~/app/
```

아래 명령어로 서버를 실행해보자. 이 때 `--spring.profiles.active=set1` 옵션이 있기 때문에 8081 포트로 동작하게 된다. 

```shell
java -jar bookreservationserver-0.0.1-SNAPSHOT.jar --spring.config.location=file:config/prod-application.yaml --spring.profiles.active=set1
```

다음 그림의 파란색 박스를 보면 8081 포트로 실행됨을 알 수 있다.

![](../assets/img/nonstop-deploy/run-spring-profile.png)

> 서버 실행 명령어는 쉘 스크립트로 옮길 것이다. 이후 과정에서 나올 것이다.


# nginx 설치 및 실행

---

아래 명령어로 nginx를 설치한다.

```shell
sudo apt-get install nginx
```

아래 명령어로 nginx를 실행한다.

```shell
sudo service nginx start
```

아래 명령어로 nginx가 잘 실행되었는지 확인한다.

```shell
sudo service nginx status
```

![](../assets/img/nonstop-deploy/nginx-status.png)


nginx는 포트 80번으로 구동된다. 따라서 nginx가 구동되는 ec2의 보안 그룹에서 80번 포트를 개방하자.

그 후에 웹 브라우저로 ec2 서버에 접속하면 다음과 같이 nginx의 기본 홈페이지 화면을 볼 수 있다.

![](../assets/img/nonstop-deploy/nginx-ec2.png)

# nginx 설정하기

---

> nginx가 스프링에게 사용자의 요청을 전달하기 위해 몇 가지 설정을 해줘야 한다.

아래 명령어로 설정 파일을 연다.
```
sudo vim /etc/nginx/sites-enabled/default
```

그리고 두 줄을 아래 사진처럼 추가한다.

```shell
include /etc/nginx/conf.d/service-url.inc;
proxy_pass $service_url;
```

![](../assets/img/nonstop-deploy/proxy-nginx.png)

> /etc/nginx/sites-enabled/default 라는 폴더가 없으면 /etc/nginx/nginx.conf 에서 해당 부분을 추가하여도 된다.
> 버전마다 좀 다른 것 같다.
> 기본적으로 nginx의 설정은 nginx.conf 이며 현재 내가 사용한 버전에서는 nginx.conf 를 몇개로 분리해놓은 것 같다.

그리고 `/etc/nginx/conf.d/` 디렉터리 안에 `service-url.inc` 파일을 생성하고 아래처럼 작성한다.
이렇게 설정하면 nginx는 사용자의 요청을 스프링에게 전달하게 된다.

```shell
set $service_url http://127.0.0.1:8081;
```

그 다음 nginx를 재시작한다.
```shell
sudo service nginx restart
```

아래 명령어를 통해 nginx에게 프로파일을 요청한다.  

```shell
curl -s localhost/profile
```

![](../assets/img/nonstop-deploy/curl-profile.png)

nginx를 통해 성공적으로 스프링의 프로파일을 확인하는 것을 확인할 수 있다.

# 재시동 및 전환 쉘 스크립트 작성

---

> 위에서 nginx로 요청을 보내면 스프링 서버로 가는 것을 확인하였다. 
> 이제는 쉘 스크립트를 통해 스프링 서버를 재시동하고, 이를 nginx가 가리키도록 하자. 
> 이 과정에서 작성할 쉘 스크립트는 2개이다.

## 재시동 쉘 스크립트

재시동 쉘 스크립트는 nginx가 가리키지 않는 서버의 port를 확인 후, 재시동하는 스크립트이다.

`~/app/` 디렉터리 안에 `deploy.sh` 파일을 만들고 아래와 같이 작성하자.

```shell
#!/bin/bash
echo "> 현재 구동중인 profile 확인"
CURRENT_PROFILE=$(curl -s http://localhost/profile)
echo "> $CURRENT_PROFILE"

if [ $CURRENT_PROFILE == set1 ]
then
  IDLE_PROFILE=set2
  IDLE_PORT=8082
elif [ $CURRENT_PROFILE == set2 ]
then
  IDLE_PROFILE=set1
  IDLE_PORT=8081
else
  echo "> 일치하는 Profile이 없습니다. Profile: $CURRENT_PROFILE"
  echo "> set1을 할당합니다. IDLE_PROFILE: set1"
  IDLE_PROFILE=set1
  IDLE_PORT=8081
fi

echo "> $IDLE_PROFILE 배포"
sudo fuser -k -n tcp $IDLE_PORT
sudo nohup java -jar /home/ubuntu/app/bookreservationserver-0.0.1-SNAPSHOT.jar --spring.config.location=file:/home/ubuntu/app/config/prod-application.yaml --spring.profiles.active=$IDLE_PROFILE &

echo "> $IDLE_PROFILE 10초 후 Health check 시작"
echo "> curl -s http://localhost:$IDLE_PORT/health "
sleep 10

for retry_count in {1..10}
do
  response=$(curl -s http://localhost:$IDLE_PORT/actuator/health)
  up_count=$(echo $response | grep 'UP' | wc -l)

  if [ $up_count -ge 1 ]
  then
    echo "> Health check 성공"
    break
  else
    echo "> Health check의 응답을 알 수 없거나 혹은 status가 UP이 아닙니다."
    echo "> Health check: ${response}"
  fi

  if [ $retry_count -eq 10 ]
  then
    echo "> Health check 실패. "
    echo "> Nginx에 연결하지 않고 배포를 종료합니다."
    exit 1
  fi

  echo "> Health check 연결 실패. 재시도..."
  sleep 10
done

echo "> 스위칭을 시도합니다..."
sleep 10

/home/ubuntu/app/switch.sh
```

위 스크립트의 흐름은 다음과 같다.

1. 현재 nginx가 가리키는 스프링의 profile을 확인하고, 가리키지 않는 포트를 `IDLE_PORT`로 지정한다.

2. `sudo fuser -k -n tcp $IDLE_PORT`는 `IDLE_PORT`로 구동중인 스프링 서버가 있으면 중단한다.

3. 새롭게 스프링 서버를 가동한다. `nohup` 커멘드를 앞에 붙이면 스프링 로그가 보이지 않으며, `nohup.out`에 이를 기록한다.

4. 일정시간 후에 10번동안 health check를 한다. 10번 시도 후에도 실패하면 code 1로 나간다. 만약 실패하지 않으면 반복문을 빠져나간다.

5. health check 성공하면 전환 스크립트를 실행한다.

health check 기능을 수행하기 위해서 다음 라이브러리를 `build.gradle`에 추가한다.
```
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

actuator는 스프링의 상태를 확인할 수 있는 라이브러리로, 설치 후 `/actuator/health`로 get 요청을 하면 상태를 확인할 수 있다.

## 전환 스크립트

전환(스위칭) 스크립트는 nginx가 가리키는 스프링 서버를 바꾸는 스크립트이다.

`~/app` 디렉터리 안에 `switch.sh` 파일 생성 후 다음과 같이 작성한다.

```shell
#!/bin/bash
echo "> 현재 구동중인 Port 확인"
CURRENT_PROFILE=$(curl -s http://localhost/profile)

if [ $CURRENT_PROFILE == set1 ]
then
  IDLE_PORT=8082
elif [ $CURRENT_PROFILE == set2 ]
then
  IDLE_PORT=8081
else
  echo "> 일치하는 Profile이 없습니다. Profile:$CURRENT_PROFILE"
  echo "> 8081을 할당합니다."
  IDLE_PORT=8081
fi

PROXY_PORT=$(curl -s http://localhost/profile)
echo "> 현재 구동중인 Port: $PROXY_PORT"

echo "> 전환할 Port : $IDLE_PORT"
echo "> Port 전환"
echo "set \$service_url http://127.0.0.1:${IDLE_PORT};" | sudo tee /etc/nginx/conf.d/service-url.inc

echo "> Nginx Reload"
sudo service nginx reload
```

위 스크립트의 흐름은 다음과 같다.

1. nginx가 구동중인 프로필을 확인 후, 새롭게 가리킬 포트를 `IDLE_PORT`에 설정한다.

2. `/etc/nginx/conf.d/service-url.inc` 파일에 포트 부분을 `IDLE_PORT`로 바꾼다.

3. nginx가 새로 바꾼 설정을 읽도록 리로드한다.

## 쉘 실행해보기 

2개의 쉘 스크립트에 실행 권한을 주기 위해 다음 명령어 2줄을 실행한다.

```shell
chmod +x deploy.sh
chmod +x switch.sh
```

그 다음 `deploy.sh` 파일을 실행한다.

![](../assets/img/nonstop-deploy/run-shell.png)

성공적으로 set2 프로필로 스프링 서버가 구동되고, nginx가 이를 가리킴을 확인할 수 있다.

> 2대의 스프링 서버 중 nginx가 가리키지 않는 하나를 재시동하고, 이를 nginx가 가리키는 것을 하였다.
> 이제는 젠킨스를 통해 github의 변화가 있을 때마다 배포, 재시동, 전환까지 되도록 해보자.  

# 젠킨스 설치 및 확인

---

> 앞에서 사용한 ec2 인스턴스 대신 젠킨스용으로 새로운 ec2 서버를 사용한다.

새로운 ec2 인스턴스에 아래 4개의 명령어를 순서대로 수행하여 젠킨스를 설치한다. 

```shell
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
```

아래 명령어로 젠킨스를 실행한다.

```shell
sudo systemctl start jenkins
```

젠킨스는 8080포트로 동작하므로, ec2 인스턴스의 보안그룹에서 8080포트를 열어주자.
그리고 8080포트로 접속한다.

![](../assets/img/nonstop-deploy/jenkins-initial.png)

위 사진처럼 결과가 나온다. `/var/lib/jenkins/secrets/initialAdminPassword`을 열어 초기 비밀번호를 확인하고 위 칸에 적어주고 `Continue` 버튼을 누른다.

그러면 아래와 같은 화면이 나오는데 왼쪽을 클릭하자.

![](../assets/img/nonstop-deploy/jenkins-init2.png)

설치되는데 시간이 좀 걸리니 안심하고 기다리자. 설치가 완료되면 자동으로 다음 화면으로 넘어간다.

![](../assets/img/nonstop-deploy/jenkins-init-install.png)

아래는 설치 후 admin user를 설정하는 화면이다. 원하는 계정과 암호로 설정한 후 넘어가자.

![](../assets/img/nonstop-deploy/jenkins-admin.png)

아래는 젠킨스가 기본적으로 구동되는 url을 설정하는 화면이다. ec2의 url이 자동적으로 적혀져 있을 것이다. 그냥 넘어가면 된다.

![](../assets/img/nonstop-deploy/jenkins-url.png)

드디어 설정이 완료되었다. `Start using jenkins` 버튼을 누르고 다음으로 넘어가자

![](../assets/img/nonstop-deploy/jenkins-finit.png)

아래는 초기 설정을 마친 젠킨스의 첫화면이다. 

![](../assets/img/nonstop-deploy/jenkins-first.png)

# 젠킨스로 CI (continues integration) 구축하기

---

> github의 master 브랜치에 변화가 생기면 자동으로 pull 하는 CI를 구축해보자.

## 깃허브 서버와의 연동

먼저 깃허브 서버와 연동하는 걸 보자.

아래 그림처러 깃허브에 들어가서 자신의 프로필을 선택하고 `Settings`를 선택하자. 

![](../assets/img/nonstop-deploy/github-setting.png)

`Developer settings` 을 클릭한다.

![](../assets/img/nonstop-deploy/github-setting2.png)

`Personal access tokens`에 들어간다.

![](../assets/img/nonstop-deploy/github-setting3.png)

`Generate new token` 버튼을 누룬다.

![](../assets/img/nonstop-deploy/github-setting4.png)

아래 그림처럼 설정하고 토큰을 생성한다.

![](../assets/img/nonstop-deploy/github-setting5.png)

그러면 아래와 같이 토큰 생성이 된다. 이 토큰을 복사해두자.

![](../assets/img/nonstop-deploy/github-setting6.png)

젠킨스로 접속하여 `Jenkins 관리 > 시스템 설정`으로 들어간다. 

![](../assets/img/nonstop-deploy/jenkins-setting.png)

아래처럼 `Github` 섹션에서 `Add Github Server > Github Server`를 클릭하자.

![](../assets/img/nonstop-deploy/jenkins-github1.png)

그러면 아래와 같은 화면이 나온다. 아래와 같이 내용을 채우고 `Credentials` 부분의 `Add > Jenkins`를 클릭하자.

![](../assets/img/nonstop-deploy/jenkins-github2.png)

`Kind`를 Secret text로 설정하고, `Secret`에 위에서 복사한 토큰을 입력하자. 그리고 `ID`에 자신의 ID를 입력한 후 `Add` 버튼을 눌러 완성하자.

![](../assets/img/nonstop-deploy/jenkins-github3.png)

방금 만든 `Credentials`로 설정하고 `Test connection`을 클릭해 제대로 작동되는지 확인하자.

![](../assets/img/nonstop-deploy/jenkins-github4.png)

정상적으로 작동한다. 이제 깃허브 리포지터리와 연동을 해보자.

## 깃허브 리포지터리와 연동

젠킨스 메인 화면에 가서 `새로운 Item`을 클릭하자.

![](../assets/img/nonstop-deploy/jenkins-newitem.png)

원하는 이름으로 설정한 후 `Freestyle project`를 누르고 `OK`를 누르자.

![](../assets/img/nonstop-deploy/jenkins-webhook1.png)

아래와 같이 아이템이 만들어지게 된다. `구성`을 클릭하자.

![](../assets/img/nonstop-deploy/item-setting.png)

`소스 코드 관리 섹션`에 가서 아래 그림처럼 Github 리포지터리 URL을 기입후 Credentials 생성을 하자.

![](../assets/img/nonstop-deploy/jenkins-webhook2.png)

`Kind`는 Username with password로 설정하고, `Username`에 github id, `Password`에 github password를 입력하자

![](../assets/img/nonstop-deploy/jenkins-webhook3.png)

위에서 만든 Credentials를 선택하자.

![](../assets/img/nonstop-deploy/jenkins-webhook4.png)

그리고 빌드 유발 섹션에서 다음과 같이 체크하고 저장하자.

![](../assets/img/nonstop-deploy/jenkins-webhook5.png)

깃허브 리포지터리의 Settings 눌러 들어가서 webhook 탭을 클릭하자.

![](../assets/img/nonstop-deploy/webhook1.png)

Add webhook 버튼을 누르자.

![](../assets/img/nonstop-deploy/webhook2.png)

Payload URL에 젠킨스가 구동되는 ec2 ip 또는 DNS를 적고 뒤에 `github-webhook/` 를 붙여주자.
그리고 다음과 같이 설정한 후 `Update webhook` 버튼을 누르자.

**꼭 github-webhook 뒤에 `/`를 붙여줘야한다.**

![](../assets/img/nonstop-deploy/webhook3.png)

그러면 다음과 같은 화면이 나온다. 시간이 조금 지난뒤 웹페이지를 리로드하면 초록색 체크를 확인할 수 있다.

![](../assets/img/nonstop-deploy/webhook4.png)

이제 실제 코드를 push하면 자동으로 pull 되는지 확인해보자.

pull request로 코드를 작성하여 merge 해보자.

![](../assets/img/nonstop-deploy/push.png)

젠킨스의 아이템의 `Build History` 에 새로운 히스토리가 생긴다. 그 히스토리를 클릭하고, `Console Output`에 들어가면 아래와 같은 결과를 볼 수 있다.

![](../assets/img/nonstop-deploy/push-2.png)

`/var/lib/jenkins/workspace/nonstop-ci-cd`에 github 내용이 pull 된 것을 확인할 수 있다.

![](../assets/img/nonstop-deploy/push-3.png)

> 기본적으로 master 브랜치의 코드를 가져오며 변경하려면 아이템의 `구성` 에 들어가서 `소스 코드 관리` 에서 바꿀 수 있다.

# CD (continues deploy) 및 쉘 스크립트 실행 구축

---

> CI를 통해 가져온 서버 코드를 gradle로 실행파일을 만든 후, nginx와 스프링 서버가 있는 ec2에 배포 후, 앞서 만들었던 deploy.sh을 실행하자.

## gradle로 실행파일 만들기

젠킨스 메인페이지에서 `젠킨스 관리 > global tool configuration`으로 들어간다.

`Gradle` 섹션을 찾아 `Add Gradle` 버튼을 누른다. 

![](../assets/img/nonstop-deploy/gradle1.png)

그러면 아래와 같은 화면이 나오는데 원하는 gradle version을 선택하고 저장한다.

![](../assets/img/nonstop-deploy/gradle2.png)

아이템의 `구성` 에 들어가 `Build` 섹션에 찾는다.

그리고 아래 그림 처럼 `Add build step > Invoke Gradle Script`를 선택한다.

![](../assets/img/nonstop-deploy/gradle-3.png)

아래 사진과 같이 설정한다.

![](../assets/img/nonstop-deploy/gradle4.png)

그리고 오른쪽 하단에 있는 `고급..` 버튼을 누르고 아래와 같이 설정하고 저장한다.

![](../assets/img/nonstop-deploy/gradle5.png)

이렇게 설정하면 CI를 하고 자동으로 `/var/lib/jenkins/workspace/nonstop-ci-cd/build/libs` 에 실행 파일을 생성한다.

## ssh로 배포 및 쉘 스크립트 실행

`젠킨스 관리 > 플러그인 관리` 로 들어가 `Publish Over SSH` 플러그인 을 설치한다.

![](../assets/img/nonstop-deploy/ssh1.png)

`젠킨스 관리 > 시스템 설정` 으로 들어가 `Publish over SSH`를 아래 사진처럼 구성한다.

![](../assets/img/nonstop-deploy/ssh-2.png)

`Key`는 서버가 구동중인 ec2 인스턴스의 .pem 파일 내용을 복붙하면 된다.

`Hostname`은 서버가 구동중인 ec2 인스턴스의 ip를 적으면 된다.

`Username`에는 접속하려는 유저와, `Remote Directory`에는 접속했을 때의 기본 디렉터리를 설정한다.

그리고 아이템의 `구성`에 들어가서 `빌드 후 조치 추가 > Send build artifacts over SSH`를 누른다.

![](../assets/img/nonstop-deploy/ssh3.png)

아래 사진처럼 구성한다.

![](../assets/img/nonstop-deploy/ssh4.png)

`Source files`는 배포하려는 파일들이다. 

`Remove prefix`는 배포하려는 파일이 속해있는 디렉터리 정보를 제거하기 위해 필요하다.

`Remote directory`는 배포하려는 디렉터리의 위치다.

`Exec command`는 ssh 배포후 실행하는 명령어이다. deploy.sh를 실행한다. 

`> /dev/null 2>&1`는 표준 출력과 표준 입력을 버리는 것이다. 이를 붙이지 않으면, 젠킨스가 쉘 스크립트 수행 후 ㅎ빠져나오지 못하기 때문에 꼭 붙여주자.

> 한가지 주의할 점은 Exec command에는 절대 경로를 설정해야한다는 점이다. 
> 만약 쉘 스크립트를 실행한다면 쉘 스크립트 안의 명령어들도 모두 절대 경로여야 한다.

이제 코드를 푸시하게 되면 아래 사진처럼 결과를 볼 수 있다. 정상적으로 배포, 실행까지 이루어진 것을 볼 수 있다.

![](../assets/img/nonstop-deploy/ssh5.png)


# Slack Notification

---

원래는 이것까지 포스트에 다루려 했으나 글이 너무 길어지는 것 같아서 다루지는 않겠다. 크게 어렵지 않으니 다른 블로그를 찾아보면 좋을 것 같다.

실제로 구축하게 되면 아래와 같이 결과를 알림받게 된다.

아래 알림은 빌드 실패일 때다.

![](../assets/img/nonstop-deploy/noti1.png)

아래 알림은 배포 또는 실행 스크립트 수행  실패했을 때이다.

![](../assets/img/nonstop-deploy/noti2.png)

아래 알림은 모두 정상적으로 성공했을 때이다.

![](../assets/img/nonstop-deploy/noti3.png)

# 참조

---

* https://jojoldu.tistory.com/267
  
* https://goddaehee.tistory.com/258

