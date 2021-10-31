---
layout: post
author: Cristina Negrean
title: 'Testing a Spring Cloud Stream Microservice'
image: /img/spring-cloud-stream-test.png
tags: [Spring Cloud Stream, Test Driven Development, AMQP, Spring Boot]
category: Spring Cloud Stream
---

In a [previous blog post](https://cristina.tech/2017/08/09/dresses-stream-processing-and-apache-kafka) I have written on how to use Spring Cloud Stream framework for developing an event driven microservice that stream processes dresses & ratings messages from a Dockerized Apache Kafka installation.

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
* `spring-cloud-stream-test-support`: includes a [TestSupportBinder](https://github.com/spring-cloud/spring-cloud-stream/blob/master/spring-cloud-stream-test-support/src/main/java/org/springframework/cloud/stream/test/binder/TestSupportBinder.java), which leaves a channel unmodified so that tests can interact with channels directly and reliably assert on what is received. You do not need connectivity to a streaming platform or messaging system.
* `com.h2database:h2`: runtime dependency for running tests against [H2](http://www.h2database.com/html/main.html) embedded, in-memory database. There is always the dilemma
whether to run tests against your regular database, in my case [PostgreSQL](https://www.postgresql.org/),
or a test reserved database, as H2. In both cases there are trade-offs. Using your application database for
testing leads that test data will end up polluting application data, most likely integration tests will be much slower (as off persisting to filesystem) than with an in-memory database. However you'll get the full-fledged capabilities in data types and functions (i.e. H2 does not support [PostgreSQL date-time functions](https://www.postgresql.org/docs/9.3/static/functions-datetime.html) as `age(timestamp)` used by the `trending dresses` data analytics with SQL functionality).

### Let's see some code

Listing 2:  [src/test/java/cristina/tech/fancydress/BootifulDressIntegrationTests.java](https://raw.githubusercontent.com/cristinanegrean/spring-cloud-stream-kafka/master/src/test/java/cristina/tech/fancydress/BootifulDressIntegrationTests.java)

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT, classes = BootifulDressApplication.class)
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

    @Autowired
    private DressInboundChannels dressInboundChannels;

    @Autowired
    private DressRepository dressRepository;

    @Autowired
    private RatingRepository ratingRepository;

    @Autowired
    private TestRestTemplate restTemplate;    

    @Test
    public void contextLoadsAndWiring() {
        assertNotNull(this.dressInboundChannels.idresses());
        assertNotNull(this.dressInboundChannels.iratings());
    }

    /**
     * Stream, store a DressMessageEvent. Browse the dress as a HTTP Resource, check no average rating.
     */
    @Test
    public void streamStoreBrowseDressCheckAverageRating() {
        // send a test dress message event to stream listener component
        dressInboundChannels.idresses().send(new GenericMessage<>(getDressMessageEvent(TEST_DRESS_ONE)));
        assertDressStored(TEST_DRESS_ONE, 0);
        browseDressUriCheckContent(TEST_DRESS_ONE, 0);
    }

    /**
     * Stream, store a RatingMessageEvent and also a DressMessageEvent. Browse the dress as a HTTP Resource, check average rating.
     */
    @Test
    public void streamStoreRatingAndDressBrowseDressCheckAverageRating() {
        // send a test rating message event to stream listener component
        this.dressInboundChannels.iratings().send(new GenericMessage<>(getRatingMessageEvent(TEST_DRESS_TWO)));

        // send a test dress message event to stream listener component
        this.dressInboundChannels.idresses().send(new GenericMessage<>(getDressMessageEvent(TEST_DRESS_TWO)));

        // assert rating is stored
        Rating rating = ratingRepository.findOne(TEST_RATING_ID);
        assertThat(rating.getStars()).isEqualTo(TEST_RATING_STARS);
        assertNotNull(rating.getEventTime());

        // assert that dress is stored, check that service call updated automatically average rating
        assertDressStored(TEST_DRESS_TWO, TEST_RATING_STARS);
        browseDressUriCheckContent(TEST_DRESS_TWO, TEST_RATING_STARS);
    }

    /**
     * Assert that a dress has been created in persistent store with given uuid
     * @param dressId The unique identifier of the dress expected to be found in data store
     * @param expectedAverageRating Expected average rating of persisted dress data
     */
    private void assertDressStored(String dressId, int expectedAverageRating) {
        Optional<Dress> storeOptionalDress = dressRepository.findById(dressId);
        assertThat(storeOptionalDress.isPresent());
        assertThat(storeOptionalDress.get().getId()).isEqualTo(dressId);
        assertThat(ratingRepository.getAverageRating(dressId)).isEqualTo(expectedAverageRating);
        assertThat(storeOptionalDress.get().getAverageRating()).isEqualTo(expectedAverageRating);
    }

    /**
     * Assert I can browse the dress details via the REST API.
     * @param dressId The unique identifier of the dress expected to be found in data store
     * @param expectedAverageRating Expected average rating of persisted dress data
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

Zooming in Listing 2 integration tests:
* `@RunWith(SpringRunner.class)` annotation tells JUnit to run using Spring's testing support.
* `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT, classes = BootifulDressApplication.class)` annotation bootstraps a Tomcat (or Jetty or Undertow, depending on your use-case) application container at test runtime, on a random available server port, and starts the microservice. Basically it allows you to perform functional, integration tests as you would do against a live application. The difference is that they're automated and are run automatically at each build.
* The bound interface `DressInboundChannels` is injected into the test so we can have access to two input (messages entering the module) channels: `idresses` and `iratings`.
Spring Cloud Stream will create an implementation of the `DressInboundChannels` interface for you at runtime as asserted in `contextLoadsAndWiring` test case.
* `DressRepository` and `RatingRepository` are as well injected into integration test with the purpose of
performing read operations against the test, in-memory H2 database. We are functionally validating that
`DressMessageEvent` and `RatingMessageEvent` events dispatched to `idresses` and `iratings`
input channels are ending up persisted correctly into the data store via assertions in `assertDressStored(String dressId, int expectedAverageRating)`
* `TestRestTemplate` is a convenience alternative to Spring’s RestTemplate that is useful in integration tests. As the microservice has both AMQP role - by implementing a consumer for Kafka topics - and a Web role - exposes a REST API to browse, search dresses data, assertions in `browseDressUriCheckContent(String dressId, int expectedAverageRating)` will partly validate the Web role of the microservice.

### Testing components in isolation, Slice and Dice, @MockBean

There was a recent Twitter poll initiated by [Josh Long](https://spring.io/team/jlong), and besides inquiring for your favorite assertion framework for the JVM, it was advising on approach to use whenever testing Spring applications components in isolation. So let's see what is out there.

 <img class="img-responsive" src="{{site.baseurl }}/img/posts/tdd/poll_fav_JVM_assertion_framework.png" alt="Poll favorite JVM assertion framework"/>

 ><img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/>In addition to `@SpringBootTest` a number of other annotations are also provided for testing more specific slices of an application.
 >> `@JsonTest` can been used to test deserialization of [dress_created.json](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/test/resources/cristina/tech/fancydress/worker/event/dress_created.json), [dress_updated.json](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/test/resources/cristina/tech/fancydress/worker/event/dress_updated.json) and [dress_rated.json](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/test/resources/cristina/tech/fancydress/worker/event/dress_rated.json) to Java object types `DressMessageEvent.java` and `RatingMessageEvent.java`. See examples in tests:  
 [DressMessageEventTest.java](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/test/java/cristina/tech/fancydress/worker/event/DressMessageEventTest.java), [RatingMessageEventTest.java](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/test/java/cristina/tech/fancydress/worker/event/RatingMessageEventTest.java)
 >> `@DataJpaTest` can be used when a test focuses only on Java Persistency API (JPA) components, examples:
 [DressRepositoryCrudTests.java](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/test/java/cristina/tech/fancydress/store/repository/DressRepositoryCrudTests.java) and
 [RatingRepositoryCrudTests.java](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/test/java/cristina/tech/fancydress/store/repository/RatingRepositoryCrudTests.java)
 >> `@WebMvcTest` annotation can be used when a test focuses only on Spring MVC components.
Typically `@WebMvcTest` is used in combination with `@MockBean` to create any collaborators required by your `@RestController` or simply `@Controller` beans. See Listing 3 for an example of usage of `@WebMvcTest` in combination with `@MockBean` annotation for testing in isolation the `trending dresses` REST endpoint.

Listing 3:  [src/test/java/cristina/tech/fancydress/store/view/DressDetailViewTest.java](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/test/java/cristina/tech/fancydress/store/view/DressDetailViewTest.java)

```java
@RunWith(SpringRunner.class)
@WebMvcTest(TrendingRestController.class)
public class DressDetailViewTest {

    @Autowired
    private MockMvc mvc;

    @MockBean
    private TrendingDressesService trendingDressesService;

    private static final String TEST_JSON =
            "[{\"id\":\"NY221C002-C11\",\"price\":57.04,\"name\":\"Jumper dress - grey\",\"season\":\"WINTER\",\"color\":\"Grey\",\"averageRating\":3,\"brandName\":\"Native Youth\"}]";

    @Test
    public void testJsonView() throws Exception {
        DressDetailView dressDetailView = new DressDetailView();
        dressDetailView.setId("NY221C002-C11");

        dressDetailView.setName("Jumper dress - grey");
        dressDetailView.setSeason("WINTER");
        dressDetailView.setColor("Grey");
        dressDetailView.setPrice(new BigDecimal(57.04).setScale(2, BigDecimal.ROUND_CEILING));
        dressDetailView.setAverageRating((short) 3);
        dressDetailView.setBrandName("Native Youth");

        List<DressDetailView> dressDetailViews = new ArrayList<>(1);
        dressDetailViews.add(dressDetailView);

        given(this.trendingDressesService.getTrending(1, "30 second"))
                .willReturn(dressDetailViews);
        this.mvc.perform(get("/trending?count=1").accept(MediaType.APPLICATION_JSON_UTF8))
                .andExpect(status().isOk()).andExpect(content().string(TEST_JSON));
    }

}		
```

`@MockBean` can be leveraged as well along with `@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)`
and `@RunWith(SpringRunner.class)` annotations for unit testing.
It integrates well with [Mockito](http://site.mockito.org/) or the new Behavior Driven Development style of writing tests in Mockito.

Listing 4:  [src/test/java/cristina/tech/fancydress/worker/event/DressEventStreamTest.java](https://github.com/cristinanegrean/spring-cloud-stream-kafka/blob/master/src/test/java/cristina/tech/fancydress/worker/event/DressEventStreamTest.java) provides an example on how I've unit tested the Spring Cloud Stream `@StreamListener` operation.

```java

import cristina.tech.fancydress.worker.domain.Brand;
import cristina.tech.fancydress.worker.domain.Dress;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK, classes = BootifulDressApplication.class)
public class DressEventStreamTest {

    @MockBean
    private DressEventStoreService dressEventStoreService;

    @MockBean
    private RatingEventStoreService ratingEventStoreService;

    @MockBean
    private DressRepository dressRepository;

    @MockBean
    private BrandRepository brandRepository;

    @Autowired
    private DressEventStream dressEventStream;

    @Test
    public void testReceiveDressMessageEventSuccessfulApply() {
        DressMessageEvent dressMessageEvent = new DressMessageEvent();
        dressMessageEvent.setEventType(DressEventType.UPDATED);
        dressMessageEvent.setPayloadKey("dressy-ten");
        Dress dress = new Dress();
        dress.setId("dressy-ten");
        dress.setName("new-name");
        Brand brand = new Brand();
        brand.setName("revamped");
        dress.setBrand(brand);
        dressMessageEvent.setPayload(dress);

        cristina.tech.fancydress.store.domain.Brand storedBrand = new cristina.tech.fancydress.store.domain.Brand(brand.getName(), null);
        cristina.tech.fancydress.store.domain.Dress dressToBeUpdated = new cristina.tech.fancydress.store.domain.Dress(dress.getId());
        when(dressRepository.findById(dressMessageEvent.getPayloadKey())).thenReturn(Optional.of(dressToBeUpdated));
        when(brandRepository.findByName(brand.getName())).thenReturn(Optional.of(storedBrand));
        when(dressEventStoreService.apply(dressMessageEvent)).thenReturn(true);

        dressEventStream.receiveDressMessageEvent(dressMessageEvent);

        verify(dressEventStoreService, times(1)).apply(dressMessageEvent);
    }

}
```

### Why should you care about writing tests?

<img class="img-responsive" src="{{site.baseurl }}/img/posts/tdd/test_coverage.png" alt="Test Coverage Comic"/>

In case you never make mistakes, good for you, and you probably shouldn't care. It also means that your peers should never end up fixing issues in any of the code you've written. Or that you will most likely
be the one whom will ever interact with the code again to refactor it or extend its functionality.

One of my consistent findings during working in computer software industry is that software engineers & developers that do test their own code, and practice to some extent test driven development, have more empathy towards their peers and pride in their work.
They do care if their code will be thrown away or totally rewritten because of poor test coverage. They do care if their code is written in a manner that encourages testability and readability: “programs must be written for people to read, and only incidentally for machines to execute”.

Nonetheless, in the end you get to choose what you spend your brain’s CPU cycles on! Happy testing & thanks for reading.
