### STOMP
WebSocket은 두가지 타입의 메시지(텍스트, 바이너리)를 정의하고 있지만 메시지의 내용까지는 정의하고 있지 않다.
이를 위해 STOMP 프로토콜은 WebSocket위에서 클라이언트와 서버가 각각 보낼수 있는 메시지 종류, 형식, 각 메시지의 내용을 정의하는 메커니즘을 정의한다.
STOMP는 WebSocket, TCP같은 어떠한 양방향 스트리밍 네트워크 프로토콜에서도 사용될수 있다.
STOMP는 텍스트 지향 프로토콜이지만, Message Payload에는 Text또는 Binary 데이터를 포함할수 있다.
STOMP는 HTTP위에서 동작하는 Frame 기반의 프로토콜이며, Frame은 아래와 같은 형식을 가지고 있다.
```
COMMAND
header1:value1
header2:value2

Body^@
```
클라이언트는 SEND 또는 SUBSCRIBE 명령을 사용하여 메시지의 내용 및 수신 대상을 설명하는 헤더와 함께 메시지를 send하거나 subsecribe할수 있다.
이러한 publish-subscribe 메커니즘은 broker를 통해 다른 연결된 클라이언트들에게 메시지를 보내거나 서버가 특정 작업을 수행하도록 메시지를 보내는것을 할수 있게해준다.
만약 스프링에서 지원하는 STOMP를 사용하게 된다면,
스프링 WebSocket 애플리케이션은 STOMP Broker로 동작한다.
메시지가 @Controller 메시지 핸들링 메소드에게 라우팅 되거나 in-memory broker에게 전달되도록 해서 subscribe중인 다른 클라이언트들에게 메시지를 전달할수 있다. 뿐만 아니라
스프링은 RabbitMQ, ActiveMQ 같은 외부 Messaging System를 STOMP Broker로 사용할수 있도록 지원하고 있다.
스프링은 외부 STOMP Broker와 TCP커넥션을 유지하는데, 외부 STOMP Broker는 서버-클라이언트 사이의 매개체로 동작한다.
스프링은 메시지를 외부 브로커에게 전달하고, 브로커는 WebSocket으로 연결된 클라이언트에게 메시지를 전달하는 구조이다.
이러한 구조 덕분에, 스프링 웹 애플리케이션은 'HTTP 기반의 보안 설정'과 '공통된 검증'등을 적용할수 있게 된다.

만약 클라이언트가 특정 경로에 대해서 아래와 같이 Subscribe한다면, 서버는 원할때마다 클라이언트에게 메시지를 전송할수 있다.
```
SUBSCRIBE
id:sub-1
destination:/topic/price.stock.*

^@
```
클라이언트 또한 메시지를 보낼수 있는데, 서버는 @MessgaeMapping메소드를 통해 해당 메시지를 처리할수 있다.
```
SEND
destination:/queue/trade
content-type:application/json
content-length:44

{"action":"BUY","ticker":"MMM","shares",44}^@
```

STOMP 스펙에서는 Destination 정보를 불분명하게 정의하였는데, 이는  STOMP 구현체에서 문자열 구문에 따라 직접 의미를 부여하도록 하기 위해서이다. 따라서, Destination 정보는 STOMP 서버 구현체마다 달라질수 있기 때문에 각 구현체의 스펙을 살펴봐야 한다.
하지만,일반적으로 **/topic/**은 public-subscribe(one-to-many)을 의미하고,**/queue/**은 point-to-point(one-to-one) 메시지 교환을 의미한다.
STOMP 서버는 모든 subscriber들에게 메시지를 브로드캐스팅 하기 위해 MESSAGE COMMAND를 사용할수 있다.

```
MESSAGE
message-id:nxahklf6-1
subscription:sub-1
destination:/topic/price.stock.MMM

{"ticker":"MMM","price":129.45}^@
```
서버의 모든 메시지는 특정 클라이언트 subscription에 응답해야만 하며, 서버 메시지의 subscription-id 헤더는 클라이언트가 subscribe한 id헤더와 일치해야 한다.

### STOMP Benefits
Spring Framework 및 Spring Security는 STOMP프로토콜을 사용하여, WebSocket만 이용할때보다 더 풍부한 프로그래밍 모델을 제공할수 있다.
* 메시징 프로토콜을 만들고, 메시지 형식을 커스터마이징할 필요가 없다.
* RabbitMQ,ActiveMQ 같은 message broker를 subscription 관리와 메시지 브로드캐스팅에 사용할수 있다.
* WebSocket 기반으로 한 커넥션마다 WebSocketHandler를 구현하는것보다, STOMP Destination 헤더를 기준으로  @Controller 객체의 @MethodMapping 메서드로 메시지를 라우팅 하도록 해서 조직적으로 관리할수 있다.
* STOMP의 Destination 및 Message Type를 기반으로 메시지를 보호하기 위해,Spring Security를 사용할수 있다.

### Enable STOMP
STOMP는 spring-messaging과 spring-websocket 모듈을 통해 이용가능하다.
기본적으로 커넥션을 위한 STOMP Endpoint를 설정해야 한다.
```java
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio").withSockJS();
    }
    //WebSocket 또는 SockJS 클라이언트가 WebSocket Handshake로 커넥션을 생성할 경로.

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.setApplicationDestinationPrefixes("/app");
        //Destination 헤더가 /app으로 시작하는 STOMP메시지들은 @Controller클래스의 @MessageMapping메소드로 라우팅 된다.
        config.enableSimpleBroker("/topic", "/queue");
        //subscription 과 broadcasting를 위해 내장 message broker를 사용하고 destination 헤더가 /topic 또는 /queue로 시작하는 메시지들을 브로커에게 라우팅한다.
    }
}
```
```
내장 simple broker에게 /topic 그리고 /queue prefix는 어떠한 특별한 의미가 없다.
단지 pub-sub 과 point-to-point messaging를 구별하기 위한 관례일뿐이다.
외부 브로커를 사용할경우, 지원하는 STOMP destination과 prefix를 이해하기 위해 브로커의 STOMP페이지를 확인해야 한다.
```

SockJS로 브라우저에서 커넥트하기 위해서는 sockjs-client 라이브러리를 사용한다. 최근에는 webstomp-client라이브러리를 많이 사용한다.

아래는 SockJS를 기반으로한 STOMP프로토콜을 이용해서 서버와 통신하는 예제이다.

```JavaScript
var socket = new SockJS("/spring-websocket-portfolio/portfolio");
var stompClient = webstomp.over(socket);

stompClient.connect({}, function(frame) {
}

```
SockJS 없이 WebSocket을 통해서 커넥트 할수도 있다.
```JavaScript
var socket = new WebSocket("/spring-websocket-portfolio/portfolio");
var stompClient = Stomp.over(socket);

stompClient.connect({}, function(frame) {
}
```
위의 코드에서 stompClient가 login과 passcode 헤더를 명시할 필요가 없었다는것에 주목하자. 만약 클라이언트에서 설정했더라도 서버측에서 무시했을것이다.

### Flow of Messages
일단 STOMP Endpoint를 노출하면, 스프링 애플리케이션은 연결된 클라이언트에 대한 STOMP Broker가 된다.
이번 섹션에서는 서버 사이드에서의 메시지 흐름을 설명한다.

![](https://docs.spring.io/spring-framework/docs/current/reference/html/images/message-flow-simple-broker.png)


spring-message모듈은 스프링 프레임워크의 통합된 메시징 애플리케이션을 위한 근본적인 지원을 한다. 다음 목록에서 몇가지 사용가능한 메세징 추상화에 대해 설명한다.
* Message: headers와 payload를 포함하는 메시지의 representation.
* MessageHandler: Message 처리에 대한 계약
* MessageChannel: Producers와 Consumers 간 느슨한 연결을 가능하게 하는 메시지 전송 계약
* SubscribableChannel: MessageHandler 구독자를 위한 MessageChannel이다.
즉, Subscribers를 관리하고, 해당 채널에 전송된 메시지를 처리할 Subscribers를 호출한다.
* ExecutorSubscribableChannel: Executor를 사용해서 메시지를 전달하는 SubscribableChannel이다.
즉, 각 구독자(Subsecribers)에게 메시지를 보내는 SubscribableChannel이다.
* clientInboundChannel:
WebSocket 클라이언트로 부터 받은 메시지 전달 위함
* clientOutboundChannel:
서버가 WebSocket 클라이언트에게 메시지 보내기 위함
* brokerChannel: 서버의 애플리케이션 코드 내에서 브로커에게 메시지를 전달한다.

외부 broker을 사용할 경우 구성 요소
![](https://docs.spring.io/spring-framework/docs/current/reference/html/images/message-flow-broker-relay.png)
[https://docs.spring.io/spring-framework/docs/current/reference/html/images/message-flow-broker-relay.png](https://docs.spring.io/spring-framework/docs/current/reference/html/images/message-flow-broker-relay.png)

앞의 두 다이어그램 주요 차이점은 broker relay를 통해 TCP를 통해 외부 STOP broker에게 메시지를 전달하고 TCP를 통해 Broker Relay 이용, broker에서 subscribed clients에게 메시지를 전달하는것이다.

#### 동작 흐름
1. WebSocket 커넥션으로 부터 메시지를 전달 받는다.
2. STOMP Frame으로 디코드 한다.
3. Spring Message representation으로 변환한다.
4. 추가 처리를 위해 clientInboundChannel로 보낸다. <br>
4-1. destination header가 /app 으로 시작하면 @MessageMaping 으로 매핑된 컨트롤러를 호출한다.<br>
4-2. 반면에 /topic, /queue메세지들은 message broker로 바로 라우팅된다.
