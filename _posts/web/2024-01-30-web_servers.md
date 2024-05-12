---
title: 웹 서버와 포트, 소켓, 그리고 스레드
date: 2024-01-24 12:00:00 +/-TTTT
categories:
comments: true
tags:
---

# 웹 서버와 포트, 소켓, 그리고 스레드

<br/>

지금까지 웹서버를 설정할 때 아무 생각 없이 포트 설정을 해 왔다.  
Nestjs 라면 아래와 같이,

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000); //  port
}
bootstrap();
```

SpringBoot라면

```
//application.properties
server.port=8080
```

이런 식으로 말이다.

이번에 네트워크 관련 스터디를 하면서 포트와 소켓, 그리고 스레드에 대해서 몇가지 의문들이 들었는데, 이번 포스팅에서 하나씩 해결해보려고 한다.

---

먼저 한가지 질문으로 시작해보려고 한다.

> 소켓은 하나의 포트와 매칭되는가?
> {: .prompt-tip }

그렇다.
포트와 IP 주소를 조합하여 하나의 소켓을 구성한다.

> 하나의 요청은 하나의 소켓을 갖는다?
> {: .prompt-tip }

당연하게도 그럴것 같다. 서로 다른 요청들을 구분하고 TCP 연결을 맻으려면 서로 다른 포트(소켓)을 가져야 한다.

## 웹서버를 구축할 때 설정한 포트는 하나인데?

여러개의 요청을 동시에 처리하려면 당연히 여러개의 포트를 사용할것이다.  
**우리가 웹서버 애플리케이션에 배정한 포트는 하나인데, 어떻게 여러개의 동시 요청들을 처리하는걸까**?

### Listening Port

사실 우리가 서버를 구축할 때 주는 포트번호는 Listening Port라는 특수한 포트이다.  
이 포트는 클라이언트들로부터 들어오는 연결 요청을 기다린다.  
즉, 우리가 설정한 포트는 어디까지나 중재자 역할을 할 뿐, 직접 요청을 처리하는것이 아니다.

<br/>

### Ports and Sockets

웹서버는 listening port를 이용해서 요청을 기다린다.  
요청이 들어올 경우, 받은 요청에 **ephemeral port(~1024의 예약 포트를 제외한 나머지)** 라고 하는 임시 커넥션 포트를 사용하여 동적으로 소켓을 만들어준다.

생성된 소켓은 톰캣과 같은 멀티스레드 웹서버 환경에서는 보유하고 있는 스레드 풀에서 스레드를 배정받아 요청을 처리하고,  
Nodejs 와 같은 싱글스레드 환경이라면 이벤트 루프가 request handler를 호출하여 해당 요청을 처리해준다.

<br/>

### Concurrency

포트개수가 많긴 하지만 결국 6만개밖에 없는게 아닌가? 만약 그 이상의 연결이 들어온다면 웹서버는 요청을 받을 수 없는걸까?

그렇지 않다.
웹서버는 5개의 원소를 가진 tuple을 통해 connection을 구분한다.  
Source IP, Source Port, Destination IP, Destination Port, Protocol
이중 하나라도 다르다면 별도의 connection으로 취급하는 것이다.

즉, 두 서버간 총 64K의 Connection이 가능하다는 의미이다.

따라서 포트 하나만 있더라도 서버의 리소스만 충분하다면 Source IP 주소가 모두 다르기 때문에 문제가 없다.

<br/>

### 쉬어가기 - Processes and Threads

옛날 웹서버들은 사실 하나의 연결당 하나의 프로세스를 배정하는 process-per-connection 정책을 사용하였다.  
다만 IPC와 프로세스 생성 / 해제의 오버헤드로 인해 프로세스가 아닌 스레드를 배정하는  
thread-per-connection 정책을 사용하거나, nodejs와 같은 비동기 방식을 채택하였다.

<br/>

### HTTP/1.1 - Connection Overhead

연결 요청마다 매번 새로운 소켓을 생성하고 TCP 요청을 수립하는것은 당연히 비효율적일 것이다.  
이를 해결하기 위해 HTTP/1.1 스펙에서는 Keep-alive라는 옵션이 추가되어, 연결을 재사용하는것이 가능하게 되었다.

<br/>

### Thread per connection vs Thread per request

다시 스레드로 돌아가보자.  
연결 하나당 스레드 하나가 배정된다면, 연결 사이사이에 스레드가 놀지 않을까?

사실 현재 웹서버들은 하나의 "연결"에 대해서 스레드를 배정하지 않고,
하나의 "요청" 단위로 스레드를 배정한다(Thread per request)

요청 단위로 스레드를 배정한다면 생성비용과 해제비용은 어떻게 감당할까?  
앞에서 잠깐 이야기했듯, 멀티스레드 웹서버들은 스레드풀을 사용하여 관리한다.  
이게 가능한 이유는 http의 stateless한 특성 때문이라고 이해할 수 있다.

{: .question}
생각해보기 - KeepAlive & Thread per request?

HTTP/1.1의 KeepAlive는 추가적인 요청에 대해서 연결을 재활용하기 위해 바로 스레드가 반납되지 않고 잠깐 살아있게 된다.

이게 Thread per request 모델과 공생이 가능할까?

스레드를 재활용하는것은 엄격하게 보면 Thread per connection이 아닐까?
