---
layout: post
author: Cristina Negrean
title: 'Dresses, Stream Processing and Kafka'
image: /img/bootiful_dress.gif
tags: [Spring Cloud Stream, Apache Kafka, Java Consumer, Python Producer, Docker, Dockerized Spring Boot Application]
category: Stream Processing
comments: false
---

In this blog post I will show you how to adopt a full-fledged stream processing framework:
[Spring Cloud Stream](https://cloud.spring.io/spring-cloud-stream/) to build a message & event driven microservice: `Bootiful dress service`.

The microservice relies on a dedicated broker, [Apache Kafka](http://kafka.apache.org/),
responsible for distributing the events: `Dress Created Event`, `Dress Updated Event`
and `Dress Rated Event`. The three events, messages are represented as JSON strings, UTF-8 encoded,
and are sourced to Kafka via a Python producer script: `producer.py`, leveraging a dataset of dresses:
`data/dresses.p`.

Using a broker, asynchronous communication approach between producer and consumer (also called pub/sub from publisher/subscriber), has particular advantages over a feed, synchronous approach:
* you have a central place where your events are stored, feeds are not accessible when the producing application is down
* scaling is easier with a broker – what happens if you suddenly need to double your consuming applications because of load? Who subcribes to the feed? If both subscribe, events are processed twice. With a broker like [Apache Kafka](http://kafka.apache.org/) you easily create consumer groups, and each event is only processed by one application of this group.

As always there is also a trade-off or disadvantage, and that is managing infrastructure. `Bootiful dress service` uses [Docker](https://www.docker.com/docker-mac) to automate infrastructure away and integrates nicely with [Spring Profiles](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html) to
segregate configuration between the three environments, profiles supported by the microservice: `development`, `docker` and `test`.

There are sooo many moving parts here, so lets see a visual of the `Bootiful dress service` and it's dependencies:

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/spring-cloud-stream/bootiful_dresses_service_overview.png" alt="Bootiful Dresses Service Overview"/>

* Kafka itself
* A Python Producer
* The actual stream processing job (worker) written in Java, on top of Spring Cloud Stream framework, packaged as a DevOps friendly, uber-fat-jar (Spring Boot)
* A database, [PostgreSQL](https://www.postgresql.org/) for outputs that will be queries by
a web request/response app app, and which takes the output of the stream processing job
* Same database, [PostgreSQL](https://www.postgresql.org/) for lookups and aggregations.
PostgreSQL having support for both [window](https://www.postgresql.org/docs/current/static/functions-window.html) and
[aggregate](https://www.postgresql.org/docs/current/static/tutorial-agg.html) functions. Windowing, time, and out-of-order events are a tricky aspect of stream processing.
* The request/response app (web) written in Java, on top of [Spring Data REST](http://projects.spring.io/spring-data-rest/), that serves live requests to your users or customers.
For sake of simplicity the worker and web are now packaged together, but they could be split in two different
services, with own deployment and CI (Continuous Integration) cycle.

## Skip the details and get me started right away

In case you can not wait and want to try out the Dockerized Spring Boot Application right away, simply follow the next steps. You need to have [Java 8 SDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html), [Git](https://git-scm.com/downloads) and [Docker](https://www.docker.com/docker-mac) installed on your computer.

{% highlight bash %}
$ git clone https://github.com/cristinanegrean/spring-cloud-stream-kafka
$ cd spring-cloud-stream-kafka
$ ./gradlew clean build buildDocker
$ docker-compose up
{% endhighlight %}

Start the Producer Python Script that produces data for dresses and dress ratings on two different Kafka topics named `dresses` and `ratings`:

Python requirements installation on OS X:

```
$ brew install python3
$ brew link python3
$ python3 -m ensurepip
```

Edit `.bash_profile`:

```
export KAFKA_HOST_PORT=localhost:9092
export PYTHON_HOME=/usr/local/bin/python3
export PATH=$PATH:$PYTHON_HOME
```

As of the Kafka advertised hostname, when running Kafka infrastructure with Docker,
you'll have to add below to `/etc/hosts`:

```
127.0.0.1       kafka
```

Install packages and start the script:

```
$ python3 -m pip install numpy click kafka-python
$ cd spring-cloud-stream-kafka/src/main/resources/
$ sudo python3 producer.py
```
And here you go, you should be all setup to stream process dresses :)

<img class="img-responsive" src="{{site.baseurl }}/img/posts/spring-cloud-stream/dockerized-bootiful-dress-service.png" alt="Dress Message Events Stream Processing"/>

<br/>
