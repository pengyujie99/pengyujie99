---
layout:     post
title:      "Java的一些方法"
subtitle:   "方法"
date:       2021-11-06 12:00:00
author:     "Pengyujie"
header-img: "img/tag-bg.jpg"
tags:
    - 方法
    - Java
---



数组对象化

```java
List<JSONParam> list = Arrays.asList(params);//将数组转为 list然后通过 parm将其对象化
Map<String, String> paramMap = new HashMap<String, String>();
for (JSONParam param : list) {
    if(param==null){
        continue;
    }
    paramMap.put(param.getName(), param.getValue());
}
```



一个对象 输出打印(利用反射)

~~~java
Class<? extends YunyingManagerBaseBean> aClass = updateOneById.getClass();
		for (Field field : aClass.getDeclaredFields()) {
			field.setAccessible(true); //设置为true可以直接通过 取消安全检测 反射调优
			String fieldName = field.getName();
			Object value = null;
			try {
				value = field.get(updateOneById);
				System.out.println(fieldName+":"+value);
			} catch (IllegalAccessException e) {
				e.printStackTrace();
			}
		}
~~~

