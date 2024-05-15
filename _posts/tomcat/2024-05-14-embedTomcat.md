---
title: Embed Tomcat
author: ninuxGithub
layout: post
date: 2024-05-14 13:27:29
description: "tomcat request2"
tag: tomcat
---

## 引言

    使用tomcat 遇到的问题分享， -->分析问题-->引入话题
    借鉴EmbeddedTomcat 来实现SupplierTomcatServer

## tomcat组件的概览

![tomcat 内部组件结构](/images/posts/tomcat-inner-structure.png)


![组件](/images/posts/Tomcat请求流程图.png)


![组件](/images/posts/Tomcat-组件流程图.png)

    Server-->
        -->Services
            -->Connector -->processHandler(构造器 ProtocolHandler.create)-->org.apache.coyote.http11.Http11NioProtocol
                         -->Adapter (initInternal)

    启动engine ， 会启动host 在通过lifecycle 来完成项目的启动部署(HostConfig)-->发布的本质(创建一个Context对象挂载到Host)-->Context.start


    EngineValve1-->EngineValve2->....->StandardEngineValve
        -->HostValve1-->HostValve2->....->StandardHostValve
            -->ContextValve1-->ContextValve2->....->StandardContextValve
                -->WrapperValve1-->WrapperValve2->....->StandardWrapperValve


## 请求转换如何转为为HttpServletRequest?

```shell
curl --location 'https://xxxx/supplier-endpoint/ota' \
--header 'Authorization: Basic xxxxxxxx=' \
--header 'Content-Type: application/xml' \
--data '<OTA_HotelRateAmountNotifRQ xmlns="http://www.opentravel.org/OTA/2003/05" EchoToken="1257418761962" TimeStamp="2011-11-05T11:40:42.026+01:00" Target="Production" Version="3.000" PrimaryLangID="en">
    <RateAmountMessages HotelCode="3019">
        <RateAmountMessage LocatorID="158881">
            <StatusApplicationControl Start="2024-05-01" End="2024-05-10" IsRoom="true" InvTypeCode="QEC" RatePlanCode="3KD" />
            <Rates>
                <Rate>
                     <BaseByGuestAmts>
                <BaseByGuestAmt AmountBeforeTax="75750" CurrencyCode="CNY" DecimalPlaces="2" NumberOfGuests="1"></BaseByGuestAmt>
                <BaseByGuestAmt AmountBeforeTax="84000" CurrencyCode="CNY" DecimalPlaces="2" NumberOfGuests="2"></BaseByGuestAmt>
              </BaseByGuestAmts>
                    <MealsIncluded MealPlanCodes="14"/>
                </Rate>
            </Rates>
        </RateAmountMessage>
    </RateAmountMessages>
</OTA_HotelRateAmountNotifRQ>'


curl --location 'https://alpha2.bookings.premierinn.com/csp/sql/Agency.Booking2.cls?soap_method=ProcessMessage' \
--form 'soap_method="ProcessMessage"' \
--form 'XMLIn="<messages xmlns=\"http://www.redskyit.com\">
  <Login UserName=\"xxx\" Password=\"xxx\" />
   <Parameters MessageType=\"AvailabilityRequest\">
    <Rooms NumberofRooms=\"1\">
      <RoomDetails Number=\"1\" Adults=\"1\" Children=\"1\" Cots=\"No\" Disabled=\"No\" Double=\"No\"/>
    </Rooms>
    <CompanyName Code=\"DE050\"/>
    <HotelCode>BANBRI</HotelCode>
    <StayDateRange Start=\"2024-04-28\" End=\"2024-04-29\"/>
    <CellCode>HUBSB,DISMD,DCSAL,XMLMD,DISBB,XMLBB,ZIPNF,ZIPFB,HUBFB</CellCode>
  </Parameters>
</messages>"'
```

    javax.servlet.http.HttpServletRequestWrapper (HttpServletRequest) getParameter(name) 查看方法的调用链
        -->org.apache.catalina.connector.RequestFacade
        -->org.apache.catalina.connector.Request
        -->org.apache.coyote.Request


也就是curl 原始的请求会将请求首先转换为coyote.Request, 当应用层的HttpServletRequest请调用方法的时候会来调用底层的request 的对应的方法(Facade)


## tomcat 响应压缩？
    response 是否开启压缩， 根据CompressionConfig 的配置 1: on, 2: force, 0:off,
    触发条件：长度>=2048, accept-encoding 包含Gzip ，结合useCompression 方法返回的是不是true, 是true就压缩
    通过GzipOutPutFilter 进行对response 的压缩


## Servlet 是如何被加载到tomcat 里面的？
    提示信息：org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedContext#load
    费启动加载的：org.apache.catalina.core.StandardWrapperValve.invoke
    ServletClass 需要被添加到Wrapper 里面去， 通过ContextConfig( 是一个lifeCycle 在启动context 的时候执行)
    org.apache.catalina.core.StandardWrapper.setServletClass 下个端点追踪ServletClass 是如何设置进去的， 以及创建Wrapper.
    ServletRegistrationBean/FilterRegistrationBean (观察他们的父类的接口) 相当于web.xml 里面配置servlet mapping, name 的配置

    重新认识ServletContextInitializer  -->ServletContextInitializerBeans 通过beanFactory 加载spring 容器呢的所有的ServletContextInitializer

    ==>分析TomcatStarter 的initializers是怎么来的？
     -->context.addServletContainerInitializer(starter, NO_CLASSES);添加到tomcat context 里面去 ，作为tomcat context 的initializers
     -->TomcatStarter 里面的initializers 是来自于匿名内部类，如下：

```java
public class ServletWebServerApplicationContext{
    private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
        return new ServletContextInitializer(){

            @Override
            public void onStartup(ServletContext servletContext) throws ServletException{

            }
        };
    }


    private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
        return this::selfInitialize;
    }

    private void selfInitialize(ServletContext servletContext) throws ServletException {
        prepareWebApplicationContext(servletContext);
        registerApplicationScope(servletContext);
        WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);
        for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
            beans.onStartup(servletContext);
        }
    }
}
```


## Tomcat 如何启动Server , Engine, Connector 的顺序和时机？
    TomcatWebService 在initiate 的
    时候将Connector 从service 中移除， 防止绑定到了protocolHandler 有请求进入了 人那通过调用tomcat.start()
    来完成engine 的启动 但是此时的状态是不是Avail , tomcat server service 里面的connector对应也被暂时移除了，
    所以无法绑定协议处理器 等到spring spring 容器加载完毕后， finishBeanFactoryInitialization(
    beanFactory) 调用下一个方法finishRefresh()
    来调用通过lifecycle 来调用启动webServer 启动webServer 之前需要将之前的connector 在设置会service 里面去，在add
    方法里面就会启动connector ， 从可以正常接收tomcat 请求了

启动顺序
```shell
tomcat --> server [init]
            --> server start[init] (remove connector)
            --> engine start
            --> spring context finish bean init
            --> add Connector to Service and start-->receive request
```




## Netty Reactor & Proactor
    https://www.xiaolincoding.com/os/8_network_system/reactor.html#reactor
    https://www.xiaolincoding.com/os/8_network_system/reactor.html#proactor


    