---
layout:     post
title:      "ElasticSearch"
subtitle:   "es搜索分析引擎"
date:       2021-05-16
author:     "Pengyujie"
header-img: "img/tag-bg.jpg"
tags:
    - Java
    - SpringBoot
    - 中间件
---

>ElasticSearch（es）是一个可以处理大量数据的分布式的搜索分析引擎，它是面向文档，一切都JSON。



## 安装使用

### 安装ElasticSearch

  1、官网下载  

  2、压缩包移动到你需要放置的目录  

  3、解压之后即可使用

进入bin下运行elasticsearch.bat。
访问localhost：9200 成功即安装成功。


### 解决可视化问题  安装 head

再配置文件中进行配置跨域，然后重启。
![image-20230410161747401](2021-05-16-ElasticSearch.assets/image-20230410161747401.png)


到gitee下载elasticsearch head插件可视化 进入head文件
```linux
cnpm install 
npm run start
```
![image-20230410161757299](2021-05-16-ElasticSearch.assets/image-20230410161757299.png)
访问localhost：9100 去访问9200 如下则成功！！
![image-20230410161804672](2021-05-16-ElasticSearch.assets/image-20230410161804672.png)

### 安装kibanna

1、官网下载，解压即可使用，
其中汉化可以在yml里面配置
![image-20230410161811654](2021-05-16-ElasticSearch.assets/image-20230410161811654.png)
2、运行
3、访问
![image-20230410161817375](2021-05-16-ElasticSearch.assets/image-20230410161817375.png)
可以在这里进行索引的管理。

### es与mysql

现在可以将es的索引理解为一个"数据库"
```sql
索引	 数据库
types	 表
document 行
fields   字段
```

## RestFul 风格

Put更新/Get查询/Delete删除/Post新增

Put更新
```java
post /索引名/类型名/文档id/_update
{请求体}
put /索引名/类型名/文档id
```
Get查询
```java
get /索引名/类型名/文档id 通过文档id查询
get/索引名/类型名/文档id/_search  查询所有数据
post/索引名/类型名/_search?name:"zhangsan"  查询name为张三的数据
```
Delete删除
```java
delete /索引名/类型名/文档id
```
Post新增
```java
post /索引名/类型名
put /索引名/类型名/文档id
{请求体}
```

## 仿京东搜索与springboot整合

### 一、创建springboot项目时选择es，额外导入依赖。
```xml
 <!-- 阿里fastjson包JSON转换-->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.47</version>
        </dependency>
<!-- jsoup包 解析网页 爬取数据-->
        <!-- https://mvnrepository.com/artifact/org.jsoup/jsoup -->
        <dependency>
            <groupId>org.jsoup</groupId>
            <artifactId>jsoup</artifactId>
            <version>1.13.1</version>
        </dependency>
```
### 二、编写实体类和HtmlParseUtil工具类。
```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Content {
    private String title;
    private String img;
    private String price;
}
```
```java
public class HtmlParseUtil {
    public List<Content> parseJD(String keywords) throws IOException {
        String url = "https://search.jd.com/Search?keyword="+keywords;
        //解析网页 jsoup返回的Document对象就是浏览器的Document对象
        Document document = Jsoup.parse(new URL(url),30000);
        //通过id获取所有信息
        Element element = document.getElementById("J_goodsList");
        //获取所有的li元素
        Elements elements = element.getElementsByTag("li");

        //定义一个Content的数组
        ArrayList<Content>  goodList =  new ArrayList<>();

        //遍历li
        for (Element el : elements) {
            String img =el.getElementsByTag("img").eq(0).attr("data-lazy-img");
            String price =el.getElementsByClass("p-price").eq(0).text();
            String title =el.getElementsByClass("p-name").eq(0).text();
            Content content = new Content();
            content.setImg(img);
            content.setPrice(price);
            content.setTitle(title);
            goodList.add(content);
        }
        return goodList;
    }
}
```

### 三、编写业务Service和Controller将爬取的数据以实体类形式存储到索引以及查询高亮显示。
```java
@Service
public class ContentService {
    @Autowired
    private RestHighLevelClient restHighLevelClient;
    //将数据放入es索引中
    public Boolean parseContent(String keywords) throws IOException {
        List<Content> contents= new HtmlParseUtil().parseJD(keywords);
        BulkRequest bulkRequest = new BulkRequest();
        bulkRequest.timeout("2m");
        for (int i = 0; i < contents.size(); i++) {
            bulkRequest.add(new IndexRequest("jd_goods")
            .source(JSON.toJSONString(contents.get(i)), XContentType.JSON));
        }
        BulkResponse bulk = restHighLevelClient.bulk(bulkRequest, RequestOptions.DEFAULT);
        return  bulk.hasFailures();
    }

    //实现 条件搜索高亮显示
    public List<Map<String,Object>> searchHighLightPage(String keyword, int pageNo, int pageSize) throws IOException {
        if(pageNo<=1){
            pageNo =1;
        }
        //搜索
        SearchRequest searchRequest = new SearchRequest("jd_goods");
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //分页
        sourceBuilder.from(pageNo);
        sourceBuilder.size(pageSize);
        //高亮设置
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.field("title");
        //highlightBuilder.requireFieldMatch(false);//多个高亮显示
        highlightBuilder.preTags("<span style='color:red'>");
        highlightBuilder.postTags("</span>");
        sourceBuilder.highlighter(highlightBuilder);
        //match匹配
        MatchQueryBuilder matchQueryBuilder = new MatchQueryBuilder("title",keyword);
        sourceBuilder.query(matchQueryBuilder);
        sourceBuilder.timeout(new TimeValue(60, TimeUnit.SECONDS));
        //执行搜索
        searchRequest.source(sourceBuilder);
        SearchResponse searchResponse = restHighLevelClient.search(searchRequest, RequestOptions.DEFAULT);
        //解析结果
        ArrayList<Map<String,Object>> list = new ArrayList<>();
        for (SearchHit hit : searchResponse.getHits().getHits()) {//所有的数据在hits中
            //解析高亮字段
            Map<String, HighlightField> highlightFields = hit.getHighlightFields();
            HighlightField title = highlightFields.get("title");
            Map<String, Object> sourceAsMap = hit.getSourceAsMap();
            if(title!=null){
                Text[] fragments = title.fragments();
                String n_title = "";
                for (Text text : fragments) {
                    n_title +=text;
                }
                sourceAsMap.put("title",n_title);
            }
            list.add(sourceAsMap);
        }
        return list;
    }
}
```
```java
@RestController
public class ContentController {

    @Autowired
    private ContentService contentService;
    @GetMapping("/parse/{keyword}")
    public Boolean parse(@PathVariable String keyword) throws IOException {
        Boolean aBoolean = contentService.parseContent(keyword);//将数据存入 es索引
        return aBoolean;
    }

    @GetMapping("/searchHigh/{keyword}/{pageNo}/{pageSize}")
    public List<Map<String,Object>> searchHigh(@PathVariable("keyword") String keyword,
                                           @PathVariable("pageNo") int pageNo,
                                           @PathVariable("pageSize") int pageSize) throws IOException {
        if (pageNo<=1){
            pageNo=1;
        }
        List<Map<String, Object>> maps = contentService.searchHighLightPage(keyword, pageNo, pageSize);
        return maps;
    }
}
```
![image-20230410161828470](2021-05-16-ElasticSearch.assets/image-20230410161828470.png)









## SpringBoot快速整合

首先已经启动了对应的elasticsearch

1.导入对应依赖

pom.xml文件配置

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.0</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.example</groupId>
    <artifactId>elasticsearch</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>elasticsearch</name>
    <description>Demo project for Spring Boot</description>
    <properties>
        <java.version>1.8</java.version>
        <elasticsearch.version>7.17.3</elasticsearch.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!--es-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.47</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
~~~





2.项目代码

~~~yml
spring:
  elasticsearch:
    uris: 127.0.0.1:9200


  #musql
  datasource:
    url: jdbc:mysql://localhost:3306/a_excel?serverTimezone=GMT%2B8&useUnicode=true&characterEncoding=UTF-8
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver

~~~



~~~java
package com.example.elasticsearch.document;

import lombok.AllArgsConstructor;
import lombok.Data;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;
import org.springframework.data.elasticsearch.annotations.FieldType;

@Document(indexName = "product",createIndex = true)
@Data
@AllArgsConstructor
public class Product {
    @Id
    @Field(type = FieldType.Integer,store = true,index = true)
    private Integer id;
    @Field(type = FieldType.Text,store = true,index = true,analyzer = "ik_max_word",searchAnalyzer = "ik_max_word")
    private String productName;
    @Field(type = FieldType.Text,store = true,index = true,analyzer = "ik_max_word",searchAnalyzer = "ik_max_word")
    private String productDesc;

}
~~~



~~~java
package com.example.elasticsearch.repository;

import com.example.elasticsearch.document.Product;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface ProductRepository extends ElasticsearchRepository<Product,Integer> {

}

~~~



3.测试

~~~java
package com.example.elasticsearch;

import com.example.elasticsearch.document.Product;
import com.example.elasticsearch.repository.ProductRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.Optional;

@SpringBootTest
class ElasticsearchApplicationTests {
    @Autowired
    private ProductRepository repository;


    //新建文档
    @Test
    public void addDocument(){
        Product product = new Product(1, "小米手机", "暖手宝");
        repository.save(product);
    }
    //更新文档
    @Test
    public void updateDocument(){
        Product product = new Product(1, "小米手机2", "暖手宝2");
        repository.save(product);
    }
    //查询所有文档
    @Test
    public void findAllDocument(){
        Iterable<Product> all = repository.findAll();
        for (Product product : all) {
            System.out.println(product);
        }
    }
    //根据id查询文档
    @Test
    public void findDocumentById(){
        Optional<Product> product = repository.findById(1);
        System.out.println(product.get());
    }
    //根据id删除文档
    @Test
    public void deleteDocument(){
        repository.deleteById(1);
    }
}
~~~



