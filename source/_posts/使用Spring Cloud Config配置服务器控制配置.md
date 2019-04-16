---
title: 使用Spring Cloud Config配置服务器控制配置
date: 2019/02/19 19:37:07 
tags: [SpringCloud,SpringCloudConfig]
categories: SpringCloud
---

# 使用Spring Cloud Config配置服务器控制配置

## Spring Cloud Config介绍
Spring Cloud Config是Sping-Cloud下用于分布式配置管理的组件，分成了两个角色Config-Server和Config-Client；

Config-Server端（即配置服务器）集中式存储/管理配置文件，并对外提供接口方便Config-Client访问，接口使用HTTP的方式对外提供访问；Config-Server存储/管理的配置文件可以来自本地文件，远程Git仓库以及远程Svn仓库

Config-Client（需要获取配置文件的各种微服务）通过接口获取配置文件，然后可以在应用中使用；

Spring Cloud配置服务器是基于REST的应用程序，它不是独立服务器，开发人员可以选择将它嵌入现有的springboot应用程序中，也可以在嵌入它的服务器中启动新的springboot项目
<!--more-->
## 构建Spring Cloud配置服务器————Config-Server

### 添加依赖
首先为springcloud配置服务器添加maven依赖


		<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>

关于pom文件的详细信息可以在文章开头给出的GitHub中找到。

### 创建引导类
为springcloud配置服务器创建一个引导类ConfigServerApplication

    import org.springframework.boot.SpringApplication;
	import org.springframework.boot.autoconfigure.SpringBootApplication;
	import org.springframework.cloud.config.server.EnableConfigServer;

	@SpringBootApplication
	@EnableConfigServer
	public class ConfigServerApplication {

	    public static void main(String[] args) {
	        SpringApplication.run(ConfigServerApplication.class, args);
	    }

	}

`@EnableConfigServer`注解使得该应用程序成为一个spring cloud config服务器

### 添加配置文件application.yml
    spring:
  		application:
    		name: config-server
  		profiles:
    		active:
      			native
  		cloud:
    		config:
      			server:
        			native:
          				search-locations: classpath:configs/,configs/config-client
	server:
  		port: 8888

在这里指出了该服务的名称为config-server，但这类配置应该写入bootstrap.yml文件中，这里因为不会影响测试功能所以写在application.yml中。具体原因会在后面解释。

指定了配置文件为文件系统，即从配置服务器本地读取。

接着指出了本地配置文件的读取位置为`classpath:configs/,configs/config-client`

最后则是配置服务器的默认监听端口8888

#### 从Git仓库远程读取配置文件
若需要使用Git作为配置文件的远程仓库则可以将配置文件application.yml对应属性更改为

	spring.cloud.config.server.git.uri=https://github.com/XXX/XXX #表示对应的Git存储库的URL
	spring.cloud.config.server.git.searchPaths=config-client #表示Git中查找配置文件的路径
	#Git的用户名和密码，若为公开Git仓库则可以为空，若为私人Git仓库则必须填写
	spring.cloud.config.server.git.username=
	spring.cloud.config.server.git.password=
	

### 准备config-client要使用的配置文件
在本地resources中创建上一步中所涉及的目录configs，在configs下创建文件config-client-dev.yml,内容如下

	test: "This is a spring cloud config test"

这里为做测试，故自定义一个配置属性，将为config-client中注入属性做准备。
类似的配置文件可以有`config-client.yml / config-client-pro.yml / config-client-test.yml`

### 启动配置服务器
此时已经可以启动配置服务器了，可以在工程所在目录下的命令行中使用`mvn spring-boot:run`运行。
此时在浏览器中可以通过多种方式查看配置文件的信息，例如`http://localhost:8888/config-client/dev`，结果显示如下，则说明成功。

![](https://i.imgur.com/WAH5CQb.png)

支持的查看方式有如下几种：

	/{application}/{profile}[/{label}]
	/{application}-{profile}.yml
	/{label}/{application}-{profile}.yml
	/{application}-{profile}.properties
	/{label}/{application}-{profile}.properties

- application对应相应服务的名称，本案例是指config-client，后文创建config-client服务时会在bootstrap.yml文件中指出；
- profile表示使用哪种环境的配置文件，这里可以是dev，test，pro，甚至是default表示上面提到的config-client.yml文件，即没有代表环境的后缀；
- label是可选的标签，git仓库默认值master，svn仓库默认值是trunk；

## config-client端的搭建

### 引入依赖
在新的config-client项目的pom文件中添加如下依赖

		<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-client</artifactId>
        </dependency>

### 创建配置文件bootstrap.yml
	spring:
	  application:
	    name: config-client
	  cloud:
	    config:
	      uri: http://localhost:8888
	      fail-fast: true
	  profiles:
	    active: dev
	server:
	  port: 8080


在配置中分别指出了该应用程序的名称和配置文件所在路径，以及配置文件的环境为dev即开发环境。即对应上文中描述的几种类型。开放了监听端口8080

注意这里的spring.application.name属性和spring.cloud.config.uri/fail-fast属性的配置写在bootstrap.yml文件中是规范的，若将spring.cloud.config.uri/fail-fast属性写入application.yml文件中，则会导致项目在启动过程中报错，原因就是找不到对应的配置文件，在这里就是上文中的config-client-dev.yml。

### 写入bootstrap.yml的原因
将spring.application.name属性和spring.cloud.config.uri属性的配置写在bootstrap.yml文件原因是在启动过程中，加载bootstrap.yml文件的顺序要优先于application.yml。

详细原因可以参考这篇文章：https://www.cnblogs.com/BlogNetSpace/p/8469033.html

### 在config-client引导类中添加API接口

	import org.springframework.beans.factory.annotation.Value;
	import org.springframework.boot.SpringApplication;
	import org.springframework.boot.autoconfigure.SpringBootApplication;
	import org.springframework.web.bind.annotation.GetMapping;
	import org.springframework.web.bind.annotation.RestController;
	
	@SpringBootApplication
	@RestController
	public class ConfigClientApplication {
	
	    @Value("${test}")
	    String test;
	
	    @GetMapping("/test")
	    public String test(){
	        return test;
	    }
	    public static void main(String[] args) {
	        SpringApplication.run(ConfigClientApplication.class, args);
	    }
	
	}

@Value读取配置文件中的test属性，即上文在config-server中的配置文件config-client-dev.yml中的test属性。

## 测试
![](https://i.imgur.com/D6cOmpb.png)

可以看到浏览器中返回的结果，与我们想要读取的配置文件中的内容一致。

这就是Spring Cloud Config 的一个简单演示，仅仅通过一些简单步骤就实现了配置文件和应用程序的完全分离，这在实际生产中的作用是巨大的。

个人博客地址:https://alexaccele.github.io/


