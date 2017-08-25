---
layout: post
author: Cristina Negrean
title: 'Testing a Spring Cloud Stream Microservice'
image: /img/spring-cloud-stream-test.png
tags: [Spring Cloud Stream, Test Driven Development, AMQP]
category: Spring Cloud Stream
---

In a [previous blog post](https://cristina.tech/2017/08/09/dresses-stream-processing-and-apache-kafka) I have written on how to use Spring Cloud Stream for developing an event driven microservice that stream processes dresses and rating messages from a Dockerized Apache Kafka installation.
This guide provides insight on how to leverage test support in Spring Cloud Stream and Spring Boot to slice and integration test it.

To start off, the first thing you'll need is to add test scope dependencies to your build specification:

Listing 1:  [build.gradle](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/build.gradle)

```
dependencies {
	  ...

    testCompile('org.springframework.boot:spring-boot-starter-test')
    testCompile('org.springframework.cloud:spring-cloud-stream-test-support')
    testRuntime('com.h2database:h2')
}
```

Above listing shows an excerpt of the [Gradle](https://gradle.org/) build specification file of the [Bootiful dress service](https://github.com/cristinanegrean/spring-cloud-stream-kafka) with the three needed dependencies:
* `spring-boot-starter-test`: compile time dependency that imports Spring Boot test modules, as well as: [JUnit](http://junit.org/junit4/), [AssertJ](http://joel-costigliola.github.io/assertj/), [Hamcrest](http://hamcrest.org/), [Mockito](http://site.mockito.org/), [JSONassert](http://jsonassert.skyscreamer.org/), [JsonPath](http://jsonpath.com/).
* `spring-cloud-stream-test-support`: compile time dependency to provision a set of classes to ease testing of Spring Cloud Stream modules
* `com.h2database:h2`: runtime dependency for running tests against [H2](http://www.h2database.com/html/main.html) embedded, in-memory database. There is always the dilemma
whether running tests against your regular database, in my case [PostgreSQL](https://www.postgresql.org/)
or a test reserved database as H2. In both cases there are trade-offs. Using your application database for
testing leads that test data will end up polluting application data, most likely integration tests will be much slower (as off persisting to filesystem) than with an in-memory database. However you'll get the full-fledged capabilities in data types and functions: i.e. H2 does not support [PostgreSQL date-time functions](https://www.postgresql.org/docs/9.3/static/functions-datetime.html) as age(timestamp).

### Let's see some code

Listing 2:  [src/test/java/cristina/tech/fancydress/BootifulDressIntegrationTests.java](https://raw.githubusercontent.com/cristinanegrean/spring-cloud-stream-kafka/master/src/test/java/cristina/tech/fancydress/BootifulDressIntegrationTests.java)

```java
package cristina.tech.fancydress;


import cristina.tech.fancydress.store.domain.Brand;
import cristina.tech.fancydress.store.domain.Dress;
import cristina.tech.fancydress.store.domain.Rating;
import cristina.tech.fancydress.store.repository.DressRepository;
import cristina.tech.fancydress.store.repository.RatingRepository;
import cristina.tech.fancydress.worker.event.DressEventType;
import cristina.tech.fancydress.worker.event.DressInboundChannels;
import cristina.tech.fancydress.worker.event.DressMessageEvent;
import cristina.tech.fancydress.worker.event.RatingMessageEvent;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.hateoas.Resource;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.messaging.support.GenericMessage;
import org.springframework.test.annotation.DirtiesContext;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.web.util.UriComponentsBuilder;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.Assert.assertNotNull;
import static org.springframework.http.HttpMethod.GET;

/**
 * Integration Tests: Streaming, Sink store to persistence layer, browse dresses via REST API.
 */
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT, classes = BootifulDressApplication.class)
@DirtiesContext
public class BootifulDressIntegrationTests {

    private static final String TEST_DRESS_ONE = "test-dress-one";
    private static final String TEST_DRESS_TWO = "test-dress-two";
    private static final String TEST_BRAND = "perfectly basics";
    private static final String TEST_SEASON = "all-around-year-no-seasonality";
    private static final String TEST_COLOR = "blue-crush";
    private static final String TEST_NAME = "pyjama dress";
    private static final BigDecimal TEST_PRICE = new BigDecimal(10.99);

    private static final String TEST_RATING_ID = "test-rating-id";
    private static final int TEST_RATING_STARS = 3;

    /**
     * TestRestTemplate is a convenience alternative to Spring’s RestTemplate that is useful in integration tests.
     * Redirects will not be followed (so you can assert the response location).
     * Cookies will be ignored (so the template is stateless)
     */
    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private DressInboundChannels dressInboundChannels;

    @Autowired
    private DressRepository dressRepository;

    @Autowired
    private RatingRepository ratingRepository;

    @Test
    public void contextLoads() {
        assertNotNull(this.dressInboundChannels.idresses());
        assertNotNull(this.dressInboundChannels.iratings());
    }

    @Test
    /**
    * Stream, store a DressMessageEvent. Browse the dress as a HTTP Resource.
    */
    public void streamStoreBrowseDress() {
        // send a test dress message event to stream listener component
        this.dressInboundChannels.idresses().send(new GenericMessage<>(getDressMessageEvent(TEST_DRESS_ONE)));

        assertDressStored(TEST_DRESS_ONE, 0);
        browseDressUriCheckContent(TEST_DRESS_ONE, 0);
    }

    @Test
    /**
     * Stream, store a RatingMessageEvent and also a DressMessageEvent. Browse the dress by id, check average rating.
     */
    public void streamStoreRatingAndDressBrowseDressCheckAverageRating() {
        // send a test rating message event to stream listener component
        this.dressInboundChannels.iratings().send(new GenericMessage<>(getRatingMessageEvent(TEST_DRESS_TWO)));

        // send a test dress message event to stream listener component
        this.dressInboundChannels.idresses().send(new GenericMessage<>(getDressMessageEvent(TEST_DRESS_TWO)));

        // assert rating is stored
        Rating rating = ratingRepository.findOne(TEST_RATING_ID);
        assertThat(rating.getStars()).isEqualTo(TEST_RATING_STARS);
        assertThat(rating.getEventTime()).isNotNull();

        // assert that dress is stored, check that service call updated automatically average rating
        assertDressStored(TEST_DRESS_TWO, TEST_RATING_STARS);
        browseDressUriCheckContent(TEST_DRESS_TWO, TEST_RATING_STARS);
    }

    /**
     * Assert that a dress has been created in persistent store with given uuid
     */
    private void assertDressStored(String expectedId, Integer expectedAverageRating) {
        Optional<Dress> storeOptionalDress = dressRepository.findById(expectedId);
        assertThat(storeOptionalDress.isPresent());
        assertThat(storeOptionalDress.get().getId()).isEqualTo(expectedId);
        assertThat(ratingRepository.getAverageRating(expectedId)).isEqualTo(expectedAverageRating);
        assertThat(storeOptionalDress.get().getAverageRating()).isEqualTo(expectedAverageRating);
    }

    /**
     * Assert I can browse the dress details via the REST API.
     */
    private void browseDressUriCheckContent(String dressId, int expectedAverageRating) {
        ParameterizedTypeReference<Resource<Dress>> responseType = new ParameterizedTypeReference<Resource<Dress>>() {
        };
        ResponseEntity<Resource<Dress>> dressUriSearch =
                restTemplate.exchange(UriComponentsBuilder.fromPath("/dresses/" + dressId)
                        .build().toString(), GET, null, responseType);

        // assert response on client rest template exchange
        assertThat(dressUriSearch.getStatusCode().value()).isEqualTo(HttpStatus.OK.value());
        assertThat(dressUriSearch.getHeaders().getContentType().toString()).isEqualTo("application/hal+json;charset=UTF-8");

        // assert body matches dress message event details
        assertThat(dressUriSearch.getBody().getContent().getBrand().getName()).isEqualTo(TEST_BRAND);
        assertThat(dressUriSearch.getBody().getContent().getName()).isEqualTo(TEST_NAME);
        assertThat(dressUriSearch.getBody().getContent().getSeason()).isEqualTo(TEST_SEASON);
        assertThat(dressUriSearch.getBody().getContent().getAverageRating()).isEqualTo(expectedAverageRating);
        assertThat(dressUriSearch.getBody().getContent().getColor()).isEqualTo(TEST_COLOR);
        assertThat(dressUriSearch.getBody().getContent().getPrice().doubleValue()).isEqualTo(TEST_PRICE.doubleValue());
    }

    private static DressMessageEvent getDressMessageEvent(String dressId) {
        DressMessageEvent createDressEvent = new DressMessageEvent();
        createDressEvent.setStatus(DressStatus.CREATED);
        createDressEvent.setPayloadKey(dressId);
        cristina.tech.fancydress.worker.domain.Dress dress = new cristina.tech.fancydress.worker.domain.Dress();
        cristina.tech.fancydress.worker.domain.Brand brand = new cristina.tech.fancydress.worker.domain.Brand();
        brand.setName(TEST_BRAND);
        dress.setId(dressId);
        dress.setBrand(brand);
        dress.setSeason(TEST_SEASON);
        dress.setPrice(TEST_PRICE);
        dress.setColor(TEST_COLOR);
        dress.setName(TEST_NAME);
        createDressEvent.setPayload(dress);
        createDressEvent.setEventType(DressEventType.CREATED);
        createDressEvent.setTimestamp(Instant.now().toEpochMilli());

        return createDressEvent;
    }

    private static RatingMessageEvent getRatingMessageEvent(String dressId) {
        RatingMessageEvent dressRatedEvent = new RatingMessageEvent();
        dressRatedEvent.setPayloadKey(TEST_RATING_ID);
        cristina.tech.fancydress.worker.domain.Rating rating = new cristina.tech.fancydress.worker.domain.Rating();
        rating.setDressId(dressId);
        rating.setRatingId(TEST_RATING_ID);
        rating.setStars(TEST_RATING_STARS);
        dressRatedEvent.setPayload(rating);
        dressRatedEvent.setEventType(DressEventType.RATED);
        dressRatedEvent.setTimestamp(Instant.now().toEpochMilli());

        return dressRatedEvent;
    }
}
```

Listing 2 annotation `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT, classes = BootifulDressApplication.class)` takes care that when you run `BootifulDressIntegrationTests`, Spring
bootstraps a Tomcat (or Jetty or Undertow, depending on your use-case) application container, on a random or specifically defined server port, basically permitting you to perform functionality integration tests as you would do against a live application. The difference is that they're automated and are run automatically by a Continuous Integration server.

### What's your favorite JVM assertion library?

There was a recent Twitter poll initiated by [Josh Long](https://spring.io/team/jlong) and I was surprised to see 13% of the respondents are `high-performers` that they afford testing in
production. 😃😄😆

<img class="img-responsive" src="{{site.baseurl }}/img/posts/tdd/poll_fav_JVM_assertion_framework.png" alt="Poll favorite JVM assertion framework"/>

For all the human kinds, like me, there are quite a few assertion libraries that you get imported out-of-the-box with only adding `spring-boot-starter-test` as test compile dependency (see Listing 1). Chances are that you are fluent in at least one of them. And that you practice test driven development to some pragmatic extent. In case you don't, no worries, read on and I will introduce you to my current favorites ones: [AssertJ](http://joel-costigliola.github.io/assertj/) and [Mockito](http://site.mockito.org/).
