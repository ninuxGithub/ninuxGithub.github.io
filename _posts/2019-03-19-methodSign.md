---
title: 公共aip方法签名
author: ninuxGithub
layout: post
date: 2019-3-19 08:55:20
description: "公共aip方法签名"
tag: dubbo
---


### 目的
    
    建立一个demo 测试公共aip客户端参数请求的时候加入签名， 后端服务器验证签名；
    
    代码寄托在github: https://github.com/ninuxGithub/spring-boot-dubbo-zookeeper.git
   
   
### 实现思路
    
    1.首先分析 客户端请求的时候携带的参数是否符合目标方法入参（是否参数名称正确， 是否参数列表顺序一致， 是否个数一致）
    解决方案： 启动项目的时候扫描公共的api接口， 提取出接口的信息
    包括方法名称， 入参列表等待
    
    
    2.服务器签名的验证
    服务端通过过滤器拦截请求（拦截的路径可以通过模糊匹配 可以对位那些公共api的公共路径 例如： /commonApi/*）
    
    简单的来说就是对外的接口都必须是以： ‘/commonApi/xxxx’ 打头
    
    然后就是获取request 的请求参数校验参数 ， 如果验签通过， 那么通过过滤器 chain.doFilter
    
    
 
### 细节描述
    
    扫描接口：方法实现ApplicationContextAware 获取到ApplicationContext  从applicatonContext 扫描bean的信息
    通过反射机制解析类的结构
       
```java
package com.example.consumer.config;

import org.apache.commons.collections4.MapUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 *
 * 扫描@RestController or @Controller的注解  获取方法上有@RequestMapping的方法
 *
 * 获取方法名称 ， 获取参数列表 封装到map
 * @author shenzm
 * @date 2019-3-18
 * @description 作用
 */

@Component
public class ScanBeanService implements ApplicationContextAware {

    private static final Logger logger = LoggerFactory.getLogger(ScanBeanService.class);


    private Map<String, List<String>> methodName2ParamsTable = new HashMap<>();

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        scan(applicationContext);
    }


    /**
     * 扫描工程里面所有对外的接口
     *
     * @param applicationContext
     */
    private void scan(ApplicationContext applicationContext) {
        //TODO @Controller 未添加
        Map<String, Object> beansWithAnnotation = applicationContext.getBeansWithAnnotation(RestController.class);
        if (MapUtils.isNotEmpty(beansWithAnnotation)) {
            for (Object bean : beansWithAnnotation.values()) {
                Method[] declaredMethods = bean.getClass().getDeclaredMethods();
                for (Method method : declaredMethods) {
                    RequestMapping declaredAnnotation = method.getDeclaredAnnotation(RequestMapping.class);
                    String methodName = method.getName();
                    if (null != declaredAnnotation) {
                        List<String> paramsTable = methodName2ParamsTable.get(methodName);
                        if (null == paramsTable) {
                            paramsTable = new ArrayList<>();
                        }
                        Parameter[] parameters = method.getParameters();
                        for (Parameter parameter : parameters) {
                            paramsTable.add(parameter.getName());
                        }
                        methodName2ParamsTable.put(methodName, paramsTable);
                    }
                }
            }
            logger.info("扫描系统的对外接口： {}",methodName2ParamsTable);
        }
    }


    public Map<String, List<String>> getMethodName2ParamsTable() {
        return methodName2ParamsTable;
    }
}

```    
    
    
    
    客户端请求common API
           
```java
package com.example.provider.controller;

import com.example.provider.utils.*;
import com.qcloud.Common.Sign;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.UnsupportedEncodingException;
import java.security.InvalidKeyException;
import java.security.NoSuchAlgorithmException;
import java.util.HashMap;
import java.util.Map;
import java.util.TreeMap;

/**
 * @author shenzm
 * @date 2019-3-18
 * @description 作用
 */

@RestController
public class ApiTestController {

    private static final Logger logger = LoggerFactory.getLogger(ApiTestController.class);

    /**通过注册中心获取*/
    private String SecretId = "12345678";

    /**通过注册中心获取*/
    private String SecretKey ="abcdefg";

    private static JsonBinder jsonBinder = JsonBinder.buildNormalBinder();

    @RequestMapping("/api/test")
    public Map<String, Object> test() {
        Map<String, Object> resultMap = new HashMap<>();
        try {
            TreeMap<String, Object> paramMap = getCommParams("sayHello");
            paramMap.put("name", "java");
            String signUrl = Sign.makeSignPlainText(paramMap, "GET", "127.0.0.1:9091", "/commonApi/hello");
            String signature = Sign.sign(signUrl, SecretKey, "HmacSHA256");
            paramMap.put("Signature", signature);
            HttpResult httpResult = HttpClientUtil.doGet("http://127.0.0.1:9091/commonApi/hello", paramMap);
            String responseBody = httpResult.getBody();
            Map<String, Object> jsonMap = jsonBinder.fromJson(responseBody, HashMap.class);
            resultMap.put("jsonMap", jsonMap);
        } catch (Exception e) {
            resultMap.put("msg", e.getMessage());
        }
        return resultMap;


    }

    /**
     * 签名的公共的请求参数 （必须携带）
     * @param Action
     * @return
     */
    private TreeMap<String, Object> getCommParams(String Action) {
        TreeMap<String, Object> paramMap = new TreeMap<String, Object>();
        paramMap = new TreeMap<String, Object>();
        paramMap.put("Action", Action);
        paramMap.put("Nonce", NumberUtil.getRandomNum());
        paramMap.put("Region", "sh");
        paramMap.put("SecretId", SecretId);
        paramMap.put("SignatureMethod", "HmacSHA256");
        paramMap.put("Timestamp", DateUtil.getTimeStamp());
        return paramMap;
    }
}

```


    最后就是客户端的验签
       
```java
package com.example.consumer.config;

import com.alibaba.fastjson.JSONObject;
import com.example.consumer.bean.Message;
import com.qcloud.Common.Sign;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.OutputStreamWriter;
import java.io.PrintWriter;
import java.nio.charset.Charset;
import java.util.*;

/**
 * 对请求路径符合： /commonApi/* 的所有的路径进行过滤
 *
 * 在过滤器的doFilter 方法进行签名的过滤
 *
 * @author shenzm
 * @date 2019-3-18
 * @description 作用
 */

@Component
@WebFilter(urlPatterns = "/commonApi/*", filterName = "apiAccessFilter")
public class ApiAccessFilter implements Filter {

    private static final Logger logger = LoggerFactory.getLogger(ApiAccessFilter.class);

    private static final String SIGNATURE = "Signature";

    private static final String SIGNATUREMETHOD = "SignatureMethod";

    @Autowired
    private ScanBeanService scanBeanService;


    /**
     * 公共的参数列表  是有序的
     */
    private List<String> paramList = new ArrayList<>(Arrays.asList("Action", "Nonce", "Region", "SecretId", "SignatureMethod", "Timestamp"));

    private Set<String> commonParams = new HashSet<>(paramList);

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    /**
     * 通过请求的参数secretId 查询对应的用户的SecretKey
     * @param secretId
     * @return
     */
    public String mockQuerydb(String secretId){
        if("12345678".equals(secretId)){
            return "abcdefg";
        }
        return null;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {

        String errJson = null;
        boolean valid = true;

        TreeMap<String, Object> paramMap = new TreeMap<>();

        String action = request.getParameter("Action");
        String secretId = request.getParameter("SecretId");
        if (StringUtils.isNoneBlank(action) && StringUtils.isNoneBlank(secretId)) {
            Map<String, List<String>> beanName2ParamsTable = scanBeanService.getMethodName2ParamsTable();
            if (beanName2ParamsTable.containsKey(action)) {
                List<String> paramsTable = beanName2ParamsTable.get(action);
                Map<String, String> name2value = new HashMap<>();

                //将方法的参数列表加入到共工的参数里面的集合里面去
                paramList.addAll(paramsTable);

                //开始获取请求的参数， 判断非空
                for (String paramName : paramList) {
                    String paramValue = request.getParameter(paramName);
                    if (paramName.equals(SIGNATURE)) {
                        continue;
                    }
                    if (null == paramValue || "".equals(paramValue.trim())) {
                        errJson = "请求参数：" + paramName + "不可以为null或空字符串";
                        valid = false;
                        break;
                    }
                    paramMap.put(paramName, paramValue);
                }

                if (valid) {
                    HttpServletRequest req = (HttpServletRequest) request;
                    String remoteHost = request.getRemoteHost();
                    int serverPort = request.getServerPort();
                    String requestHost = remoteHost + ":" + serverPort;
                    String requestURI = req.getRequestURI();
                    String requestMethod = req.getMethod();
                    String signatureMethod = req.getParameter(SIGNATUREMETHOD);
                    String reqestSignature = req.getParameter(SIGNATURE);

                    logger.info("requestHost : {}",requestHost);
                    logger.info("requestURI :{}",requestURI);
                    logger.info("requestMethod:{}",requestMethod);
                    logger.info("signatureMethod:{}",signatureMethod);
                    logger.info("reqestSignature:{}",reqestSignature);

                    try {
                        String signUrl = Sign.makeSignPlainText(paramMap, requestMethod, requestHost, requestURI);
                        String signature = Sign.sign(signUrl, mockQuerydb(secretId), signatureMethod);

                        logger.info("reqestSignature : {} signature :{}", reqestSignature, signature);

                        //对比签名是否一致
                        if (reqestSignature.equals(signature)) {
                            valid = true;
                            logger.info("签名验证通过，调用=====>chain.doFilter");
                        } else {
                            valid = false;
                            errJson = "签名不正确";
                        }
                    } catch (Exception e) {
                        valid = false;
                        errJson = e.getMessage();
                    }

                }

            } else {
                valid = false;
                errJson = "不存在的Action名称";
            }
        } else {
            valid = false;
            errJson = "请求参数必须包含参数：Action";
        }

        if (!valid) {
            writeErrJson(response, new Message(errJson));
        } else {
            chain.doFilter((HttpServletRequest)request, (HttpServletResponse)response);
        }
    }

    private void writeErrJson(ServletResponse response, Message message) throws IOException {
        OutputStreamWriter osw = new OutputStreamWriter(response.getOutputStream(), Charset.forName("UTF-8"));
        PrintWriter pw = new PrintWriter(osw, true);
        pw.write(JSONObject.toJSONString(message));
        pw.flush();
        pw.close();
        osw.close();
    }

    @Override
    public void destroy() {

    }
}

```   


![方法签名](/images/posts/methodSign.jpg) 


### 友情提示
   
    pom 借助了腾讯云的签名的jar

    项目启动的方法
    本地启动activeMQ , zookeeper 
    然后启动 ProviderApplication --> ConsumerApplication