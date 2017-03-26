---
layout: post
author: Cristina Negrean
title: 'Expose a discoverable REST API of your domain model using HAL JSON as media type'
image: /img/open-travel-rest-api.gif
tags: [Spring Data REST, Spring Initializr, Spring Boot, Flyway, Spring Data JPA, ALPS, JSON HAL]
category: Spring Data REST, Back-end API, Wanderlust
comments: true
---
## Background

[REpresentational State Transfer (REST)](https://martinfowler.com/articles/richardsonMaturityModel.html) is an architectural style inspired by the Web and nowadays Hypermedia-driven CRUD REST services are at the backbone of a [microservice](https://martinfowler.com/articles/microservices.html) internal architecture.
Benefits as scalability, ease of testing and deployment, as well as the chance to eliminate long-term commitment to a single technology stack, have convinced some big enterprise players
– like [Amazon](https://www.infoq.com/news/2015/12/microservices-amazon), [Netflix](https://www.nginx.com/blog/microservices-at-netflix-architectural-best-practices/),
[eBay](http://highscalability.com/blog/2015/12/1/deep-lessons-from-google-and-ebay-on-building-ecosystems-of.html),
[Spotify](https://www.infoq.com/news/2015/12/microservices-spotify)
– to begin transitioning their monolithic applications to [microservices](https://martinfowler.com/articles/microservices.html).

## Purpose
If we want to accelerate microservices adoption rather than code at monolithic applications, it’s essential that we can create a basic service with a minimum of effort.
Writing a REST-based microservice that accesses a data model typically involves writing the Data Access Objects (DAO) to access the data repository and writing the REST methods independently.
This often means that you are responsible for writing all of the SQL queries or JPQL(Java Persistence Query Language).
The [Spring Data REST](http://projects.spring.io/spring-data-rest/) framework provides you with the ability to create data repository implementations automatically at runtime,
from a data repository interface, and expose these data repositories over HTTP as a REST service.
It takes the features of [Spring HATEOAS](http://projects.spring.io/spring-hateoas/) and [Spring Data JPA](http://projects.spring.io/spring-data-jpa/) and combines them automatically.

This guide will give you a high-level walk through of my process of creating a microservice - called [Wanderlust](https://github.com/cristinanegrean/wanderlust-open-travel-api) - that exposes a discoverable REST API of a simple open-travel domain model using [JSON HAL](https://tools.ietf.org/html/draft-kelly-json-hal-08) as media type.

## Minimum Viable Product
* list, search by country or name travel destinations
* create, validate, update and delete a travel destination
* list, search travel agents by name
* create, validate, update and delete a travel agent
* list, search holiday packages by destination country
* create, validate a holiday package for a travel destination
* pagination and sorting for list and search operations

## Technology Stack

* [Java 8 SDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
* [Postgresql 9](https://www.postgresql.org/)
* [Gradle](https://gradle.org/)
* [Git](https://git-scm.com/downloads)
* [cURL 7.0+](https://curl.haxx.se/download.html) (cURL is installed by default on most UNIX and Linux distributions. To install cURL on a Windows 64-bit machine,
click [here](http://www.oracle.com/webfolder/technetwork/tutorials/obe/cloud/objectstorage/creating_containers_REST_API/files/installing_curl_command_line_tool_on_windows.html)) or [Postman](https://www.getpostman.com/)
* Your favorite Java code editor. At the moment my favorite one is [IntelliJ IDEA](https://www.jetbrains.com/idea/)

When running on OS X, installion steps can be found in the project's [README](https://raw.githubusercontent.com/cristinanegrean/wanderlust-open-travel-api/master/README.md).

## "Bootiful Wanderlust" with Spring Boot and Spring Initializr

Sometimes the hardest part of any project is getting started. You have to setup a directory structure for various project artifacts, create a build file and populate it with all library dependencies. This is where [Spring Initializr](https://start.spring.io/) comes into handy. [Spring Initializr](https://start.spring.io/) is ultimately a web application that can generate a [Spring Boot](http://projects.spring.io/spring-boot/) project structure for you. It doesn’t generate any application code, but it will give you a basic project structure and either a [Maven](https://maven.apache.org/) or a [Gradle](https://gradle.org/) build specification to build your code with. All you need to do is write the application code.
[Spring Boot](http://projects.spring.io/spring-boot/), on the other hand, helps package the autonomous data microservice application as an uber-self-runnable-JAR (Java Application Archive) with an embedded application container ([Apache Tomcat](http://tomcat.apache.org/) - defaultly autoconfigured, [Jetty](http://www.eclipse.org/jetty/), (JBoss Undertow)[http://undertow.io/]). [Spring Boot](http://projects.spring.io/spring-boot/) also became "de-facto" way to create Spring applications.

### Why Spring Boot?
* It's suitable for Cloud Native Applications that follow the [12factor](http://12factor.net) patterns (developed by the Netflix engineering team)
* Productivity increases by reducing time of development and deployment
* Enterprise-production-ready Spring applications
* Non-functional requirements, such as Spring Boot Actuator (a module that brings metrics, health checks and management features)

The most straightforward way to use the [Spring Initializr](https://start.spring.io/) is to point your web browser to [http://start.spring.io](http://start.spring.io). Fill in basic project information on the left side of the form, search and add dependencies listed in the figure below on the right side of the form and you should be all set to download the project template.

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/wanderlust-api/SpringInitializr.png" alt="Spring Initializr"/>

The <i class="blue">Generate Project</i> button will download the template project.

## "REST-Assured": Building Continuous Delivery confidence with Test Driven Development

 Before even starting to write a line of code, is a good practice to think about the approach to take to automate code quality checks so that you can code and refactor with confidence. And at the same time strive to minimize the number of defects!
 While I will show you how to unit and integration test the back-end API in a later post, I'll start by adding the [tooling](https://github.com/jacoco/jacoco) that calculates the percentage of code accessed by tests to the project build specification file (build.gradle).
 The goal is to have each time when building the project, JaCoCo (Java Code Coverage) produce an XML report in the build directory:  `wanderlust-open-travel-api/build/reports/jacoco/test/jacocoTestReport.xml` to be used with the Continuous Integration (CI) server of your choice - for me that is [Travis](https://travis-ci.org/) - to visually quantify test coverage and its [evolution](https://coveralls.io/github/cristinanegrean/wanderlust-open-travel-api) between chronological software increments.

 <img class="img-responsive" src="{{ site.baseurl }}/img/posts/wanderlust-api/TravisCIBadges.png" alt="Test Coverage Badge with JaCoCo Report XML, GitHub, Travis CI and Coveralls"/>

 `wanderlust-open-travel-api/build/reports/jacoco/test/html/index.html`

  <img class="img-responsive" src="{{ site.baseurl }}/img/posts/wanderlust-api/Jacoco.png" alt="Drill-down on test coverage per code package"/>

 Go ahead and open the project template structure generated with Spring Initializr in your favorite IDE and add the <i class="blue">jacoco</i> and <i class="blue">coveralls</i> dependencies to the Gradle Build specification. The result should look similar to the listing below.

Listing 1: [build.gradle](https://github.com/cristinanegrean/wanderlust-open-travel-api/blob/master/build.gradle)

{% highlight gradle %}  
group 'cristina.tech'
version '1.0.0-SNAPSHOT'

repositories {
    mavenCentral()
}

buildscript {
    ext {
        springBootVersion = '1.5.2.RELEASE'
    }
    repositories {
        mavenCentral()
        maven { url "https://repo.spring.io/snapshot" }
        maven { url "https://repo.spring.io/milestone" }
        maven { url "https://plugins.gradle.org/m2/" }
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath("org.kt3k.gradle.plugin:coveralls-gradle-plugin:2.8.1") //Send coverage data to coveralls.io
    }
}

apply plugin: 'java'
apply plugin: 'idea'
apply plugin: 'org.springframework.boot'
apply plugin: 'jacoco'
apply plugin: 'com.github.kt3k.coveralls'


jar {
    baseName = 'wanderlust'
    version = '1.0.0-SNAPSHOT'
}

sourceCompatibility = 1.8
targetCompatibility = 1.8

repositories {
    mavenCentral()
    maven { url "https://repo.spring.io/snapshot" }
    maven { url "https://repo.spring.io/milestone" }
}

dependencies {
    compile('org.springframework.boot:spring-boot-starter-actuator') //Production ready features to help you monitor and manage your application
    compile('org.springframework.boot:spring-boot-starter-data-jpa') //Java Persistence API including spring-data-jpa, spring-orm and Hibernate
    compile('org.springframework.boot:spring-boot-starter-data-rest') //Exposing Spring Data repositories over REST via spring-data-rest-webmvc
    compile('com.fasterxml.jackson.datatype:jackson-datatype-jsr310') //Instruct Jackson JSON serializer/deserializer to work with Java 8 Date & Time API

    testCompile('org.springframework.boot:spring-boot-starter-test') //Imports both Spring Boot test modules as well has JUnit, AssertJ, Hamcrest, Mockito, JSONassert, JsonPath 

    runtime('org.springframework.data:spring-data-rest-hal-browser')  //Browsing Spring Data REST repositories in your browser
    runtime('org.flywaydb:flyway-core') //Flyway Database Migrations library
    runtime('org.postgresql:postgresql') //PostgreSQL jdbc driver
    runtime('com.zaxxer:HikariCP') //high-performance, production-quality JDBC connection pool

    testRuntime('com.h2database:h2') //H2 in-memory database for tests (with embedded support)
}

jacoco {
    toolVersion = "0.7.9"
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

task wrapper(type: Wrapper) {
    gradleVersion = '3.4.1'
}

tasks.coveralls {
    onlyIf { System.env.'CI' }
}
{% endhighlight %}

The Gradle IDEA plugin - `apply plugin: 'idea'` - generates files for working with the project in IntelliJ IDEA. To generate the project files, simply open a terminal window, navigate to the project root directory and run the `idea` task.

```
Cristinas-iMac:wanderlust-open-travel-api cristina$ ./gradlew idea
:ideaModule
:ideaProject
:ideaWorkspace
:idea
```

## Creating and Initializing PostgreSQL Relational Data Store with Flyway and SQL

One of the design principles of microservices architecture is to have a separate data store for each microservice. If you got so far, you should already have a [PostgreSQL](https://www.postgresql.org/) data store instance running on your machine with a created database named <i class="blue">wanderlust</i>, as well as USERNAME_POSTGRES and PWD_POSTGRES environment variables configured and resolvable according to the project's [README](https://raw.githubusercontent.com/cristinanegrean/wanderlust-open-travel-api/master/README.md)

To check whether your connection to the database is working properly, you can open a command line tool and type in:
```bash
psql -h localhost wanderlust
```

You should see an output similar with to below:

```
psql (9.5.3)
Type "help" for help.

wanderlust=# help
You are using psql, the command-line interface to PostgreSQL.
```
Once that is working, we need to instruct [Spring Boot](http://projects.spring.io/spring-boot/) to use [PostgreSQL](https://www.postgresql.org/) for storing the microservice data. As you might already know, Spring Boot is using by default [H2](http://www.h2database.com/html/main.html) in-memory database. In-memory databases are useful in the early development stages in local environments, but they have lots of restrictions. As the development progresses, you would most probably require a persistent data store to develop and test your application before deploying it to use a production database server. Spring Boot integrates with all major RDBMS and NoSQL databases as: [MySQL](https://www.mysql.com/), [PostgreSQL](https://www.postgresql.org/), Oracle, [MongoDB](https://www.mongodb.com/), [Cassandra](http://cassandra.apache.org/), even Microsoft SQL Server. Isn't Microsoft taking major steps towards open source? :)

To use PostgreSQL, you will need the proper database drivers. The required JARs should already be provisioned due to the fact that we have included it as a dependency with Spring Initializr. Furthermore we would like to use PostgreSQL with a lightweight, high-performance JDBC connection pool technology like [HikariCP](https://brettwooldridge.github.io/HikariCP/) to prepare the service for production-grade environment concurrency.

This last one can be added to <i class="blue">build.gradle</i> file:

{% highlight gradle %}
  runtime('com.zaxxer:HikariCP')
{% endhighlight %}

Additionally we will check that Spring Boot's AutoConfiguration feature has yielded the proper connection properties and tweak them accordingly.

<div class="col-lg-6 col-md-6 col-sm-6 col-xs-12">
  <img class="img-responsive" src="{{ site.baseurl }}/img/posts/wanderlust-api/datasource_config.png" alt="Spring Boot Properties for postgres Profile "/>
</div>

Listing 2:
[application-postgres.properties](https://github.com/cristinanegrean/wanderlust-open-travel-api/blob/master/src/main/resources/application-postgres.properties)

<div class="col-lg-6 col-md-6 col-sm-6 col-xs-12">

{% highlight properties %}
# Datasource configuration
spring.datasource.url=jdbc:postgresql://localhost:5432/wanderlust
spring.datasource.driverClassName=org.postgresql.Driver
spring.datasource.username=${USERNAME_POSTGRES}
spring.datasource.password=${PWD_POSTGRES}
spring.datasource.hikari.maximumPoolSize=10
spring.datasource.hikari.connectionTimeout=1500
{% endhighlight %}

</div>

Finally we have come to the point where we can model the <i class="blue">wanderlust</i> database.

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/wanderlust-api/schema.png" alt="Spring Boot Properties for postgres Profile "/>

Listing 3: [src/main/resources/db/migration/V0__DDL_wanderlust.sql](https://github.com/cristinanegrean/wanderlust-open-travel-api/blob/master/src/main/resources/db/migration/V0__DDL_wanderlust.sql)

```sql
DROP TABLE IF EXISTS schema_version;
DROP TABLE IF EXISTS travel_agent_holiday_packages;
DROP TABLE IF EXISTS travel_agents;
DROP TABLE IF EXISTS holiday_packages;
DROP TABLE IF EXISTS destination_facts;
DROP TABLE IF EXISTS destinations;

CREATE TABLE destinations (
  id          SERIAL    NOT NULL PRIMARY KEY,
  name        TEXT      NOT NULL UNIQUE,
  country     TEXT      NOT NULL,
  description TEXT,
  created_at  TIMESTAMP NOT NULL DEFAULT now(),
  modified_at TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE destination_facts (
  destination SERIAL REFERENCES destinations (id) ON DELETE CASCADE,
  fact        TEXT NOT NULL
);

CREATE TABLE travel_agents (
  id             SERIAL    NOT NULL PRIMARY KEY,
  name           TEXT      NOT NULL UNIQUE,
  country        TEXT      NOT NULL,
  street_address TEXT,
  postal_code    TEXT      NOT NULL,
  city           TEXT,
  email          TEXT      NOT NULL UNIQUE,
  phone          TEXT,
  fax            TEXT,
  website        TEXT,
  created_at     TIMESTAMP NOT NULL DEFAULT now(),
  modified_at    TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE holiday_packages (
  id                    SERIAL    NOT NULL PRIMARY KEY,
  destination           SERIAL    NOT NULL REFERENCES destinations (id) ON DELETE CASCADE,
  start_on              DATE,
  days_count            SMALLINT,
  depart_from           TEXT,
  price                 NUMERIC,
  flight_included       BOOLEAN   NOT NULL DEFAULT FALSE,
  accomodation_included BOOLEAN   NOT NULL DEFAULT FALSE,
  package_info          TEXT,
  created_at            TIMESTAMP NOT NULL DEFAULT now(),
  modified_at           TIMESTAMP NOT NULL DEFAULT now(),
  UNIQUE (destination, start_on, days_count, depart_from, price, flight_included, accomodation_included)
);

CREATE TABLE travel_agent_holiday_packages (
  travel_agent    SERIAL NOT NULL REFERENCES travel_agents (id) ON DELETE CASCADE,
  holiday_package SERIAL NOT NULL REFERENCES holiday_packages (id) ON DELETE CASCADE,
  PRIMARY KEY (travel_agent, holiday_package)
)
```

Listing 4: [src/main/resources/db/migration/V1__DML_wanderlust.sql](https://github.com/cristinanegrean/wanderlust-open-travel-api/blob/master/src/main/resources/db/migration/V1__DML_wanderlust.sql)

```sql
INSERT INTO destinations
VALUES ((SELECT nextval('destinations_id_seq')), 'Quintessential Japan', 'Japan', 'Life-changing-experience');
INSERT INTO destination_facts VALUES
  ((SELECT currval('destinations_id_seq')),
   'There are over 5.5 million vending machines in Japan selling everything from umbrellas and cigarettes to canned bread and hot noodles.'),
  ((SELECT currval('destinations_id_seq')),
   'Japan''s birth rate has plummeted so significantly that adult nappies (diapers) outsell babies'' nappies, which are also sold in vending machines.'),
  ((SELECT currval('destinations_id_seq')),
   'It is estimated that more paper is used for manga comics than for toilet paper in Japan. (Surprise: both are sold in vending machines as well.)'),
  ((SELECT currval('destinations_id_seq')),
   'One of the world''s most famous pilgrimage routes after the Camino de Santiago is Japan''s Kumano Kodo near Osaka.');

INSERT INTO travel_agents (id, name, country, postal_code, email, website) VALUES
  ((SELECT nextval('travel_agents_id_seq')), 'Shoestring', 'NL', '1114 AA', 'info@shoestring.nl',
   'https://shoestring.nl');

INSERT INTO holiday_packages (id, destination, depart_from, package_info) VALUES
  ((SELECT nextval('holiday_packages_id_seq')), 1, 'Amsterdam Schipol', 'New horizons in the land of the rising sun');

INSERT INTO travel_agent_holiday_packages
VALUES ((SELECT currval('travel_agents_id_seq')), (SELECT currval('holiday_packages_id_seq')));
```

Database scripts will be run automatically by [Flywaydb](https://flywaydb.org) upon service start. Before starting the service, I will change the default server port. Spring Boot default is 8080 and it supplies you with the `server.port` property for configuring the port on which our Spring Boot application should run.

Listing 5: [src/main/resources/application.properties](https://github.com/cristinanegrean/wanderlust-open-travel-api/blob/master/src/main/resources/application.properties)

{% highlight properties %}
info.app.name=wanderlust
info.app.description=Create a RESTful API with Spring Data REST
info.app.version=1.0.0
logging.level.org.springframework.data.jpa=INFO
logging.level.org.springframework.data.rest=INFO
flyway.validate.on.migrate=false
spring.data.rest.base.path=/api/opentravel
server.port=9000
spring.jackson.serialization.WRITE_DATES_AS_TIMESTAMPS = false
{% endhighlight %}

To start the service, open a terminal window and type `./gradlew bootRun` :

```
$ Cristinas-iMac:wanderlust-open-travel-api cristina$ ./gradlew bootRun
```

You may be wondering: Wait a minute, what is this `bootRun` command? How can I install it? If you revisit <i class="blue">Listing 1: build.gradle</i>, line
`apply plugin: 'org.springframework.boot'` provides Spring Boot support in Gradle. [The Spring Boot Gradle Plugin](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/html/build-tool-plugins-gradle-plugin.html) allows you to package executable jar or war archives. The default option for packaging in Spring Initializr is jar (Java ARchive) and that is also reflected in <i class="blue">Listing 1: build.gradle</i>

{% highlight gradle %}
jar {
    baseName = 'wanderlust'
    version = '1.0.0-SNAPSHOT'
}
{% endhighlight %}

Listing 6: [src/main/java/cristina/tech/blog/travel/WanderlustApplication.java](https://github.com/cristinanegrean/wanderlust-open-travel-api/blob/master/src/main/java/cristina/tech/blog/travel/WanderlustApplication.java)

```java
package cristina.tech.blog.travel;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.data.jpa.convert.threeten.Jsr310JpaConverters;

@SpringBootApplication
@EntityScan(
        basePackageClasses = { WanderlustApplication.class, Jsr310JpaConverters.class }
)
public class WanderlustApplication {

    public static void main(String[] args) {
        SpringApplication.run(WanderlustApplication.class, args);
    }
}
```

Listing 6 shows you the entry point for a Spring Boot application in Java. The `main()` method uses Spring Boot’s SpringApplication.run() method to launch an application. Did you notice that there wasn’t a single line of XML? No web.xml file either. This web application is 100% pure Java and you didn’t have to deal with configuring any plumbing or infrastructure. The `run()` method call accepts two parameters - the class annotated with `@SpringBootApplication` and the application's arguments.

You can check the schema version and installation details via [Spring Actuator endpoint](http://localhost:9000/flyway) or command line:

```sql
wanderlust=# select * from schema_version;
version_rank | installed_rank | version |  description   | type |         script        |  checksum   | installed_by |        installed_on        | execution_time | success
------------+----------------+---------+----------------+------+------------------------+-------------+--------------+----------------------------+----------------+---------
          1 |              1 | 0       | DDL wanderlust | SQL  | V0__DDL_wanderlust.sql | -1654223572 | cristina     | 2016-10-03 12:31:50.914628 |             47 | t
          2 |              2 | 1       | DML wanderlust | SQL  | V1__DML_wanderlust.sql |  -674519781 | cristina     | 2016-10-03 12:31:51.035453 |             12 | t
(2 rows)
```

[Flywaydb](https://flywaydb.org) is a great tool that makes database migrations automation easy. It deserves in itself a separate blog post. I particularly like [this one](http://zoltanaltfatter.com/2016/03/14/database-migration-with-flyway/)!

## Coding the Domain Model

The key abstraction of information in REST is a resource. Any information that can be named can be a resource: a travel destination, a holiday package, a tour operator/agent.
<i class="blue">"In other words, any concept that might be the target of an author's hypertext reference must fit within the definition of a resource. A resource is a conceptual mapping to a set of entities, not the entity that corresponds to the mapping at any particular point in time.”</i> - Roy Fielding’s dissertation

In this blog post I will exemplify `Destination` class which is a simple model of a travel destination resource, with four basic attributes: the name, country, description and off course some funny facts about the destination. Geodata (GIS) and photos anyone? ;) It is annotated with `@Entity`, indicating that it is a [JPA](http://www.oracle.com/technetwork/java/javaee/tech/persistence-jsp-140049.html) entity, mapping to persistent storage: `wanderlust.destinations` and `wanderlust.destination_facts` schema tables.

The `Destination` class encapsulates validation constraints with the use of [Hibernate Validator](http://hibernate.org/validator/) `@NotEmpty` and
[JSR 303/349 Bean Validation](http://beanvalidation.org/) `@Size`,`@Pattern` annotations. As you can see the name and the country of the destination are mandatory fields.
I have used the message attribute of the bean validation annotations to provide API clients with descriptive validation error messages.

Listing 7: [src/main/java/cristina/tech/blog/travel/domain/Destination.java](https://github.com/cristinanegrean/wanderlust-open-travel-api/blob/master/src/main/java/cristina/tech/blog/travel/domain/Destination.java)

```java
package cristina.tech.blog.travel.domain;

import org.hibernate.validator.constraints.NotEmpty;

import javax.persistence.CollectionTable;
import javax.persistence.Column;
import javax.persistence.ElementCollection;
import javax.persistence.Entity;
import javax.persistence.FetchType;
import javax.persistence.JoinColumn;
import javax.persistence.Table;
import javax.validation.constraints.Pattern;
import javax.validation.constraints.Size;
import java.util.List;

@Entity
@Table(name = "destinations")
public class Destination extends AbstractEntity {

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

    /** Getters and setters used by unit and integration tests. */
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

    /** Default C-tor needed by Jackson JSON. */
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
```

Additionally it inherits common attributes as: id (unique resource identifier), createdAt, modifiedAt from `AbstractEntity`. The last two are used in the context of database auditing: tracking and logging events related to all persistent entities.

Listing 8: [src/main/java/cristina/tech/blog/travel/domain/AbstractEntity.java](https://github.com/cristinanegrean/wanderlust-open-travel-api/blob/master/src/main/java/cristina/tech/blog/travel/domain/AbstractEntity.java)

```java
package cristina.tech.blog.travel.domain;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonTypeId;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;

import javax.persistence.Column;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.MappedSuperclass;
import javax.persistence.PrePersist;
import javax.persistence.PreUpdate;
import java.io.Serializable;
import java.time.LocalDateTime;

@MappedSuperclass
@JsonIgnoreProperties({"createdAt", "modifiedAt"})
@JsonInclude(JsonInclude.Include.NON_EMPTY)
@JsonAutoDetect(fieldVisibility = JsonAutoDetect.Visibility.ANY)
public abstract class AbstractEntity implements Serializable {

    private static final long serialVersionUID = 1126074635410771212L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @JsonTypeId
    protected Integer id;

    @Column(name = "created_at")
    @CreatedDate
    protected LocalDateTime createdAt;

    @Column(name = "modified_at")
    @LastModifiedDate
    protected LocalDateTime modifiedAt;

    @PrePersist
    @PreUpdate
    public void prePersist() {
        LocalDateTime now = LocalDateTime.now();

        if (createdAt == null) {
            createdAt = now;
        }
        modifiedAt = now;
    }

    protected AbstractEntity() {
    }

    public Integer getId() {
        return id;
    }
}
```

<i class="blue">"REST components communicate by transferring a representation of the data in a format matching one of the evolving set of standard data types.”</i> - Fielding and Taylor. And by `standard data types` it is referred to a media type, something you would specify in `Accept` or `Content-Type` HTTP headers. Thus there is some magic glue needed to translate a `Destination` domain object to a [json](http://www.json.org/) (JavaScript Object Notation) representation, for example.

Spring Data REST does that automatically for you by using <i class="blue">Spring Data REST’s ObjectMapper</i>, which has been specially configured to use intelligent serializers that can turn domain objects into links and back again. For JSON representation there are 2 such intelligent serializers supported in Spring Data REST: [Jackson JSON](https://github.com/FasterXML/jackson) and [GSON](https://github.com/google/gson). Jackson is being auto configured as default by Spring Boot. <i class="blue">Spring Data REST’s ObjectMapper</i> will try and serialize unmanaged beans as normal POJOs and it will try and create links to managed beans where that’s necessary. But if your domain model doesn’t easily lend itself to reading or writing plain JSON, you may want to configure Jackson’s ObjectMapper with your own custom type mappings and (de)serializers.

In my case, <i class="blue">Spring Data REST’s ObjectMapper</i> did the trick. I however have chosen to instruct [Jackson JSON](https://github.com/FasterXML/jackson) to simplify the view of my resources to not include `null` or empty attributes by using annotation `@JsonInclude(JsonInclude.Include.NON_EMPTY)` on `AbstractEntity`. The other excerpt was not including the auditing attributes, created and modified date, in the JSON representation: see `@JsonIgnoreProperties({"createdAt", "modifiedAt"})` on `AbstractEntity`. The two date fields are initialized by application events that occur inside the persistence mechanism. As such the `@PrePersist` and `@PreUpdate` callback annotations will initialize the creation and modified timestamps of the API domain objects.

Coding the other two domain objects: `Holiday` and `Agent` is similar, you could try to do the exercise yourself based on the schema design or check the code listing [here](https://github.com/cristinanegrean/wanderlust-open-travel-api/tree/master/src/main/java/cristina/tech/blog/travel/domain).

## Coding the Repositories

The general idea of Spring Data REST is that builds on top of Spring Data repositories and automatically exports those as REST resources. I've created several repositories, one for each entity: `DestinationRepository`, `HolidayRepository` and `AgentRepository`. All repositories are Java interfaces extending from [PagingAndSortingRepository](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/PagingAndSortingRepository.html) interface, defined in Spring Data Commons, fact that ensures that common CRUD operations as: `findAll`, `findOne`, `save`, `exists`, `count`, `delete`, `deleteAll` will be exposed as HTTP operations on all domain objects. Additionally it provides out-of-the-box pagination and sorting support.

Listing 9: [src/main/java/cristina/tech/blog/travel/DestinationRepository.java](https://github.com/cristinanegrean/wanderlust-open-travel-api/blob/master/src/main/java/cristina/tech/blog/travel/DestinationRepository.java)

```java
package cristina.tech.blog.travel;

import cristina.tech.blog.travel.domain.Destination;
import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.repository.query.Param;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

import java.util.List;
import java.util.Optional;

@RepositoryRestResource(collectionResourceRel = "destinations", path = "destinations")
public interface DestinationRepository extends PagingAndSortingRepository<Destination, Integer> {

    List<Destination> findByCountry(@Param("country") String country);

    Optional<Destination> findByName(@Param("name") String name);
}
```
Listing 9 showcases the `DestinationRepository` which encapsulates various operations involving `Destination` objects. Spring Boot automatically spins up Spring Data JPA to create a concrete implementation of the DestinationRepository, at runtime, and configure it to talk to our back-end PostgreSQL database using JPA. `@RepositoryRestResource` is not required for a repository to be exported. It is only used here to document to the export details, such as RESTful endpoints at: <i class="blue">/destinations</i> (the collection resource) and <i class="blue">/destinations/{id}</i> (the item resource)

By default, Spring Data REST will expose your CrudRepository using the name of the domain class, lower-cased, and apply the [Evo Inflector](https://github.com/atteo/evo-inflector) to pluralize this word.

I have also defined two custom queries to retrieve a list of `Destination` objects based on the country and find zero or one destination by name. Note that there is a UNIQUE database constraint on destination name. These two queries will be automatically exposed as RESTful endpoints: <i class="blue">/destinations/search/findByCountry?country={country}</i>,
respectively: <i class="blue">/destinations/search/findByName?name={name}</i>

The listing of the `HolidayRepository` and `AgentRepository` interfaces can be found [here](https://github.com/cristinanegrean/wanderlust-open-travel-api/tree/master/src/main/java/cristina/tech/blog/travel).

## Running the API

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/wanderlust-api/destinations_postman.png" alt="Testing the API with Postman"/>

"Ta da"! The screenshot above shows the back-end API in action, more specifically the <i class="blue">/destinations</i> endpoint.

As the screenshot shows there are many more RESTful endpoints exposed by the wanderlust service. I have created a test collection for the other relevant endpoints and you can give them a spin yourself with [Postman](https://www.getpostman.com/). The test collection can be downloaded [here](https://github.com/cristinanegrean/wanderlust-open-travel-api/blob/master/Wanderlust_OpenTravelAPI_Postman_collection.json).

Manual testing can be a drag and as a developer, we like to automate testing as much as possible. Listing below shows how to automate integration testing of HTTP operations on the `Destination` resource.

Listing 10: [src/test/java/cristina/tech/blog/travel/domain/WanderlustIntegrationTests.java](https://github.com/cristinanegrean/wanderlust-open-travel-api/blob/master/src/test/java/cristina/tech/blog/travel/domain/WanderlustIntegrationTests.java)

```java
package cristina.tech.blog.travel.domain;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.hateoas.Resource;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.web.util.UriComponentsBuilder;

import java.io.IOException;
import java.net.URI;
import java.util.ArrayList;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.http.HttpMethod.GET;
import static org.springframework.http.HttpMethod.PATCH;
import static org.springframework.http.HttpMethod.PUT;


@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class WanderlustIntegrationTests {
    private static final String KOTOR_FACT        =
            "At 1300 metres deep, the Grand Canyon of Tara River is actually the deepest canyon in Europe and second largest in the world after the Colorado canyon in the USA.";
    private static final String KOTOR_DESCRIPTION = "The gem of Montenegro";

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    public void destinations() throws IOException {
        Destination destinationKotor = new Destination("Kotor", "ME");

        // POST, create destination
        URI uri = restTemplate.postForLocation("/destinations", destinationKotor);

        // setup API response type to be a Destination HTTP Resource
        ParameterizedTypeReference<Resource<Destination>> responseType = new ParameterizedTypeReference<Resource<Destination>>() {};

        // GET by URI, id as URI Path Param, i.e.: http://localhost:54294/destinations/1, assert Response HTTP code 200: OK
        ResponseEntity<Resource<Destination>> getDestinationByURI = restTemplate.exchange(uri.toString(), GET, null, responseType);
        assertThat(destinationKotor.getCountry()).isEqualTo(getDestinationByURI.getBody().getContent().getCountry());
        assertThat(HttpStatus.OK.value()).isEqualTo(getDestinationByURI.getStatusCode().value());

        // PUT, replace destination, assert assert Response HTTP code 200: OK
        List<String> facts = new ArrayList<>(1);
        facts.add(KOTOR_FACT);
        destinationKotor.setFacts(facts);
        ResponseEntity<Resource<Destination>> putDestination = restTemplate.exchange(uri.toString(), PUT, new HttpEntity<>(destinationKotor), responseType);
        assertThat(destinationKotor.getFacts()).isEqualTo(putDestination.getBody().getContent().getFacts());
        assertThat(getDestinationByURI.getStatusCodeValue()).isEqualTo(HttpStatus.OK.value());

        // PATCH, update destination, does not work with exchange method, see: https://jira.spring.io/browse/SPR-15347
        destinationKotor.setDescription(KOTOR_DESCRIPTION);
        restTemplate.postForLocation(UriComponentsBuilder.fromUri(uri).queryParam("_method", PATCH.name()).build().toString(), destinationKotor);

        // GET by name, filter on destination name, assert return HTTP code 200
        ResponseEntity<Resource<Destination>> getDestinationByName =
                restTemplate.exchange(UriComponentsBuilder.fromPath("/destinations/search/findByName").queryParam("name", "Kotor").build().toString(), GET, null, responseType);
        assertThat(destinationKotor.getDescription()).isEqualTo(getDestinationByName.getBody().getContent().getDescription());
        assertThat(getDestinationByName.getStatusCode().value()).isEqualTo(HttpStatus.OK.value());

        // DELETE
        restTemplate.delete(uri.toString());

        // GET by country, assert destination does not exist anymore
        ResponseEntity<Resource<Destination>> getDestinationByCountry =
                restTemplate.exchange(UriComponentsBuilder.fromPath("/destinations/search/findByCountry").queryParam("country", "ME").build().toString(), GET, null, responseType);
        assertThat(getDestinationByCountry.getBody().getContent().getId()).isEqualTo(null);
    }
}
```

## HAL and Resource discoverability

A core principle of [HATEOAS](https://spring.io/understanding/HATEOAS) (Hypermedia as the Engine of Application State) is that resources should be discoverable through the publication of links that point to the available resources. There are a few competing de-facto standards of how to represent links in JSON. By default, Spring Data REST uses HAL to render responses. HAL defines links to be contained in a property of the returned document.

If you can remember from <i class="blue">Listing 1: build.gradle</i>, we have added a Gradle runtime dependency for browsing Spring Data REST repositories in your Web browser It is an API browser for the hal+json media type.

When you point your browser to location: http://localhost:9000/api/opentravel/, you should see something similar to below. The HAL Browser makes for quite a nice Wanderlust REST API documentation, by listing: links, headers, URI templates, response body.

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/wanderlust-api/HAL_Browser.png" alt="Learn more of the REST API with the HAL browser"/>

## RESTing in the APLS

<i class="blue">"ALPS is a data format for defining simple descriptions of application-level semantics, similar in complexity to HTML microformats. An ALPS document can be used as a profile to explain the application semantics of a document with an application-agnostic media type (such as HTML, HAL, Collection+JSON, Siren, etc.). This increases the reusability of profile documents across media types."</i> - M. Admundsen / L. Richardson / M. Foster

```
$ Cristinas-iMac:wanderlust-open-travel-api cristina$ curl http://localhost:9000/api/opentravel/profile
{
  "_links" : {
    "self" : {
      "href" : "http://localhost:9000/api/opentravel/profile"
    },
    "holidays" : {
      "href" : "http://localhost:9000/api/opentravel/profile/holidays"
    },
    "destinations" : {
      "href" : "http://localhost:9000/api/opentravel/profile/destinations"
    },
    "agents" : {
      "href" : "http://localhost:9000/api/opentravel/profile/agents"
    }
  }
}
$ Cristinas-iMac:wanderlust-open-travel-api cristina$ curl http://localhost:9000/api/opentravel/profile/destinations
{
  "alps" : {
    "version" : "1.0",
    "descriptors" : [ {
      "id" : "destination-representation",
      "href" : "http://localhost:9000/api/opentravel/profile/destinations",
      "descriptors" : [ {
        "name" : "name",
        "type" : "SEMANTIC"
      }, {
        "name" : "country",
        "type" : "SEMANTIC"
      }, {
        "name" : "description",
        "type" : "SEMANTIC"
      }, {
        "name" : "facts",
        "type" : "SEMANTIC"
      }, {
        "name" : "createdAt",
        "type" : "SEMANTIC"
      }, {
        "name" : "modifiedAt",
        "type" : "SEMANTIC"
      } ]
    }, {
      "id" : "get-destinations",
      "name" : "destinations",
      "type" : "SAFE",
      "rt" : "#destination-representation",
      "descriptors" : [ {
        "name" : "page",
        "doc" : {
          "value" : "The page to return.",
          "format" : "TEXT"
        },
        "type" : "SEMANTIC"
      }, {
        "name" : "size",
        "doc" : {
          "value" : "The size of the page to return.",
          "format" : "TEXT"
        },
        "type" : "SEMANTIC"
      }, {
        "name" : "sort",
        "doc" : {
          "value" : "The sorting criteria to use to calculate the content of the page.",
          "format" : "TEXT"
        },
        "type" : "SEMANTIC"
      } ]
    }, {
      "id" : "create-destinations",
      "name" : "destinations",
      "type" : "UNSAFE",
      "rt" : "#destination-representation"
    }, {
      "id" : "get-destination",
      "name" : "destination",
      "type" : "SAFE",
      "rt" : "#destination-representation"
    }, {
      "id" : "patch-destination",
      "name" : "destination",
      "type" : "UNSAFE",
      "rt" : "#destination-representation"
    }, {
      "id" : "delete-destination",
      "name" : "destination",
      "type" : "IDEMPOTENT",
      "rt" : "#destination-representation"
    }, {
      "id" : "update-destination",
      "name" : "destination",
      "type" : "IDEMPOTENT",
      "rt" : "#destination-representation"
    }, {
      "name" : "findByName",
      "type" : "SAFE",
      "descriptors" : [ {
        "name" : "name",
        "type" : "SEMANTIC"
      } ]
    }, {
      "name" : "findByCountry",
      "type" : "SAFE",
      "descriptors" : [ {
        "name" : "country",
        "type" : "SEMANTIC"
      } ]
    } ]
  }
}
```

## Source Code
You can find the full code base of my Incubator Wanderlust OpenTravel API at my public [GitHub repo](https://github.com/cristinanegrean/wanderlust-open-travel-api).
Feel free to clone the project and give wanderlust-api a spin!

```
$ git clone https://github.com/cristinanegrean/wanderlust-open-travel-api
$ cd wanderlust-open-travel-api
$ ./gradlew clean build
$ java -jar build/libs/wanderlust-1.0.0-SNAPSHOT.jar --spring.profiles.active=postgres
```
