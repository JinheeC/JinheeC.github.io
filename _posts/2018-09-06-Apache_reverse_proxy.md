---
title: "Apache Reverse Proxy 접근 암호화와 Port Forwarding"
categories: 
- Apache
excerpt: |
  
feature_text: |
  Apache Reverse Proxy 로 Basic Auth, Port Forwarding 하기
feature_image: "https://picsum.photos/2560/600?random"
---

* table of contents
{:toc .toc}

## Reverse Proxy 란
![Alt text](https://monosnap.com/image/5MYrF51Wtm8JNXLSziPQ3kKPsEs4BZ) 
위의 [youtube 캡쳐](https://www.youtube.com/watch?v=Dgf9uBDX0-g)가 가장 쉽게 Reverse Proxy 를 설명한것 같다.
Forward Proxy 와의 가장 큰 차이점은 유저(접속하려는 자)는 프록시 서버로 요청한다는 것을 모른다는 점이다. Forward 의 경우 프록시임을 알고 프록시에게 요청해서 프록시가 다시 해당 서버에 요청을 하고 그 결과를 유저에게 전달해주는 방식이지만, Reverse 의 경우 프록시임을 모르고 그림에서의 example.com 으로 요청을 하면 실제 example.com 도메인을 가지고 있는 프록시 서버는 myapp.exmaple.com으로 다시 요청을 해준다. (필요하다면 포트도 원하는 포트로 요청이 된다.) 그 후 리턴받은 결과를 프록시가 유저에게 전달해주는 것을 Reverse Proxy 라고 한다.

Reverse Proxy 를 사용하는 가장 큰 이유는 보안이라고 한다. 망에 접근할 수 있는 서버에 제한을 두기위해 사용하기도 하고 포트 포워딩을 위해 사용하기도 한다.

## 접근 암호화하기
리버스 프록시를 해야했던 이유가 접근을 인증된 사용자에게만 허용하게 하기 위해서였다. 
그러기 위해서 Basic auth 로 간단히 ID/PW 만 입력하면 접근할 수 있도록 설정을 했고 이 글의 모든 설명은 Apache 기준 설명이다.

### 암호파일 만들기
암호파일이라는 것은 원하는 ID/PW 가 있으면 PW 를 평문(Plain text)으로 보관할 수 없기 때문에 PW 를 암호화 시킨 파일이라고 보면 된다. 이런 암호파일을 만들어야 하는데 만드는 암호화 도구는 이미 apache 가 설치된 위치에 가면 찾을 수 있다. 아파치 설치 위치로 가서 `/apache/bin/` 하위에 htpasswd 이라는 파일이 존재하는 것을 확인할수 있다. 이 파일로 만들 수 있는데 아래와 같이 설정한다.
```
./htpasswd -c password 보관 위치 userId
```
password 보관 위치는 원하는데로 설정할 수 있고 예를 들어 `/usr/local/apache/passwd/passwords` 라고 설정하면 passwd 폴더 밑에 passwords 라고 파일이 생길 것이고 userId 는 위에서 ID 에 해당한다. 	`-c` 옵션은 생성하겠다는 의미이다. 만약 passwords 파일에 user 를 추가하려는 의도라면 `-c` 를 지우고 위의 명령어를 입력해 주면 된다.
위 명령어를 입력하면 pasword 를 입력하라고 하는데 위의 PW 로 설정하고 싶은 것을 입력하면 된다.
입력이 정상적으로 완료되면 아래와 같이 출력되고 파일이 생길 것이다. 
``` plain
# htpasswd -c /usr/local/apache/passwd/passwords rbowen
New password: mypassword
Re-type new password: mypassword
Adding password for user rbowen
```

#### 기존 암호파일에 유저 추가하기
```
./htpasswd password_보관_위치 newUser
```
새로 만드는게 아니고 이전에 만들어뒀던 passwords 파일에 추가하고자 한다면 -c 를 없애고 만들면 된다.

#### 여러명의 user 를 위한 암호파일 만들기
여러개의 User Id 를 허용하기 위해서는 아래와 같이 passwords 파일을 만들기 전 하나의 파일이 더 필요하다. vi 편집기 등 편한 편집기를 사용해서 파일을 만들어주면된다.
```
GroupName: user1 user2 user3 user4
```
암호는 위의 기존 파일에 유저 추가하기 처럼 암호를 추가해 줘야 한다.
이렇게 group파일 까지 추가했다면 이제 아래에 여러명의 user 를 위한 설정을 이어나가면 된다.

### Reverse Proxy 설정하기
이제 암호화 파일도 있으니 설정만 해주면 된다. 설정은 `httpd.conf` 파일을 아래와 같이 편집하면 된다. `httpd.conf` 파일은 apache 설치 디렉토리에서 conf 디렉토리 하위에 위치한다. 
```
<Location />
    Order deny,allow
    Allow from all
    AuthType Basic
    AuthName "password required"
    AuthUserFile 암호파일 위치
    Require user userID
</Location>
```
위의 설정은 httpd.conf 파일의 아무곳에나 추가해도 좋지만 영역을 잘못 넣을 수도 있고 나중에 어느부분을 추가한지 쉽게 파악하기 위해 제일 마지막에 추가하는것이 좋다. authName 은 아무거나 상관없고 암호파일 위치는 위에서 미리 만들어 둔 passwords 위치면 된다. 예를 들어 위에서는 `/usr/local/apache/passwd/passwords`에 해당한다.

#### 여러명의 user를 위한 설정
여러명을 위한 경우 위에서 암호파일도 여러명을 위해 만들었을 것이다. 그럴경우 아래와 같이 설정을 추가하면 된다.

```
<Location />
    Order deny,allow
    Allow from all
    AuthType Basic
    AuthName "password required"
    AuthUserFile 암호파일_위치
    AuthGroupFile 그룹파일_위치
	Require group GroupName
</Location>
```

## 포트포워딩 하기
기본 인증 접근과 함께 포트도 바꿔줘야하는 경우도 있을 수 있다. 예를 들면 `example.com` 에서 실제로 어플리케이션이 떠있는 8080포트 `example.com:8080` 으로 포워딩 해줘야 하는 상황이 있을 수 있는데 그럴경우 아래의 설정을 추가해주면 된다.

```
<VirtualHost *:80>
    ProxyPreserveHost On
    ServerName example.com
    ProxyPass / http://example.com:8080/
    ProxyPassReverse / http://example.com:8080/
</VirtualHost>
```

## 최종 설정 파일 형태

위에서 설정했던 포트 포워딩, basic auth 를 모두 추가한 최종 파일 형태는 아래와 같다.
```
<VirtualHost *:80>
    ProxyPreserveHost On
    ServerName example.com
    ProxyPass / http://example.com:8080/
    ProxyPassReverse / http://example.com:8080/
    <Proxy *>  // <Location /> 도 가능
        Order deny,allow
        Allow from all
        AuthType Basic
        AuthName "password required"
        AuthUserFile 위치
        Require user userID
    </Proxy>
</VirtualHost>
```
