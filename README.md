# ELK-Stack-Logging

How to use the ELK Stack with Spring Boot - An example

Reference video - https://www.youtube.com/watch?v=O5ou6lBwWYw&t=618s
Reference website - https://www.javainuse.com/spring/springboot-microservice-elk 

Steps to use :
Go on to the website for Elastic Stack[https://www.elastic.co/downloads/elasticsearch]
Download Elastic search, Logstash and Kibana pertaining to the operating system that you are using.[Here, it is written according to Windows]
Run elastic search from the command prompt by cd to the bin location of elastic search on the local system -> run the elasticsearch.bat file.
To check if it is running successfully, type  http://localhost:9200/  in the browser.
Run kibana from the command prompt by cd to the bin location of kibana on the local system -> run the kibana.bat file.
To check if it is running successfully, type http://localhost:5601/ in the browser.
Create a configuration file called logstash.conf in the bin folder of logstash.
We will be creating a simple spring boot application to generate logs. Open Eclipse for the environment.
Define the pom.xml file as follows -
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.javainuse</groupId>
	<artifactId>spring-boot-elk</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>spring-boot-elk</name>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.6.RELEASE</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>
<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>

Create the Spring Boot Bootstrap class as follows - 
package com.javainuse;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class HelloWorldSpringBootApplication {

	public static void main(String[] args) {
		SpringApplication.run(HelloWorldSpringBootApplication.class, args);
	}
}

Define the controller to expose REST API.
package com.javainuse;

import java.io.PrintWriter;
import java.io.StringWriter;
import java.util.Date;

import org.apache.log4j.Level;
import org.apache.log4j.Logger;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RestController
class ELKController {
	private static final Logger LOG = Logger.getLogger(ELKController.class.getName());

	@Autowired
	RestTemplate restTemplete;

	@Bean
	RestTemplate restTemplate() {
		return new RestTemplate();
}

	@RequestMapping(value = "/elk")
	public String helloWorld() {
		String response = "Welcome to JavaInUse" + new Date();
		LOG.log(Level.INFO, response);

		return response;
	}

	@RequestMapping(value = "/exception")
	public String exception() {
		String response = "";
		try {
			throw new Exception("Exception has occured....");
		} catch (Exception e) {
			e.printStackTrace();
			LOG.error(e);

			StringWriter sw = new StringWriter();
			PrintWriter pw = new PrintWriter(sw);
			e.printStackTrace(pw);
			String stackTrace = sw.toString();
			LOG.error("Exception - " + stackTrace);
			response = stackTrace;
}

		return response;
	}
}
Specify the name and location of the log file to be created in the application.properties file
logging.file=C:/elk/spring-boot-elk.log

Configure the logstash pipeline. Add a file in the logstash folder ->bin ->logstash.conf
input {
  file {
    type => "java"
    path => "C:/elk/spring-boot-elk.log"
    codec => multiline {
      pattern => "^%{YEAR}-%{MONTHNUM}-%{MONTHDAY} %{TIME}.*"
      negate => "true"
      what => "previous"
    }
  }
}
 
filter {
  #If log line contains tab character followed by 'at' then we will tag that entry as stacktrace
  if [message] =~ "\tat" {
    grok {
      match => ["message", "^(\tat)"]
      add_tag => ["stacktrace"]
    }
  }
 
}
 
output {
  stdout {
    codec => rubydebug
  }
 
  # Sending properly parsed log events to elasticsearch
  elasticsearch {
    hosts => ["localhost:9200"]
  }
}

Start the logstash using the command prompt as follows - 
logstash -f logstash.conf
Start the spring boot application by running the HelloWorldSpringBootApplication as a java application. Logs will be generated in C:/elk folder.

Goto localhost:8080/elk and localhost:8080/exception to generate logs

Logs generated will be visible in the file present in the elk folder

Go to Kibana UI console in local host and create and Index pattern to see the indexed data
