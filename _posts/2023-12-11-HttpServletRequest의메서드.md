---
title : HttpServletRequest의 메서드
date : 2023-12-11 23:00 +09:00
categories : [Web, Spring]
tags : [Spring, Web, Java, 패스트캠퍼스]

---

# 1. HttpServletRequest의 메서드

## 1. getPrameter(Parameter Name)

→ 매개변수로 받은 Parameter의 value를 return함

```java
request.getParameter("이름")
```

## 2. getPrameterNames()

→ name만 return

```java
Enumeration(==Iterater) enum = request.getParameterNames();
```

## 3. getPrameterMap()

→ map 형태로 return  

```java
Map paraMap = request.getPrameterMap();

//ex) key value
//	"year" "2023"  
```

## 4. getPrameterValues()

→ name이 모두 같은경우 ex) ?**year**=2021&**year**=2022&**year**=2023

```java
String[] yearArr = request.getParameterValues("year");
```

