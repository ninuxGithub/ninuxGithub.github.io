---
title: 绕过https安全验证发送post请求
author: ninuxGithub
layout: post
date: 2019-9-17 13:39:13
description: "https发送post请求"
tag: java
---


## https请求

    最近对接客户的需求，要求通过post发送数据给客户端的后台； 特别的地方就是客户的服务器是
    https://ip:port/xxxxx  自己弄的https; 没有绑定域名
    
    如果是https绑定了域名就不存在请求的时候安全验证一说了， 但是正式可能是自己弄的时候后， 在发送post数据的时候就需要绕过
    https的安全验证了；
    
    这个工能弄了大半天， 借鉴了和多博客的方法， 但是不是很理想；
    
    走过的坑太多了

    控制权的请求接口


```java
@RestController
public class HttpsController {


    @RequestMapping(value = "/https", method = RequestMethod.POST)
    public String https(HttpServletRequest request) {
        Enumeration<String> names = request.getParameterNames();
        List<String> lists = new ArrayList<>();
        while (names.hasMoreElements()) {
            lists.add(names.nextElement());
        }
        System.out.println(lists);

        String name = request.getParameter("name");
        Integer age = Integer.valueOf(request.getParameter("age"));
        String content = request.getParameter("content");
        System.out.println(content);
        JSONObject jsonObject = JSONObject.parseObject(content);
        String name2 = jsonObject.get("name").toString();
        Integer age2 = Integer.valueOf(jsonObject.get("age").toString());
        return name + age + " " + content + "  " + name2 + "  " + age2;
    }

}

```


       spring boot 使用restTemplate的配置



```java
package com.example.study.config;

import org.apache.http.config.Registry;
import org.apache.http.config.RegistryBuilder;
import org.apache.http.conn.socket.ConnectionSocketFactory;
import org.apache.http.conn.socket.PlainConnectionSocketFactory;
import org.apache.http.conn.ssl.NoopHostnameVerifier;
import org.apache.http.conn.ssl.SSLConnectionSocketFactory;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.DefaultHttpRequestRetryHandler;
import org.apache.http.impl.client.HttpClientBuilder;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.apache.http.ssl.SSLContextBuilder;
import org.apache.http.ssl.TrustStrategy;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.DefaultResponseErrorHandler;
import org.springframework.web.client.RestTemplate;

import javax.net.ssl.HostnameVerifier;
import javax.net.ssl.SSLContext;
import java.security.KeyManagementException;
import java.security.KeyStoreException;
import java.security.NoSuchAlgorithmException;
import java.security.cert.CertificateException;
import java.security.cert.X509Certificate;

/**
 * @author shenzm
 * @date 2019-9-16 12:12
 * @since 1.0
 */

@Configuration
public class RestTemplateConfig {

    private static final Logger logger = LoggerFactory.getLogger(RestTemplateConfig.class);


    @Bean("httpsRestTemplate")
    public RestTemplate httpsRestTemplate() {
        RestTemplate restTemplate = new RestTemplate(clientHttpRequestFactory());
        restTemplate.setErrorHandler(new DefaultResponseErrorHandler());
        return restTemplate;
    }

    @Bean
    public HttpComponentsClientHttpRequestFactory clientHttpRequestFactory() {
        try {
            HttpClientBuilder httpClientBuilder = HttpClientBuilder.create();

            SSLContext sslContext = new SSLContextBuilder().loadTrustMaterial(null, new TrustStrategy() {
                @Override
                public boolean isTrusted(X509Certificate[] arg, String s) throws CertificateException {
                    return true;
                }
            }).build();

            httpClientBuilder.setSSLContext(sslContext);

            HostnameVerifier hostnameVerifier = NoopHostnameVerifier.INSTANCE;

            SSLConnectionSocketFactory sslSocketFactory = new SSLConnectionSocketFactory(sslContext, hostnameVerifier);
            // 注册http和https请求
            Registry<ConnectionSocketFactory> socketFactoryRegistry = RegistryBuilder.<ConnectionSocketFactory>create()
                    .register("http", PlainConnectionSocketFactory.getSocketFactory())
                    .register("https", sslSocketFactory)
                    .build();
            // 开始设置连接池
            PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager(socketFactoryRegistry);
            // 最大连接数2700
            connectionManager.setMaxTotal(2700);
            // 同路由并发数100
            connectionManager.setDefaultMaxPerRoute(100);
            //设置连接管理器
            httpClientBuilder.setConnectionManager(connectionManager);
            // 重试次数
            httpClientBuilder.setRetryHandler(new DefaultHttpRequestRetryHandler(3, true));

            CloseableHttpClient httpClient = httpClientBuilder.build();
            // httpClient连接配置
            HttpComponentsClientHttpRequestFactory httpRequestFactory = new HttpComponentsClientHttpRequestFactory(httpClient);
            // 连接超时150s
            httpRequestFactory.setConnectTimeout(150_000);
            // 数据读取超时时间200s
            httpRequestFactory.setReadTimeout(200_000);
            // 连接不够用的等待时间20s
            httpRequestFactory.setConnectionRequestTimeout(20_000);
            //返回client工厂
            return httpRequestFactory;
        } catch (KeyManagementException | NoSuchAlgorithmException | KeyStoreException e) {
            logger.error("初始化HTTP连接池出错", e);
        }
        return null;
    }
}

```


        测试类： 三种方式来绕过https安全验证


```java

package com.example.study;

import org.apache.http.HttpResponse;
import org.apache.http.HttpStatus;
import org.apache.http.NameValuePair;
import org.apache.http.client.ClientProtocolException;
import org.apache.http.client.entity.UrlEncodedFormEntity;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.client.utils.URLEncodedUtils;
import org.apache.http.conn.ssl.SSLConnectionSocketFactory;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.message.BasicNameValuePair;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.client.RestTemplate;

import javax.net.ssl.*;
import java.io.*;
import java.net.URL;
import java.security.KeyManagementException;
import java.security.NoSuchAlgorithmException;
import java.security.cert.CertificateException;
import java.security.cert.X509Certificate;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;


@RunWith(SpringRunner.class)
@SpringBootTest(classes = SpringBootStudyApplication.class)
@SuppressWarnings("all")
public class HttpClientTest {


    //post 请求地址， 复杂的参数请求： 包含name, age , content 三个参数
    private static final String url = "https://xxxxx:8443/https";


    /**
     * 经过https安全认证处理
     */
    @Autowired
    @Qualifier("httpsRestTemplate")
    RestTemplate restTemplate;


    @Test
    public void sendHttpsRequest() {
        HashMap<String, Object> hashMap = new HashMap<String, Object>() {{
            put("name", "java");
            put("age", 18);
            put("content", "{\"name\":\"java\",\"age\":\"20\"}");
        }};

        MultiValueMap<String, Object> requestParams = new LinkedMultiValueMap<String, Object>();
        requestParams.add("name", "java");
        requestParams.add("age", 18);

        requestParams.add("content", new HashMap<String, Object>() {{
            put("name", "java");
            put("age", 20);
        }});

        //HttpsURLConnection
        String response = sendPost(url, hashMap);
        System.out.println("HttpsURLConnection : " + response);

        //HttpClient
        String s = doPost(url, hashMap);
        System.out.println("httpClient : " + s);

        //restTemplate
        String result = restTemplate.postForObject(url, requestParams, String.class);
        System.out.println("restTemplate : " + result);
    }


    public static CloseableHttpClient createSSLClientDefault() {
        try {
            SSLContext sslcontext = SSLContext.getInstance("TLS");
            sslcontext.init(null, new TrustManager[]{myX509TrustManager}, null);
            HttpsURLConnection.setDefaultSSLSocketFactory(sslcontext.getSocketFactory());

            // Create all-trusting host name verifier
            HostnameVerifier allHostsValid = new HostnameVerifier() {
                public boolean verify(String hostname, SSLSession session) {
                    return true;
                }
            };

            // Install the all-trusting host verifier
            HttpsURLConnection.setDefaultHostnameVerifier(allHostsValid);
            SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(sslcontext, allHostsValid);
            return HttpClients.custom().setSSLSocketFactory(sslsf).build();
        } catch (KeyManagementException e) {
            e.printStackTrace();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;

    }


    public static String doPost(String url, HashMap<String, Object> params) {
        //构建POST请求   请求地址请更换为自己的。
        HttpPost post = new HttpPost(url);
        InputStream inputStream = null;
        String result = "";
        try {
            //使用之前写的方法创建httpClient实例
            //disableSslVerification();
            CloseableHttpClient httpClient = createSSLClientDefault();
            List<NameValuePair> paramsList = new ArrayList<>();
            for (String key : params.keySet()) {
                paramsList.add(new BasicNameValuePair(key, params.get(key).toString()));
            }
            System.out.println(paramsList);
            post.setEntity(new UrlEncodedFormEntity(paramsList, "UTF-8"));
            post.setHeader("Content-type", "application/x-www-form-urlencoded");
            post.setHeader("User-Agent", "Mozilla/4.0 (compatible; MSIE 5.0; Windows NT; DigExt)");
            HttpResponse response = httpClient.execute(post);
            org.apache.http.HttpEntity responseEntity = response.getEntity();
            if (null != responseEntity && response.getStatusLine().getStatusCode() == HttpStatus.SC_OK) {
                inputStream = responseEntity.getContent();
                byte[] buffer = new byte[1024];
                int len;
                StringBuffer sb = new StringBuffer();
                while ((len = inputStream.read(buffer)) != -1) {
                    sb.append(new String(buffer, 0, len, "UTF-8"));
                }
                String content = sb.toString();

                if (content.length() > 0) {
                    return  content;
                }
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (ClientProtocolException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (inputStream != null) {
                try {
                    inputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return result;
    }


    private static void disableSslVerification() {
        try {
            // Create a trust manager that does not validate certificate chains
            TrustManager[] trustAllCerts = new TrustManager[]{new X509TrustManager() {
                public java.security.cert.X509Certificate[] getAcceptedIssuers() {
                    return null;
                }

                public void checkClientTrusted(X509Certificate[] certs, String authType) {
                }

                public void checkServerTrusted(X509Certificate[] certs, String authType) {
                }
            }
            };

            // Install the all-trusting trust manager
            SSLContext sc = SSLContext.getInstance("SSL");
            sc.init(null, trustAllCerts, new java.security.SecureRandom());
            HttpsURLConnection.setDefaultSSLSocketFactory(sc.getSocketFactory());

            // Create all-trusting host name verifier
            HostnameVerifier allHostsValid = new HostnameVerifier() {
                public boolean verify(String hostname, SSLSession session) {
                    return true;
                }
            };

            // Install the all-trusting host verifier
            HttpsURLConnection.setDefaultHostnameVerifier(allHostsValid);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (KeyManagementException e) {
            e.printStackTrace();
        }
    }

    private static TrustManager myX509TrustManager = new X509TrustManager() {

        @Override
        public X509Certificate[] getAcceptedIssuers() {
            return null;
        }

        @Override
        public void checkServerTrusted(X509Certificate[] chain, String authType)
                throws CertificateException {
        }

        @Override
        public void checkClientTrusted(X509Certificate[] chain, String authType)
                throws CertificateException {
        }
    };


    private static String sendPost(String url, Map<String, Object> params) {
        disableSslVerification();
        HttpsURLConnection connection = null;

        List<NameValuePair> param = new ArrayList<>();
        for (String key : params.keySet()) {
            param.add(new BasicNameValuePair(key, params.get(key).toString()));
        }

        try {
            String requestParams = URLEncodedUtils.format(param, "UTF-8");
            URL postUrl = new URL(url + "?" + requestParams);
            SSLContext sslcontext = SSLContext.getInstance("TLS");
            sslcontext.init(null, new TrustManager[]{myX509TrustManager}, null);
            connection = (HttpsURLConnection) postUrl.openConnection();
            connection.setRequestProperty("Content-Type", "application/Json; charset=UTF-8");
            connection.setConnectTimeout(30000);
            connection.setReadTimeout(30000);
            connection.setDoOutput(true);
            connection.setDoInput(true);
            connection.setRequestMethod("POST");    // 默认是 GET方式
            connection.setUseCaches(false);         // Post 请求不能使用缓存
            connection.setInstanceFollowRedirects(true);   //设置本次连接是否自动重定向
            connection.setRequestProperty("User-Agent", "Mozilla/5.0");
            connection.setRequestProperty("Content-Type", "application/x-www-form-urlencoded");
            connection.addRequestProperty("Connection", "Keep-Alive");//设置与服务器保持连接
            connection.setRequestProperty("Accept-Language", "zh-CN,zh;0.9");
            connection.connect();
            connection.setSSLSocketFactory(sslcontext.getSocketFactory());


            int responseCode = connection.getResponseCode();
            if (responseCode == 200) {
                final BufferedReader in = new BufferedReader(new InputStreamReader(connection.getInputStream()));
                String line;
                final StringBuffer stringBuffer = new StringBuffer(255);

                synchronized (stringBuffer) {
                    while ((line = in.readLine()) != null) {
                        stringBuffer.append(line);
                        stringBuffer.append("\n");
                    }
                    System.out.println(stringBuffer.toString());
                    return stringBuffer.toString();
                }
            }
            return null;

        } catch (final IOException e) {
            e.printStackTrace();
            return null;
        } catch (final Exception e1) {
            e1.printStackTrace();
            return null;
        } finally {
            if (connection != null) {
                connection.disconnect();
            }
        }
    }


}

```    
    
    
    
