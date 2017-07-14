---
title: SpringMVC整合Freemarker
date: 2016-09-12 21:02:56
tags:
- Java
- SpringMVC
- FreeMarker
categories:
- Java
- 环境配置
---

## 添加jar包
　　添加freemarker的jar，还需要额外添加spring-content-support的jar包，不然会报错。

　　然后再Spring的配置文件中添加对freemarker的配置

``` xml
<!-- 配置freeMarker的模板路径 -->  
 <bean class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">  
    <property name="templateLoaderPath" value="WEB-INF/ftl/" />  
    <property name="defaultEncoding" value="UTF-8" />  
 </bean>  
 <!-- freemarker视图解析器 -->  
 <bean class="org.springframework.web.servlet.view.freemarker.FreeMarkerViewResolver">  
    <property name="suffix" value=".html" />  
    <property name="contentType" value="text/html;charset=UTF-8" />  
    <!-- 此变量值为pageContext.request, 页面使用方法：rc.contextPath -->  
    <property name="requestContextAttribute" value="rc" />  
 </bean> 
```

　　这样就配置好了对freemarker的支持。

　　做一下测试，写一个User类：

``` java
package com.my.springmvc.bean;

public class User {
	private String username;
	private String password;
	public String getUsername() {
		return username;
	}
	public void setUsername(String username) {
		this.username = username;
	}
	public String getPassword() {
		return password;
	}
	public void setPassword(String password) {
		this.password = password;
	}
	
}
```

　　一个FreeMarkerController类：

``` java
package com.my.springmvc.controller;

import java.util.ArrayList;
import java.util.List;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.servlet.ModelAndView;

import com.my.springmvc.bean.User;

@Controller
@RequestMapping("/home")
public class FreeMarkerController {
	
	@RequestMapping("/index")
	public ModelAndView Add(HttpServletRequest request,HttpServletResponse response){
		User user = new User();
		user.setUsername("sg");
		user.setPassword("1234");
		List<User> users  = new ArrayList<User>();
		users.add(user);
		
		ModelAndView mv = new ModelAndView();
		mv.setViewName("index");
		mv.addObject("users",users);
		return mv;
	}
}

```

然后再`WEB-INF/ftl`目录下创建一个`index.html`文件：

``` html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>another</title>
</head>
<body>  
<#list users as user>  
username : ${user.username}<br/>  
password : ${user.password}  
</#list>  
</body>  
</html>
```

　　结果：
![这里写图片描述](http://img.blog.csdn.net/20160504204151660)