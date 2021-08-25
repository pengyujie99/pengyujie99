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



## 支付接口环境

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



ApiUtil工具类(sdk中的方法)

~~~java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package com.saobei.open.sdk.util;

import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import com.saobei.open.sdk.model.requst.trade.SaobeiWapPayRequest;
import com.saobei.open.sdk.model.requst.trade.preauth.SaobeiPreAuthQrH5Request;
import java.net.URLEncoder;
import java.util.Iterator;
import java.util.TreeMap;

public class ApiUtil {
    public ApiUtil() {
    }

    public static <T> JSONObject convertRequest(T t, String key, String keyname) {//请求参数 秘钥 秘钥名称（access_token）
        JSONObject obj = JSON.parseObject(JSON.toJSONString(t));
        String key_sign = SignBeanUtil.FilterNullSign(t, keyname, key, "key_sign", "utf-8");
        obj.put("key_sign", key_sign);
        return obj;
    }
}

~~~



~~~java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package com.saobei.open.sdk.util;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class SignBeanUtil {
    public SignBeanUtil() {
    }

    public static SignBeanUtil.Sign SignTool() {
        return new SignBeanUtil.Sign((SignBeanUtil.Sign)null);
    }

    public static String FilterNullSign(Object object, String keyname, String keyvalue, String Filter, String charset) {
        SignBeanUtil.Sign signContent = SignTool();
        String sign = null;

        try {
            Field[] fields = null;
            fields = getBeanFields(object.getClass(), fields);

            for(int i = 0; i < fields.length; ++i) {
                Field f = fields[i];
                String name = f.getName();
                f.setAccessible(true);
                Object valueObject = f.get(object);
                if (valueObject != null && (Filter == null || Filter.equals("") || !Filter.equals(name))) {
                    String value = valueObject.toString();
                    if (value != null) {
                        signContent.putParam(name, value);
                    }
                }
            }

            signContent.putLastParam(keyname, keyvalue);
            String parm = signContent.getSignStr();
            sign = MD5Util.MD5Encode(parm, charset);
        } catch (Exception var13) {
            var13.printStackTrace();
        }

        return sign;
    }

    public static Field[] getBeanFields(Class cls, Field[] fs) {
        fs = (Field[])ArrayUtils.addAll(fs, cls.getDeclaredFields());
        if (cls.getSuperclass() != null) {
            Class clsSup = cls.getSuperclass();
            fs = getBeanFields(clsSup, fs);
        }

        return fs;
    }

    public static class Sign {
        private Map<String, String> map;
        private String lastStr;

        private Sign() {
            this.lastStr = "";
            this.map = new HashMap();
        }

        public SignBeanUtil.Sign putParam(String key, String value) {
            this.map.put(key, value);
            return this;
        }

        public SignBeanUtil.Sign putParamFilter(String key, String value) {
            if (!ApiUtil.isEmpty(value)) {
                this.map.put(key, value);
            }

            return this;
        }

        public SignBeanUtil.Sign putParamFilterNull(String key, String value) {
            if (value != null) {
                this.map.put(key, value);
            }

            return this;
        }

        public SignBeanUtil.Sign putLastParam(String key, String value) {
            this.lastStr = "&" + key + "=" + value;
            return this;
        }

        public String getParam(String key) {
            return (String)this.map.get(key);
        }

        public String getSignStr() {
            List<String> keys = new ArrayList(this.map.keySet());
            Collections.sort(keys);
            String prestr = "";

            for(int i = 0; i < keys.size(); ++i) {
                String key = (String)keys.get(i);
                String value = (String)this.map.get(key);
                if (i == keys.size() - 1) {
                    prestr = prestr + key + "=" + value;
                } else {
                    prestr = prestr + key + "=" + value + "&";
                }
            }

            return prestr + this.lastStr;
        }
    }
}
~~~





## 付款码支付 

自己写的一个demo

main方法

~~~java
package com.example.demo.barcodepay;

import com.example.demo.barcodepay.dao.Barcodepay;
import com.example.demo.barcodepay.service.PayPost;
import com.example.demo.barcodepay.service.impl.PayPostImpl;

public class BarcodePay {
    public static void main(String[] args) {

        Barcodepay barcodepay = new Barcodepay();

        barcodepay.setPay_ver("100");
        barcodepay.setPay_type("010");
        barcodepay.setService_id("010");
        barcodepay.setMerchant_no("816107297000001");
        barcodepay.setTerminal_id("33335474");
        barcodepay.setTerminal_trace("123456");
        barcodepay.setTerminal_time("20210812120000");
        barcodepay.setAuth_no("102156498523457625");
        barcodepay.setTotal_fee("1");

        String result;
        PayPost payPost = new PayPostImpl();
        result = payPost.post("http://test.lcsw.cn:8045/lcsw/pay/100/barcodepay", barcodepay,"access_token=bc4a32bc9a7845aa8cd919c424dbbf28");
        System.out.println(result);
    }
}

~~~



dao层：

~~~java
package com.example.demo.barcodepay.dao;

import lombok.Data;

@Data
public class Barcodepay {

    private String pay_ver;
    private String pay_type;
    private String service_id;
    private String merchant_no;
    private String terminal_id;
    private String terminal_trace;
    private String terminal_time;
    private String auth_no;
    private String total_fee;
    private String sub_appid;
    private String order_body;
    private String attach;
    private String goods_detail;
    private String goods_tag;

}

~~~



service层：

接口

~~~java
package com.example.demo.barcodepay.service;

public interface PayPost<K> {
    String post(String url, K barcodepay, String str);
}

~~~

实现：

~~~java
package com.example.demo.barcodepay.service.impl;

import com.alibaba.fastjson.JSONObject;
import com.example.demo.MD.MDTest;
import com.example.demo.MD.service.HcPost;
import com.example.demo.MD.service.HcPostImpl;
import com.example.demo.barcodepay.service.PayPost;
import lombok.extern.slf4j.Slf4j;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

@Slf4j
public class PayPostImpl<K> implements PayPost<K>{
    @Override
    public String post(String url, K barcodepay, String str) {
        MDTest mdTest = new MDTest();
        //字典序 将数据保存到一个list中，然后Collections.sort(list) 此时list中已经字典序了 接下来将值以url形式排好 赋值给str1
        //文档序就是不进行Collections.sort(list)
        //ArrayList<String> list = new ArrayList<>();
        ArrayList<String> list1 = new ArrayList<>();
        Class<?> clazz1 = barcodepay.getClass();
        for (Field field : clazz1.getDeclaredFields()) {
            field.setAccessible(true);
            String fieldName = field.getName();
            Object value = null;
            try {
                value = field.get(barcodepay);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
            if((fieldName!="attach")&&(fieldName!="order_body")
                    &&(fieldName!="sub_appid")&&(fieldName!="goods_detail")
                    &&(fieldName!="goods_tag")){//这些非必传
                list1.add(fieldName+"="+value);
            }
        }


        log.info("==========================");
        //log.info("list:"+list.toString());
        log.info("需要签名的参数:"+list1.toString());

        //Collections.sort(list1);//如果是字典序 就需要加上这段代码
        String str1=null;
        //将值以url形式排序
        for (int i = 0; i < list1.size(); i++) {
            if(i==0){
                str1 = list1.get(i);
            }else{
                str1 = str1+"&"+ list1.get(i);
            }
        }


        String str3="&"+str;
        log.info("签名前的拼接字符串:"+str3);
        //MD5加密
        String ks = mdTest.encryption(str1+str3);//调用方法生成MD5加密后 赋值给ks（key_sign）
        log.info("签名:"+ks);

        //Map<String, Object> map = new HashMap<String, Object>();

        Map<String, Object> map1 = new HashMap<String,Object>();
        Class<?> clazz = barcodepay.getClass();
        for (Field field : clazz.getDeclaredFields()) {
            field.setAccessible(true);
            String fieldName = field.getName();
            Object value = null;
            try {
                value = field.get(barcodepay);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
            if((fieldName!="attach")&&(fieldName!="order_body")
                    &&(fieldName!="sub_appid")&&(fieldName!="goods_detail")
                    &&(fieldName!="goods_tag")){
                map1.put(fieldName, value);
            }
        }

        map1.put("key_sign", ks);
        
        Object o = JSONObject.toJSON(map1);
        //log.info("map:"+map.toString());
        log.info("请求参数String格式:"+map1.toString());
        log.info("请求参数JSON格式:"+o);
        log.info("==========================");

        HcPost hcPost =new HcPostImpl();
        String response = hcPost.response(url, o);
        return response;
    }
}
~~~



接口：

~~~java
package com.example.demo.barcodepay.service;

public interface HcPost {
    String response(String url,Object o);
}
~~~



第一种实现（使用HttpClient）：

~~~java
package com.example.demo.barcodepay.service.impl;

import com.example.demo.MD.service.HcPost;
import org.apache.http.HttpEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

public class HcPostImpl implements HcPost {
    @Override
    public String response(String url, Object o) {
        try {
            CloseableHttpClient httpClient = HttpClients.createDefault();//创建一个httpclicent对象
            HttpPost httpPost = new HttpPost(url);//创建一个httppost对象 url为url
            //StringEntity se = new StringEntity(entity, "UTF-8"); //请求体类型
            StringEntity s = new StringEntity(o.toString(), "UTF-8");
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
}
~~~





第二种实现（使用io流）

~~~java
package com.example.demo.invoice.service.impl;

import com.example.demo.MD.service.HcPost;
import lombok.extern.slf4j.Slf4j;
import org.apache.http.HttpEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

import java.io.BufferedOutputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.MalformedURLException;
import java.net.URL;

@Slf4j
//利用流进行传递的请求参数 和返回参数
public class HcStreamPostImpl implements HcPost {
    @Override
    public String response(String url, Object o) {
        String request = o.toString();
        URL url_URL =null;
        try {
            url_URL = new URL(url);
        } catch (MalformedURLException e) {
            e.printStackTrace();
        }
        System.out.println("请求参数:" + request);//查看发送请求参数
        String resp = null;
        String content_type = null;
        try {
            HttpURLConnection connection = (HttpURLConnection) url_URL.openConnection();
            connection.setDoInput(true);
            connection.setDoOutput(true);
            connection.setRequestMethod("POST");
            connection.setUseCaches(false);
            connection.setInstanceFollowRedirects(true);
            connection.setRequestProperty("Content-Type", "application/json");
            connection.connect();
            //传过去的消息
            try (BufferedOutputStream bos = new BufferedOutputStream(connection.getOutputStream())) {
                bos.write(request.getBytes("UTF-8"));
                bos.flush();
            }
            //传回来的消息
            try (InputStreamReader reader = new InputStreamReader(connection.getInputStream(), "UTF-8")) {
                char[] buf = new char[1024];
                StringBuffer buffer = new StringBuffer();
                int count = 0;
                while ((count = reader.read(buf)) != -1) {
                    buffer.append(buf, 0, count);
                }
                resp = buffer.toString();
                content_type = connection.getContentType();
            }
            connection.disconnect();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            //response = JSONObject.fromObject(resp);
        }
        System.out.println("响应类型：" + content_type);
        System.out.println("响应参数：" + resp);
        return resp;
    }

}
~~~







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

<img src="/img/notes/HTTPClient/2.png">



### POST

```java
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

<img src="/img/notes/HTTPClient/3.png">





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





## 付款码支付 



自己写的一个demo

main方法

~~~java
package com.example.demo.barcodepay;

import com.example.demo.barcodepay.dao.Barcodepay;
import com.example.demo.barcodepay.service.PayPost;
import com.example.demo.barcodepay.service.impl.PayPostImpl;

public class BarcodePay {
    public static void main(String[] args) {

        Barcodepay barcodepay = new Barcodepay();

        barcodepay.setPay_ver("100");
        barcodepay.setPay_type("010");
        barcodepay.setService_id("010");
        barcodepay.setMerchant_no("816107297000001");
        barcodepay.setTerminal_id("33335474");
        barcodepay.setTerminal_trace("123456");
        barcodepay.setTerminal_time("20210812120000");
        barcodepay.setAuth_no("102156498523457625");
        barcodepay.setTotal_fee("1");

        String result;
        PayPost payPost = new PayPostImpl();
        result = payPost.post("http://test.lcsw.cn:8045/lcsw/pay/100/barcodepay", barcodepay,"access_token=bc4a32bc9a7845aa8cd919c424dbbf28");
        System.out.println(result);
    }
}

~~~



dao层：

~~~java
package com.example.demo.barcodepay.dao;

import lombok.Data;

@Data
public class Barcodepay {

    private String pay_ver;
    private String pay_type;
    private String service_id;
    private String merchant_no;
    private String terminal_id;
    private String terminal_trace;
    private String terminal_time;
    private String auth_no;
    private String total_fee;
    private String sub_appid;
    private String order_body;
    private String attach;
    private String goods_detail;
    private String goods_tag;

}

~~~



service层：

接口

~~~java
package com.example.demo.barcodepay.service;

public interface PayPost<K> {
    String post(String url, K barcodepay, String str);
}

~~~

实现：

~~~java
package com.example.demo.barcodepay.service.impl;

import com.alibaba.fastjson.JSONObject;
import com.example.demo.MD.MDTest;
import com.example.demo.MD.service.HcPost;
import com.example.demo.MD.service.HcPostImpl;
import com.example.demo.barcodepay.service.PayPost;
import lombok.extern.slf4j.Slf4j;

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

@Slf4j
public class PayPostImpl<K> implements PayPost<K>{
    @Override
    public String post(String url, K barcodepay, String str) {
        MDTest mdTest = new MDTest();
        //字典序 将数据保存到一个list中，然后Collections.sort(list) 此时list中已经字典序了 接下来将值以url形式排好 赋值给str1
        //文档序就是不进行Collections.sort(list)
        //ArrayList<String> list = new ArrayList<>();
        ArrayList<String> list1 = new ArrayList<>();
        Class<?> clazz1 = barcodepay.getClass();
        for (Field field : clazz1.getDeclaredFields()) {
            field.setAccessible(true);
            String fieldName = field.getName();
            Object value = null;
            try {
                value = field.get(barcodepay);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
            if((fieldName!="attach")&&(fieldName!="order_body")
                    &&(fieldName!="sub_appid")&&(fieldName!="goods_detail")
                    &&(fieldName!="goods_tag")){//这些非必传
                list1.add(fieldName+"="+value);
            }
        }


        log.info("==========================");
        //log.info("list:"+list.toString());
        log.info("list1:"+list1.toString());
        log.info("==========================");

        //Collections.sort(list1);//如果是字典序 就需要加上这段代码
        String str1=null;
        //将值以url形式排序
        for (int i = 0; i < list1.size(); i++) {
            if(i==0){
                str1 = list1.get(i);
            }else{
                str1 = str1+"&"+ list1.get(i);
            }
        }


        System.out.println(str1);
        String str3="&"+str;
        //MD5加密
        String ks = mdTest.encryption(str1+str3);//调用方法生成MD5加密后 赋值给ks（key_sign）
        System.out.println("ks:"+ks);

        //Map<String, Object> map = new HashMap<String, Object>();

        Map<String, Object> map1 = new HashMap<String,Object>();
        Class<?> clazz = barcodepay.getClass();
        for (Field field : clazz.getDeclaredFields()) {
            field.setAccessible(true);
            String fieldName = field.getName();
            Object value = null;
            try {
                value = field.get(barcodepay);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
            if((fieldName!="attach")&&(fieldName!="order_body")
                    &&(fieldName!="sub_appid")&&(fieldName!="goods_detail")
                    &&(fieldName!="goods_tag")){
                map1.put(fieldName, value);
            }
        }

        map1.put("key_sign", ks);
        
        Object o = JSONObject.toJSON(map1);
        log.info("==========================");
        //log.info("map:"+map.toString());
        log.info("请求参数String格式:"+map1.toString());
        log.info("请求参数JSON格式:"+o);
        log.info("==========================");

        HcPost hcPost =new HcPostImpl();
        String response = hcPost.response(url, o);
        return response;
    }
}
~~~



接口：

~~~java
package com.example.demo.barcodepay.service;

public interface HcPost {
    String response(String url,Object o);
}
~~~



实现：

~~~java
package com.example.demo.barcodepay.service.impl;

import com.example.demo.MD.service.HcPost;
import org.apache.http.HttpEntity;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

public class HcPostImpl implements HcPost {
    @Override
    public String response(String url, Object o) {
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
}
~~~

