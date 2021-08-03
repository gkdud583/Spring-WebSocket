
**모든 내용은 명품 JAVA Programming 책을 정리한 내용입니다.**

# 소켓 프로그래밍

## 소켓(socket)
통신하는 두 응용프로그램 간의 통신 링크의 각 끝단으로서, TCP/IP의 네트워크 기능을 활용하여 다른 컴퓨터의 소켓과 데이터를 주고 받는다.

## 소켓 통신
응용프로그램은 소켓과 연결한 후 소켓에 데이터를 주기만 하면, 소켓이 상대방 응용프로그램에 연결된 소켓에 데이터를 보낸다. 또는 응용프로그램은 연결된 소켓으로 부터 도착한 데이터를 단순히 받기만 하면 된다.

## 소켓과 서버 클라이언트 통신
통신은 서버가 먼저 클라이언트의 접속을 기다리고, 클라이언트에서 서버에 접속하면, 그떄부터 서버나 클라이언트가 데이터를 서로 주고 받을수 있다. 서버나 클라이언트가 보내는 순서를 정하거나 순서에 상관없이 데이터를 전송하는것은 개발자가 프로그램을 작성하기에 달렸다.
## 서버 소켓과 클라이언트 소켓
**1. 서버 소켓**      
서버소켓은 서버 응용프로그램이 사용자의 접속을 기다리는 목적으로만 사용   
<br>
**2. 클라이언트 소켓**   
클라이언트 응용프로그램에서는 클라이언트 소켓을 이용하여 서버에 접속한다.   
<br>
**3. 서버소켓**  
서버소켓은 클라이언트가 접속해오면, 클라이언트 소켓을 추가로 만들어 상대 클라이언트와 통신하게 한다.

## 서버에서 클라이언트 소켓들의 포트 공유   
  **1.  포트 공유**   
  서버쪽의 통신 프로그램은 각각 독립된 소켓을 이용하여 클라이언트와 통신을 수행한다.
  클라이언트가 처음 서버 소켓에 연결될떄, 운영체제는 클라이언트 IP주소와 포트번호를 저장하고 기억해둔다. 그 후 서버 컴퓨터의 운영체제는 클라이언트로부터 데이터 패킷을 받게 되면, 패킷 속에 들어있는 클라이언트의 IP주소와 포트 번호를 참고하여, 서버에 있는 클라이언트 소켓을 찾아 그곳으로 데이터를 보낸다. <br>        
**2. 소켓을 이용한 서버 클라이언트 통신 프로그램 구성**  
  1. 서버응용프로그램은 ServetSocket 클래스를 이용하여 서버 소켓 객체를 생성하고(new ServerSocket(서버 port)) 클라이언트의 접속을 받기 위해 기다린다.   
  2. 클라이언트 응용프로그램은 Socket클래스를 이용하여 클라이언트 소켓 객체를 생성하고(new Socekt(서버IP, 서버 port)) 서버에 접속을 시도한다.   소켓 객체를 생성할때, 접속 할 서버 소켓의 IP주소와 포트번호를 지정한다.
  3. 서버는 클라이언트로부터 접속 요청을 받으면, accept() 메소드에서 접속된 클라이언트와 통신하도록 전용 클라이언트 소켓을 따로 생성한다.   
  4. 서버와 클라이언트 모두 소켓으로부터 입출력 스트림을 얻어내고 데이터를 주고 받을 준비를 한다.   
  5. 서버에 생성된 클라이언트 전용 소켓과 클라이언트의 소켓이 상호 연결된 채 스트림을 이용하여 양방향으로 데이터를 주고 받는다.   
  6. 서버는 클라이언트가 접속해올때마다 accept() 메소드에서 따로 전용 클라이언트 소켓을 생성하여 클라이언트와 통신하도록 한다. 통신이 끝나면 소켓을 닫는다.   

## Socket클래스, 클라이언트 소켓
  **1. 클라이언트 소켓 생성 및 서버 접속**

  ```java
  //클라이언트의 포트는 사용되지 않는 포트 중에서 자동으로 선택된다.
  Socket clientSocket = new Socket("128.12.1.1", 5550); // 128.12.1.1 서버에 접속
  ````
  **2. 네트워크 입출력 스트림 생성**      
  소켓이 만들어지고 서버와 연결이 된 후에는 Socket클래스의 getInputStream() 과 getOutputStream() 메소드를 이용하여 서버와 데이터를 주고 받을 소켓 스트림을 얻어내고 이를 버퍼 스트림에 연결한다.

  ```java
  BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
  BufferedWriter out =new BufferedWriter(new OutputStreamWriter(clientSocket.getOutputStream()));

  ```
**3. 서버로 데이터 전송**   
이제 버퍼 출력 스트림 out을 통해 데이터를 전송해보자.
```java
out.write("hello"+"\n");
out.flush();
````
**4. 서버로부터 데이터 수신**
```java
int x = in.read(); // 한 개의 문자 수신
String line = in.readLine(); // 한 행의 문자열 수신
```
**5. 데이터 송수신 종료**
```java
socket.close();
```

## ServerSocket 클래스, 서버 소켓
ServerSocket은 클라이언트로부터 연결 요청을 기다리는 목적으로만 사용되며,   
서버가 클라이언트의 연결 요청을 수락하면 Socket 객체를 별도로 생성하고, 이 Socket객체가 클라이언트와 데이터를 주고 받는다. ServerSocket은 데이터의 송수신에 사용되지 않는다.

**1. 서버 소켓 생성**   
포트번호는 클라이언트의 접속을 기다릴 자신의 포트번호이다.

```java
ServerSocket listener = new ServerSocket(9999);
```

**2. 클라이언트로부터 접속 대기**    
ServerSocket 클래스의 accept() 메소드를 이용하여 클라이언트로부터 연결 요청을 기다린다. accept() 메소드가 연결을 수락하면 Socket 객체를 하나 별도로 생성하여 리턴한다.

```java
Socket socket = listener.accept();
````

**3. 네트워크 입출력 스트림 생성**   
클라이언트로 데이터를 주고 받기 위한 스트림 객체는, ServerSocket의 accept()메소드로부터 얻은 accept객체의 getInputStream() 과 getOutputStream() 메소드를 이용하여 얻어낸다.
```java
BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
BufferedWriter out = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
```
**4. 클라이언트로부터 데이터 수신**
```java
int x = in.read();
String line = in.readLine();
```
**5. 클라이언트로 데이터 전송**
```java
out.write("Hi!, Client" + "\n");
out.flush();
````
**6. 데이터 송수신 종료**
```java
socket.close();
```
**7. 서버 응용프로그램 종료**   
더 이상 클라이언트의 접속을 받지 않고 서버 응용프로그램을 종료하고자 하는 경우 ServerSocket을 종료시킨다.

```java
ServetSocket.close();
````
