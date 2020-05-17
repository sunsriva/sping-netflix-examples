# Api Gateway

Let’s imagine you are building an online store that uses the Microservice architecture pattern and that you are implementing the product details page. You need to develop multiple versions of the product details user interface.

In addition, the online store must expose product details via a REST API for use by 3rd party applications.

Since the online store uses the Microservice architecture pattern the product details data is spread over multiple services. For example,

* Product Info Service - basic information about the product such as title, author
* Pricing Service - product price
* Order service - purchase history for product
* Inventory service - product availability
* Review service - customer reviews

So, the problem is, how do the clients of a Microservices-based application access the individual services?

Solution is to implement an API gateway that is the single entry point for all clients.

Before understanding gateways and their responsibilities, let's first look at how a proxy works. A proxy server acts as a bridge which makes internal networks invisible to the internet. There are two types of proxy servers: a forward proxy and a reverse proxy.

A forward proxy is internet facing and retrieves data from the internet. A reverse proxy, on the other hand, sits in the internal network and accepts requests from the internet and forwards them to your servers in the internal network. A gateway is a reverse proxy pattern which protects access to your servers on the private network though they are not mutually exclusive.


There are many features of an API gateway :

* Security: You might think that you have set up security layers to your architecture like encrypting requests with HTTPS. I have set up a firewall for my private network. I have added authentication to my requests, etc., but there are additional security aspects which your gateway manages and helps in negotiating requests from the client.

* CORS: A gateway can come to the rescue by implementing CORS (Cross-Origin Resource Sharing) filters and having the capability of handling Cross-Origin requests. CORS is a mechanism which enables cross-domain requests and allows restricted resources. 

* DDoS and SQL Injection: Since all the traffic is routed through the gateway, there is an added advantage that insecure requests are filtered out. Many gateways are good at sanitizing inputs to common threats like SQL injection.

* Authorization and Authentication: Since gateways are entry points to your requests, it is always a better place to authorize and authenticate your end users. This helps in keeping your backend services intact without even the request reaching the business layers. 
The best way to manage authorization and authentication at an API gateway is to use OAuth and establish a handshake.

* Certificate Management: API gateways can manage certificates using their own keyStore and trustStore. Many commercial gateways have a provision to create/import certs into stores and enforce SSL between the client and the gateway.


# Spring Cloud Gateway

Spring Cloud Gateway aims to provide a simple, yet effective way to route to APIs and provide cross cutting concerns to them such as: security, monitoring/metrics, and resiliency.

To include Spring Cloud Gateway in your project, use the starter with a group ID of org.springframework.cloud and an artifact ID of spring-cloud-starter-gateway.

## Glossary

* Route: The basic building block of the gateway. It is defined by an ID, a destination URI, a collection of predicates, and a collection of filters. A route is matched if the aggregate predicate is true.

* Predicate: This is a Java 8 Function Predicate. The input type is a Spring Framework ServerWebExchange. This lets you match on anything from the HTTP request, such as headers or parameters.

* Filter: These are instances of Spring Framework GatewayFilter that have been constructed with a specific factory. Here, you can modify requests and responses before or after sending the downstream request.


## How It Works

![spring_cloud_gateway_diagram](./static/spring_cloud_gateway_diagram.png)

Clients make requests to Spring Cloud Gateway. If the Gateway Handler Mapping determines that a request matches a route, it is sent to the Gateway Web Handler. This handler runs the request through a filter chain that is specific to the request. The reason the filters are divided by the dotted line is that filters can run logic both before and after the proxy request is sent. All “pre” filter logic is executed. Then the proxy request is made. After the proxy request is made, the “post” filter logic is run.

## Configuring Route Predicate Factories and Gateway Filter Factories

There are two ways to configure predicates and filters: shortcuts and fully expanded arguments. Most examples below use the shortcut way.

Shortcut configuration is recognized by the filter name, followed by an equals sign (=), followed by argument values separated by commas (,).

```
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - Cookie=mycookie,mycookievalue
```        

Fully expanded arguments appear more like standard yaml configuration with name/value pairs. Typically, there will be a name key and an args key. The args key is a map of key value pairs to configure the predicate or filter.

```

application.yml

spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - name: Cookie
          args:
            name: mycookie
            regexp: mycookievalue

```


# Router and Filter: Zuul



Netflix uses Zuul for the following:

* Authentication
* Insights
* Stress Testing
* Canary Testing
* Dynamic Routing
* Service Migration
* Load Shedding
* Security
* Static Response handling
* Active/Active traffic management

Zuul’s rule engine lets rules and filters be written in essentially any JVM language, with built-in support for Java


## How to Include Zuul

To include Zuul in your project, use the starter with a group ID of org.springframework.cloud and a artifact ID of spring-cloud-starter-netflix-zuul

## Embedded Zuul Reverse Proxy

Spring Cloud has created an embedded Zuul proxy to ease the development of a common use case where a UI application wants to make proxy calls to one or more back end services. This feature is useful for a user interface to proxy to the back end services it requires, avoiding the need to manage CORS and authentication concerns independently for all the back ends.

To enable it, annotate a Spring Boot main class with @EnableZuulProxy. Doing so causes local calls to be forwarded to the appropriate service. By convention, a service with an ID of users receives requests from the proxy located at /users (with the prefix stripped). The proxy uses Ribbon to locate an instance to which to forward through discovery. All requests are executed in a hystrix command, so failures appear in Hystrix metrics. Once the circuit is open, the proxy does not try to contact the service.

```
 zuul:
  routes:
    users:
      path: /myusers/**
      serviceId: users_service
```

The preceding example means that HTTP calls to /myusers get forwarded to the users_service service. The route must have a path that can be specified as an ant-style pattern, so /myusers/* only matches one level, but /myusers/** matches hierarchically.


##  Cookies and Sensitive Headers

You can share headers between services in the same system, but you probably do not want sensitive headers leaking downstream into external servers. You can specify a list of ignored headers as part of the route configuration. Cookies play a special role, because they have well defined semantics in browsers, and they are always to be treated as sensitive. If the consumer of your proxy is a browser, then cookies for downstream services also cause problems for the user, because they all get jumbled up together (all downstream services look like they come from the same place).



The sensitive headers can be configured as a comma-separated list per route, as shown in the following example:
```
 zuul:
  routes:
    users:
      path: /myusers/**
      sensitiveHeaders: Cookie,Set-Cookie,Authorization
      url: https://downstream

```

## Management Endpoints

By default, if you use @EnableZuulProxy with the Spring Boot Actuator, you enable two additional endpoints:

* Routes

* Filters




A GET to the routes endpoint at /routes returns a list of the mapped routes:
GET /routes
```
{
  /stores/**: "http://localhost:8081"
}

```



Additional route details can be requested by adding the ?format=details query string to /routes. Doing so produces the following output:
GET /routes/details
```
{
  "/stores/**": {
    "id": "stores",
    "fullPath": "/stores/**",
    "location": "http://localhost:8081",
    "path": "/**",
    "prefix": "/stores",
    "retryable": false,
    "customSensitiveHeaders": false,
    "prefixStripped": true
  }
}

```

A POST to /routes forces a refresh of the existing routes (for example, when there have been changes in the service catalog). You can disable this endpoint by setting endpoints.routes.enabled to false.

Filters Endpoint

A GET to the filters endpoint at /filters returns a map of Zuul filters by type. For each filter type in the map, you get a list of all the filters of that type, along with their details.
