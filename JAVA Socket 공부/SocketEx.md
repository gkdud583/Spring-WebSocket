
**모든 내용은 명품 JAVA Programming 책을 정리한 내용입니다.**

## 단방향 소켓 통신
```java
import java.io.*;
import java.net.*;
import java.util.*;


public class ClientEx {
    public static void main(String[]args){
        BufferedReader in = null;
        BufferedWriter out = null;
        Socket socket = null;
        Scanner scanner = new Scanner(System.in);
        try{
            socket = new Socket("localhost",9999);
            in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            out = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
            while(true){
                System.out.print("보내기>>");
                String outputMessage = scanner.nextLine();
                if(outputMessage.equals("")){
                    continue;
                }

                if(outputMessage.equalsIgnoreCase("bye")){
                    out.write(outputMessage+"\n");
                    out.flush();
                    break;
                }
                out.write(outputMessage+"\n");
                out.flush();
                String inputMessage = in.readLine();
                System.out.println("서버: " + inputMessage);

            }
        }catch(IOException e){
            System.out.println(e.getMessage());
        }finally{
            try{
                scanner.close();
                if(socket != null)
                    socket.close();
            }catch(IOException e){
                System.out.println("서버와 채팅 중 오류가 발생했습니다.");
            }
        }
    }

}

```
```java
import java.io.*;
import java.net.*;
import java.util.*;



public class ServerEx {
    public static void main(String[]args){
        BufferedReader in = null;
        BufferedWriter out = null;
        ServerSocket listener = null;
        Socket socket = null;
        Scanner scanner = new Scanner(System.in);

        try{
            listener = new ServerSocket(9999);
            System.out.println("연결을 기다리고 있습니다...");
            socket = listener.accept();
            System.out.println("연결되었습니다.");
            in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            out = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
            while(true){
                String inputMessage = in.readLine();
                if(inputMessage.equalsIgnoreCase("bye")){
                    System.out.println("클라이언트에서 bye로 연결을 종료하였음");
                    break;
                }
                System.out.println("클라이언트: " + inputMessage);
                System.out.println("보내기>>");
                String outputMessage = scanner.nextLine();
                out.write(outputMessage+"\n");
                out.flush();
            }
        }catch(IOException e){
                System.out.println(e.getMessage());
        }finally{
            try{
                scanner.close();
                socket.close();
                listener.close();
            }catch(IOException e){
                System.out.println("클라이언트와 채팅 중 오류가 발생했습니다.");
            }
        }
    }
}

```

## 양방향 소켓 통신
Server - ThreadRcv(읽기 위한 스레드), ThreadSend(쓰기 위한 스레드) 생성   
Client - ThreadRcv(읽기 위한 스레드), ThreadSend(쓰기 위한 스레드) 생성

둘중에 하나의 스레드가 죽기 전에 interrupt() 호출해서 나머지 하나의 스레드도 죽도록 함.

```java
import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

public class Server {
    public static void main(String[]args){
        ServerSocket listener = null;
        Socket socket = null;
        try{
            listener = new ServerSocket(9999);
            socket = listener.accept();
        }catch(IOException e){
            System.out.println(e.getMessage());
        }
        ThreadSend sendSer = new ThreadSend(socket); //보내는쪽 쓰레드
        ThreadRcv rcvSer = new ThreadRcv(socket); //받는쪽 쓰레드
        sendSer.setTh(rcvSer);
        rcvSer.setTh(sendSer);
        sendSer.start();
        rcvSer.start();
        try{
            listener.close();
        }catch(IOException e){
            System.out.println(e.getMessage());
        }


    }

}

```
```java
import java.io.IOException;
import java.net.Socket;

public class Client {
    public static void main(String[]args){
        Socket socket = null;
        try{
            socket = new Socket("localhost",9999);

        }catch(IOException e){
            System.out.println(e.getMessage());
        }
        ThreadSend sendSer = new ThreadSend(socket); //보내는쪽 쓰레드
        ThreadRcv rcvSer = new ThreadRcv(socket); //받는쪽 쓰레드
        // sendSer.setTh(rcvSer);
        sendSer.setTh(rcvSer);
        rcvSer.setTh(sendSer);

        sendSer.start();
        rcvSer.start();




    }
}

```
```java
import java.io.BufferedReader;
import java.io.BufferedWriter;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStreamWriter;
import java.net.Socket;

class ThreadSend extends Thread{

    ThreadRcv th = null;
    BufferedReader br = null;
    BufferedWriter out = null;
    Socket socket = null;
    public ThreadSend(Socket socket){
        this.socket = socket;
        try{
            br = new BufferedReader(new InputStreamReader(System.in));
            this.out = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
        }catch(IOException e)
        {
            System.out.println(e.getMessage());
        }
    }
    public void setTh(ThreadRcv th){
        this.th = th;
    }

    @Override
    public void run() {
        while(true){
            try{
                // wait until we have data to complete a readLine()
                while(!br.ready()){
                    sleep(200);
                }
                String outputMessage = br.readLine();
                out.write(outputMessage+"\n");
                out.flush();
                if(outputMessage.equals("bye")){
                    if(th.isAlive())
                        th.interrupt();
                    out.close();
                    return;
                }
            }catch(IOException e){
                return;
            }catch(InterruptedException e){
                return;
            }

        }
    }
}
```

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.Socket;

class ThreadRcv extends Thread{
    ThreadSend th = null;
    BufferedReader in = null;
    Socket socket = null;
    public ThreadRcv(Socket socket){
        this.socket = socket;
        try{
            in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
        }catch(IOException e){
            System.out.println(e.getMessage());

        }

    }
    public void setTh(ThreadSend th){
        this.th = th;
    }
    @Override
    public void run() {
        while(true){
            try{
                while(!in.ready()){
                    sleep(200);
                }
                String inputMessage = in.readLine();
                System.out.println("상대방: " + inputMessage);
                if(inputMessage.equals("bye")){
                    if(th.isAlive())
                        th.interrupt();
                    in.close();
                    return;
                }
            }catch(IOException e){
                return;
            }catch(InterruptedException e){
                return;
            }
        }
    }


}
```
