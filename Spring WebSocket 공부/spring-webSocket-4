### Authentication
WebSocket messaging session을 통한 모든 STOMP는 HTTP 요청으로 시작한다.
WebSocket HandShake 과정으로 HTTP요청은 WebSocket으로 Upgrade한다. (SockJS 경우에는 일련의 SockJS HTTP transport 요청을 보낸다.)
많은 웹 어플리케이션은 이미 HTTP 요청을 보호하기 위해 인증 및 권한을 제공하고 있다.
일반적으로 로그인 페이지, HTTP basic 인증 등의 메커니즘을 사용함으로써 spring security를 통해 사용자를 인증한다.
인증된 사용자에 대한 보안 컨텍스트는 HTTP 세션에 저장되고, 그 다음 요청부터는 쿠키 기반으로 동일하게 연결된다.
WebSocket handshake 또는 SockJS HTTP transport 요청의 경우, 일반적으로 HttpServiceRequest#getUserPrincipal() 으로 접근 가능한 인증된 사용자가 이미 존재한다.
스프링은 자동으로 사용자와 사용자를 위해 만든 WebSocket 또는 SockJS 세션을 연결 짓고, user header를 통해 세션에 전달되는 모든 STOMP메시지를 연관짓는다
일반적인 웹 애플리케이션은 보안을 위해 이미 하고 있는 것 외에는 추가적으로 무엇을 할 필요가 없다.

사용자는 쿠키 기반 HTTP session(후에 사용자를 위해 만들어진 WebSocket 또는 SockJS 세션과 연관되어짐)을 통해 유지되는 보안 컨텍스트와 함께 HTTP 요청 수준에서 인증된다.
이후 애플리케이션을 통과하는 모든 메시지는 사용자 헤더를 가지고 인증확인 과정을 거친다.

TCP 기반  STOMP프로토콜은 CONNECT frame에 login과 passcode헤더를 가지고 있다.
하지만, WebSocket통한 STOMP에서 기본적으로 스프링은 사용자가 이미 HTTP 전송 수준에서 인증되었다고 가정하기 대문에 STOMP 프로토콜 수준의 인증 헤더를 무시한다.
따라서, WebSocket, SockJS세션은 서버에 메시지를 보내기 위해 반드시 이미 인증된 사용자를 포함하고 있어야만 한다.

### Token Authentication
Spring Security OAuth는 JWT(JSON Web Token)같은 토큰 기반 보안을 지원한다.
WebSocket 기반 STOMP 프로토콜을 비롯한 웹 애플리케이션에서 Token 기반 보안을 사용할수 있다.
쿠키 기반 세션이 항상 최적의 방법이 될수는 없다.
예를 들어 어플리케이션이 서버 사이드 세션을 유지하지 않을수도 있고 모바일 어플리케이션에서는 일반적으로 인증 헤더를 선호한다.
WebSocket protocol, RFC 6455는
webSocket Handshake과정에서 서버가 클라이언트를 인증하는 방법을 규정하고 있지 않다.
또한 브라우저 클라이언트는 오직 표준 인증 헤더(기본 HTTP 인 ) 또는 쿠키만을 사용할수 있고 사용자 정의 헤더를 사용할수 없다.
뿐만 아니라, SockJS JavaScript는 SockJS transport요청과 함께 HTTP 헤더를 전달할 방법을 제공하지 않는다. 대신 이는 토큰을 보내는데 사용할수 있는 쿼리 파라미터를 보내는것을 허용한다. 하지만 이방법은 토큰이 URL과 함께 서버 로그에 실수로 기록될수 있다.
하지만 서버 애플리케이션은 HTTP수준에서 쿠키 사용 없이 인증할 마땅한 대안이 없다. 따라서 쿠키 사용 대신에 STOMP Messaging프로토콜 수준의 헤더를 이용해서 인증하는것을 생각해 볼수 있다. 그렇게 하려면 아래와 같이 두 단계가 필요하다.

1. STOMP클라이언트는 CONNECT프레임에 pass인증 헤더를 추가해야 한다.
2. ChannelIntercepter사용해서 인증헤더를 처리한다.

다음은 커스텀 인증 인터셉터를 등록하기 위한 서버 사이드 설정 예제이다.

```java
@Configuration
@EnableWebSocketMessageBroker
public class MyConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(new ChannelInterceptor() {
            @Override
            public Message<?> preSend(Message<?> message, MessageChannel channel) {
                StompHeaderAccessor accessor =
                        MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
                if (StompCommand.CONNECT.equals(accessor.getCommand())) {
                    Authentication user = ... ; // access authentication header(s)
                    accessor.setUser(user);
                }
                return message;
            }
        });
    }
}
```
인터셉터는 오직 CONNECT 메시지를 기반으로 인증하고, 사용자 헤더를 추가하는 기능을 제공한다.
스프링은 인증된 사용자를 저장하고 동일한 세션으로 전달된 STOMP 메시지에 인증된 사용자 정보를 추가해준다.
메시지 기반으로 Spring Security인증을 사용하는 경우에는 반드시 ChannelInterceptor을 Spring Security 보다 앞 쪽 순서에 설정해야 한다. 이는 WebSocketMessageBrokerConfigurer의 구현체에서 @Order(Ordered.HIGHEST_PRECEDENCE + 99)을 추가함으로써 가능하다.

### User Destinations
스프링의 STOMP는 /user 접두사를 가진 destination 헤더를 인식해서 애플리케이션이 특정한 사용자에게 메시지를 보낼수 있도록 지원한다.
예를 들어 사용자가 /user/queue/position-updates을 구독한다고 가정할때, 이 Destination은 UserDestinationMessageHandler에 의해 처리 되고 각 사용자 세션마다 고유한 Destination으로 변환된다. (/queue/position-updates-user123 같은)
이를 통해 동일한 Destination에 동시로 구독하는 다른 사용자와 충돌이 발생하지 않도록 보장한다.

송신측은 /user/{username}/queue/position-updates 같은 형식의 Destination에 메시지를 보낸다.
이후 UserDestinationMessageHandler에 의해 변환되어 하나 이상의 Destination으로 전달된다.
덕분에, 어플리케이션내의 컴포넌트들은 사용자의 이름과 Destination에 대한 정보 없이도 특정 사용자에게 메시지를 보낼수 있다.
구체적으로 이는 애노테이션과 메시징 템플릿을 이용해서 지원된다.

메세지 핸들링하는 메서드는 @SendToUser를 이용해서 특정 사용자에게 메시지를 보낼수 있다.
```java
@Controller
public class PortfolioController {

    @MessageMapping("/trade")
    @SendToUser("/queue/position-updates")
    public TradeResult executeTrade(Trade trade, Principal principal) {
        // ...
        return tradeResult;
    }
}
```
만약 사용자가 하나 이상의 세션을 가지고 있다면, 기본적으로 Destination에 구독한 모든 세션으로 메시지가보내진다.
그러나, 때때로 오직 메시지를 받은 하나의 세션에만 전송하고 싶은 경우에는 broadcast 속성을 false로 설정하면 된다.
```Java
@Controller
public class MyController {

    @MessageMapping("/action")
    public void handleAction() throws Exception{
        // raise MyBusinessException here
    }

    @MessageExceptionHandler
    @SendToUser(destinations="/queue/errors", broadcast=false)
    public ApplicationError handleException(MyBusinessException exception) {
        // ...
        return appError;
    }
}
```
먄약 다중 애플리케이션 서버 시나리오에서 사용자가 한 서버에 연결되어 있기 때문에, 다른 서버에서 /user Destination은 확인되지 않은 상태로 남아있을수 있다.
이런 경우, 다른 서버가 확인되지 않은 메시지를 브로드 캐스트 시도하도록 Destination을 설정할수 있다.
아래와 같이, MessageBrokerRegistry에 userDestinationBroadcast 속성을 설정해주면 된다.

```Java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
	@Override
	public void registerStompEndpoints(StompEndpointRegistry registry) {
		registry.addEndpoint("/test").setAllowedOrigins("*").withSockJS();
	}

	@Override
	public void configureMessageBroker(MessageBrokerRegistry registry) {
		registry
			.setApplicationDestinationPrefixes("/simple")
			.enableStompBrokerRelay("/topic", "/queue")
			.setUserDestinationBroadcast("/topic/unresolved-user-destination");
	}
}
```

### Order of Messages
broker로부터 받은 메시지는 clientOutboundChannel에 publish된다.
채널은 ThreadPoolExecutor에 의해 보관되며 메시지는 서로 다른 Thread에서 처리되기 때문에, 클라이언트가 수신한 메시지 순서는 clientOutboundChannel에 publication된 순서와 정확하게 일치한다고 보장할수 없다.

만약 해당 이슈가 있다면 setPreservePublishOrder속성을 설정하면 된다.
```JAVA@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
	@Override
	public void registerStompEndpoints(StompEndpointRegistry registry) {
		registry.addEndpoint("/test").setAllowedOrigins("*").withSockJS();
	}

	@Override
	public void configureMessageBroker(MessageBrokerRegistry registry) {
		registry
			.setPreservePublishOrder(true)
			.setApplicationDestinationPrefixes("/simple")
			.enableSimpleBroker("/topic", "/queue");
	}
}
```
설정이 되었다면 같은 클라이언트 세션을 통해서 보낼 메시지들은 한번에 하나씩 clientOutboundChannel에 publish된다.
따라서, clientOutboutChannel채널에 publication된 순서와 클라이언트가 수신한 메시지 순서를 정확하게 일치시키도록 보장할수 있다.
다만 약간의 성능 오버헤드가 발생할수 있기 떄문에 오직 필요한 경우에만 사용해야 한다.

### Events
ApplicationContext가 발생시키는 여러 Events는 스프링의 ApplicationListener를 통해서 전달 받을수 있다.

* BrokerAvailabilityEvent:
브로커가 사용할수 있게 되거나 사용할수 없게 될때 발생하는 이벤트이다.
simple broker는 시작과 동시에 이용 가능하고 어플리케이션이 동작하는 동안 계속 사용가능하지만, STOMP Broker Relay는 외부 broker와 연결이 끊길수 있다. (예를 들어 외부 broker가 재시작 하는 경우) 이런 경우 broker relay는 System TCP Connection을 다시 연결해야 한다. 결과적으로 이러한 이벤트는 브로커와의 연결 상태가 변경될때마다 발생한다.
SimpMessagingTemplate 을 사용하는 컴포넌트는 이러한 이벤트를 구독하여 브로커를 사용할수 없는 경우 메시지를 보내지 않도록 해야한다. 어떤 경우라도, 메시지를 보낼때 MessageDeliveryException을 처리할수 있도록 해야만 한다.

* SessionConnectEvent:
새로운 클라이언트 세션의 시작을 나타내는 STOMP CONNECT가 수신될때 발생하는 이벤트이다.
이 이벤트에는 session ID, 사용자 정보, 클라이언트가 보낸 사용자 정의 헤더를 포함하여 연결을 나타내는 메시지가 포함되기 때문에 사용자 추적에 용이하다.
이러한 이벤트를 구독한 컴포넌트들은 포함된 메시지를 SimpMessageHeaderAccessor 또는
StompMessageHeaderAccessor으로 Wrapping할수 있다.

* SessionConnectedEvent:
broker가 CONNECT에 대한 응답으로 STOMP CONNECTED 프레임을 보냈을때 SessionConnectEvent이벤트 발생 직후에 발생하는 이벤트이다. 이때 STOMP 세션은 완전히 수립된것으로 간주할수 있다.

* SessionSubscribeEvent:
새로운 STOMP SUBSCRIBE가 수신될때 발생하는 이벤트이다.

* SessionUnsubscribeEvent:
새로운 STOMP UNSUBSCRIBE가 수신될때 발생하는 이벤트이다.

* SessionDisconnectEvent: STOMP 세션이 끝난 경우에 발생한다.
클라이언트로부터 DISCONNECT 메시지를 받거나 WebSocket 세션이 닫히는 경우에 자동으로 이벤트가 발생한다. 경우에 따라 해다 이벤트가 두 번 이상 발생하기 때문에, 컴포넌트는 다양한 disconnect이벤트와 멱득성을 가져야 한다.

### Interception
Events는 STOMP connection의 라이프 사이클에 대한 알림을 제공하지만 모든 클라이언트 메시지에 대한 알림은 제공하지 않는다. 어플리케이션은 ChannelInterceptor을 등록하여 메시지와 processing chain의 모든 부분을 가로챌수 있다.
다음은 클라이언트로부터 inbound message를 가로채는 예제이다.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(new MyChannelInterceptor());
    }
}
```
커스텀 ChannelInterceptor은 메시지에 대한 정보를 얻기 위해  StompHeaderAccessor 또는 SimpMessageHeaderAccessor을 사용할수 있다.
```JAVA
public class MyChannelInterceptor implements ChannelInterceptor {

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
        StompCommand command = accessor.getStompCommand();
        // ...
        return message;
    }
}
```
어플리케이션은 또한 메시지를 별도의 Thread에서 처리하는  ExecutorChannelInterceptor(ChannelInterceptor하위 인터페이스)도 구현할수 있다.
ChannelInterceptor는 채널로 메시지가 보내질때마다 한번씩 호출되지만, ExecutorChannelInterceptor는 각 MessageHandler의 Thread에서 Hook을 제공한다.
즉, MessageHandler의 handleMessage()메서드가 호출되기 이전과 이후에 ExecutorChannelInterceptor의 beforeHandle() 와 afterMessageHandled() 메서드가 호출되는데, 실제로 각 메서드는 MessageHandler가 동작중인 Thread에서 실행된다.

### WebSocket Scope
각각의 WebSocket 세션은 속성에 대한 Map을 가지고 있다. Map은 inbound 클라이언트 메시지의 헤더에 추가되기 때문에, 컨트롤러 메소드에서 접근가능하다.
```java
@Controller
public class MyController {

    @MessageMapping("/action")
    public void handle(SimpMessageHeaderAccessor headerAccessor) {
        Map<String, Object> attrs = headerAccessor.getSessionAttributes();
        // ...
    }
}
```
WebSocket scope에서 스프링 빈을 선언할수 있다.
clientInboundChannel에 등록된 channel interceptor 와 컨트롤러에서 WebSocket 스코프 빈을 주입받을수 있다.
webSocket scope를 가진 Bean들은
일반적으로 싱글톤이지만, 개별 WebSocket세션보다 오래 유지된다.
그러므로 websocket scope를 가진 Bean은 아래 예제와 같이 Proxy Mode로 사용해야 한다.

```java
@Component
@Scope(scopeName = "websocket", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class MyBean {

    @PostConstruct
    public void init() {
        // Invoked after dependencies injected
    }

    // ...

    @PreDestroy
    public void destroy() {
        // Invoked when the WebSocket session ends
    }
}

@Controller
public class MyController {

    private final MyBean myBean;

    @Autowired
    public MyController(MyBean myBean) {
        this.myBean = myBean;
    }

    @MessageMapping("/action")
    public void handle() {
        // this.myBean from the current WebSocket session
    }
}
```
다른 커스텀 스코프와 마찬가지로, 스프링은 컨트롤러가 처음으로 MyBean에 접근했을때 MyBean 인스턴스를 생성,초기화 하고 WebSocket session attribute에 저장한다.
그다음, 세션이 종료될때까지 동일한 인스턴스가 반환된다. WebSocket 스코프 빈은 스프링이 라이프 사이클 단계마다 호출해주는 PostConstruct, PreDestroy 메서드를 가질수 있다.
