---
title: Jxls export excel
author: ninuxGithub
layout: post
date: 2022-12-22 11:21:04
description: "Jxls export excel"
tag: centos
---
### Jxls export excel 
    直接上一个demo
    
![template.xlsx](/images/posts/premierinn-product-template.xlsx)


```java
@RestController
public class TestController {

    @RequestMapping("/test")
    public String test() throws IOException {
        // template
        String templateResource = "templates/template.xlsx";
        try (InputStream template = getClass().getClassLoader().getResourceAsStream(templateResource)) {
            // output to
            String property = System.getProperty("java.io.tmpdir");
            File out = new File(property+"/common-functions-result.xlsx");
            if (out.exists()) out.delete();
            try (OutputStream output = new FileOutputStream(out)) {
                // data
                PoiContext context = new PoiContext();
                context.putVar("its", mockDataList());
                //-- inject common-functions
                //context.putVar("fn", CommonFunctions.getSingleton());

                //-- other data
                context.putVar("num", new BigDecimal("123.456"));
                context.putVar("str", "123");

                // render
                JxlsHelper.getInstance().processTemplate(template, output, context);
                return out.getAbsolutePath();
            } catch (Exception ex) {
                throw new RuntimeException(ex.getMessage());
            }
        }
    }


    private List<Map<String, Object>> mockDataList() {
        List<Map<String, Object>> dataList = Lists.newArrayList();
        for (int i = 0; i < 10; i++) {
            Map<String, Object> rowMap = new HashMap<>();
            rowMap.put("hotel", "1434");
            rowMap.put("ratePlan", "mock" + i);
            rowMap.put("status", "+");
            dataList.add(rowMap);
        }
        return dataList;
    }
}
```

### 解决报错
    如果使用了动态环境配置：  filtering= true 
    就会将模板修改了，报错

    NotOfficeXmlFileException: No valid entries or contents found, this is not a valid OOXML (Office Open XML) file
    

    解决办法如下， 修改pom 配置， pom 会在build时候将模板修改了， 观察模板的kb大小就可以发现

```xml
<project>
    <build>
        <resources>
            <resource>
                <directory>src/main/profiles/${env}</directory>
            </resource>
            <resource>
                <directory>src/main/java</directory>
            </resource>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
                <includes>
                    <include>**/*.*</include>
                </includes>
                <excludes>
                    <exclude>templates/*.xlsx</exclude>
                </excludes>
            </resource>
            
            <!--这个解决压缩-->
            <resource>
                <directory>src/main/resources</directory>
                <includes>
                    <include>templates/*.xlsx</include>
                </includes>
            </resource>
        </resources>
        <testResources>
            <testResource>
                <directory>src/test/java</directory>
                <excludes>
                    <exclude>**/*.java</exclude>
                </excludes>
            </testResource>
            <testResource>
                <directory>src/test/resources</directory>
            </testResource>
        </testResources>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.2.3</version>
                <configuration>
                    <failOnMissingWebXml>false</failOnMissingWebXml>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-resources-plugin</artifactId>
                <version>2.6</version>
                <configuration>
                    <encoding>UTF-8</encoding>
                    <nonFilteredFileExtensions>
                        <!--这个解决压缩-->
                        <nonFilteredFileExtension>xlsx</nonFilteredFileExtension>
                    </nonFilteredFileExtensions>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```