
# Currency Microservice

A brief description of what this project does and who it's for








## 🛠 Built With
Spring Boot, Spring Cloud, Spring Cloud Gateway, Distributed Tracing with Zipkin,
Load balancing with Eureka-Feign-Spring Cloud Load Balancer, Resilience4j (Circuit Breaker), Docker (Containerize Microservices),



## Currency Conversion Microservice

Currency conversion microservice invoke the currency exchange microservice and do the calculation after getting the response.

Instead of rest template we used Feign to communicate currency conversion microservice with currency exchange microservice. Feign is very important when we talk about microservices because of making it easy to call rest api. Feign also helps us to do load balancing very easily.

### Service Registry or Naming Serve

In microservice architecture all the instances of all the microservices would register in the service registry. Currency exchange, conversion and all the other services would register with the service registry.

When currency conversion microservice wants to talk with currency exchange microservice, it would ask service registry to get the address of currency exchange microservice. When service registry returns the address, then currency conversion microservice can send request to exchange microservice. 

All the instances would register with the naming server or service registry. When currency conversion microservice try to find what are the active instances of exchange microservice asking naming server, and then get the instances and load balance between them. 



## Spring Cloud Netflix Eureka

- For creating a naming server we can use spring-cloud-netflix Eureka Server. To avoid eureka server to register with itself, we have to add few attributes in the application.properties file


```bash
  eukera.client.register-with-eureka=false
  eureka.client.fetch-registry=false
```

<img width="1000" alt="Screen Shot 2022-08-04 at 7 48 35 pm" src="https://user-images.githubusercontent.com/44027826/182821379-d0c2078c-b9de-4f57-b238-515c3378fd78.png">


<img width="1000" alt="Screen Shot 2022-08-04 at 7 55 47 pm" src="https://user-images.githubusercontent.com/44027826/182822361-bdae86db-b2de-4baf-a7b9-ef8192aa515b.png">


- We have to add a dependency in all the client microservices in pom.xml file to register with eureka server

```bash
  spring-cloud-starter-netflix-eureka-client
```

- We have to configure the eureka server url in all the microservice’s application.properties file to register all the services with eureka server.

```bash
  eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka
```

- Load balancing between multiple instance
  - currency exchange fom currency conversion microservice
  - Add Image 1 Here
  - We want FeignClient in currency conversion service, [@FeignClient(name=”currency-exchange”)] to talk with eureka and pick up the instances of currency exchange and do load balancing between them. We don’t have to specify URL in FeignClient just the name of the application is enough. 
  - Here inside the currency conversion microservice, there is a load balancer component which talk with naming server, finding the instances and then do automatic load balance between them. This is what is called client site load balancing which is happening through Feign.

### API Gateway - Spring Cloud Routing

You can call any microservices registered with eureka through the API gateway. If we want to implement any authentications, we can develop them in api gateway and we can forward only those request which are authenticated towards microservices.

  - Register APi gateway with eureka server
  - Enable service discovery in application.properties file
  ```bash
    spring.cloud.gateway.discovery.locator.enabled=true
  ```

### Circuit Breaker: Resilience4j

Microservice 1 -> Microservice 2 -> Microservice 3 -> Microservice 4 -> Microservice 5

If a service is down we can return a fallback default response from another microservice. To reduce the load we can implement a circuit breaker pattern.  If any chances microservice 4 is down, instead of repeatedly hitting it and causing it situation more bad, we can return default response without hitting it again and again. We can also retry few times if there is temporary failure from microservice 4, and after failing multiple times I can return default response back. We can also implement rate limiting, only certain numbers of calls to specific microservice in a specific period of times.

Following annotation we can use right before controller:

  ```bash
    @Retry(name = “default”)  //Default configuration - It would retry three times
  ```

We can add specific number of retry configurations and fallback method

```bash
  @Retry(name = “sample-api”, fallbackMethod = “hardCodedResponse”)
  	public String hardCodedResponse(Exception exception){
	    return “Fallback Response”;
    }
```
In application.properties file
```bash
  resilience4j.retry.instances.sample-api.maxAttempts = 5
```
We can configure what should be the interval between retry

```bash
  resilience4j.retry.instances.sample-api.waitDuration=1s
```

Using watch command we can fire a lot of request to any specific microservice. 
If we want to allow specific number of calls in a specific time period we acn use @RateLimiter before api method. - 2 requests in every 10 seconds
```bash
  @RateLimiter(name = “default”)
```
In application properties file: 
```bash
  resilience4j.ratelimiter.instances.default.limitForPeriod=2
  resilience4j.ratelimiter.instances.default.limitRefreshPeriod=10s
```
We can also configure how many concurrent calls are allowed to any specific api method. Each api in microservice we can configure @Bulkhead
```bash
  @Bulkhead(name = “sample-api”)
  resilience4j.bulkhead.instances.sample-api.maxConcurrentCalls=10
```

### Distributed Tracing

When request goes from one microservice to another microservice, and if it is required to debug problems or tracking request across multiple microservices, that’s where we go for a feature called distributed tracing. 


<img width="1000" alt="Screen Shot 2022-08-04 at 7 55 47 pm" src="https://user-images.githubusercontent.com/44027826/182821009-0017ce47-ab0b-4f4b-bd6a-0276c811e78b.png">


All the microservices send all the information to a single distributed tracing server. This distributed tracing server would store everything in the database. Distributed tracing server provides us an interface which would allow us to trace the request across multiple microservices. We have used distributed tracing server zipkin in a docker container. 

```bash
  docker run -p 9411:9411openzipkin/zipkin:2.23 
```
```bash
  spring-cloud-starter-sleuth -> Each request is assigned a unique id by sleuth framework.
```

### RabbitMQ

What if distributed tracing server is down? We can have a queue in between to avoid losing request data. All the microservices can send information to rabbit mq. And later distributed tracing server can pick information one by one from rabbit mq. Even if distributed tracing server is down, microservices can send request information to rabbit mq.

Request Sampling: We don’t want to trace every request that goes between microservices. We sample a percentage of request. Since it would impact on performance if we want to track every requests. In application.properties file:
```bash  
  spring.sleuth.sampler.probability = 1.0 / 0.5 / 0.25
```


