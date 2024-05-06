**Requirements**

+ < 요구조건 #1: 자바 API를 이용해서 클라이언트, 서버 프로그램 만들기>
+ < 요구조건 #2: 클라이언트로부터 수식을 받아서 서버에서 해결 후 결과 클라이언트에게 전송>
+ < 요구조건 #3: 아스키코드 기반 메세지 포맷 정의하기>
+ < 요구조건 #4: 네개의 연산자 : 더하기, 빼기, 곱하기, 나누기>
+ < 요구조건 #5: 서버로부터의 답변은 answer혹은 error message>
+ < 요구조건 #6: 서버는 다수의 클라이언트를 감당>
+ < 요구조건 #7: 서버 정보는 configuration file에 저장>

**Architecture Diagram**

![image](https://github.com/daedalus-KM/Computer-Newworm/assets/85052989/e231a6ca-7fd3-4ae6-91e1-d5a3593c8704)

**Protocol**

Request
+ [수식 정수 정수]의 형태로 구성되며 각각은 공백으로 구분.
+ 수식은 ADD, MIN, MUL, DIV로 구성되며 각각은 더하기 빼기 곱하기 나누기의 사칙연산을 의미
+ client에서 연결 종료할 경우 [FIN]이라는 request를 서버에게 보냄

Response
+ [상태코드 상태 정답(optional)]의 형태로 구성되며 각각은 공백으로 구분
+ 상태코드

  + 0 : client로부터 정상적인 수식이 전달된 경우
  + 1 : client로부터 0으로 나누려고 하는 수식이 전달된 경우
  + 2 : client로부터 너무 적은 argument가 전달된 경우 ex: ADD 3
  + 3 : client로부터 너무 많은 argument가 전달된 경우 ex: ADD 3 5 6
  + 4 : client로부터 server측에서 요청을 처리하는 과정에서 문제가 생긴 경우

+ 각 상태코드에 대한 상태
  + 0 : OK
  + 1 : Divide by zero
  + 2 : Too Less Argument
  + 3 : Too Many Argument
  + 4 : Server Error로 구성

**Sourse Code**
+ Server

```
/*
 * File:CalcServer.java
 * Descriptio of this file: this file get encoded math expression from client, and calclate that expresison
 * and transport answer to user. this process is processed using thread, so server can cover multiple user.
 */
package Thread;
import java.io.*;
import java.net.*;
import java.util.*;
import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class CalcServer {
    /*
     * description: calc is interpret encoded math expression, and calculate that expression. 
       and encode answer with defined protocol
       status code stat has 0 ~ 4 value.
     * parameter exp is encoded math expression
     */
    public static String calc(String exp){
        StringTokenizer st = new StringTokenizer(exp, " ");
        if(st.countTokens() < 3){
            return "2 Too Less Argument";
        }
        if(st.countTokens() > 3){
            return "3 Too Many Argument";
        }
        String res = "";
        String opcode = st.nextToken();
        int op1 = Integer.parseInt(st.nextToken());
        int op2 = Integer.parseInt(st.nextToken());
        switch(opcode){
            case "ADD":
                res = "0 OK " + Integer.toString(op1 + op2);
                break;
            case "MIN":
                res = "0 OK " + Integer.toString(op1 - op2);
                break;
            case "MUL":
                res = "0 OK " + Integer.toString(op1 * op2);
                break;
            case "DIV":
                if(op2 == 0){
                    res = "1 divide by zero";
                }
                else{
                    res = "0 OK " + Integer.toString(op1 / op2);
                }
                break;
            default:
                res = "4 Server error";
        }
        return res;
    }

    public static void main(String[] args) throws IOException{
        ServerSocket listener = new ServerSocket(9999);
        System.out.println("waiting to be connected.......");
        ExecutorService pool = Executors.newFixedThreadPool(20);
        while(true){
            Socket socket = listener.accept();
            pool.execute(new MyThread(socket));
        }
    }

    /*
     * Each Thread get encoded math expression from client, and proccessing that using calc method
     */
    private static class MyThread implements Runnable {
        private Socket socket;

        MyThread(Socket socket) {
            this.socket = socket;
        }
        //Thread for covering multiple user
        @Override
        public void run(){
            System.out.println("Connected: " + socket);
            try{
                BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream())); // to communicate with client
                BufferedWriter out = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
                while(true){
                    String InputMessage = in.readLine(); //request from client
                    System.out.println(InputMessage);
                    if(InputMessage.equalsIgnoreCase("fin")){
                        System.out.println("Connection closed from user");
                        break;
                    }
                    String res = calc(InputMessage); //calculate answer or determine that this request is invalid (error)
                    System.out.println(res);
                    out.write(res + "\n"); // write to client
                    out.flush();
                }
            } catch(IOException e){
                System.out.println(e.getMessage());
            } finally{
                try{
                    if(socket != null){
                        socket.close();
                        System.out.println("Closed: " + socket);
                    }
                } catch(IOException e){
                    System.out.println("Error appear while communicate with client");
                }
            }  
        }
    }
}


```
+ Client
```
/*
 * File:CalcClient.java
 * Description of this file: this file connect server with "server_info.txt", and get math expression from user, 
 * and change expression into valid protocol. and get answer from server, finally print answer.
 */
package Thread;

import java.io.*;
import java.net.*;
import java.util.*;

public class CalcClient {
    /*
     * Description: ToProtocol is change math expression into protocol that I defined.
     * parameter exp is math expression
     */
    public static String ToProtocol(String exp){
        StringTokenizer st = new StringTokenizer(exp, " ");
        String res = "";
        String temp = st.nextToken();
        String opcode = st.nextToken();
        switch(opcode){
            case "+":
                res = "ADD " + temp;
                while(st.countTokens() > 0){
                    res += " ";
                    temp = st.nextToken();
                    res += temp;
                }
                break;
            case "-":
                res = "MIN " + temp;;
                while(st.countTokens() > 0){
                    res += " ";
                    temp = st.nextToken();
                    res += temp;
                }
                break;
            case "*":
                res = "MUL " + temp;;
                while(st.countTokens() > 0){
                    res += " ";
                    temp = st.nextToken();
                    res += temp;
                }
                break;
            case "/":
                res = "DIV " + temp;;
                while(st.countTokens() > 0){
                    res += " ";
                    temp = st.nextToken();
                    res += temp;
                }
                break;
            default:
                res = "error! invalid operator";
        }
        return res;
    }

    /*
     * Description: ProcesResponse is for process response from server. that may be answer of math expression or error message.
     * parameter response is server response.
     */
    public static void ProcessResponse(String response){
        StringTokenizer st = new StringTokenizer(response, " ");
        int statCode = Integer.parseInt(st.nextToken());
        if(statCode == 0){ //success
            String status = st.nextToken();
            String answer = st.nextToken();
            System.out.println("Answer: " + answer);
            return;
        }
        else{
            switch(statCode){ //interpret server response
            case 1:
                System.out.println("can't divide by 0");
                break;
            case 2:
                System.out.println("Too less argument!");
                break;
            case 3:
                System.out.println("Too many argument!");
                break;
            case 4:
                System.out.println("Server Error!");
                break;
            default:
                System.out.println("Unexpected Error!");
            }
            return;
        }
    }

    /*
     * Descriptin: main method read server IP address and port # from "server_info.txt", and connect with server and communicate using
     * ToProtocol method and ProcessResponse method.
     */
    public static void main(String[] args){
        BufferedReader in = null;
        BufferedWriter out = null;
        Socket socket = null;
        Scanner scanner = new Scanner(System.in); // to get use input
        String Ip;
        int Port;
        try{
            String path = System.getProperty("user.dir");
            path = path + "\\server_info.txt"; //server information file path
            Server_info config = new Server_info();
            Configuration.readInfoFromFile(path, config);

            Ip = config.getIp();
            Port = config.getPort();

            socket = new Socket(Ip, Port);
            in = new BufferedReader(new InputStreamReader(socket.getInputStream())); //to communicate with server
            out = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
            while(true){
                System.out.print("enter expression with space. (ex: 24 + 42)>>");
                String outputMessage = scanner.nextLine();
                if(outputMessage.equalsIgnoreCase("bye")){
                    out.write("FIN"); 
                    out.flush();
                    break;
                }
                outputMessage = ToProtocol(outputMessage); //transform math expression to protocol 
                out.write(outputMessage + "\n"); // write to server
                out.flush();
                String inputMessage = in.readLine(); // response from server
                ProcessResponse(inputMessage); // interpret server response
            }
        } catch(NumberFormatException e1){
            Port = -1;
        }
        catch(IOException e){
            System.out.println(e.getMessage());
        } finally{
            try{
                scanner.close();
                if(socket != null){
                    socket.close();
                }
            } catch(IOException e){
                    System.out.println("error apear while communicate server.");
            }
        }
    }
    
    
}
/*
 * Description : this class to get server IP and port# with file path. 
   will set server inforamtion. if file doesn't exist, will set default value
 * parameter: filepath is file path that has server information (IP address, Port number), info is object for saving server information
 
 */
class Configuration {
    public static void readInfoFromFile(String filePath, Server_info info) {
        
        try (BufferedReader reader = new BufferedReader(new FileReader(filePath))) {
            String line = reader.readLine();
            String[] parts = line.split(" ");

            info.setIp(parts[0]);
            info.setPort(Integer.parseInt(parts[1]));
        } catch (IOException e) {
            e.printStackTrace();
            info.setIp("localhost"); // default value
            info.setPort(9999);
        }
    }
}

/*
 * Description: Server_info is for saving sever information in main method
 */
class Server_info {
    private String Ip;
    private int Port;

    public String getIp() {
        return Ip;
    }

    public void setIp(String Ip) {
        this.Ip = Ip;
    }

    public int getPort() {
        return Port;
    }

    public void setPort(int Port) {
        this.Port = Port;
    }
}

```
**Output**
+ 상황 1 : 일반적인 상황에서의 동작. Server는 Client로부터 받은 요청과 본인의 응답을 모두 출력
  + Client Side

![image](https://github.com/daedalus-KM/Computer-Newworm/assets/85052989/fb4d4469-c390-4cb0-814a-0dc9578442a1)
  + Server Side

![image](https://github.com/daedalus-KM/Computer-Newworm/assets/85052989/cffe4dcc-e1e7-4d05-b7bd-d1fbbee5055b)

+ 상황 2 : 예외 발생 상황. 0으로 나누려는 상황, 인자가 너무 많은 상황, 인자가 너무 적은 상황
  + Client Side

![image](https://github.com/daedalus-KM/Computer-Newworm/assets/85052989/019a81b6-2f89-43ad-9a2a-ebcd7c17334e)
  + Server Side

![image](https://github.com/daedalus-KM/Computer-Newworm/assets/85052989/77870bdc-de9c-4a84-bbc1-e27d402d11c2)

+ 상황 3: 다수의 유저가 접속한 상황. Thread를 이용해 여러 유저를 커버
  + Server Side

![image](https://github.com/daedalus-KM/Computer-Newworm/assets/85052989/fbe3c944-1fe5-4fa9-8399-40fb5881fe05)


