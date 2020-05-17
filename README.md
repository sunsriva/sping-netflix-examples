# Spring Cloud Netflix

Spring Cloud Netflix project provides Netflix OSS(Netflix OSS is a set of frameworks and libraries that Netflix wrote to solve some interesting distributed-systems problems) integrations for Spring Boot apps through autoconfiguration. With a few simple annotations you can quickly enable and configure the common patterns inside your application. The patterns provided include Service Discovery (Eureka), Circuit Breaker (Hystrix), Intelligent Routing (Zuul) and Client Side Load Balancing (Ribbon).


# Service Discovery: Eureka Clients

Let’s imagine that you are writing some code that invokes a service that has a REST API. In order to make a request, your code needs to know the network location (IP address and port) of a service instance. In a traditional application running on physical hardware, the network locations of service instances are relatively static.

In a modern, cloud-based microservices application, Service instances have dynamically assigned network locations. Moreover, the set of service instances changes dynamically because of auto-scaling, failures, and upgrades. Your client code needs to use a more elaborate service discovery mechanism.

There are two main service discovery patterns: client-side discovery and server-side discovery.

* Client-Side: When using client-side discovery, the client is responsible for determining the network locations of available service instances and load balancing requests across them. The client queries a service registry, which is a database of available service instances. The client then uses a load-balancing algorithm to select one of the available service instances and makes a request.The network location of a service instance is registered with the service registry when it starts up. It is removed from the service registry when the instance terminates. The service instance’s registration is typically refreshed periodically using a heartbeat mechanism.

* Server-side: The other approach to service discovery is the server-side discovery pattern.The client makes a request to a service via a load balancer. The load balancer queries the service registry and routes each request to an available service instance. As with client-side discovery, service instances are registered and deregistered with the service registry.

The service registry is a key part of service discovery. It is a database containing the network locations of service instances. A service registry needs to be highly available and up to date. 


** Eureka ** is the Netflix Service Discovery Server and Client. The server can be configured and deployed to be highly available, with each server replicating state about the registered services to the others.


## How to Include Eureka Client

To include the Eureka Client in your project, use the starter with a group ID of org.springframework.cloud and an artifact ID of spring-cloud-starter-netflix-eureka-client

## Registering with Eureka

When a client registers with Eureka, it provides meta-data about itself — such as host, port, health indicator URL, home page, and other details. Eureka receives heartbeat messages from each instance belonging to a service. If the heartbeat fails over a configurable timetable, the instance is normally removed from the registry.

```
@SpringBootApplication
@RestController
public class Application {

    @RequestMapping("/")
    public String home() {
        return "Hello world";
    }

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}
```

NOTE: the preceding example shows a normal Spring Boot application. By having spring-cloud-starter-netflix-eureka-client on the classpath, your application automatically registers with the Eureka Server. Configuration is required to locate the Eureka server

```
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```



The default application name (that is, the service ID), virtual host, and non-secure port (taken from the Environment) are ${spring.application.name}, ${spring.application.name} and ${server.port}, respectively.

Having spring-cloud-starter-netflix-eureka-client on the classpath makes the app into both a Eureka “instance” (that is, it registers itself) and a “client” (it can query the registry to locate other services).


## Registering a Secure Application

If your app wants to be contacted over HTTPS, you can set two flags in the EurekaInstanceConfig:

* eureka.instance.[nonSecurePortEnabled]=[false]

* eureka.instance.[securePortEnabled]=[true]



Doing so makes Eureka publish instance information that shows an explicit preference for secure communication. The Spring Cloud DiscoveryClient always returns a URI starting with https for a service configured this way. Similarly, when a service is configured this way, the Eureka (native) instance information has a secure health check URL.

Because of the way Eureka works internally, it still publishes a non-secure URL for the status and home pages unless you also override those explicitly. 

```
eureka:
  instance:
    statusPageUrl: https://${eureka.hostname}/info
    healthCheckUrl: https://${eureka.hostname}/health
    homePageUrl: https://${eureka.hostname}/
```

## Eureka’s Health Checks

By default, Eureka uses the client heartbeat to determine if a client is up. Unless specified otherwise, the Discovery Client does not propagate the current health check status of the application, per the Spring Boot Actuator. Consequently, after successful registration, Eureka always announces that the application is in 'UP' state. This behavior can be altered by enabling Eureka health checks

```
eureka:
  client:
    healthcheck:
      enabled: true
```

NOTE: eureka.client.healthcheck.enabled=true should only be set in application.yml

## Refreshing Eureka Clients

By default, the EurekaClient bean is refreshable, meaning the Eureka client properties can be changed and refreshed. When a refresh occurs clients will be unregistered from the Eureka server and there might be a brief moment of time where all instance of a given service are not available. One way to eliminate this from happening is to disable the ability to refresh Eureka clients. To do this set eureka.client.refresh.enable=false.

#  Service Discovery: Eureka Server

To include Eureka Server in your project, use the starter with a group ID of org.springframework.cloud and an artifact ID of spring-cloud-starter-netflix-eureka-server.

 
## How to Run a Eureka Server

@SpringBootApplication
@EnableEurekaServer
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}



The Eureka server does not have a back end store, but the service instances in the registry all have to send heartbeats to keep their registrations up to date (so this can be done in memory). Clients also have an in-memory cache of Eureka registrations (so they do not have to go to the registry for every request to a service).

By default, every Eureka server is also a Eureka client and requires (at least one) service URL to locate a peer. If you do not provide it, the service runs and works, but it fills your logs with a lot of noise about not being able to register with the peer.

## Standalone Mode

The combination of the two caches (client and server) and the heartbeats make a standalone Eureka server fairly resilient to failure, as long as there is some sort of monitor or elastic runtime (such as Cloud Foundry) keeping it alive. In standalone mode, you might prefer to switch off the client side behavior so that it does not keep trying and failing to reach its peers. 

##  Peer Awareness

Eureka can be made even more resilient and available by running multiple instances and asking them to register with each other. In fact, this is the default behavior, so all you need to do to make it work is add a valid serviceUrl to a peer.

```
---
spring:
  profiles: peer1
eureka:
  instance:
    hostname: peer1
  client:
    serviceUrl:
      defaultZone: https://peer2/eureka/

---
spring:
  profiles: peer2
eureka:
  instance:
    hostname: peer2
  client:
    serviceUrl:
      defaultZone: https://peer1/eureka/
```      

## Securing The Eureka Server

You can secure your Eureka server simply by adding Spring Security to your server’s classpath via spring-boot-starter-security. By default when Spring Security is on the classpath it will require that a valid CSRF token be sent with every request to the app. Eureka clients will not generally possess a valid cross site request forgery (CSRF) token you will need to disable this requirement for the /eureka/** endpoints. For example:

```
@EnableWebSecurity
class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().ignoringAntMatchers("/eureka/**");
        super.configure(http);
    }
}

```
## Disabling Ribbon with Eureka Server and Client starters

spring-cloud-starter-netflix-eureka-server and spring-cloud-starter-netflix-eureka-client come along with a spring-cloud-starter-netflix-ribbon. Since Ribbon load-balancer is now in maintenance mode, we suggest switching to using the Spring Cloud LoadBalancer, also included in Eureka starters, instead.

In order to that, you can set the value of spring.cloud.loadbalancer.ribbon.enabled property to false.

```


<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.cloud</groupId>
                    <artifactId>spring-cloud-starter-ribbon</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>com.netflix.ribbon</groupId>
                    <artifactId>ribbon-eureka</artifactId>
                </exclusion>
            </exclusions>
</dependency>

```

# Circuit Breaker

Software systems make remote calls to software running in different processes, usually on different machines across a network. One of the big differences between in-memory calls and remote calls is that remote calls can fail, or hang without a response until some timeout limit is reached. What's worse, if you have many callers on an unresponsive supplier, you can run out of critical resources leading to cascading failures across multiple systems.

The circuit breaker pattern is the solution to this problem. The basic idea behind the circuit breaker is very simple. You wrap a protected function call in a circuit breaker object, which monitors for failures. Once the failures reach a certain threshold, the circuit breaker trips, and all further calls to the circuit breaker return with an error or with some alternative service or default message, without the protected call being made at all. This will make sure system is responsive and threads are not waiting for an unresponsive call.
Different States of the Circuit Breaker

The circuit breaker has three distinct states: Closed, Open, and Half-Open:

*    Closed – When everything is normal, the circuit breaker remains in the closed state and all calls pass through to the services. When the number of failures exceeds a predetermined threshold the breaker trips, and it goes into the Open state.
*   Open – The circuit breaker returns an error for calls without executing the function.
*   Half-Open – After a timeout period, the circuit switches to a half-open state to test if the underlying problem still exists. If a single call fails in this half-open state, the breaker is once again tripped. If it succeeds, the circuit breaker resets back to the normal, closed state. 

You can implement the circuit breaker pattern with Netflix Hystrix. 

## How to Include Hystrix

To include Hystrix in your project, use the starter with a group ID of org.springframework.cloud and a artifact ID of spring-cloud-starter-netflix-hystrix.

The following example shows a minimal Eureka server with a Hystrix circuit breaker:

```
@SpringBootApplication
@EnableCircuitBreaker
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}

@Component
public class StoreIntegration {

    @HystrixCommand(fallbackMethod = "defaultStores")
    public Object getStores(Map<String, Object> parameters) {
        //do stuff that might fail
    }

    public Object defaultStores(Map<String, Object> parameters) {
        return /* something useful */;
    }
}

```

When you apply a circuit breaker to a method, Hystrix watches for failing calls to that method, and, if failures build up to a threshold, Hystrix opens the circuit so that subsequent calls automatically fail. While the circuit is open, Hystrix redirects calls to the method, and they are passed to your specified fallback method.

Spring Cloud Netflix Hystrix looks for any method annotated with the @HystrixCommand annotation and wraps that method in a proxy connected to a circuit breaker so that Hystrix can monitor it. This currently works only in a class marked with @Component or @Service.

##  Hystrix Metrics Stream

To enable the Hystrix metrics stream, include a dependency on ** spring-boot-starter-actuator ** and set ** management.endpoints.web.exposure.include: hystrix.stream. ** Doing so exposes the /actuator/hystrix.stream as a management endpoint.