title: 带泛型的json转换
author: Dylan
tags:
  - json
  - 泛型
categories:
  - 编程语言
date: 2018-09-14 10:35:00
---
### 前言
jackson的类库很多，有google的Gson，有马云的fastJson,还有spring boot用的fasterxml，等等，还有很多。这里使用google的Gson，介绍带有泛型的bean转换。
1. 普通转换

```java
//定义一个bean
public class UserResponse{

	private String username;
    
   ...getter and setter 
}

@Test
public void test(){
	Gson json = new GsonBuilder().serializeNulls().setDateFormat("yyyy-MM-dd HH:mm:ss").create();
	String jsonStr="{\"username\":\"foo\"}";
   UserResponse userResponse = json.fromJson(jsonStr,UserResponse.class);
}
```

2. 带泛型转换

```java
public class Response<T>｛

	private String code;
   private String desc;
   private T response;
   
   ...getter and setter
｝

public class UserResponse{
	private String username;
    
   ...getter and setter 
}

@Test
public void test(){
	Gson json = new GsonBuilder().serializeNulls().setDateFormat("yyyy-MM-dd HH:mm:ss").create();
	String jsonStr="{\"code\": \"100\",\"desc\": \"成功\",\"response\": {\"username\": \"foo\"}}";
   Response<UserResponse> response = json.fromJson(jsonStr,new TypeToken<Response<UserResponse>>(){}.getType());
}
```