출처

[https://velog.io/@koseungbin/WebSocket](https://velog.io/@koseungbin/WebSocket)<br>
[https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#websocket](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#websocket)<br>
[https://supawer0728.github.io/2018/03/30/spring-websocket/](https://supawer0728.github.io/2018/03/30/spring-websocket/)





## WebSocket
서버와 클라이언트 간에 표준화된 전이중 통신방법을 제공한다.
기존의 다른 TCP기반 프로토콜과 다르게, HTTP요청 기반으로 Handshake과정을 거쳐 커넥션을 생성한다.
덕분에, 초기 WebSocket Handshake요청은 추가적인 방화벽 설정없이 80,443포트를 사용하여 양방향 통신이 가능하다.
handshake 성공 이후에 클라이언트와 서버 간 메시지를 계속해서 주고 받기 위해 TCP커넥션은 유지한다.

## HTTP vs WebSocket
HTTP에서 어플리케이션은 많은 URI로 설계되고 어플리케이션과 상호작용 하기 위해 클라이언트는 이러한 URI로 접근하는 request-response 스타일이다.
반대로 WeSocket은 초기 연결을 위해서 하나의 URL만 사용하고 이후 모든 어플리케이션 메시지는 같은 TCP 커넥션을 통해 이뤄진다.
WebSocket은 HTTP와 달리 low-level 통신 프로토콜로
메시지 내용에 의미를 두지 않기 때문에, 클라이언트 서버 간 임의로 의미를 부여하지 않으면 메시지를 처리 할 방법이 없다.
이러한 문제를 해결해주는것이 STOMP 프토토콜이다.
STOMP 프로토콜을 이용하면 단순한 binary,text가 아닌 규격을 갖춘 (format)message를 보낼수 있다.

## When to Use WebSockets
메시지의 양이 상대적으로 적다면, HTTP streaming 또는 polling이 효과적인 해결책이 될수 있다.
낮은 지연시간, 높은 빈도수,메시지의 양이 많은 경우에는 WebSocket을 사용하는것이 좋다.

### Traditional polling
```
고전적인 Polling방식은 새로운 정보가 있는지 확인하기 위해 주기적으로 HTTP 요청을 보낸다.
이러한 방식은 지속적으로 요청을 보내기 때문에, 매번 커넥션을 생성하기 위한 handshake 비용이 많아지며 서버에 부담을 주게 된다.
```

### Long Polling
```
클라이언트는 서버에 요청을 보내고, 서버는 변경 사항이 있는 경우에만 응답하여 커넥션을 종료한다.
커넥션은 무한히 대기할수 없으므로, 브라우저는 약 5분 정도 대기하며 중간 프록시에 따라 더 짧게 커넥션이 종료될수도 있다.
변경 사항이 불규칙적인 간격으로 일어나는 경우 효율적이나, 변경 사항의 빈도가 잦다면 기존 Traditional Polling과 차이가없으므로 서버의 부담이 증가하게 된다.
```


### HTTP Streaming
```
Long Polling과 동일하게 HTTP요청을 보내지만, 변경 사항을 클라이언트에 응답한 이후에도 커넥션을 종료하지 않고 유지한다.
서버가 현재 업데이트된 데이터를 모두 전달해야만, 클라이언트에서 다음 업데이트된 데이터의 시작 위치를 알수 있기 때문에 서버의 부담을 줄일수 있지만 동시처리가 어려워진다.
```


## WebSocket API
Spring프레임워크는 WebSocket API를 제공한다.
WebSocketHandler를 구현하거나
TextWebSocketHandler 또는 BinaryWebSocketHandler을 상속받아서 WebSocket 서버를 만들수 있다.
WebSocketHandler를 사용할때 WebSocket 세션(JSR-356)은
동시전송을 지원하지 않기 때문에 STOMP 메시징 프로토콜을 이용해서 메시지 전송을 동기화 하거나, WebSocketSession을 ConcurrentWebSocketSessionDecorator으로 Wrapping해야 한다. ConcurrentWebSocketSessionDecorator은 오직 하나의 스레드만 메시지를 전송하도록 보장해주기 때문이다.

```java
import org.springframework.web.socket.WebSocketHandler;
import org.springframework.web.socket.WebSocketSession;
import org.springframework.web.socket.TextMessage;

public class MyHandler extends TextWebSocketHandler {

    @Override
    public void handleTextMessage(WebSocketSession session, TextMessage message) {
        // ...
    }

}
```
```Java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(), "/myHandler");
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }

}
```

## WebSocket Handshake
초기  HTTP WebSocket handshake request를 커스터마이징 하는 가장 쉬운 방법은 handshake 전/후에 호출되는  메서드인, HandshakeInterceptor을 이용하는것이다. 인터셉터를 이용해서 handshake를 막거나 WebSocketSession속성을 사용할수 있다.
아래는 기본적으로 제공하는 HTTP Session을 WebSocket Session에 전달하는 HttpSessionHandshakeInterceptor 예제이다.

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new MyHandler(), "/myHandler")
            .addInterceptors(new HttpSessionHandshakeInterceptor());
    }

}
```


## Allowed Origins
Spring프레임워크는 WebSocket, SockJS에 대해 디폴트로 Same-Origin요청을 지원한다.

### Same-Origin 정책
```
두 URL의 프로토콜,포트(명시한 경우),호스트가 모두 같아야 동일한 출처라고 말한다.

```
[출처](https://developer.mozilla.org/ko/docs/Web/Security/Same-origin_policy)

스프링에서는 각 핸들러마다 지원할 도메인을 지원할수 있도록 설정하는 방법을 제공한다.

```java
import org.springframework.web.socket.config.annotation.EnableWebSocket;
import org.springframework.web.socket.config.annotation.WebSocketConfigurer;
import org.springframework.web.socket.config.annotation.WebSocketHandlerRegistry;

@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(), "/myHandler").setAllowedOrigins("https://mydomain.com");
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }

}
```

## SockJS Fallback
클라이언트와 서버 중간에 있는 프록시가 WebSocket 통신을 못하게 할수 있다. Upgrade 헤더를 전달하지 못하거나 유휴 상태인것으로 보이는 연결을 닫을수 있기 때문이다. 이에 대한 해결책이 WebSocket emulation이다.

### Websocket emulation
처음엔 WebSocket사용을 시도하다가 연결이 실패한 경우 HTTP 기반의 다른 기술(HTTP streaming, HTTP Long Polling)로 전환해 다시 연결을 시도하는것이다. 이러한 기능을 제공하는것이 SockJS 프로토콜이다.
즉, SockJS의 목적은 어플리케이션이 WebSocket API를 사용하도록 하지만 사용할수 없는 경우 런타임에 어플리케이션의 코드 변경 없이 WebSocket이 아닌 대안으로 fall back하는것이다.

### 동작 순서
1. SockJS 클라이언트는 GET /info 요청으로 서버로부터 기본 정보를 얻는다.
```
https://host:port/myApp/myEndpoint/{server-id}/{session-id}{transport}
{server-id}: 클러스터에서 요청을 라우팅하는데 유용하지만 다른 경우에 사용되지 않는다.
{session-id}: SockJS 세션에 속하는 HTTP요청을 연관시킨다.
transport는 전송타입을 가리킨다. (ex. WebSocket,xhr-streaming,..)
```

2. 정보를 토대로 통신에 사용할 프로토콜을 결정한다. 만약 가능하다면 WebSocket을 사용하고, 그렇지 않다면 HTTP streaming 또는 HTTP long polling을 사용한다.

3. WebSocket 통신은 WebSocket handshake를 위해 HTTP 요청 한번이 필요하다.
Ajax/XHR Streamig의 경우는 장기 실행 요청을 유지하여 서버에서 클라이언트로 전달하기 위한 메시지를 응답으로 전달 받는다. 이후, 클라이언트에서 서버로 새로운 요청을 보내야 할 경우에는 기존의 커넥션을 종료하고 HTTP POST요청을 보내어 커넥션을 유지한다.
Long polling은 이와 유사하나, 서버에서 클라이언트로 전송한후  현재 요청을 끝낸다.

SockJS는 message frame 크기를 최소화 하기 위해 노력한다.
예를 들어, 서버는 "open" frame을 뜻하는 문자 'o'를 보내고 메시지는 다음의 형태로 전달받는다.
```
a["message1", "message2"]
```
 문자 'h'는 커넥션 유지 여부를 확인하는 "hearbeat" frame 뜻한다.

 문자 'c'는 세션을 닫는 "close" frame을 뜻한다.

 ## SockJS 사용
 다음의 예처럼 자바 설정을 통해 SockJS를 사용할수 있다. 스프링에서 제공하는 WebSocket API와 SockJS는 Spring MVC에 독립적이지만 관련된 설정들은 DispatcherServlet에 포함되어야 한다.
 SockJSHttpRequestHandler의 도움으로 다른 HTTP serving 환경에 간단하게 통합시킬수 있다.


 ```java
 @Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(myHandler(), "/myHandler").withSockJS();
    }

    @Bean
    public WebSocketHandler myHandler() {
        return new MyHandler();
    }

}
 ```
 클라이언트 측에서는 sockjs-client라이브러리를 사용하는데, 서버와 통신하여 브라우저에 따른 최적의 전송 옵션을 선택한다.

### IE 8, 9 호환
SockJS를 사용하는 가장 큰 이유이다.
SockJS 클라이언트는 XDomainRequest를 사용함으로써 IE8,9에서 Ajax/XHR streaming을 지원하지만, 쿠키 전송은 지원하지 않는다.
따라서 서버측의 Cookie 필요 여부에 따라 HTTP Streaming, HTTP Long Polling에서 사용하는 기술이 달라진다.
* Cookie가 필요하다면,iframe기반 기술 사용한다.
* Cookie가 필요하지 않다면, XDomainRequest사용한다.

SockJS 클라이언트가 첫번째로 요청한 GET /info  요청에대한 응답을 토대로 transport 타입을 결정한다.
Spring의 SockJS는 sessionCookieNeeded속성을 지원한다.
자바 어플리케이션은 대부분 JSESSIONID쿠키에 의존하기 때문에 이 속성은 기본적으로 TRUE이다. 옵션 사용을 원치 않는 경우 옵션을 끄면 된다.
iframe기반 transport를 사용하는 경우 HTTP 응답 헤더 X-Frame-Options에 지정한  DENY, SAMEORIGIN, 또는  ALLOW-FROM <origin> 페이지들에 대해서만 iframe을 렌더링하도록 할수 있다.
* DENY는 모든 iframe에서 사용될수 없다.
* SAMEORIGIN은 동일한 출처, 즉 같은 도메인인 경우에만 허용
* ALLOW-FROM <origin>는 지정한 도메인 URI에 대해서만 허용

만약 iframe-based기반 transport를 사용하고 X-Frame-Options를 응답 헤더에포함 한다면, 헤더의 값은  SAMEORIGIN 이거나 또는 ALLOW-FROM <origin>에 SockJS 클라이언트 도메인을 지정 하여야 한다.
스프링은 SAMEORIGIN을 지원하기 위해 SockJS-Client 접근 경로를 설정할수 있도록 아래와 같이 제공한다.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio").withSockJS()
                .setClientLibraryUrl("http://localhost:8080/myapp/js/sockjs-client.js");
    }

    // ...

}
```

#### CORS(교차 출처 리소스 공유)
[출처](https://developer.mozilla.org/ko/docs/Web/HTTP/CORS)

CORS는 추가 HTTP헤더를 사용하여, 한 출처에서 실행중인 웹 애플리케이션이 다른 출처의 선택한 자원에 접근할수 있는 권한을 부여하도록 브라우저에서 알려주는 체제이다. 웹 애플리케이션은 리소스가 자신의 출처(도메인, 프로토콜, 포트) 와 다를때 교차 출처 HTTP 요청을 실행한다.  최신 브라우저는 XMLHttpRequest 또는 Fetch와 같은 API에서 CORS를 사용하여 교차 출처 HTTP요청의 위험을 완화한다.
보안상의 이유로, 브라우저는 스크립트에서 시작한 교차 출처 HTTP 요청을 제한한다. 예를 들어, XMLHttpRequest와 Fetch API는 동일 출처 정책을 따른다.
즉, 이 API를 사용하는 웹 애플리케이션은 자신의 출처와 동일한 리소스만 불러올수 있으며, 다른 출처의 리소스를 불러 오려면 그 출처에서 올바른 CORS 헤더를 포함한 응답을 반환해야 한다.

#### XDomainRequest
[출처1](https://homoefficio.github.io/2015/07/21/Cross-Origin-Resource-Sharing/)
[출처2](https://developer.mozilla.org/ko/docs/Glossary/XHR_(XMLHttpRequest))

<br>W3C표준이 아니며, 비동기 CORS통신을 위해 Microsoft에서 만든 객체이다.
* XDR은 setRequestHeader가 없다.
* XDR, XHR(AJAX요청을 생성하는 JavaScript API)을 구분하려면 obj.contentType을 사용한다.(XHR에는 이게 없음, XHR도 CORS를 지원한다.)
* XDR은 http와 https프로토콜만 가능

CORS(Cross-Origin Resource Sharing)를 쓰면  AJAX로도 Same Origin Policy의 제약을 넘어 다른 도메인의 자원을 사용할수 있다.
CORS를 사용하려면 클라이언트에서 Access-control-** 류의 HTTP Header를 서버에 보내야 하고, 서버도 Access-Control-** 류의 HTTP Header를 클라이언트에 회신하게 되어 있어야 한다.

### Heartbeats
서버는 프록시가 연결이 끊겼다고 생각하지 않도록 주기적으로 hearbeat 메시지를 보내야 한다.
Spring SockJS설정에서  hearbeatTime속성을 통해 빈도를 설정할수 있다.
기본적으로 마지막 메세지가 전송된 이후 25초후에 hearbeat를 전송한다.
또한 Spring SockJS는 heartbeats tasks를 스케줄링 하기 위해 TaskScheduler를 설정할수 있다.

### Client Disconnects
HTTP streaming 과 HTTP long polling SockJS transport는 일반요청보다 더 긴 커넥션을 요구한다.
Servlet 3 asynchronous는 Servlet Container Thread가 종료되고도 요청을 처리하며 다른 스레드가 지속적으로 응답에 write할 수 있도록 지원한다.
여기서 문제는 Servlet API가 사라진 클라이언트에 대한 알림을 제공하지 않는다는것이다.   하지만, 다행히 Servlet Container 는 응답에 write를 시도할 경우 예외를 발생시키고,
Spring의 SockJS가 기본적으로 25초 마다 heartbeats를 보내기 때문에, 클라이언트의 연결 여부를 일정 시간 안에 파악할수 있다.

### SocketJS, CORS
CORS를 허용 한다면, SockJS 프로토콜은 XHR Streaming과 polling transport에서 corss-domain 지원을 위해 CORS를 사용한다.
따라서 응답에서 CORS헤더가 발견되지 않을 경우 CORS헤더가 자동으로 추가 된다. 만약 어플리케이션이 이미 CORS를 설정한 경우에는 스프링의 SocketJsService는 건너뛴다.
스프링의 SockJsService의 setSuppressCors설정을 통해 이러한 CORS헤더 추가를 막을수 있다.

SockJS에서는  CORS헤더에 아래의 값이 필요하다.
* Access-Control-Allow-Origin: Origin요청 헤더의 값으로 초기화된다.
* Access-Control-Allow-Credentials:
항상 true로 설정된다.
* Access-Control-Request-Headers:
실제 요청이 만들어질때 클라이언트가 보낼수 있는 HTTP headers를 서버에게 알리는 용도로, 브라우저가 preflight request를 보내는 경우에 사용된다.
SockJS에서는 Request와 동일한 헤더로 설정한다.
* Access-Control-Allow-Methods:
서버가 지원하는 Transports타입의 HTTP METHOD를 설정한다.
* Access-Control-Max-Age:
preflight request결과를 얼마나 캐시할지를 나타내고 1년으로 설정된다.

### SockJsClient
Spring은 브라우저를 사용하지 않고 SockJS endpoint에 연결할수 있는 SockJS java 클라이언트를 제공한다.
이것은 네트워크 프록시가 WebSocket 프로토콜을 막을수 있는 공용 네트워크에서 두 서버간에 양방향 통신이 필요한 경우에 유용하다. SockJS java 클라이언트는 또한 테스팅에도 매우 유용하다.(많은 수의 동시 사용자를 시뮬레이션하는 경우)
SockJS java client는 websocket, xhr-streaming,xhr-polling transports를 지원하고 나머지는 브라우저에서만 지원된다.
아래 예시는 SockJS client를 이용해서 **ws://example.com:8080/sockjs** 서버와 연결하는 작업이다.
XhrTransport는 클라이언트 관점에서 서버에 연결 하기 위해 사용되는  URL이외에 다른 차이가 없기 때문에 xhr-streaming, xhr-polling 둘다 지원한다.
XhrTransport 구현체인 RestTemplateXhrTransport는 HTTP 요청을 위해 스프링의 RestTemplate를 사용한다.
WebSocket 설정은 WebSocketTransport 객체를 통해 이루어지는데, JSR-356 runtime에서 제공하는 StandardWebSocketClient 사용하도록 지정한다.


```java
List<Transport> transports = new ArrayList<>(2);
transports.add(new WebSocketTransport(new StandardWebSocketClient()));
transports.add(new RestTemplateXhrTransport());

SockJsClient sockJsClient = new SockJsClient(transports);
sockJsClient.doHandshake(new MyWebSocketHandler(), "ws://example.com:8080/sockjs");
```

많은 수의 동시 사용자를 시뮬레이션하고자 하는 경우, XHR transports가  충분한 수의 커넥션과 스레드를 허용하도록 하기 위해 구성할수 있다.

다음은 jetty를 이용한 예제이다.
```java
HttpClient jettyHttpClient = new HttpClient();
jettyHttpClient.setMaxConnectionsPerDestination(1000);
jettyHttpClient.setExecutor(new QueuedThreadPool(1000));
```

다음의 예시는 서버 사이드 SockJS 관련 속성들을 보여주는 예시이다.
* streamBytesLimit:
단일 HTTP 스트리밍 요청을 통해 전송될수 있는 최대 바이트 수를 의미한다.
* HttpMessageCacheSize:
클라이언트의 다음 HTTP 폴링 요청을 기다리는 동안에, 서버가 클라이언트로 전송하기 위해 메시지들을 세션에 캐시할수 있는 개수이다.
* DisconnectDelay:
클라이언트가 연결이 끊긴것으로 간주되는 시간을 의미한다.




```java
@Configuration
public class WebSocketConfig extends WebSocketMessageBrokerConfigurationSupport {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/sockjs").withSockJS()
            .setStreamBytesLimit(512 * 1024)
            //streamBytesLimit을 설정.
            .setHttpMessageCacheSize(1000)
            //httpMessageCacheSize설정.
            .setDisconnectDelay(30 * 1000);
            //disconnectDelay설정.
    }

    // ...
}
```
