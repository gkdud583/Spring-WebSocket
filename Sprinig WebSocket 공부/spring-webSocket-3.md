@Controller 컨트롤러는 STOMP 메시지를 처리하고 brokerChannel을 통해 메시지를 message broker에게 보낸다.
그리고 broker는 clientOutboundChannel을 통해 매칭된 subscribers에게 메시지를 브로드캐스팅한다.

다음 예제는 메시지 처리과정을 보여준다.
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio");
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/app");
        registry.enableSimpleBroker("/topic");
    }
}

@Controller
public class GreetingController {

    @MessageMapping("/greeting") {
    public String handle(String greeting) {
        return "[" + getTimestamp() + ": " + greeting;
    }
}
```
#### 동작 과정
1. 클라이언트가  **http://localhost:8080/portfolio**에 커넥트한다. WebSocket 커넥션이 성공하면, STOMP frame들을 해당 커넥션으로 전송하기 시작한다.
2. 클라이언트는 **/topic/greeting**경로로 **SUBSCRIBE Frame**을 보낸다.
서버는 프레임을 수신하여 디코딩 한 후,메시지는 clientInboundChannel로 보내진다.
그후에 클라이언트의 subscription 정보를 저장하고 있는 message broker로 라우팅되고 해당 클라이언트의 구독 정보를 저장한다.
3. 이후 클라이언트는 /app/greeting으로 **SEND frame** 을 보낸다.
**/app** prefix는 해당 메시지가 @MessageMapping 메소드를 가진 컨트롤러로 라우팅 될수 있도록 도움을 준다.
**/app** prefix가 벗겨진후, destination의 **/greeing**은  @MessageMapping 가진 메서드 **GreetingController**로 라우팅된다.
4. **GreetingController** 반환 값은 스프링의 Message로 변환된다. Messgae의 payload는 컨트롤러의 반환값을 기반으로 하고, 기본적으로 destination header는 **/topic/greeting**이 된다.
(/app -> /topic으로 바뀐것)
변환된 메시지는 brokerChannel로 보내지고 message broker에 의해 처리된다.
5. message broker는 매칭된 모든 subscribers을 찾고, clientOutboundChannel을 통해 각각에 **MESSAGE frame**을 보낸다.
clientOutboundChannel에서는 메시지를 STOMP frame으로 인코딩하고, 연결된 WebSocket 커넥션으로 프레임을 전송한다.

### Annotated Controllers
어플리케이션은 클라이언트가 보내는 메시지를 처리하기 위해 @Controller 클래스를 사용할수 있다. 이러한 컨트롤러는 @MessageMapping, @SubscribeMapping,@ExceptionHandler메서드를 사용할수 있다.

#### @MessageMapping
@MessageMapping 메서드는 지정한 경로를 기반으로 메시지를 라우팅할수 있다.
@MessageMapping은 메서드 뿐만 아니라 타입레벨, 즉 클래스에서도 설정할수 있는데 이는 컨트롤러 안에서 공통된 경로를 제공하기 위해서 사용된다.
기본적으로 매핑은 Path패턴으로 구성하고, Template변수도 지원한다.
(ex, /thing/{id})
Template변수는 @DestinationVariable 메소드 아규먼트를 통해 사용할수 있다.
[Supported Method Arguments](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#supported-method-arguments)

#### Return Values
기본적으로 @MessageMapping 메서드의 반환값은 일치하는 MessageConverter를 통해 payload로 직렬화 되고, 구독자에게 브로드캐스팅되는 BrokerChannel에 메시지로 전송된다. 전송되는 메세지의 destination 헤더는 클라이언트로부터 전달 받은 destination 헤더값에 접두사만 /topic으로 변경된값으로 설정된다.
Output message의 target destination을 커스터마이징 하고자 할 경우 @SendTo를 사용할수 있다.
@SendToUser는 Output message를 오직 input message와 관련된 사용자에게만 보내고자 할 경우 사용한다.
같은 메소드 또는 클래스에서  @SendTo,@SendToUser를 모두 사용할수 있다.
만약 @MessageMapping 메서드에서 메시지를 비동기적으로 처리하고 싶은 경우에는 ListenableFuture, CompletableFuture 또는 CompletionStage객체를 반환하면 된다.

#### SubscribeMapping
@SubscribeMapping은 @MessageMapping과 유사하지만 subscription message만 매핑한다는 차이점이 있다.
@SubscribeMapping은 기본적으로 brokerChannel을 통해 broker로 전달되는게 아니라 clientOutboundChannel을 통해 클라이언트로 바로 메시지가 보내진다. @SendTo 또는 @SendToUser을 오버라이드 해서 broker로 메시지가 보내지도록 할수도 있다.

***그럼 @SubscribeMapping은 언제 사용될까?***
broker가 /topic, /queue 로 매핑되고 어플리케이션 컨트롤러가 /app으로 매핑된다고 할떄 broker가 /topic,/queue에 대한 모든 subsciption을 저장하고 있으므로 어플리케이션이 개입할 필요가 없다.
그런데 클라이언트가 /app 접두사를 가진 목적지로 구독 요청 보내는 상황을 가정할때, @SubscribeMapping을 사용한다면, 컨트롤러는 브로커  통과 없이 return value를 구독에 대한 응답으로 보낸다.
즉, @SubscribeMapping은 브로커에 구독정보를 저장하지 않을뿐더러 구독 정보를 재활용 하지도 않는 일회성 용도로 사용된다. 일회성 request-reply교환인것이다.


### MessageExceptionHandler
@MessageMapping 메소드에서 exception을 처리 하기위해 @MessageExceptionHandler메소드를 지원한다. 발생한 예외는 Method Argument을 통해 접근할수 있다.
```java
@Controller
public class MyController {

    // ...

    @MessageExceptionHandler
    public ApplicationError handleException(MyException exception) {
        // ...
        return appError;
    }
}
```
일반적으로 @MessageExceptionHandler메소드는 선언된 @Controller 클래스내에 적용된다.
만약 글로벌하게 예외처리 메소드를 적용하고 싶으면, @ControllerAdvice 컨트롤러에 선언하면 된다.

### Sending Messages
어플리케이션에서 연결된 클라이언트에게 메시지를 보내야 한다면 어떻게 해야 할까?
brokerChannel을 통해 메시지를 보낼수도 있지만
가장 쉬운 방법은 **SimpleMessagingTemplate**을 주입하고 이것을 이용해서 메시지를 보내는것이다.
```java
@Controller
public class GreetingController {

    private SimpMessagingTemplate template;

    @Autowired
    public GreetingController(SimpMessagingTemplate template) {
        this.template = template;
    }

    @RequestMapping(path="/greetings", method=POST)
    public void greet(String greeting) {
        String text = "[" + getTimestamp() + "]:" + greeting;
        this.template.convertAndSend("/topic/greetings", text);
    }

}
```

### Simple Broker
내장된 simple message broker는 클라이언트로부터 subscription 요청을 메모리에 저장하고 처리한다. 그리고 Destination 헤더와 일치하는 클라이언트 커넥션에 메시지를 브로드캐스팅한다.
만약 TaskScheduler와 함께 설정한다면 simple broker는 STOMP hearbeats를 지원한다.
자신만의 스케줄러를 구현해서 사용하거나 기본적으로 등록되는 스케줄러를 사용할수도 있다.
다음은 자신이 구현한 TaskScheduler를 등록해서 사용하는 에제이다.
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private TaskScheduler messageBrokerTaskScheduler;

    @Autowired
    public void setMessageBrokerTaskScheduler(TaskScheduler taskScheduler) {
        this.messageBrokerTaskScheduler = taskScheduler;
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {

        registry.enableSimpleBroker("/queue/", "/topic/")
                .setHeartbeatValue(new long[] {10000, 20000})
                .setTaskScheduler(this.messageBrokerTaskScheduler);

        // ...
    }
}
```
### External Broker
simple broker는 처음 시작용으로는 좋지만 일부의 STOMP COMMAND만 지원한다는 단점이 있다. (acks, receipts, 등등 지원X)
또한, 연결된 구독자들에게 메시지 전송하는 경우에 Simple Message Broker는 단순한 반복문(Loop)에 의존하기 때문에 클러스터링에 적합하지 않다는단점이 있다.
완전한 기능을 갖춘 message broker를 사용함으로써 어플리케이션을 업그레이드 할수 있다.
message broker를 설치한 다음
Spring 설정에서 simple broker대신 STOMP broker relay를 활성화할수 있다.
다음 예시는 완전한 기능을 갖춘 broker를 사용하도록 구성한예제이다.
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/portfolio").withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableStompBrokerRelay("/topic", "/queue");
        registry.setApplicationDestinationPrefixes("/app");
    }

}
```
이전 구성에서 STOMP broker relay는 메시지를 외부 메시지 브로커로 포워딩하는 스프링의 MessageHandler(StompBrokerRelayMessageHandler)이다.
StompBrokerRelayMessageHandler는 아래순서로 동작한다.

1. broker로 TCP 커넥션을 수립한다.
(각 클라이언트마다 독립된 TCP커넥션을 사용하는데, 이는 session-id메시지 헤더로 식별한다.)
2. 모든 메시지를 브로커에게 전달한 다음,
3. 브로커로부터 수신한  모든 메시지는  각각의 session-id를 메시지 헤더에 더하고 WebSocket session을 통해 클라이언트에게 보낸다.

기본적으로 메시지를 양방향으로 전달하는 "relay" 역할을 한다.

애플리케이션의 컨트롤러, 서비스등의 컴포넌트등에서도 구독중인 WebSocket클라이언트들에게 메시지를 브로드캐스트 하기 위해 broker relay에 메시지를 보낼수도 있다.

### Connecting to a Broker
StompBrokerRelayMessageHandler는 기본적으로 메시지 브로커와 단 하나의 System TCP Connection을 수립한다.
이러한 커넥션은 메시지를 수신하기 위한것이 아니라 서버 애플리케이션이 메시지 브로커에게 메시지를 전달하기 위한 용도이다.
스프링은 STOMP Credentials(STOMP Frame login, passcode headers)를 설정할수 있도록 메서드를 제공한다.
메서드들은 systemLogin, systemPasscode속성을 설정하는데, 디폴트 값은 모두 guest이다.

STOMP broker relay는 또한 연결된 WebSocket 클라이언트마다 TCP 커넥션을을 수립한다.
스프링은 클라이언트 대신 생성된 모든 TCP커넥션에 STOMP credentials 설정 가능하도록 메서드를 제공한다.
해당 메서드들은 clientLogin, clientPassCode 속성을 설정하는데, 디폴트 값은 모두 guest이다.
STOMP broker relay는 클라이언트 대신 해서 브로커로 전달하는 모든 CONNECT Frame에 항상 login 및 passcode 헤더를 설정하기 때문에, WebSocket 클라이언트는 해당 헤더를 설정 할 필요가 없다.(설정 했더라도 무시된다.)
WebSocket 클라이언트는 대신 HTTP 인증을 사용해 WebSocket Endpoint를 보호하고 클라이언트 식별자를 설정해야 한다.
STOMP broker relay는 System TCP connection을 통해 message broker와 heartbeat를 주고 받는다.
보내고 받는 hearbeat 간격을 설정할수 있다. (기본은 10초)
만약 브로커와 연결이 끊어진다면, STOMP broker relay는 성공할때까지 5초마다 재연결을 시도한다.
스프링 빈은 broker와의 System connection이 끊어지거나 재연결 될 때 알림을 받기 위해  ApplicationListener<BrokerAvailabilityEvent>을 구현할수 있다.
이를 통해 System TCP Connection이 비활성화 된 경우에는 메시지 전송을 중지할수 있다.
기본적으로 STOMP broker relay는 항상 같은 호스트, 같은 포트로 재연결을 시도한다.
만약 여러 URL을 가지고 매번 다르게 연결 시도를 하고 싶다면, 고정된 Host, port대신하여 주소 공급자를 설정할수 있다.

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig extends AbstractWebSocketMessageBrokerConfigurer {

    // ...

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.enableStompBrokerRelay("/queue/", "/topic/").setTcpClient(createTcpClient());
        registry.setApplicationDestinationPrefixes("/app");
    }

    private ReactorNettyTcpClient<byte[]> createTcpClient() {
        return new ReactorNettyTcpClient<>(
                client -> client.addressSupplier(() -> ... ),
                new StompReactorNettyCodec());
    }
}
```
STOMP broker relay에 VirtualHost 속성을 설정한다면, VirtualHost 속성값은 CONNECT 프레임의 host헤더로 세팅되어 유용하게 사용된다.
예를 들어, "클라우드 기반으로 STOMP 메시지 브로커를 서비스하는 호스트"와 "TCP커넥션이 수립되는 실제 호스트"가 다른 클라우드 환경에서 유용하게 사용된다.


### Dots as Separators
메시지는 AntPathMatcher와 일치하는 @MessageMapping 메서드로 라우팅된다.
기본적으로 패턴은 구분자로 '/'를 사용한다. 이는 HTTP URL과 유사하기 때문에 웹 어플리케이션에서 좋은 컨벤션이다. 하지만 메시징 시스템에 적합한 컨벤션을 적용하고 싶다면, '.'구분자를 사용하면 된다.
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    // ...

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setPathMatcher(new AntPathMatcher("."));
        registry.enableStompBrokerRelay("/queue", "/topic");
        registry.setApplicationDestinationPrefixes("/app");
    }
}
```
```java
@Controller
@MessageMapping("red")
public class RedController {

    @MessageMapping("blue.{green}")
    public void handleGreen(@DestinationVariable String green) {
        // ...
    }
}
```
클라이언트는 /app/red.blue.green123으로 메시지를 보낼수 있게 된다.

앞의 예제를 보면, broker relay에 대한 접두사는 전적으로 외부 메시지 브로커 스펙을 따르기 때문에, 스프링에서는 직접적으로 변경할수 없다는것을 알수 있다.
반면에, 스프링 소켓 모듈에 내장된 simple Broker는 설정한 PathMatcher에 의존한다.
그래서 구분자를 변경하면 이러한 변화는 브로커뿐만 아니라 브로커가 subsciption패턴으로 메시지와 구독자를 매칭하는 방법도 변경된다.
