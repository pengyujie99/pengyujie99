---
layout:     post
title:      "HttpClient"
subtitle:   "HttpClient访问接口"
date:       2021-06-25
author:     "Pengyujie"
header-img: "img/tag-bg.jpg"
hidden: true
tags:
  - Java
  - HTTP
  - HttpClient

---





> 多应用于对接第三方，和爬虫。

> HttpClient是Apache Jakarta Common下的子项目，用来提供高效的、最新的、功能丰富的支持HTTP协议的客户端编程工具包，并且它支持HTTP协议最新的版本和建议。HttpClient已经应用在很多的项目中。



### 支付接口环境

举例：

```
url
http://test.lcsw.cn:8045/lcsw/pay/100/barcodepay

参数
pay_ver=100&pay_type=020&service_id=010&merchant_no=824205541000001&terminal_id=33333378&terminal_trace=1234567890&terminal_time=20210622120010&auth_no=134679352090241590&total_fee=1

token令牌
access_token=48131d0d4d294dbdbd6cbea6f407ea40

key_sign参数
key_sign=md5(pay_ver=100&pay_type=020&service_id=010&merchant_no=824205541000001&terminal_id=33333378&terminal_trace=1234567890&terminal_time=20210622120010&auth_no=134679352090241590&total_fee=1&access_token=48131d0d4d294dbdbd6cbea6f407ea40);


总结：
访问url根据文档所写，参数为规定参数，其中token的key为accss_token，key_sign参数为前二者拼接起来之后进行MD5加密。
```



注意：POST请求时实际是请求体去进行传参，参数是以JSON格式(如下)进行传输，由map类型转换为json格式。需要将每个参数以map类型进行存储。

```java
{
 "pay_ver":"100",
 "pay_type":"010",
 "service_id":"020",
 "merchant_no":"889521000000001",
 "terminal_id":"10000001",
 "terminal_trace":"1234567890",
 "terminal_time":"20161001120010",
 "key_sign":"bc3e6ac8355bf3a4c2cf9cd00d6065b8"
}
```






### 利用HttpClient实现接口访问

- MDTest（MD5加密 工具类）

```java
package com.example.demo.MD;

import java.io.UnsupportedEncodingException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

public class MDTest {

    public String encryption(String plainText) {//UTF-8编码，32位md5加密转换
        String re_md5 = ""; //用来存储最后UTF-8编码，32位md5加密转换后的数据

        MessageDigest md5 = null;
        try {
            md5 = MessageDigest.getInstance("MD5");
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        try {
            md5.update((plainText).getBytes("UTF-8"));//utf-8编码
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }

        byte b[] = md5.digest();

        int i;
        StringBuffer buf = new StringBuffer("");

        for(int offset=0; offset<b.length; offset++){
            i = b[offset];
            if(i<0){
                i+=256;
            }
            if(i<16){
                buf.append("0");
            }
            buf.append(Integer.toHexString(i));
        }
        re_md5=buf.toString();
        return re_md5;

    }
}

```



POST

```JAVA
package com.example.demo.MD;

import com.alibaba.fastjson.JSONObject;
import org.apache.http.HttpEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

import java.util.HashMap;
import java.util.Map;

public class Post {
    // 发送请求HTTP-POST请求 url:请求地址; entity:json格式请求参数
    public static String post(String url) {//[{},{},{}]
        MDTest mdTest = new MDTest();
        String str1="pay_ver=100&pay_type=010&service_id=010&merchant_no=824205541000001&terminal_id=33333378&terminal_trace=1234567890&terminal_time=20210622120010&auth_no=";
        String str2="134513503683520441";
        String str3="&total_fee=1&access_token=48131d0d4d294dbdbd6cbea6f407ea40";
        String ks = mdTest.encryption(str1+str2+str3);//调用方法生成MD5加密后 赋值给ks（key_sign）

        Map<String, Object> map = new HashMap<String, Object>();

        map.put("pay_ver", "100");
        map.put("pay_type", "010");
        map.put("service_id", "010");
        map.put("merchant_no", "824205541000001");
        map.put("terminal_id", "33333378");
        map.put("terminal_trace", "1234567890");
        map.put("terminal_time", "20210622120010");
        map.put("auth_no", "134513503683520441");
        map.put("total_fee", "1");
        map.put("key_sign", ks);


        Object o = JSONObject.toJSON(map);

        System.out.println(o);


        try {
            CloseableHttpClient httpClient = HttpClients.createDefault();//创建一个httpclicent对象
            HttpPost httpPost = new HttpPost(url);//创建一个httppost对象 url为url
            //StringEntity se = new StringEntity(entity, "UTF-8"); //请求体类型
            StringEntity s = new StringEntity(o.toString());
            s.setContentEncoding("UTF-8");
            s.setContentType("application/json");//发送json数据需要设置contentType设为application/json
            httpPost.setEntity(s); //httppost对象的Entity设为s
            CloseableHttpResponse response = httpClient.execute(httpPost);//响应
            HttpEntity entity1 = response.getEntity();
            String resStr = null;
            if (entity1 != null) {
                resStr = EntityUtils.toString(entity1, "UTF-8");
            }
            httpClient.close();
            response.close();
            return resStr;//打印响应的数据
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "";
    }

    public static void main(String[] args) {
        String result;
        result=post("http://test.lcsw.cn:8045/lcsw/pay/100/barcodepay");          //1.1. 付款码支付
//        result=post("http://test.lcsw.cn:8045/lcsw/pay/100/prepay");              //1.2. 扫码支付（预支付）
//        result=post("http://test.lcsw.cn:8045/lcsw/pay/100/jspay");               //1.3. 公众号预支付（统一下单）
//        result=post("http://test.lcsw.cn:8045/lcsw/pay/100/minipay");             //1.4. 小程序支付接
//        result=post("http://test.lcsw.cn:8045/lcsw/pay/110/facepay");           //1.5. 自助收银支付接口（仅支持微信）
//        result=post("http://test.lcsw.cn:8045/lcsw/pay/110/faceinfo");          //1.6. 自助收银SDK调用凭证获取接口
//        result=post("http://test.lcsw.cn:8045/lcsw/pay/110/authcodetoopenid");  //1.7. 授权码查询 OPENID 接口


        System.out.println(result);
    }
}


```



GET

```JAVA
package com.example.demo.MD;


import org.apache.http.HttpEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;


import java.io.IOException;


public class Get {
    public static void main(String[] args) throws IOException {
        //
        String url = "http://test.lcsw.cn:8045/lcsw/wx/jsapi/authopenid";//地址
        String merchant_no ="merchant_no=824205541000001";//参数1
        String terminal_no ="terminal_no=33333378";//参数2
        String redirect_uri ="redirect_uri=https%3A%2F%2Fwww.example.com%2Fpay.html";//参数3
        MDTest mdTest = new MDTest();
        String token="access_token=48131d0d4d294dbdbd6cbea6f407ea40";
        String encryption = mdTest.encryption(merchant_no+"&" + redirect_uri +"&"+ terminal_no +"&"+ token);
        String key_sign =encryption;//参数4 key_sign


        String result;
        result=get(url+"?"+merchant_no+"&"+terminal_no+"&"+redirect_uri+"&"+key_sign);
        System.out.println(result);


    }
    public static  String get(String url) {
        try {
            CloseableHttpClient httpClient = HttpClients.createDefault();//创建一个httpclicent对象
            HttpGet httpGet = new HttpGet(url);
            CloseableHttpResponse response = httpClient.execute(httpGet);
            HttpEntity entity = response.getEntity();
            String resStr =null;
            if(entity != null){
                resStr= EntityUtils.toString(entity, "UTF-8");


            }
            httpClient.close();
            response.close();
            System.out.println("----------------");
            return resStr;//打印响应的数据


        }catch (IOException e) {
            e.printStackTrace();
        }


        return "";//打印null
    }
}
```



- post方式 签名字符串,拼装所有传递参数（字典序）+令牌，UTF-8编码，32位md5加密转换 

```java
package com.example.demo.MD;

import com.alibaba.fastjson.JSONObject;
import org.apache.http.HttpEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

import java.util.*;

public class PostQR {
    // 发送请求HTTP-POST请求 url:请求地址; entity:json格式请求参数
    public static String post(String url) {//[{},{},{}]
        MDTest mdTest = new MDTest();

        String pay_ver="pay_ver=110";
        String pay_type="pay_type=000";
        String service_id="service_id=016";
        String merchant_no="merchant_no=824205541000001";
        String terminal_id="terminal_id=33333378";
        String terminal_trace="terminal_trace=1234567890";
        String terminal_time="terminal_time=20210622120010";
        String total_fee="total_fee=1";

        //字典序 将数据保存到一个list中，然后Collections.sort(list) 此时list中已经字典序了 接下来将值以url形式排好 赋值给str1
        ArrayList<String> list = new ArrayList<>();
        list.add(pay_ver);
        list.add(pay_type);
        list.add(service_id);
        list.add(merchant_no);
        list.add(terminal_id);
        list.add(terminal_trace);
        list.add(terminal_time);
        list.add(total_fee);

        Collections.sort(list);
        String str1=null;
        for (int i = 0; i < list.size(); i++) {
            if(i==0){
                str1 = list.get(i);
            }else{
                str1 = str1+"&"+ list.get(i);
            }
        }


        String str3="&access_token=48131d0d4d294dbdbd6cbea6f407ea40";


        //MD5加密
        String ks = mdTest.encryption(str1+str3);//调用方法生成MD5加密后 赋值给ks（key_sign）

        Map<String, Object> map = new HashMap<String, Object>();

        map.put("pay_ver", "110");
        map.put("pay_type", "000");
        map.put("service_id", "016");
        map.put("merchant_no", "824205541000001");
        map.put("terminal_id", "33333378");
        map.put("terminal_trace", "1234567890");
        map.put("terminal_time", "20210622120010");
        //map.put("auth_no", "134570358617084359");
        map.put("total_fee", "1");
        map.put("key_sign", ks);


        Object o = JSONObject.toJSON(map);

        System.out.println(o);


        try {
            CloseableHttpClient httpClient = HttpClients.createDefault();//创建一个httpclicent对象
            HttpPost httpPost = new HttpPost(url);//创建一个httppost对象 url为url
            //StringEntity se = new StringEntity(entity, "UTF-8"); //请求体类型
            StringEntity s = new StringEntity(o.toString());
            s.setContentEncoding("UTF-8");
            s.setContentType("application/json");//发送json数据需要设置contentType设为application/json
            httpPost.setEntity(s); //httppost对象的Entity设为s
            CloseableHttpResponse response = httpClient.execute(httpPost);//响应
            HttpEntity entity1 = response.getEntity();
            String resStr = null;
            if (entity1 != null) {
                resStr = EntityUtils.toString(entity1, "UTF-8");
            }
            httpClient.close();
            response.close();
            return resStr;//打印响应的数据
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "";
    }

    public static void main(String[] args) {
        String result;
        result=post("http://test.lcsw.cn:8045/lcsw/pay/110/qrpay");              //2. 聚合码支付接口
        System.out.println(result);
    }
}

```



```java
{"return_code":"01","return_msg":"聚合码预支付请求成功！","result_code":"01","merchant_name":"范琴","merchant_no":"824205541000001","terminal_id":"33333378","terminal_trace":"1234567890","terminal_time":"20210622120010","total_fee":"1","qr_url":"http://test.lcsw.cn:8045/lcsw/qrurl/9f598qp441","attach":null,"key_sign":"c298b0d07c96eda0ef5a5b04cd28661f"}
```

<img src="/img/notes/HttpClient/1.png">







## 编写简单接口进行GET和POST访问

- Factory   (接口)

```java
package com.example.demo.api;

public interface Factory {
    String tesla();
    String BC();
    String BM();
}
```



- FactoryImpl （实现接口）

```java
package com.example.demo.api.impl;

import com.example.demo.api.Factory;
import org.springframework.stereotype.Service;

@Service
public class FactoryImpl implements Factory{
    @Override
    public String tesla() {
        return "tesla";
    }

    @Override
    public String BC() {
        return "bc";
    }

    @Override
    public String BM() {
        return "bm";
    }
}
```



- CarPojo (pojo)

```java
package com.example.demo.api.pojo;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class CarPojo {
    String name;
    String text;
}
```



- CarController (controller)

```java
package com.example.demo.api.controller;

import com.example.demo.api.Factory;
import com.example.demo.api.pojo.CarPojo;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
public class CarController {
    @Autowired
    private Factory factory;
    @GetMapping("/car/Param")//get方法 http://localhost:8080/car/Param?name=bc
    public String car0(@RequestParam String name){
        if(name.equals("tesla")){
            return "This is a car:-->"+factory.tesla();
        }else if(name.equals("bc")){
            return "This is a car:-->"+factory.BC();
        }else{
            return "This is a car:-->"+factory.BM();
        }
    }

    @GetMapping("/car/Path/{name}")//get方法 http://localhost:8080/car/Path/bc
    public String car1(@PathVariable("name") String name){
        if(name.equals("tesla")){
            return "This is a car:-->"+factory.tesla();
        }else if(name.equals("bc")){
            return "This is a car:-->"+factory.BC();
        }else{
            return "This is a car:-->"+factory.BM();
        }
    }

    @PostMapping("/post")//post方法
    public String car2(@RequestBody CarPojo carPojo){//需要加@RequestBody 否则无法正常获取json数据
        if(carPojo.getName() =="tesla"){
            return "This is a car:-->"+factory.tesla()+"---"+"text:-->"+carPojo.getText() ;
        }else if(carPojo.getName()  =="bc"){
            return "This is a car:-->"+factory.BC()+"---"+"text:-->"+carPojo.getText();
        }else{
            return "This is a car:-->"+factory.BM()+"---"+"text:-->"+carPojo.getText();
        }

    }
}
```

### GET

```java
package com.test.demo.get;

import org.apache.http.HttpEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

import java.io.IOException;

public class Get {
    public static String get(String url) throws IOException {
        CloseableHttpClient httpClient = HttpClients.createDefault();//创建一个CloseableHttpClient对象
        HttpGet httpGet = new HttpGet(url);
        CloseableHttpResponse response = httpClient.execute(httpGet);
        HttpEntity entity = response.getEntity();
        String resStr;
        if(entity!=null){
            resStr = EntityUtils.toString(entity,"UTF-8");
            httpClient.close();
            response.close();
            return resStr;
        }

        return null;
    }

    public static void main(String[] args) throws IOException {
        String result;
        String url = "http://localhost:8080/car/Param";
        String name = "name=bc";

        result = get(url+"?"+name);
        System.out.println(result);
    }
}
```

<img src="/img/notes/HttpClient/2.png">



### POST

```JAVA
package com.test.demo.post;

import com.alibaba.fastjson.JSONObject;
import org.apache.http.HttpEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

import java.util.HashMap;
import java.util.Map;


public class Post {
    // 发送请求HTTP-POST请求 url:请求地址; entity:json格式请求参数
    public static String post(String url) {//[{},{},{}]


        Map<String, Object> map = new HashMap<String, Object>();

        map.put("name", "bc");
        map.put("text", "hello! post");


        Object o = JSONObject.toJSON(map);
        System.out.println(o);


        try {
            CloseableHttpClient httpClient = HttpClients.createDefault();//创建一个httpclicent对象
            HttpPost httpPost = new HttpPost(url);//创建一个httppost对象 url为url
            //StringEntity se = new StringEntity(entity, "UTF-8"); //请求体类型
            StringEntity s = new StringEntity(o.toString());
            s.setContentEncoding("UTF-8");
            s.setContentType("application/json");//发送json数据需要设置contentType设为application/json
            httpPost.setEntity(s); //httppost对象的Entity设为s
            CloseableHttpResponse response = httpClient.execute(httpPost);//响应
            HttpEntity entity1 = response.getEntity();
            String resStr = null;
            if (entity1 != null) {
                resStr = EntityUtils.toString(entity1, "UTF-8");
            }
            httpClient.close();
            response.close();
            return resStr;//打印响应的数据
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "";
    }

    public static void main(String[] args) {
        String result;
        result=post("http://localhost:8080/post");
        System.out.println(result);
    }
}

```

<img src="/img/notes/HttpClient/3.png">





## MD5和字典序

- MD5工具类

```java
package com.example.demo.MD;

import java.io.UnsupportedEncodingException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

public class MDTest {

    public String encryption(String plainText) {//UTF-8编码，32位md5加密转换
        String re_md5 = ""; //用来存储最后UTF-8编码，32位md5加密转换后的数据

        MessageDigest md5 = null;
        try {
            md5 = MessageDigest.getInstance("MD5");
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        try {
            md5.update((plainText).getBytes("UTF-8"));//对plainText数据进行utf-8编码和MD5加密
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }

        byte b[] = md5.digest();

        int i;
        StringBuffer buf = new StringBuffer("");//new一个stringbuffer对象

        for(int offset=0; offset<b.length; offset++){//将数组b中的数据 填充到buf中
            i = b[offset];
            if(i<0){
                i+=256;
            }
            if(i<16){
                buf.append("0");
            }
            buf.append(Integer.toHexString(i));
        }
        re_md5=buf.toString();
        return re_md5;

    }
}

```

- 字典序（只截取了完成字典序和MD5加密的代码   完整代码可以查看上面的HttpClient访问接口字典序的代码）

```java
 //字典序 将数据保存到一个list中，然后Collections.sort(list) 此时list中已经字典序了 接下来将值以url形式排好 赋值给str1
        ArrayList<String> list = new ArrayList<>();
        list.add(pay_ver);
        list.add(pay_type);
        list.add(service_id);
        list.add(merchant_no);
        list.add(terminal_id);
        list.add(terminal_trace);
        list.add(terminal_time);
        list.add(total_fee);

        Collections.sort(list);
        String str1=null;

        //将值以url形式排序
        for (int i = 0; i < list.size(); i++) {
            if(i==0){
                str1 = list.get(i);
            }else{
                str1 = str1+"&"+ list.get(i);
            }
        }

        String str3="&access_token=48131d0d4d294dbdbd6cbea6f407ea40";

        //MD5加密 调用MD5工具类中方法实现
		MDTest mdTest = new MDTest();
        String ks = mdTest.encryption(str1+str3);//调用方法生成MD5加密后 赋值给ks（key_sign）
```



