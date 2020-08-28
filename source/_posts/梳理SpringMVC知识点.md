---
title: "梳理SpringMVC知识点"
catalog: true
date: 2020-04-17 18:20:24
subtitle: ""
header-img: "/img/default.png"
tags:
- Java
- 后端
catagories:
---

SpringMVC 是Spring开发框架中的一个部分，它和struts2框架类似，都是基于MVC模型对外提供web层服务的框架。SpringMVC相比Struts2框架更轻便，效率更高，配置更少，因此目前市面上web开发SpringMVC是不可或缺的一部分。
![springmvc](/img/springmvc.png)
SpringMVC的核心模块是DispatcherServlet, 我们传统用servlet去做请求适配控制时都是在web.xml中去配置mapping关系，但在SpringMVC中，我们会把所有的请求都交给DispatcherServlet给处理，所以默认情况下，dispatcher的路径配置为‘/’就好，代表拦截所有请求。之后在DispatcherServlet中会调用getHander()方法在已注册的处理器中进行匹配，然后返回 HandleExecutionChain。(注册方式可以通过XML bean的方式，也可以通过注解`@Controler`方式。) HandleExecutionChain 这个对象中包含了所有的拦截器信息和最终的handler信息。之后在doDispatch()这个方法中通过用HandlerAdapter对象的handle方法来处理请求。实际上也就是让Controller中对应的方法执行，返回ModelAndView对象给Dispatcher，然后基于view的名字和配置好的路径和视图解析器类型进行匹配和渲染。最终将渲染了的view返回给浏览器。