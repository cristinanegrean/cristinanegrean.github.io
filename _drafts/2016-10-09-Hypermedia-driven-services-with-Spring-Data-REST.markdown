---
layout: post
author: Cristina Negrean
title: 'Expose a discoverable REST API of your domain model using HAL as media type'
image: /img/open-travel-rest-api.gif
tags: [Spring Data REST, Spring Boot, JSON HAL, Microservice Architecture, Integration, Public API]
category: Spring Data REST, Microservice Architecture
comments: true
---
## Background

REpresentational State Transfer (REST) is an architectural style inspired by the Web and nowadays Hypermedia-driven CRUD REST services are at the backbone of a microservice architecture. 
Benefits as scalability, ease of testing and deployment, as well as the chance to eliminate long-term commitment to a single technology stack, have convinced some big enterprise players 
– like [Amazon](https://www.infoq.com/news/2015/12/microservices-amazon), [Netflix](https://www.nginx.com/blog/microservices-at-netflix-architectural-best-practices/), 
[eBay](http://highscalability.com/blog/2015/12/1/deep-lessons-from-google-and-ebay-on-building-ecosystems-of.html), 
[Spotify](https://www.infoq.com/news/2015/12/microservices-spotify), [Zalando](https://tech.zalando.com/blog/goto-2016-from-monolith-to-microservices/)
– to begin transitioning their monolithic applications to microservices.
 
## Purpose
If we want to use microservices rather than monolithic applications, it’s essential that we can create a basic service with a minimum of effort. 
Writing a REST-based micro service that accesses a data model typically involves writing the Data Access Objects (DAO) to access the data repository and writing the REST methods independently. 
This often means that you are responsible for writing all of the SQL queries or JPQL. 
The [Spring Data REST](http://projects.spring.io/spring-data-rest/) framework provides you with the ability to create data repository implementations automatically at runtime, 
from a data repository interface, and expose these data repositories over HTTP as a REST service. 
Spring Data REST takes the features of [Spring HATEOAS](http://projects.spring.io/spring-hateoas/) and [Spring Data JPA](http://projects.spring.io/spring-data-jpa/) and combines them automatically.

This guide walks you through the process of creating a microservice - called [Wanderlust](https://github.com/cristinanegrean/wanderlust-open-travel-api) ;) - that exposes a discoverable REST API of an open travel domain model using HAL as media type.

## Prerequisites

* [Java 8 SDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
* [Postgresql 9](https://www.postgresql.org/)
* [Gradle](https://gradle.org/)
* [Git](https://git-scm.com/downloads)
* [cURL 7.0+](https://curl.haxx.se/download.html) (cURL is installed by default on most UNIX and Linux distributions. To install cURL on a Windows 64-bit machine, 
click [here](http://www.oracle.com/webfolder/technetwork/tutorials/obe/cloud/objectstorage/creating_containers_REST_API/files/installing_curl_command_line_tool_on_windows.html)) or [Postman](https://www.getpostman.com/)

You might want to consult the project's [README](https://raw.githubusercontent.com/cristinanegrean/wanderlust-open-travel-api/master/README.md) installation steps if you are running on OS X.

## Creating a Gradle Project with Spring Initializr

[Spring Initializr](https://start.spring.io/) provides an extensible API to generate quickstart projects.

## Configuring the build.gradle File
 {% highlight java %}  
  buildscript {
      ext {
          springBootVersion = "1.4.0.RELEASE"
      }
      repositories {
          mavenCentral()
          maven { url "https://repo.spring.io/snapshot" }
          maven { url "https://repo.spring.io/milestone" }
          maven { url "https://plugins.gradle.org/m2/" }
      }
      dependencies {
          classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
          classpath("org.kt3k.gradle.plugin:coveralls-gradle-plugin:2.6.3")
      }
  }
  
  apply plugin: "jacoco"
  apply plugin: "com.github.kt3k.coveralls"
  
  
  dependencies {
      compile("org.springframework.boot:spring-boot-starter-actuator")
      compile("org.springframework.boot:spring-boot-starter-data-jpa")
      compile("org.springframework.boot:spring-boot-starter-data-rest")
      compile("joda-time:joda-time")
      compile("org.jadira.usertype:usertype.jodatime:2.0.1") // provides mappings for joda types to use in JPA entities
  
      // optional dependency if you want to load a HAL browser 
      // when you issue an HTTP GET to the root URL: http://localhost:9000/api/opentravel/
      // compile("org.springframework.data:spring-data-rest-hal-browser")
  
      runtime("org.flywaydb:flyway-core")
      runtime("org.postgresql:postgresql")
      runtime("com.zaxxer:HikariCP")
  
      testCompile("org.springframework.boot:spring-boot-starter-test")
      testCompile("junit:junit:4.12")
  
      testRuntime("com.h2database:h2")
      // remove GSON test runtime after https://github.com/spring-projects/spring-boot/issues/6502 fix is released
      testRuntime("com.google.code.gson:gson")
      ...
  }
  
  jacoco {
      toolVersion = "0.7.7.201606060606"
  }
  
  test {
      jacoco {
          append = false
          destinationFile = file("$buildDir/jacoco.exec")
      }
  }
  
  jacocoTestReport {
      reports {
          xml.enabled true
      }
  }
  build.dependsOn jacocoTestReport
  
  coveralls {
      jacocoReportPath = "${buildDir}/reports/jacoco/test/jacocoTestReport.xml"
  }
  
  tasks.coveralls {
      onlyIf { System.env."CI" }
  }
 {% endhighlight %}
 
## Creating and Initializing Relational Datastore with Flyway
 
Database schema DDL will be generated using [Flywaydb](https://flywaydb.org) upon service start. You can check the schema version and list of DB scripts via [Spring Actuator endpoint](http://localhost:9000/flyway)
Here put schema model image

## Coding the Domain Model and the RepositoryRestResources

The key abstraction of information in REST is a resource. Any information that can be named can be a resource: a travel destination, a holiday package, a collection of holiday packages to a certain travel destination operated by a tour operator/agent.
"In other words, any concept that might be the target of an author's hypertext reference must fit within the definition of a resource. A resource is a conceptual mapping to a set of entities, not the entity that corresponds to the mapping at any particular point in time.” 
- Roy Fielding’s dissertation.

The Destination class is a simple POJO with fields, setters, and getters that represents the `destinations` table in the `wanderlust` database.
    {% highlight java %}
    @Entity
    @Table(name = "destinations")
    public class Destination extends BaseEntity {
    
        private static final long serialVersionUID = 1126074635410771215L;
    
        @NotEmpty(message = "Destination name cannot be empty")
        @Size(min = 2, max = 100, message = "Destination name must not be longer than 100 characters and shorter than 2 characters")
        @Pattern(regexp = "[a-z-A-Z- ']*", message = "Destination name has invalid characters")
        private String name;
    
        @NotEmpty(message = "How to prepare when destination country is a total surprise?")
        private String country;
    
        private String description;
    
        @ElementCollection(fetch = FetchType.EAGER)
        @CollectionTable(name = "destination_facts", joinColumns = @JoinColumn(name = "destination", referencedColumnName = "id"))
        @Column(name = "fact")
        private List<String> facts;
    
        public String getName() {
            return this.name;
        }
    
        public String getCountry() {
            return this.country;
        }
    
        public String getDescription() {
            return this.description;
        }
    
        public List<String> getFacts() {
            return this.facts;
        }
    
        public void setName(String name) {
            this.name = name;
        }
    
        public void setCountry(String country) {
            this.country = country;
        }
    
        public void setFacts(List<String> facts) {
            this.facts = facts;
        }
    
        public void setDescription(String description) {
            this.description = description;
        }
    
        public Destination() {
        }
    
        public Destination(String name, String country) {
            this.name = name;
            this.country = country;
        }
    
        public Destination(String name, String country, List<String> facts, String description) {
            this(name, country);
            this.facts = facts;
            this.description = description;
        }
    }
    {% endhighlight %}
    
## Validation


## Create The-uber—self-runnable-jar and Boot
[Spring Boot](http://projects.spring.io/spring-boot/) takes an opinionated way toward packaging and deployment of an autonomous data microservice application with an embedded application container
Describe here application.properties and the postgres profile

```
$ git clone https://github.com/cristinanegrean/rest-service-bean-validation
$ cd rest-service-bean-validation
$ ./gradlew clean build 
$ java -jar build/libs/wanderlust-1.0.0-SNAPSHOT.jar --spring.profiles.active=postgres
```

## APLS and HAL

## Testing the Application

## Source Code
You can find the full code base of my Wanderlust OpenTravel API at my public [GitHub repo](https://github.com/cristinanegrean/wanderlust-open-travel-api). 
Feel free to clone the working application and give it a spin. Looking forward to your experiences around Spring Data REST and Hypermedia-driven CRUD REST Microservices.