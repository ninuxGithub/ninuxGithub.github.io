---
title: 常用的命令
author: ninuxGithub
layout: post
date: 2023-9-19 18:33:34
description: "常用的命令"
tag: other
---

### 常用的一些命令


```shell
git 工作命令使用
	git tag ： 查看tags
	git tag 1.0.20 ： 发布前建立一个tag
	git push origin 1.0.20 : 推送到远程
	git checkout 1.0.20 获取tag分支
	
	git创建分支：
	git checkout -b name ： 创建新的分支

	git tag删除：
	git tag -d xxx :删除本地的tag
	git push origin :refs/tags/1.1.6.6 删除远程的tag


	git检出分支：
	git checkout -a : 显示远程的所有的分支    

	git 分支删除：
    git push origin --delete branchName : 删除远程的分支    
    git branch -d branchName : 删除本地的分支

    git 修改branch的名称
    1.修改本地的分支名称
    git branch -m oldbranch newbranch
    2.删除远程的分支
    git push --delete origin oldbranch
    3.将本地的分支推送到远程
    git push origin newbranch


    git 分支合并： git merge --no-ff --no-commit branchX


    git 将文件提交到其他分支的操作,在没有进行commit之前可以进行一下操作 
		1、通过git stash将工作区恢复到上次提交的内容，同时备份本地所做的修改
		git stash
		2、然后切换至B分支
		git checkout B
		3、从git栈中获取到最近一次stash进去的内容，恢复工作区的内容，获取之后，会删除栈中对应的stash
		git stash pop
		4、在进行正常的提交代码步骤即可
		git add /src/main/..
		5、git commit -m "功能开发"
		6、git pull origin   分支名称
		7、git push origin   分支名称

	git 修改项目的地址：	
	git remote set-url origin https://git.coding.net/B/demo.git  ： 修改本地项目git的提交地址

	
    maven 工作中的使用命令：
	mvn clean install -Dmaven.test.skip=true 
	mvn clean install -P dev -Dmaven.test.skip=true
	mvn clean install package -P prd-pod :项目打包
	mv */target/*endpoint*war* . 删除没用的war文件



	项目覆盖率
	mvn -U org.jacoco:jacoco-maven-plugin:0.8.5:prepare-agent verify org.jacoco:jacoco-maven-plugin:0.8.5:report install -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.validity.dates=true

	idea maven importing 配置： -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true

	maven 跳过证书验证
	mvn clean install -Dmaven.test.skip=true -Dmaven.multiModuleProjectDirectory=$MAVEN_HOME -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.validity.dates=true

	跳过test
	mvn clean install dependency:sources -DdownloadSources=true -DdownloadJavadocs=true  -Dmaven.test.skip=true -Dmaven.multiModuleProjectDirectory=$MAVEN_HOME -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.validity.dates=true

	mvnd clean install dependency:sources -DdownloadSources=true -DdownloadJavadocs=true  -Dmaven.test.skip=true -Dmaven.multiModuleProjectDirectory=$MAVEN_HOME -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.validity.dates=true

	maven下载jar源码
	mvn dependency:sources -DdownloadSources=true -DdownloadJavadocs=true -Dmaven.test.skip=true -Dmaven.multiModuleProjectDirectory=$MAVEN_HOME -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.validity.dates=true

	覆盖率报告
	mvn -U org.jacoco:jacoco-maven-plugin:0.8.5:prepare-agent verify org.jacoco:jacoco-maven-plugin:0.8.5:report install -Dmaven.multiModuleProjectDirectory=$MAVEN_HOME -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.validity.dates=true


```


### http curl 


```curl
curl -v 'http://xxx:x/' \
--header 'Content-Type: text/xml' \
--header 'Accept: text/xml' \
--retry-connrefused \
--retry 10 \
--connect-timeout 20 \
--data '<SOAP-ENV:Envelope xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/">
    <SOAP-ENV:Header>
        <Login>
            <username>xxx</username>
            <password>xxx</password>
        </Login>
    </SOAP-ENV:Header>
    <SOAP-ENV:Body>
        <CacheRequest xmlns="http://tempuri.org">
            <Parameters>
                <FirstArrivalDate>2023-08-18</FirstArrivalDate>
                <LastArrivalDate>2023-08-18</LastArrivalDate>
                <CellCodes>
                    <CellCode>XMLMD</CellCode>
                    <CellCode>XMLBB</CellCode>
                </CellCodes>
                <StayNights>
                    <Nights>1</Nights>
                </StayNights>
                <Hotels>
                    <HotelCode>xx</HotelCode>
                </Hotels>
                <CompanyName/>
            </Parameters>
        </CacheRequest>
    </SOAP-ENV:Body>
</SOAP-ENV:Envelope>'


```
