---
layout: post
author: Cristina Negrean
title: 'Dresses, Stream Processing and Apache Kafka'
image: /img/bootiful_dress.gif
tags: [Spring Cloud Stream, Apache Kafka, AMQP]
category: Spring Cloud Stream
comments: true
---

In this blog post I will show you how to adopt a full-fledged stream processing framework:
[Spring Cloud Stream](https://cloud.spring.io/spring-cloud-stream/) to build a message & event driven microservice: `Bootiful dress service`.

The microservice relies on a dedicated broker, [Apache Kafka](http://kafka.apache.org/),
responsible for distributing the events: `Dress Created Event`, `Dress Updated Event`
and `Dress Rated Event`. The three events are represented as JSON strings, UTF-8 encoded,
and are sourced to Kafka via a Python producer script: `producer.py`, leveraging a dataset of dresses:
`data/dresses.p`.

Using a broker, asynchronous communication approach between producer and consumer (also called pub/sub from publisher/subscriber), has particular advantages over a feed, synchronous approach:
* you have a central place where your events are stored, feeds are not accessible when the producing application is down
* scaling is easier with a broker – what happens if you suddenly need to double your consuming applications because of load? Who subscribes to the feed? If both subscribe, events are processed twice. With a broker like [Apache Kafka](http://kafka.apache.org/) you easily create consumer groups, and each event is only processed by one application of this group.

As always there is also a trade-off, and with a broker approach, that is managing infrastructure. `Bootiful dress service` leverages [Docker](https://www.docker.com/docker-mac) to automate infrastructure provisioning and [Spring Cloud Stream's](https://cloud.spring.io/spring-cloud-stream/) opinionated configuration of message brokers. The last came to me as a handy simplification, as I did not have weeks to deep dive into [Apache Kafka](http://kafka.apache.org/), nor did I wish to develop a low-tech solution that appealed to people who liked to roll their own.

Other reasons to consider when pulling in any heavy dependencies on a full-fledged stream processing framework, like [Spring Cloud Stream's](https://cloud.spring.io/spring-cloud-stream/) are:
* when you want to do something more involved, say `compute aggregations - i.e trending dresses` or join streams of data, inventing a solution on top of the Kafka consumer APIs is fairly involved;
* `durability` of consumer group subscriptions, that is, a binder implementation ensures that group subscriptions are persistent, and once at least one subscription for a group has been created, the group will receive messages, even if they are sent while all applications in the group are stopped;
* `connectors for various middleware vendors` including message brokers as [Apache Kafka](http://kafka.apache.org/) and/or [RabbitMQ](https://www.rabbitmq.com/), storage (relational, non-relational, filesystem);
* least but not last: `testing support`, where [Spring Cloud Stream](https://cloud.spring.io/spring-cloud-stream/) integrates nicely with [Spring Profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html) to
segregate configuration between the three environments, profiles supported by the microservice: `development`, `docker` and `test`.

There are so many moving parts here, so lets get a visual of the `Bootiful dress service` and it's dependencies:

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/spring-cloud-stream/bootiful_dresses_service_overview.png" alt="Bootiful Dresses Service Overview"/>

* Kafka itself
* A Python Producer
* The actual stream processing job (AMQP protocol, worker role) written in Java, on top of Spring Cloud Stream framework, packaged as a DevOps friendly, uber-fat-jar ([Spring Boot](https://projects.spring.io/spring-boot/))
* A database, [PostgreSQL](https://www.postgresql.org/) for persisting output of the stream processing job.
Same database used, in first instance, for lookups and aggregations as well.
* A REST API (HTTP protocol, web role) written in Java, on top of [Spring Data REST](http://projects.spring.io/spring-data-rest/), that serves live requests to users, devices and/or frontends.

`Note`: to further reduce infrastructure footprint, the HTTP (web role) and AMQP (worker role) components are packaged together into a single container deployment model.

# Skip the details and get me started right away

In case you cannot wait and want to try it out right away, simply follow the next steps. You need to have [Java 8 SDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html), [Git](https://git-scm.com/downloads) and [Docker](https://www.docker.com/docker-mac) installed on your computer.

1) Clone, build and start multi-container Docker application:

{% highlight bash %}
$ git clone https://github.com/cristinanegrean/spring-cloud-stream-kafka
$ cd spring-cloud-stream-kafka
$ ./gradlew clean build buildDocker
$ docker-compose up
{% endhighlight %}

2) Install Python, requirements installation on OS X:

```
$ brew install python3
$ brew link python3
$ python3 -m ensurepip
```

3) Edit `.bash_profile`:

```
export KAFKA_HOST_PORT=localhost:9092
export PYTHON_HOME=/usr/local/bin/python3
export PATH=$PATH:$PYTHON_HOME
```

4) As of the Kafka advertised hostname, when running Kafka infrastructure with Docker,
you'll have to add below to your `/etc/hosts` file:

```
127.0.0.1       kafka
```

5) Install packages and start the script that produces data for dresses and dress ratings on two different Kafka topics named `dresses` and `ratings`. Requirements installation on OS X:

```
$ python3 -m pip install numpy click kafka-python
$ cd spring-cloud-stream-kafka/src/main/resources/
$ sudo python3 producer.py
```

And here you go, you should be all setup to stream process dresses and dress ratings. Screenshots below show the fully dockerized application infrastructure and exploring the REST API endpoints:

<img class="img-responsive" src="{{site.baseurl }}/img/posts/spring-cloud-stream/dockerized-bootiful-dress-service.png" alt="Dress Message Events Stream Processing Logs"/>

<img class="img-responsive" src="{{site.baseurl }}/img/posts/spring-cloud-stream/container_ids.png" alt="Docker multi-container application"/>

<img class="img-responsive" src="{{site.baseurl }}/img/posts/spring-cloud-stream/connect_to_docker_container.png" alt="Dockerized PostgreSQL Server Console"/>

<img class="img-responsive" src="{{site.baseurl }}/img/posts/spring-cloud-stream/rest_api_hal_browser.png" alt="Dockerized REST API browse collection endpoint"/>

<img class="img-responsive" src="{{site.baseurl }}/img/posts/spring-cloud-stream/sorting.png" alt="Sorting"/>

<img class="img-responsive" src="{{site.baseurl }}/img/posts/spring-cloud-stream/dress_by_uuid.png" alt="Dress Endpoint"/>

# Lets See Some Code

The most important piece of code with regards to the AMQP role is the `DressEventStream.java` class (see listing 1). That's where the whole stream processing magic takes place. By adding `@EnableBinding(DressInboundChannels.class)`, the worker component gets immediate connectivity to the Kafka message broker and by adding `@StreamListener` to a method, it will receive events for stream processing. The `@Profile({"development", "docker", "test"})` annotation gives an insight that the component will be loaded as a Spring bean in pretty all environments my application supports.

Listing 1:  [src/main/java/cristina/tech/fancydress/worker/event/DressEventStream.java](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/main/java/cristina/tech/fancydress/worker/event/DressEventStream.java)

```java
package cristina.tech.fancydress.worker.event;

import cristina.tech.fancydress.store.service.DressEventStoreService;
import cristina.tech.fancydress.store.service.RatingEventStoreService;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.context.annotation.Profile;

@EnableBinding(DressInboundChannels.class)
@Profile({"development", "docker", "test"})
@Slf4j
public class DressEventStream {

    @Autowired
    private DressEventStoreService dressEventStoreService;

    @Autowired
    private RatingEventStoreService ratingEventStoreService;

    private static final String LOG_RECEIVED = "Received: ";

    @StreamListener(target = DressInboundChannels.INBOUND_DRESSES)
    public void receiveDressMessageEvent(DressMessageEvent dressMessageEvent) {
        log.info(LOG_RECEIVED + dressMessageEvent.toString());
        dressEventStoreService.apply(dressMessageEvent);
    }

    @StreamListener(target = DressInboundChannels.INBOUND_RATINGS)
    public void receiveRatingMessageEvent(RatingMessageEvent ratingMessageEvent) {
        log.info(LOG_RECEIVED + ratingMessageEvent.toString());
        ratingEventStoreService.apply(ratingMessageEvent);
    }
}
```

Now for easy addressing of the most common use cases, which involve either an input channel, an output channel, or both, Spring Cloud Stream provides three predefined interfaces out of the box:
* `Source` - can be used for an application which has a single outbound channel.
* `Sink` - can be used for an application which has a single inbound channel.
* `Processor`- can be used for an application which has both an inbound channel and an outbound channel.

Hereby the [link](http://docs.spring.io/spring-cloud-stream/docs/current/reference/htmlsingle/#__code_source_code_code_sink_code_and_code_processor_code) to the documentation. However, in my case, I have two input channels: one for the `dresses` topic and one for the `ratings` topic, and I have no output channel.
As such, I have defined my own interface `DressInboundChannels.java` (see listing 2) and I am using it
with the `@EnableBinding` annotation: `@EnableBinding(DressInboundChannels.class)`. The former will instruct Spring Cloud Stream to automatically create message channels: `idresses` and `iratings`, which will be used by the `@StreamListener`.

Listing 2:  [src/main/java/cristina/tech/fancydress/worker/event/DressInboundChannels.java](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/main/java/cristina/tech/fancydress/worker/event/DressInboundChannels.java)

```java
package cristina.tech.fancydress.worker.event;

import org.springframework.cloud.stream.annotation.Input;
import org.springframework.messaging.SubscribableChannel;

public interface DressInboundChannels  {
    String INBOUND_DRESSES = "idresses";
    String INBOUND_RATINGS = "iratings";

    @Input(INBOUND_DRESSES)
    SubscribableChannel idresses();

    @Input(INBOUND_RATINGS)
    SubscribableChannel iratings();
}
```

So now you may be asking yourself what's up with those magic strings: `idresses`
and `iratings`, where are they configured? Well, that happens in:
 [/src/main/resources/application.yml](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/main/resources/application.yml) for `development` and `docker` profiles and [/src/test/resources/application.yml](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/test/resources/application.yml) for the `test` environment Spring profile. I have kept the configuration pretty similar, so that I can perform integration tests and mimic the running application semantics. That is why I will only show the listing for the `development` and `docker` profiles below:

 Listing 3:  [/src/main/resources/application.yml](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/main/resources/application.yml)

 ```yml
 spring:
  profiles:
    active: development
server:
    port: 9000
management:
  port: 8081
  context-path: /admin
flyway:
  enabled: true
  validate-on-migrate: false
  baseline-on-migrate: true
endpoints:
  health:
    sensitive: false
  flyway:
    sensitive: false
logging:
  level:
    cristina:
      tech: debug
---
spring:
  profiles: development
  datasource:
    url: jdbc:postgresql://localhost:5432/dresses
    driverClassName: org.postgresql.Driver
    username: ${POSTGRES_USER}
    password: ${POSTGRES_PASSWORD}
    hikari:
      maximumPoolSize: 10
      connectionTimeout: 1500
  data:
    rest:
      detection-strategy: annotated
  jackson:
    serialization:
      WRITE_DATES_AS_TIMESTAMPS: false
  cloud:
    stream:
      bindings:
        idresses:
          destination: dresses
          group: dresses-group
          contentType: 'application/x-java-object;type=cristina.tech.fancydress.worker.event.DressMessageEvent'
          consumer:
            durableSubscription: true
            headerMode: raw
        iratings:
          destination: ratings
          group: ratings-group
          contentType: 'application/x-java-object;type=cristina.tech.fancydress.worker.event.RatingMessageEvent'
          consumer:
            durableSubscription: true
            headerMode: raw
---
spring:
  profiles: docker
  datasource:
    url: jdbc:postgresql://postgresdb:5432/postgres
    driverClassName: org.postgresql.Driver
    username: postgres
    password: demo
    hikari:
      maximumPoolSize: 10
      connectionTimeout: 1500
  data:
    rest:
      detection-strategy: annotated
  jackson:
    serialization:
      WRITE_DATES_AS_TIMESTAMPS: false
  cloud:
    stream:
      kafka:
        binder:
          brokers: kafka
          zkNodes: zookeeper
      bindings:
        idresses:
          destination: dresses
          group: dresses-group
          contentType: 'application/x-java-object;type=cristina.tech.fancydress.worker.event.DressMessageEvent'
          consumer:
            durableSubscription: true
            concurrency: 20
            headerMode: raw
        iratings:
          destination: ratings
          group: ratings-group
          contentType: 'application/x-java-object;type=cristina.tech.fancydress.worker.event.RatingMessageEvent'
          consumer:
            durableSubscription: true
            concurrency: 20
            headerMode: raw
 ```

 That is a hands full, but if you focus on keys `spring.cloud.stream.bindings.idresses.destination`
 and `spring.cloud.stream.bindings.iratings.destination` you should get the idea on how the Spring Cloud Stream input channel links to the appropriate Kafka topic.

 Other `important` configurations above that took some trial and error, attentive read through Spring Cloud Stream's documentation, Google search, application debugging are:
 * `spring.cloud.stream.bindings.<channelName>.consumer.headerMode=raw` As Kafka does not support message headers natively, and inbound data is coming from outside Spring Cloud Stream application, python kafka producer more precisely, Spring Cloud Stream's default: `embeddedHeaders` was giving me
 a `java.lang.StringIndexOutOfBoundsException: String index out of range: at org.springframework.cloud.stream.binder.EmbeddedHeaderUtils.extractHeaders` in an incipient setup.
 * `spring.cloud.stream.bindings.<channelName>.consumer.contentType` will not work with `'application/json'`, but you have to map the plain Java object, Jackson JSON annotated with a `'application/x-java-object;type=<JavaClassType>'` to get rid of deserialization error: `Caused by: com.fasterxml.jackson.databind.JsonMappingException: Can not deserialize instance of java.lang.String out of START_OBJECT token`. I've found this one not that well documented and intuitive, as `application/json` works just great for a Spring Cloud Stream `Sink` consumer, when the producer is a Spring Cloud Stream `Source` application.
 * `spring.cloud.stream.kafka.binder.brokers=kafka` and `spring.cloud.stream.kafka.binder.zkNodes=zookeeper` in the `docker` profile. While the defaults `localhost` work well with the `development` profile, `DOCKER_IP` environment variable interpolation did not work out well for me, but using hostnames together with an edit of the `/etc/hosts` file to map `kafka` hostname to `127.0.0.1` did the trick.

 Further the line, the `DressEventStoreService.java` and `RatingEventStoreService.java` autowired components are plain `@Service` classes handling:
 * out-of-order events: i.e. `Dress Updated Event` before `Dress Created Event`, `Dress Rated Event` before `Dress Created Event`
 * translate worker job POJO to a JPA `@Entity`
 * and persist `@Entity` domain objects to PostgreSQL database

 In a future post, I will write about the testing support in Spring Cloud Stream and how I've leveraged it to integration test the `DressEventStream.java` stream processing component.

 Looking forward to your experiences with Spring Cloud Stream or how those match up against using [Kafka's Streams API](https://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple/), which I am keen as well on trying out these days.
