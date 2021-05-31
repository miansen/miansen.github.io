---
layout: post
title: SpringBoot 处理全局异常，包括 404 错误，返回 JSON 或者页面
date: 2020-01-21
categories: Spring全家桶
tags: SpringBoot
author: 龙德
---

* content
{:toc}

## 第一种方式：实现 HandlerExceptionResolver 接口

BaseException 是我自定义的业务异常类，包含错误码和错误信息。

```java
public class BaseExceptionHandler implements HandlerExceptionResolver {

	@Override
	public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler,
			Exception e) {
		String contentType = request.getContentType();
		// 返回 json
		if (Objects.equals("application/json", contentType)) {
		    // 使用这个类可以返回 json 数据
			MappingJackson2JsonView jsonView = new MappingJackson2JsonView();
			jsonView.setExtractValueFromSingleKeyModel(true);
			ModelAndView mv = new ModelAndView(jsonView);
			if (e instanceof BaseException) {
				BaseException be = (BaseException) e;
				response.setStatus(be.getHttpCode());
				mv.addObject(new Result<>(be.getErrorCode(), be.getMessage()));
				return mv;
			} else {
				response.setStatus(BaseErrorCodeEnum.INTERNAL_ERROR.getHttpCode());
				mv.addObject(new Result<>(BaseErrorCodeEnum.INTERNAL_ERROR.getErrorCode(),
						BaseErrorCodeEnum.INTERNAL_ERROR.getMessage()));
				return mv;
			}
		} else {
		    // 返回页面
			ModelAndView mv = new ModelAndView();
			if (e instanceof BaseException) {
				BaseException be = (BaseException) e;
				response.setStatus(be.getHttpCode());
				mv.addObject("exception", be.getMessage());
				mv.addObject("errorCode", be.getErrorCode());
				mv.setViewName("/default/front/error/error");
				return mv;
			} else {
				response.setStatus(BaseErrorCodeEnum.INTERNAL_ERROR.getHttpCode());
				mv.addObject("exception", BaseErrorCodeEnum.INTERNAL_ERROR.getMessage());
				mv.addObject("errorCode", BaseErrorCodeEnum.INTERNAL_ERROR.getErrorCode());
				mv.setViewName("/default/front/error/error");
				return mv;
			}
		}
	}

}
```

这种方式只能处理 Controller 层抛出的异常，对于 DispatcherServlet 抛出的异常无能为力。

## 第二种方式：使用 @ControllerAdvice 和 @ExceptionHandler 注解

```java
@ControllerAdvice
public class GlobalExceptionHandler {
	
	@ExceptionHandler(value = Exception.class)
	public ModelAndView defaultErrorHandler(Exception e, HttpServletRequest request, HttpServletResponse response) throws Exception {
		String contentType = request.getContentType();
		if (Objects.equals("application/json", contentType)) {
			MappingJackson2JsonView jsonView = new MappingJackson2JsonView();
			jsonView.setExtractValueFromSingleKeyModel(true);
			ModelAndView mv = new ModelAndView(jsonView);
			if (e instanceof BaseException) {
				BaseException be = (BaseException) e;
				response.setStatus(be.getHttpCode());
				mv.addObject(new Result<>(be.getErrorCode(), be.getMessage()));
				return mv;
			} else {
				response.setStatus(BaseErrorCodeEnum.INTERNAL_ERROR.getHttpCode());
				mv.addObject(new Result<>(BaseErrorCodeEnum.INTERNAL_ERROR.getErrorCode(),
						BaseErrorCodeEnum.INTERNAL_ERROR.getMessage()));
				return mv;
			}
		} else {
			ModelAndView mv = new ModelAndView();
			if (e instanceof BaseException) {
				BaseException be = (BaseException) e;
				response.setStatus(be.getHttpCode());
				mv.addObject("exception", be.getMessage());
				mv.addObject("errorCode", be.getErrorCode());
				mv.setViewName("/default/front/error/error");
				return mv;
			} else {
				response.setStatus(BaseErrorCodeEnum.INTERNAL_ERROR.getHttpCode());
				mv.addObject("exception", BaseErrorCodeEnum.INTERNAL_ERROR.getMessage());
				mv.addObject("errorCode", BaseErrorCodeEnum.INTERNAL_ERROR.getErrorCode());
				mv.setViewName("/default/front/error/error");
				return mv;
			}
		}
	  }
	
}
```

这种方式可以处理 Controller 层和 DispatcherServlet 抛出的异常，所以用这种方式就好了。

## 处理 404 错误

DispatcherServlet 中有一个 throwExceptionIfNoHandlerFound 属性，如下图所示：

![image](https://miansen.wang/assets/20200121171015.png)

它的作用是：如果找不到处理该请求的处理程序，是否抛出 NoHandlerFoundException？

它默认是 false，也就是说找不到处理器时不抛出 NoHandlerFoundException 异常，DispatcherServlet 会自己处理。

所以上面那两个全局处理类都无法捕获到 NoHandlerFoundException 异常，也就无法处理 404 错误。

你可以在 application.properties 加上这段参数，将 throwExceptionIfNoHandlerFound 属性设置为 true。

```properties
spring.mvc.throw-exception-if-no-handler-found=true
```

但是这样还是无法起作用，关键是这段代码：

![image](https://miansen.wang/assets/20200121171853.png)

这段代码表明只有找不到处理器，也就是 mappedHandler 为 null 的情况下，才会进入 noHandlerFound 方法。

然而 SpringBoot 会默认配置一个静态资源映射处理器 ResourceHttpRequestHandler，所以不会进入 noHandlerFound 方法。

![image](https://miansen.wang/assets/20200121172455.png)

幸好 SpringBoot 还提供了配置接口，我们只要在 application.properties 里加入以下配置，即可关闭默认的静态资源映射处理器。

```properties
spring.resources.add-mappings=false
```

这样，当 DispatcherServlet 找不到处理器时，又没有默认的静态资源映射处理器，同时我们把 throwExceptionIfNoHandlerFound 属性设置了 true，那么 DispatcherServlet 就会抛出 NoHandlerFoundException 异常。就会被我们上面定义的 GlobalExceptionHandler 类捕获到，返回我们自定义的处理结果。

但是这种方式有一个弊端，就是关闭静态资源映射处理器后，无法访问到静态资源，只适合纯后端接口，没有前端的项目。

如果在不影响静态资源访问的情况下，又能处理 404 错误呢？请看下面这种方式：

SpringBoot 提供了一个接口，名字叫做 ErrorController。这个接口有一个实现类 BasicErrorController，它主要定义了两个方法：

![image](https://miansen.wang/assets/20200121173532.png)

DispatcherServlet 处理一个不存在的请求时，该请求会转发到 /${context-path}/error，并且不会被 ControllerAdvice 拦截。

它默认的页面是这样的：

![image](https://miansen.wang/assets/20200121174158.png)

默认的 JSON 是这样的：

![image](https://miansen.wang/assets/20200121174245.png)

所以，我们可以自己定义一个 BasicErrorController，返回自定义的页面和 JSON 数据。

比如我是这样写的：

```java
@RequestMapping(value = "/error", produces = "text/html")
public ModelAndView errorHtml(HttpServletRequest request) {
	ModelAndView mv = new ModelAndView();
	mv.addObject("exception", BaseErrorCodeEnum.INTERNAL_ERROR.getMessage());
	mv.addObject("errorCode", getStatus(request).value());
	mv.setViewName("/default/front/error/error");
	return mv;
}

@RequestMapping(value = "/error")
@ResponseBody
public ResponseEntity<Result<String>> error(HttpServletRequest request) {
	HttpStatus status = getStatus(request);
	return new ResponseEntity<Result<String>>(
			new Result<>(String.valueOf(status.value()), BaseErrorCodeEnum.INTERNAL_ERROR.getMessage()), status);
}
```

这时候 ErrorController 接口应该有两个实现类，一个是 SpringBoot 的，还有一个是自己定义的。

![image](https://miansen.wang/assets/20200121174544.png)

但是这样也不会冲突，因为 SpringBoot 会判断，如果已经存在 ErrorController 的实现类时，它就不会加载 BasicErrorController。

源码链接：

[https://github.com/miansen/Roothub/blob/master/src/main/java/wang/miansen/roothub/common/handler/BaseExceptionHandler.java](https://github.com/miansen/Roothub/blob/master/src/main/java/wang/miansen/roothub/common/handler/BaseExceptionHandler.java)

[https://github.com/miansen/Roothub/blob/master/src/main/java/wang/miansen/roothub/common/handler/GlobalExceptionHandler.java](https://github.com/miansen/Roothub/blob/master/src/main/java/wang/miansen/roothub/common/handler/GlobalExceptionHandler.java)

[https://github.com/miansen/Roothub/blob/master/src/main/java/wang/miansen/roothub/common/controller/BaseErrorController.java](https://github.com/miansen/Roothub/blob/master/src/main/java/wang/miansen/roothub/common/controller/BaseErrorController.java)