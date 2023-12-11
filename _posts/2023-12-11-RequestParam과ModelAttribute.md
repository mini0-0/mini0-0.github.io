---
title : RequestParam과 ModelAttribute
date : 2023-12-11 23:00 +09:00
categories : [Web, Spring]
tags : [Spring, Web, Java, 패스트캠퍼스]

---

# 1. @RequestParam

- 요청의 파라미터를 연결할 매개변수에 붙이는 애너테이션

## 예제1

### RequestParamTest.java

```java
package com.fastcampus.ch2;

import java.util.Date;

import javax.servlet.http.HttpServletRequest;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;

@Controller
public class RequestParamTest {
	@RequestMapping("/requestParam")
	public String main(HttpServletRequest request) {
		String year = request.getParameter("year");
//		http://localhost/ch2/requestParam         ---->> year=null
//		http://localhost/ch2/requestParam?year=   ---->> year=""
//		http://localhost/ch2/requestParam?year    ---->> year=""
		System.out.printf("[%s]year=[%s]%n", new Date(), year);
		return "yoil";
	}

	@RequestMapping("/requestParam2")
//	public String main2(@RequestParam(name="year", required=false) String year) {   // 아래와 동일 
	public String main2(String year) {   
//		http://localhost/ch2/requestParam2         ---->> year=null
//		http://localhost/ch2/requestParam2?year    ---->> year=""
		System.out.printf("[%s]year=[%s]%n", new Date(), year);
		return "yoil";
	}

	@RequestMapping("/requestParam3")
//		public String main3(@RequestParam(name="year", required=true) String year) {   // 아래와 동일 
		public String main3(@RequestParam String year) {   
//		http://localhost/ch2/requestParam3         ---->> year=null   400 Bad Request. required=true라서 
//		http://localhost/ch2/requestParam3?year    ---->> year=""
		System.out.printf("[%s]year=[%s]%n", new Date(), year);
		return "yoil";	
	}

	@RequestMapping("/requestParam4")
	public String main4(@RequestParam(required=false) String year) {   
//		http://localhost/ch2/requestParam4         ---->> year=null 
//		http://localhost/ch2/requestParam4?year    ---->> year=""   
		System.out.printf("[%s]year=[%s]%n", new Date(), year);
		return "yoil";
	}

	@RequestMapping("/requestParam5")
	public String main5(@RequestParam(required=false, defaultValue="1") String year) {   
//		http://localhost/ch2/requestParam5         ---->> year=1   
//		http://localhost/ch2/requestParam5?year    ---->> year=1   
		System.out.printf("[%s]year=[%s]%n", new Date(), year);
		return "yoil";
	}
	
// =======================================================================
	
	@RequestMapping("/requestParam6") 
	public String main6(int year) {   
//		http://localhost/ch2/requestParam6        ---->> 500 java.lang.IllegalStateException: Optional int parameter 'year' is present but cannot be translated into a null value due to being declared as a primitive type. Consider declaring it as object wrapper for the corresponding primitive type.
//		http://localhost/ch2/requestParam6?year   ---->> 400 Bad Request, nested exception is java.lang.NumberFormatException: For input string: "" 
		System.out.printf("[%s]year=[%s]%n", new Date(), year);
		return "yoil";
	}
	
	@RequestMapping("/requestParam7") 
	public String main7(@RequestParam int year) {   
//		http://localhost/ch2/requestParam7        ---->> 400 Bad Request, Required int parameter 'year' is not present
//		http://localhost/ch2/requestParam7?year   ---->> 400 Bad Request, nested exception is java.lang.NumberFormatException: For input string: "" 
		System.out.printf("[%s]year=[%s]%n", new Date(), year);
		return "yoil";
	}

	@RequestMapping("/requestParam8") 
	public String main8(@RequestParam(required=false) int year) {   
	//	http://localhost/ch2/requestParam8        ---->> 500 java.lang.IllegalStateException: Optional int parameter 'year' is present but cannot be translated into a null value due to being declared as a primitive type. Consider declaring it as object wrapper for the corresponding primitive type.
	//	http://localhost/ch2/requestParam8?year   ---->> 400 Bad Request, nested exception is java.lang.NumberFormatException: For input string: "" 
		System.out.printf("[%s]year=[%s]%n", new Date(), year);
		return "yoil";
	}
	
	@RequestMapping("/requestParam9") 
	public String main9(@RequestParam(required=true) int year) {   
	//	http://localhost/ch2/requestParam9        ---->> 400 Bad Request, Required int parameter 'year' is not present
	//	http://localhost/ch2/requestParam9?year   ---->> 400 Bad Request, nested exception is java.lang.NumberFormatException: For input string: "" 
		System.out.printf("[%s]year=[%s]%n", new Date(), year);
		return "yoil";
	}
	
	@RequestMapping("/requestParam10")   
	public String main10(@RequestParam(required=true, defaultValue="1") int year) {   
	//	http://localhost/ch2/requestParam10        ---->> year=1   
	//	http://localhost/ch2/requestParam10?year   ---->> year=1   
		System.out.printf("[%s]year=[%s]%n", new Date(), year);
		return "yoil";
	}

	@RequestMapping("/requestParam11")   
	public String main11(@RequestParam(required=false, defaultValue="1") int year) {   
//		http://localhost/ch2/requestParam11        ---->> year=1   
//		http://localhost/ch2/requestParam11?year   ---->> year=1   
		System.out.printf("[%s]year=[%s]%n", new Date(), year);
		return "yoil";
	}
} // class
```

### 참고

- **ExceptionHandler -** Controller 계층에서 발생하는 에러를 잡아서 메서드로 처리해주는 기능이다.
- @RequestParam가 기본형, string일때는 생략가능

## 예제2

YoilMVC4.java

```java
package com.fastcampus.ch2;

import java.util.Calendar;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.ModelAndView;

@Controller
public class YoilTellerMVC4 {
    @RequestMapping("/getYoilMVC4") // http://localhost/ch2/getYoilMVC4?year=2021&month=10&day=1
    public ModelAndView main(MyDate date) { // 반환 타입이 ModelAndView 
System.out.println("date="+date);

    	// 1. ModelAndView를 생성
    	ModelAndView mv = new ModelAndView(); 
    	
    	// 2. 유효성 검사 
    	if(!isValid(date)) {
            mv.setViewName("yoilError"); // 뷰의 이름을 지정 
    	    return mv;
        }
    	
        // 3. 처리
    	char yoil = getYoil(date);

    	// 4. ModelAndView에 작업한 결과를 저장 
      	//mv.addObject("year",  date.getYear());     	
      	//mv.addObject("month", date.getMonth());     	
      	//mv.addObject("day",   date.getDay());
        mv.addObject("myDate", date);
      	mv.addObject("yoil", yoil);        
        
      	// 5. 작업 결과를 보여줄 뷰의 이름을 지정 
      	mv.setViewName("yoil"); 
      	
      	// 6. ModelAndView를 반환
      	return mv;
    }

    private char getYoil(MyDate date) {
        return getYoil(date.getYear(), date.getMonth(), date.getDay());
    }
    
    private char getYoil(int year, int month, int day) {
        Calendar cal = Calendar.getInstance();
        cal.set(year, month - 1, day);

        int dayOfWeek = cal.get(Calendar.DAY_OF_WEEK);
        return " 일월화수목금토".charAt(dayOfWeek);
    }
    
    private boolean isValid(MyDate date) {
        return isValid(date.getYear(), date.getMonth(), date.getDay());
    }
    
    private boolean isValid(int year, int month, int day) {    
    	if(year==-1 || month==-1 || day==-1) 
    		return false;
    	
    	return (1<=month && month<=12) && (1<=day && day<=31); // 간단히 체크 
    }
}
```

MyDate.java

```java
package com.fastcampus.ch2;

public class MyDate {
	private int year;
	private int month;
	private int day;
	
	public int getYear() {
		return year;
	}

	public void setYear(int year) {
		this.year = year;
	}

	public int getMonth() {
		return month;
	}

	public void setMonth(int month) {
		this.month = month;
	}

	public int getDay() {
		return day;
	}

	public void setDay(int day) {
		this.day = day;
	}

	@Override
	public String toString() {
		return String.format("[year=%d, month=%d, day=%d]", year, month, day);
	}
}
```

# 3. ModelAttribute

- 적용대상을 Model의 속성으로 자동 추가해주는 애너테이션
- 반환 타입 또는 컨트롤러 메서드의 매개변수에 적용 가능

```java
package com.fastcampus.ch2;

import java.util.Calendar;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
public class YoilTellerMVC5 {
    @ExceptionHandler(Exception.class)
	public String catcher(Exception ex) {
		System.out.println("ex="+ex);
		
		return "yoilError";
	}
    
    @RequestMapping("/getYoilMVC5") // http://localhost/ch2/getYoilMVC5?year=2021&month=10&day=1
//  public String main(@ModelAttribute("myDate") MyDate date, Model m) { // 아래와 동일 
    public String main(@ModelAttribute MyDate date, Model m) { // @ModelAttribute사용, 반환 타입은 String  
System.out.println("myDate="+date);

    	// 1. 유효성 검사 
    	if(!isValid(date))
    		return "yoilError";
    	
        // 2. 처리
    	char yoil = getYoil(date);

    	// 3. Model에 작업한 결과를 저장 
        // @ModelAttribute 덕분에 MyDate를 저장안해도 됨. View로 자동 전달됨.
//      m.addAttribute("myDate", date);     	
//      m.addAttribute("yoil", yoil);        
        
      	// 4. 작업 결과를 보여줄 뷰의 이름을 반환  
      	return "yoil";
    }
    
    private @ModelAttribute("yoil") char getYoil(MyDate date) {
    	return getYoil(date.getYear(), date.getMonth(), date.getDay());
    }
    
    private char getYoil(int year, int month, int day) {
        Calendar cal = Calendar.getInstance();
        cal.set(year, month - 1, day);

        int dayOfWeek = cal.get(Calendar.DAY_OF_WEEK);
        return " 일월화수목금토".charAt(dayOfWeek);
    }

    private boolean isValid(MyDate date) {
    	return isValid(date.getYear(), date.getMonth(), date.getDay());
    }
    
    private boolean isValid(int year, int month, int day) {    
    	if(year==-1 || month==-1 || day==-1) 
    		return false;
    	
    	return (1<=month && month<=12) && (1<=day && day<=31); // 간단히 체크 
    }
}
```

### 참고

- @ModelAttribute가 참조형 일때 생략가능

