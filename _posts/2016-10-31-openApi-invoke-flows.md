---
layout: post
title: "openApi调用流程"
description: ""
category: docs
tags: []
---
本文介绍 微信--> open --> wxrmisvr 的调用流程

## 1.微信调用open

调用的地址类似 
http://x.x.x.x:8080/open/tlb/poslLoanSignProtocolService/handleService/json

下面看一下，当open接收到这个请求时是怎么处理的
在com.allinpay.open.web.controller.RouteController 中有相关的代码

``` java
    @RequestMapping("{system}/{serviceName}/{methodName}/{formart}")
    public void service(RequestPO requestPO, HttpServletRequest req,
            HttpServletResponse resp) throws UnsupportedEncodingException {
        Map<String, String> paramMap = HttpServiceUtils
                .getServiceParameters(req);
        String ip = HttpUtils.getIpAddr(req);
        requestPO.setIp(ip);
        logger.info("收到请求：[{}],[{}]", ClassUtils.getObjectLogInfo(requestPO),
                ClassUtils.getObjectLogInfo(paramMap));
        OpenResponse response = new OpenResponse();
        String opServiceName = requestPO.getSystem() + "_"
                + requestPO.getServiceName();
        try {
            // 校验
            validator.valid(requestPO, paramMap);
            long t1 = System.currentTimeMillis();
            OpServiceInvoker invoker = opServiceInvokerFactory
                    .getOpServiceInvoker(requestPO.getSystem() + "_opService");
            String result = invoker.invokeOpService(opServiceName, requestPO.getMethodName(),
                    requestPO.getFormart(), paramMap);
            String signResult = signUtils.sign(result, requestPO);
            sendResponse(resp, signResult, requestPO.getInputCharset());
            long cost = System.currentTimeMillis() - t1;
            // log入库
            log(requestPO, req, result, signResult, cost, "true");
            return;
        } catch (Exception e) {
            logger.error("调用服务失败", e);
            response.setErrorMessage((e instanceof BusinessException || e instanceof SystemException) ? e
                    .getMessage() : ErrorMessage.SYSTEM_ERROR.getMessage());
        }
        // 发送失败响应
        response.setIsSuccess("false");
        String result = RequestConstant.XML.equalsIgnoreCase(requestPO
                .getFormart()) ? response.toXml() : response.toJson();
        sendResponse(resp, result, requestPO.getInputCharset());
        // log入库
        log(requestPO, req, result, "", 0, "false");
    }
```
上面的代码 对
http://x.x.x.x:8080/open/tlb/poslLoanSignProtocolService/handleService/json
做了解析 
将请求转发到了配置文件中tlb_opService ,tlb_poslLoanSignProtocolService 对应的服务

``` xml
<service key="tlb_poslLoanSignProtocolService+handleService">
        <serviceName>tlb_poslLoanSignProtocolService</serviceName>
        <method>handleService</method>
        <url>http://x.x.x.x:8180/wxrmisrv/openService</url>
        <isProtect>1</isProtect>
        <privateKey></privateKey>
        <status>1</status>
        <desc>合同补签</desc>
</service>
```
即调用了 
http://x.x.x.x:8180/wxrmisrv/openService?BEAN_NAMEW=tlb_poslLoanSignProtocolService&METHOD_NAME=handleService&FORMART=json

## 2.open调用wxrmisvr

看wxrmisvr中的服务是如何发布的
首先在web.xml中配置了

``` xml
<filter>
    <filter-name>openService</filter-name>
    <filter-class>com.allinpay.commons.core.open.OpenServiceFilter</filter-class>
</filter>
<filter-mapping>
     <filter-name>openService</filter-name>
     <url-pattern>/openService</url-pattern>
 </filter-mapping>
```
OpenServiceFilter 这个过滤器会处理 url为 /openService 的请求
看一下 OpenServiceFilter 的实现

``` java
  public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
    throws IOException, ServletException
  {
    HttpServletRequest req = (HttpServletRequest)request;
    req.setCharacterEncoding("utf-8");
    HttpServletResponse res = (HttpServletResponse)response;
    Map paramMap = HttpUtils.getAllStringValueParameters(req);
    this.logger.info("收到请求：[{}]", ClassUtils.getObjectLogInfo(paramMap));
    OpenResponse openResponse = invokeService(paramMap, (String)paramMap.get("BEAN_NAME"), (String)paramMap.get("METHOD_NAME"));
    res.setCharacterEncoding("utf-8");
    if ("XML".equalsIgnoreCase((String)paramMap.get("FORMART"))) {
      res.getWriter().write(URLEncoder.encode(openResponse.toXml(), "utf-8"));
      this.logger.info("响应：[{}]", openResponse.toXml());
    } else {
      res.getWriter().write(URLEncoder.encode(openResponse.toJson(), "utf-8"));
      this.logger.info("响应：[{}]", openResponse.toJson());
    }
  }

  private OpenResponse invokeService(Map<String, String> paramMap, String beanName, String methodName)
  {
    OpenResponse response = new OpenResponse();
    try {
      Object service = SpringContextHolder.getBean(beanName);
      Method method = service.getClass().getDeclaredMethod(methodName, new Class[] { Map.class });

      convertBizMap(paramMap);

      Object object = method.invoke(service, new Object[] { paramMap });
      response.setResponse(object);
      response.setIsSuccess("true");
    } catch (Exception e) {
      
    }

    return response;
  }
```

过滤器收到请求的时候解析请求 寻找对应的spring bean通过反射调用指定的方法。
spring 配置：

``` xml
    <!-- 合同补签 L121 -->
    <bean id="tlb_poslLoanSignProtocolService"
        class="com.tonghuafund.loan.rmi.service.impl.LoanSignProtocolServiceImpl">
        <property name="loanProtocolService">
            <ref bean="loanProtocolService" />
        </property>
    </bean>
```

以上。