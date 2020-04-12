title: '我的shiro之旅: 十六验证码'
author: Dylan
tags:
  - shiro
  - java
categories:
  - 编程语言
date: 2018-09-16 12:13:00
---
&emsp;&emsp;验证码几乎是所有站点都会用到的，验证码在一定程序上能有效处理一些恶意攻击。验证码的类库有很多，这里用到生成验证码的工具是SimpleCaptcha，目前在官网看到最新版的是1.2.1，[下载地址](http://sourceforge.net/projects/simplecaptcha/files/simplecaptcha-1.2.1.jar/download)。

&emsp;&emsp;首先将类库的包下载引入到项目中，写一个用于生成和校验验证码的工具类。下面给出生成验证码的工具类:

```java
import java.awt.Color;
import java.awt.Font;
import java.util.List;

import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

import nl.captcha.Captcha;
import nl.captcha.servlet.CaptchaServletUtil;
import nl.captcha.text.renderer.DefaultWordRenderer;
import nl.captcha.text.renderer.WordRenderer;

import org.apache.commons.lang.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.concom.lang.exception.InvalidCaptchaException;
import com.google.common.collect.ImmutableList;

/**
 * @author Dylan
 * @mail pickear@gmail.com
 * @time 2014年4月9日
 */
public final class HttpCaptchaHelper {

	private final static Logger log = LoggerFactory.getLogger(HttpCaptchaHelper.class);

	private final static String CAPTCHA_PARAM = "captcha";
	
	private final static List<Font> FOUNTS = ImmutableList.of(
		new Font("Courier New", Font.ITALIC, 40), 
		new Font("Arial", Font.ITALIC, 40),
		new Font("Times New Roman", Font.ITALIC, 40),
		new Font("Verdana", Font.ITALIC, 40)
	);
	private final static List<Color> COLORS = ImmutableList.of(Color.black);
	private final static WordRenderer RENDERER = new DefaultWordRenderer(COLORS, FOUNTS);
	private HttpCaptchaHelper() {}
	/**store captcha to session
	 * @param captcha
	 * @param session
	 */
	public static void storeCaptcha(Captcha captcha, HttpSession session) {

		DemonPredict.notNull(session, new InvalidCaptchaException("can not get session"));
		DemonPredict.notNull(captcha, new InvalidCaptchaException(String.format("no captcha in session 【%s】", session.getId())));
		
		synchronized (session) {
			if(log.isTraceEnabled()){
				log.trace(String.format("store captcha to session 【%s】", session.getId()));
			}
			session.setAttribute(CAPTCHA_PARAM, captcha);
		}
	}
	
	/**check captcha
	 * @param captchaCode
	 * @param session
	 */
	public static void checkCaptcha(String captchaCode,HttpSession session){
		
		if(StringUtils.isBlank(captchaCode)){
			throw new InvalidCaptchaException("captcha is blank");
		}
		
		DemonPredict.notNull(session, new InvalidCaptchaException("can not get session"));
		
		Captcha captcha = (Captcha)session.getAttribute(CAPTCHA_PARAM);
		
		DemonPredict.notNull(captcha, new InvalidCaptchaException(String.format("no captcha in session 【%s】", session.getId())));
		
		DemonPredict.isTrue(captcha.isCorrect(captchaCode), new InvalidCaptchaException(String.format("the captcha is %s ,but user input %s",captcha.getAnswer(),captchaCode)));
	
		synchronized (captcha) {
			if(log.isTraceEnabled()){
				log.trace("remove captcha from session");
			}
			session.removeAttribute(CAPTCHA_PARAM);
		}
		
	}
	
	/**
	 * @return
	 */
	public static Captcha createCaptcha(){
		Captcha captcha = new Captcha.Builder(120, 40).addText(RENDERER).addNoise().build();
		return captcha;
	}
	
	/**
	 * @param response
	 * @param session
	 */
	public static void writeAndStoreCaptcha(HttpServletResponse response, HttpSession session){
		Captcha captcha = createCaptcha();
		CaptchaServletUtil.writeImage(response, captcha.getImage());
		storeCaptcha(captcha,session);
	}
	
}
```

其中DemonPredict类是自己写的一个断言类，在这里不给出代码，读者可以写一个或者用第三方包提供的断言类。这个工具类主要是生成一个Captcha，把Captcha的image写回客户端，并缓存到session中，当check后再把Captcha从session中删除。下面再给出另一个工具类，主要是针对shiro项目，在HttpCaptchaHelper基础上进一步封装:

```java
import javax.servlet.ServletRequest;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

import org.apache.shiro.SecurityUtils;
import org.apache.shiro.web.subject.WebSubject;

import nl.captcha.Captcha;

import com.concom.lang.helper.HttpCaptchaHelper;

/**
 * @author Dylan
 * @mail pickear@gmail.com
 * @time 2014年4月9日
 */
public final class ShiroCaptchaHelper {

	
	private ShiroCaptchaHelper(){}
	
	
	/**
	 * @param captcha
	 */
	public static void storeCaptcha(Captcha captcha){
		HttpCaptchaHelper.storeCaptcha(captcha, getHttpSession());
	}
	
	
	/**
	 * @param captchaCode
	 */
	public static void checkCaptcha(String captchaCode){
		HttpCaptchaHelper.checkCaptcha(captchaCode, getHttpSession());
	}
	
	/**
	 * @param response
	 */
	public static void writeAndStoreCaptcha(HttpServletResponse response){
		HttpCaptchaHelper.writeAndStoreCaptcha(response, getHttpSession());
	}
	
	/**
	 * @return
	 */
	private static HttpSession getHttpSession(){
		ServletRequest request = ((WebSubject)SecurityUtils.getSubject()).getServletRequest(); 
		HttpSession session = ((HttpServletRequest)request).getSession(false);
		return session;
	}
}
```

下面是一个测试用的controller代码

```java
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.shiro.web.util.WebUtils;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import com.concom.security.infrastructure.helper.ShiroCaptchaHelper;

@Controller
public class TestController {
	

	@RequestMapping(value="/captcha.html",method=RequestMethod.GET)
	public void captcha(HttpServletRequest request,HttpServletResponse response){
		
		ShiroCaptchaHelper.writeAndStoreCaptcha(response);
	}
	
	@RequestMapping(value="/check.html",method=RequestMethod.POST)
	public void checkCaptcha(HttpServletRequest request,HttpServletResponse response){
		
		String captchaCode = WebUtils.getCleanParam(request, "captcha");
		
		ShiroCaptchaHelper.checkCaptcha(captchaCode);
	}


}
```

页面获取验证码的html为:

```html

<%@ page language="java" import="java.util.*" pageEncoding="UTF-8"%>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<title></title>
<script type="text/javascript">
$(function(){
	
	$(".captcha").click(function(){
		var captchaUrl =  "${ctx}/captcha.html?"+Math.random().toString();
		$("#captcha-img").attr("src",captchaUrl);
		$("#captcha").focus();
	});
	
});
</script>
</head>
<body>
	<form action="${ctx}/check.html" method="post">
	验证码 ： <input id="captcha" name="captcha" type="text"/>
	<img id="captcha-img"  class="captcha"  src="${ctx}/captcha.html" alt="看不清,请点击图片换一张" />(看不清<a id="captcha1" class="captcha" href="javascript:void(0)">换一张</a>)
	<input value="提交" type="submit" />
	</form>
</body>
</html>
```

这里有一个地方要注意的是，页面的js点击事件里加了Math.random().toString()，这样火狐和ie浏览器才可以正常改变验证码。这个，这个问题在谷歌浏览器不存在。