---
layout:     post
title:      "简单的socket通讯"
subtitle:   "仿QQ"
date:       2021-07-30
author:     "Pengyujie"
header-img: "img/tag-bg.jpg"
hidden: true
tags:
  - 网络
  - 线程
  - Socket

---

>来自韩顺平老师的socket编程，听课后的练习



### Socket通讯系统 仿QQ

> 对于其中消息类型（msgType）： 1 表示登入成功   2 表示登入失败  3 普通信息包  4 要求返回的在线用户信息列表  5 返回的在线用户列表  6 客户端请求退出 7 群发消息   8 文件消息

### 客户端

#### 实体类

消息

~~~java
package com.example.demo.kuangstudy.hsp.qq.dao;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class Message implements Serializable {
    private String sender;
    private String getter;
    private String content;
    private String sendTime;
    private String msgType;

    //文件相关属性
    private byte[] fileBytes;
    private int fileLen = 0;
    private String dest;//将文件传输到哪
    private String src;
}
~~~



用户

~~~java
package com.example.demo.kuangstudy.hsp.qq.dao;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.io.Serializable;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class User implements Serializable {
    private String username;
    private String pwd;
}
~~~



#### view类  

可以当做客户端启动类（由客户在这边输入指令服务端进行接收）

~~~java
package com.example.demo.kuangstudy.hsp.qq.view;


import com.example.demo.kuangstudy.hsp.qq.service.FileClientService;
import com.example.demo.kuangstudy.hsp.qq.service.MessageClientService;
import com.example.demo.kuangstudy.hsp.qq.service.UserClientService;

import java.util.Scanner;

public class QQView {

    private UserClientService userClientService = new UserClientService();
    private FileClientService fileClientService = new FileClientService();//用于文件传输
    private boolean isMenu = true;

    public void MainMenu(){
        while(isMenu){
            System.out.println("===========QQ通讯系统=============");
            System.out.println("\t\t 1 登入系统");
            System.out.println("\t\t 9 退出系统");
            System.out.println("=================================");
            System.out.print("请输入您的选择：");

            Scanner scanner = new Scanner(System.in);
            String key = scanner.nextLine();


            switch(key){
                case "1":
                    System.out.println("请输入用户：");
                    String username = scanner.nextLine();

                    System.out.println("请输入密码：");
                    String pwd = scanner.nextLine();

                    //需要去服务端验证用户是否存在和合法

                    if(userClientService.checkUser(username,pwd)){
                        System.out.println("================欢迎---用户："+username+"==============");
                        //二级菜单
                        while(isMenu){
                            System.out.println("\n===========QQ通讯二级菜单  用户："+username+"===============");
                            System.out.println("\t\t 1 显示在线用户列表");
                            System.out.println("\t\t 2 群发消息");
                            System.out.println("\t\t 3 私聊消息");
                            System.out.println("\t\t 4 发送文件");
                            System.out.println("\t\t 9 退出系统");
                            System.out.println("==========================================================");

//                            Scanner scanner2 = new Scanner(System.in);
                            String key2 = scanner.nextLine();

                            switch (key2){
                                case "1":
                                    userClientService.onLineFriendList();
                                    break;
                                case "2":
                                    System.out.println("群发 请输入群发消息：");
                                    String contentToAll = scanner.nextLine();
                                    MessageClientService.sendMessageToAll(username,contentToAll);
                                    break;
                                case "3":
                                    System.out.print("输入想要私聊的对象（对方需要是在线状态）：");
                                    String getter = scanner.nextLine();
                                    
                                    System.out.println("===============通话中================");
                                    String content = scanner.nextLine();
                                    //将消息发送给服务端
                                    MessageClientService.sendMessageToOne(username,getter,content);//
                                    break;
                                case "4":
                                    System.out.println("请输入你想发送文件的用户（对方需要在线）：");
                                    String name = scanner.nextLine();
                                    System.out.println("请输入需要发送文件的路径：（格式：d:\\xx.jpg）");
                                    String src = scanner.nextLine();
                                    System.out.println("请输入存入对方的路径：（格式：d:\\yy.jpg）");
                                    String dest = scanner.nextLine();
                                    fileClientService.sendFileToOne(src,dest,username,name);
                                    break;
                                case "9":
                                    userClientService.logout();//向服务端发送断开连接请求
                                    System.out.println("退出！！");
                                    isMenu = false;
                                    break;
                            }

                        }
                    }else {
                        //登入失败
                        System.out.println("登入失败！！！");
                    }
                    break;
                case "9":
                    System.out.println("退出！！");
                    isMenu = false;
                    break;
            }
        }
    }
}
~~~



#### 客户端线程

~~~java
package com.example.demo.kuangstudy.hsp.qq.service;

import com.example.demo.kuangstudy.hsp.qq.dao.Message;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.net.Socket;


public class ClientConnectServerThread extends Thread{
    private Socket socket;

    public Socket getSocket() {
        return socket;
    }

    //构造器 可以接收一个Socket对象
    public ClientConnectServerThread(Socket socket){
        this.socket = socket;
    }

    @Override
    public void run() {

        System.out.println("=======================================");
        //因为Thread 需要在后台和服务器通讯，因此我们while循环
        while(true){
            try {
                System.out.println("读取从服务端发送过来的消息...");
                ObjectInputStream ois = new ObjectInputStream(socket.getInputStream());
                Message message = (Message) ois.readObject();


                if (message.getMsgType().equals("5")){//服务端返回过来的的在线用户列表
                    System.out.println("=========在线用户列表：==========");
                    String[] onlineUsers = message.getContent().substring(1).split(" ");
                    for (String onlineUser : onlineUsers) {
                        System.out.println(onlineUser);
                    }
                }
                else if (message.getMsgType().equals("3")){//私聊消息
                    //获取从服务器传来的消息
                    System.out.println(message.getSendTime()+" "+message.getSender()+": ");
                    System.out.println(message.getContent());
                }
                else if (message.getMsgType().equals("7")){//群发信息
                    //获取从服务器传来的群发消息
                    System.out.println(message.getSendTime()+" "+message.getSender()+": （群聊）");
                    System.out.println(message.getContent());
                }
                else if (message.getMsgType().equals("8")){//文件消息
                    //获取文件消息
                    System.out.println("获取文件来自"+ message.getSender()+ "存入地址："+message.getDest());
                    FileOutputStream fileOutputStream = new FileOutputStream(message.getDest());
                    fileOutputStream.write(message.getFileBytes());
                    fileOutputStream.close();
                    System.out.println("文件接收成功！");

                }else {
                    System.out.println("消息没有类型，无法获取~");
                }


            } catch (IOException | ClassNotFoundException e) {
                e.printStackTrace();
            }

        }
    }
}
~~~



#### 登入 、退出、显示在线用户的service

~~~java
package com.example.demo.kuangstudy.hsp.qq.service;

import com.example.demo.kuangstudy.hsp.qq.dao.Message;
import com.example.demo.kuangstudy.hsp.qq.dao.User;

import java.io.*;
import java.net.InetAddress;
import java.net.Socket;

//用来验证用户登入和用户注册
public class UserClientService {
    private boolean isCheck;
    private Socket socket;
    private User user = new User();

    public boolean checkUser(String username,String pwd){//用来进行登入判断
        user.setUsername(username);
        user.setPwd(pwd);
        //连接到服务端 然后去发送user对象
        try {
            socket = new Socket(InetAddress.getByName("127.0.0.1"), 9999);
            OutputStream outputStream = socket.getOutputStream();
            //得到一个ObjectOutputStream
            ObjectOutputStream oos = new ObjectOutputStream(outputStream);
            oos.writeObject(user);//发送user给服务端


            //读取从服务端回复的Message对象
            InputStream inputStream = socket.getInputStream();
            ObjectInputStream ois = new ObjectInputStream(inputStream);
            Message ms = (Message) ois.readObject();

            if(ms.getMsgType().equals("1")){//1 表示登入成功   2 表示登入失败  3 普通信息包  4 要求返回的在线用户信息列表  5 返回在线用户列表  6 客户端请求退出

                //登入成功后 就创建一个客户端线程去实现登入的这个用户与服务端的通讯  将socket（与服务端通讯的）放入这个线程
                ClientConnectServerThread ccst = new ClientConnectServerThread(socket);
                //启动这个线程
                ccst.start();
                //为了后面的客户端扩展，我们将socket放入一个集合中
                ManageClientConnectServerThread.addClientConnectServerThread(username,ccst);
                isCheck = true;

            }else{
                socket.close();
                isCheck = false;

            }


        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
        return isCheck;
    }

    //向服务器请求在线的用户列表
    public void onLineFriendList(){
        Message message = new Message();
        message.setMsgType("4");//1 表示登入成功   2 表示登入失败  3 普通信息包  4 要求返回的在线用户信息列表  5 返回的在线用户列表  6 客户端请求退出
        message.setSender(user.getUsername());
        //拿到登入用户的socket
        Socket socketUser = ManageClientConnectServerThread.getClientConnectServerThread(user.getUsername()).getSocket();
        try {
            OutputStream os = socketUser.getOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(os);
            oos.writeObject(message);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void logout(){
        Message message = new Message();
        message.setMsgType("6");
        message.setSender(user.getUsername());

        try {
            OutputStream os = socket.getOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(os);
            oos.writeObject(message);
            System.out.println(user.getUsername()+"退出系统");
            System.exit(0);//结束进程
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
~~~





#### 文件消息处理的service

~~~java
package com.example.demo.kuangstudy.hsp.qq.service;

import com.example.demo.kuangstudy.hsp.qq.dao.Message;

import java.io.*;
import java.net.Socket;

public class FileClientService {
    public void sendFileToOne(String src,String dest,String sender,String getter){
        //读取src文件 -->message
        Message message = new Message();
        message.setMsgType("8");//表示为文件
        message.setSender(sender);
        message.setGetter(getter);
        message.setSrc(src);
        message.setDest(dest);

        FileInputStream fileInputStream = null;
        byte[] fileBytes = new byte[(int) new File(src).length()];
        try {
            fileInputStream = new FileInputStream(src);
            fileInputStream.read(fileBytes);
            message.setFileBytes(fileBytes);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if(fileInputStream!=null){
                try {
                    fileInputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        //将文件发送过去
        ClientConnectServerThread clientConnectServerThread = ManageClientConnectServerThread.getClientConnectServerThread(message.getSender());
        Socket socket = clientConnectServerThread.getSocket();
        try {
            ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());
            oos.writeObject(message);
        } catch (IOException e) {
            e.printStackTrace();
        }
        //提示信息
        System.out.println("\n"+sender+"发送文件到"+getter);
        System.out.println("\n 文件地址由"+src+"到"+dest);
    }
}
~~~



#### 私聊和群发的 service

~~~java
package com.example.demo.kuangstudy.hsp.qq.service;

import com.example.demo.kuangstudy.hsp.qq.dao.Message;

import java.io.IOException;
import java.io.ObjectOutputStream;
import java.net.Socket;
import java.util.Date;

public class MessageClientService {
    public static void sendMessageToOne(String sender,String getter,String content){
        Message message = new Message();
        message.setContent(content);
        message.setSender(sender);
        message.setGetter(getter);
        message.setMsgType("3");
        message.setSendTime(new Date().toString());//发送时间

        ClientConnectServerThread clientConnectServerThread = ManageClientConnectServerThread.getClientConnectServerThread(sender);
        Socket socket = clientConnectServerThread.getSocket();
        try {
            ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());
            oos.writeObject(message);
            System.out.println(message.getSendTime()+" "+sender+": ");
            System.out.println(content);
        } catch (IOException e) {
            e.printStackTrace();
        }

    }

    public static void sendMessageToAll(String sender,String content){
        Message message = new Message();
        message.setContent(content);
        message.setSender(sender);
        message.setMsgType("7");//群发
        message.setSendTime(new Date().toString());//发送时间

        ClientConnectServerThread clientConnectServerThread = ManageClientConnectServerThread.getClientConnectServerThread(sender);
        Socket socket = clientConnectServerThread.getSocket();
        try {
            ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());
            oos.writeObject(message);
            System.out.println(message.getSendTime()+" "+sender+": ");
            System.out.println(content);
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}
~~~



管理客户端线程

~~~java
package com.example.demo.kuangstudy.hsp.qq.service;

import java.util.HashMap;

//该类是管理客户端连接到 服务端的线程的类
public class ManageClientConnectServerThread {
    private static HashMap<String,ClientConnectServerThread> hm = new HashMap();

    //将某个线程加入到集合中
    public static void addClientConnectServerThread(String user,ClientConnectServerThread clientConnectServerThread){
        hm.put(user,clientConnectServerThread);
    }

    //将某个线程从集合中取出
    public static ClientConnectServerThread getClientConnectServerThread(String user){
        ClientConnectServerThread clientConnectServerThread = hm.get(user);
        return clientConnectServerThread;
    }
}
~~~







### 服务端

#### 实体类

对于实体类 客户端的的实体类服务端也需要一份（因为服务端与客户端可能不在一个服务器上）



#### view类   

可以当做服务端的启动类（用来接收客户端传来的消息，并进行对应的处理之后返回）

~~~java
package com.example.demo.kuangstudy.hsp.qqserver.service;

import com.example.demo.kuangstudy.hsp.qq.dao.Message;
import com.example.demo.kuangstudy.hsp.qq.dao.User;

import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.ArrayList;
import java.util.HashMap;

public class QQServer {
    private ServerSocket ss = null;
    private static HashMap<String,User> validUsers =new HashMap<>();

    static {
        validUsers.put("peng",new User("peng","123"));
        validUsers.put("yu",new User("yu","123"));
        validUsers.put("jie",new User("jie","123"));
        validUsers.put("老师",new User("老师","123"));
        validUsers.put("学生",new User("学生","123"));
    }

    public static boolean isValid(String username,String pwd){//验证是否存在该用户
        boolean isValid = false;
        User user = validUsers.get(username);
        if (user!=null&&user.getPwd().equals(pwd)){
            isValid = true;
        }
        return isValid;
    }

    public QQServer(){
        try {
            ss = new ServerSocket(9999);
            System.out.println("服务端在9999端口监听");

            //开启 推送新闻线程
            new Thread(new SendNews()).start();


            while(true){//一直监听
                //等待连接
                Socket socket = ss.accept();
                //输入流 客户端传过来的
                ObjectInputStream ois = new ObjectInputStream(socket.getInputStream());
                User user = (User) ois.readObject();

                //输出流
                OutputStream os = socket.getOutputStream();
                ObjectOutputStream oos = new ObjectOutputStream(os);
                Message message =new Message();

                //验证用户
                if(isValid(user.getUsername(),user.getPwd())){//如果存在该用户 登入成功 返回messageType为 1
                    message.setMsgType("1");
                    //然后将message返回给客户端
                    oos.writeObject(message);

                    //用户刚登入时 需要到离线消息map查看 是否有该用户的消息 有就传给用户 没有就跳过
                    HashMap<String, ArrayList<Message>> offlineMessageMap = ManageClientThread.getOfflineMessageMap();
                    if (offlineMessageMap.keySet().contains(user.getUsername())){//有该用户的离线消息
                        ArrayList<Message> messageArrayList = offlineMessageMap.get(user.getUsername());//对应用户的离线消息列表
                        for (Message message1 : messageArrayList) {//利用for循环 将list的每个message 都发送给该用户
                            ObjectOutputStream objectOutputStream = new ObjectOutputStream(socket.getOutputStream());
                            objectOutputStream.writeObject(message1);
                        }
                        offlineMessageMap.remove(user.getUsername());//将对应用户的离线消息map清除
                    }

                    //创建一个线程和客户端保持通讯 还是使用的这个socket
                    ServerConnectClientThread scct = new ServerConnectClientThread(user.getUsername(), socket);
                    scct.start();
                    //放入一个集合中
                    ManageClientThread.addServerConnectClientThread(user.getUsername(),scct);
                }else{//登入失败
                    System.out.println("用户："+user.getUsername()+"  密码："+user.getPwd()+"   验证失败！");
                    message.setMsgType("2");
                    oos.writeObject(message);
                    socket.close();

                }

            }
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }finally {
            try {
                ss.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
~~~



#### 服务端线程

~~~java
package com.example.demo.kuangstudy.hsp.qqserver.service;

import com.example.demo.kuangstudy.hsp.qq.dao.Message;
import com.example.demo.kuangstudy.hsp.qq.service.ClientConnectServerThread;
import com.example.demo.kuangstudy.hsp.qq.service.ManageClientConnectServerThread;

import java.io.*;
import java.net.Socket;
import java.util.ArrayList;
import java.util.Date;
import java.util.HashMap;

public class ServerConnectClientThread extends Thread{
    private  String username;
    private Socket socket;

    public Socket getSocket() {
        return socket;
    }

    public ServerConnectClientThread(String username, Socket socket) {
        this.username = username;
        this.socket = socket;
    }

    @Override
    public void run() {
        while(true){
            try {
                System.out.println("服务端与客户端： "+username+" 保持通信，读取数据...");
                //存储离线消息的map
                HashMap<String, ArrayList<Message>> offlineMessageMap = ManageClientThread.getOfflineMessageMap();
                //读数据
                InputStream is = socket.getInputStream();
                ObjectInputStream ois = new ObjectInputStream(is);
                Message message = (Message) ois.readObject();

                //判断获取到的message类型
                if (message.getMsgType().equals("4")){//如果是4 需要返回在线用户列表
                    System.out.println(message.getSender()+"  请求在线用户列表");

                    String onlineUser = ManageClientThread.getOnlineUser();
                    Message message2 = new Message();
                    message2.setMsgType("5");//证明返回的是在线用户列表
                    message2.setContent(onlineUser);
                    message2.setGetter(message.getSender());

                    //返回给客户端
                    OutputStream os = socket.getOutputStream();
                    ObjectOutputStream oos = new ObjectOutputStream(os);
                    oos.writeObject(message2);

                }
                if (message.getMsgType().equals("6")){//如果是6 需要断开连接
                    System.out.println(message.getSender()+" 准备退出通讯系统");
                    //删除集合中对应的线程
                    ManageClientThread.removeSocketFromHm(message.getSender());
//                    HashMap<String, ServerConnectClientThread> hm = ManageClientThread.getHm();
//                    hm.remove(message.getSender());//删除对应的线程
                    socket.close();//关闭socket连接
                    break;//退出while循环

                }
                if (message.getMsgType().equals("3")){//如果是3 正常消息
                    //转发给指定的用户 如果用户已下线 可以存入 离线map中
                    HashMap<String, ServerConnectClientThread> hm = ManageClientThread.getHm();
                    if (hm.keySet().contains(message.getGetter())){
                        //在线就直接发送过去
                        ClientConnectServerThread clientConnectServerThread = ManageClientConnectServerThread.getClientConnectServerThread(message.getGetter());
                        OutputStream os = clientConnectServerThread.getSocket().getOutputStream();
                        ObjectOutputStream oos = new ObjectOutputStream(os);
                        oos.writeObject(message);
                    }else{//用户已下线 可以存入 离线map中
                        //HashMap<String, ArrayList<Message>> offlineMessageMap = ManageClientThread.getOfflineMessageMap();
                        if (offlineMessageMap.keySet().contains(message.getGetter())){//如果离线消息中 已有该用户的离线消息 直接添加到后面 没有则新建一个map
                            offlineMessageMap.get(message.getGetter()).add(message);//添加到已有离线消息列表的后面
                        }
                        //没有 则直接新建一个map 然后将message存入 list在存入map的value
                        ArrayList<Message> messageList =new ArrayList<>();
                        messageList.add(message);
                        offlineMessageMap.put(message.getGetter(),messageList);
                    }

                }
                if (message.getMsgType().equals("7")){//如果是7 群发消息
                    //转发给所有用户
                    HashMap<String, ServerConnectClientThread> hm = ManageClientThread.getHm();
                    for (String key : hm.keySet()) {
                        if(key!=message.getSender()){
                            ServerConnectClientThread serverConnectClientThread = ManageClientThread.getServerConnectClientThread(key);
                            Socket socket = serverConnectClientThread.getSocket();
                            ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());
                            oos.writeObject(message);
                        }
                    }
                }
                if (message.getMsgType().equals("8")){//如果是8 文件消息
                    //转发给指定的用户 如果用户已下线 可以存入 离线map中
                    HashMap<String, ServerConnectClientThread> hm = ManageClientThread.getHm();
                    if (hm.keySet().contains(message.getGetter())){
                    ClientConnectServerThread clientConnectServerThread = ManageClientConnectServerThread.getClientConnectServerThread(message.getGetter());
                    Socket socket = clientConnectServerThread.getSocket();
                    ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());
                    oos.writeObject(message);
                    }else{//用户已下线 可以存入 离线map中
                        //HashMap<String, ArrayList<Message>> offlineMessageMap = ManageClientThread.getOfflineMessageMap();
                        if (offlineMessageMap.keySet().contains(message.getGetter())){//如果离线消息中 已有该用户的离线消息 直接添加到后面 没有则新建一个map
                            offlineMessageMap.get(message.getGetter()).add(message);//添加到已有离线消息列表的后面
                        }
                        //没有 则直接新建一个map 然后将message存入 list在存入map的value
                        ArrayList<Message> messageList =new ArrayList<>();
                        messageList.add(message);
                        offlineMessageMap.put(message.getGetter(),messageList);
                    }
                }


            } catch (IOException | ClassNotFoundException e) {
                e.printStackTrace();
            }
        }
    }
}
~~~



#### 管理服务端线程和离线消息

~~~java
package com.example.demo.kuangstudy.hsp.qqserver.service;

import com.example.demo.kuangstudy.hsp.qq.dao.Message;
import com.example.demo.kuangstudy.hsp.qq.service.ClientConnectServerThread;

import java.util.ArrayList;
import java.util.HashMap;

public class ManageClientThread {
    private static HashMap<String,ServerConnectClientThread> hm =new HashMap();//服务端运行中的线程 用一个map存放

    public static HashMap<String, ServerConnectClientThread> getHm() {
        return hm;
    }

    public static HashMap<String, ArrayList<Message>> getOfflineMessageMap() {//服务端中离线消息 利用一个map存放
        return OfflineMessageMap;
    }

    private static HashMap<String, ArrayList<Message>> OfflineMessageMap =new HashMap<>();//创建一个map用来存放 离线的信息，当对方用户登入之后就将对应的离线文件发送过去

    public static void removeSocketFromHm(String username){
        hm.remove(username);
    }

    //将某个线程加入到集合中
    public static void addServerConnectClientThread(String username, ServerConnectClientThread serverConnectClientThread){
        hm.put(username,serverConnectClientThread);
    }

    //将某个线程从集合中取出
    public static ServerConnectClientThread getServerConnectClientThread(String username){
        ServerConnectClientThread serverConnectClientThread = hm.get(username);
        return serverConnectClientThread;
    }

    public static String getOnlineUser(){//得到所有在线用户
        String result ="";
        for (String name : hm.keySet()) {
            result = result+" "+name;
        }
        return result;
    }
}
~~~



#### 服务端推送新闻线程

~~~java
package com.example.demo.kuangstudy.hsp.qqserver.service;

import com.example.demo.kuangstudy.hsp.qq.dao.Message;

import java.io.IOException;
import java.io.ObjectOutputStream;
import java.net.Socket;
import java.util.Date;
import java.util.HashMap;
import java.util.Scanner;

public class SendNews extends Thread{
    @Override
    public void run() {
        while(true){
            System.out.println("可以输入信息进行服务器推送~ 如果想退出推送服务可以输入“exit” 退出");
            Scanner scanner =new Scanner(System.in);
            String news = scanner.nextLine();
            if (news.equals("exit")){
                break;
            }
            Message message = new Message();
            message.setContent(news);
            message.setMsgType("7");
            message.setSender("服务器推送");
            message.setSendTime(new Date().toString());

            //得到所有在线用户的socket 并将推送消息发送过去
            HashMap<String, ServerConnectClientThread> hm = ManageClientThread.getHm();
            for (String key : hm.keySet()) {
                try {
                    Socket socket = ManageClientThread.getServerConnectClientThread(key).getSocket();
                    ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream());
                    oos.writeObject(message);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }


    }
}
~~~





### 启动

服务端

~~~java
package com.example.demo.kuangstudy.hsp.qqApplication;

import com.example.demo.kuangstudy.hsp.qqserver.service.QQServer;

public class QQServerApplication {
    public static void main(String[] args) {
        new QQServer();
    }
}
~~~



客户端

~~~java
package com.example.demo.kuangstudy.hsp.qqApplication;

import com.example.demo.kuangstudy.hsp.qq.view.QQView;

public class QQViewApplication {
    public static void main(String[] args) {
        new QQView().MainMenu();
    }
}
~~~



优化：

​		现在离线消息只能支持，私聊和文件发送， 可以对于群发消息和系统推送也加上离线消息（实现思路：可以在离线消息的map中存入一个单独的kv 用来存放所有的群发和推送，当每个用户登入都必须从其中取出消息）

​		对于消息类型的表示，可以使用一个枚举或者接口定义一个静态变量去管理，比如

~~~java
package com.example.demo.kuangstudy.hsp.qq.dao;
//	1 表示登入成功  2 表示登入失败  3 普通信息包  4 要求返回的在线用户信息列表  5 返回的在线用户列表  6 客户端请求退出 7 群发消息  8 文件消息
public interface MsgType {
    static String LOGIN_SUCCESS = "1";
    static String LOGIN_FAIL = "2";
    static String MESSAGE_NORMAL = "3";
    static String CLIENT_USER = "4";
    static String SERVER_USER = "5";
    static String CLIENT_EXIT = "6";
    static String MESSAGE_GROUP = "7";
    static String MESSAGE_FILE = "8";
}
~~~





