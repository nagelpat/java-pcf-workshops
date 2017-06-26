PCF Developers workshop
==

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Introduction](#Introduction)
- [Pivotal Cloud Foundry Technical Overview](#pivotal-cloud-foundry-technical-overview)
	- [Lab - Run Spring boot app](#run-spring-boot-app)
	- [Lab - Run web site](#run-web-site)
- [Deploying simple apps](#deploying-simple-apps)
  - [Lab - Deploy Spring boot app](#deploy-spring-boot-app)
  - [Lab - Deploy web site](#Deploy-web-site)
	- [Deploying applications with application manifest](#deploying-applications-with-application-manifest)
- [Cloud Foundry services](#cloud-foundry-services)
  - [Lab - Load flights from a database](#load-flights-from-a-provisioned-database)  
  - [Lab - Load flights fares from an external application using User Provided Services](#load-flights-fares-from-an-external-application-using-user-provided-services)
  - [Lab - Let external application access a platform provided service](#let-external-application-access-a-platform-provided-service)
- [Routes and Domains](#routes-and-domains)
  - [Lab - Private and Public routes/domains](#private-and-public-routesdomains)
  - [Lab - Blue-Green deployment](#blue-green-deployment)
  - [Lab - Routing Services](#routing-services)
- [Orgs, Spaces and Users](orgsSpacesUsers-README.md)
- [Build packs](buildpack-README.md)
  - [Lab - Adding functionality](buildpack-README.md#adding-functionality)
  - [Lab - Changing functionality](buildpack-README.md#changing-functionality)

<!-- /TOC -->
# Introduction

## Prerequisites

- Java JDK 1.8
- Maven 3.3.x
- Latest git client
- CF client (https://docs.cloudfoundry.org/cf-cli/install-go-cli.html)
- `curl` or `Postman` (https://www.getpostman.com/) or similar http client.
- Ideally, a github account but not essential.
`git clone https://github.com/pivotalservices/java-pcf-workshops.git`

# Pivotal Cloud Foundry Technical Overview

Reference documentation:
- https://docs.pivotal.io
- [Elastic Runtime concepts](http://docs.pivotal.io/pivotalcf/concepts/index.html)


# Deploying simple apps

CloudFoundry excels at the developer experience: deploy, update and scale applications on-demand regardless of the application stack (java, php, node.js, go, etc).  We are going to learn how to deploy 2 types of applications: java and a static web page without writing any logic/script to make it happen.

Reference documentation:
- [Using Apps Manager](https://docs.pivotal.io/pivotalcf/1-11/console/index.html)
- [Using cf CLI](https://docs.pivotal.io/pivotalcf/1-11/cf-cli/index.html)
- [Deploying Applications](https://docs.pivotal.io/pivotalcf/1-11/devguide/deploy-apps/deploy-app.html)
- [Deploying with manifests](https://docs.pivotal.io/pivotalcf/1-11/devguide/deploy-apps/manifest.html)

In the next sections we are going to deploy a Spring Boot application and a web site. Before we proceed with the next sections we are going to checkout the following repository which has the Java projects we are about to deploy.

1. `git clone https://github.com/nagelpat/java-pcf-workshops.git`
2. `cd java-pcf-workshops`
3. `git fetch`
4. `git checkout load-flights-from-in-memory-db`

## Deploy a Spring boot app
Deploy flight availability and make it publicly available on a given public domain

1. `cd apps/flight-availability`
3. Build the app  
  `mvn install`
4. Deploy the app  
  `cf push flight-availability -p target/flight-availability-0.0.1-SNAPSHOT.jar --random-route`
5. Check out application's details, whats the url?  
  `cf app flight-availability`  
6. Check out the health of the application (thanks to the [actuator](https://github.com/nagelpat/java-pcf-workshops/blob/master/apps/flight-availability/pom.xml#L37-L40)) `/health` endpoint:  
  `curl <url>/health`
7. Check out the environment variables of the application (thanks to the [actuator](https://github.com/nagelpat/java-pcf-workshops/blob/master/apps/flight-availability/pom.xml#L37-L40)) `/env` endpoint:  
  `curl <url>/env`

## Deploy a web site
Deploy Maven site associated to the flight availability and make it internally available on a given private domain

2. Assuming you are under `apps/flight-availability`
3. Build the site. Maven literally downloads hundreds of jars to generate the maven site with all the project reports such as javadoc, sure-fire reports, and others. For this reason, there is already a `site` folder built from `mvn site`. Have a look with `ls -l site`
4. Deploy the app 
   `cf push flight-availability-site -p site --random-route`  use this command if you are pushing the already built site
   (If you have a good internet connection, try this command instead:
   `mvn site` followed by `cf push flight-availability-site -p target/site --random-route`)
5. Check out application's details, whats the url?  
  `cf app flight-availability-site`  


## Deploying applications with application manifest

Rather than passing a potentially long list of parameters to `cf push` we are going to move those parameters to a file so that we don't need to type them everytime we want to push an application. This file is called  *Application Manifest*.

The equivalent *manifest* file for the command `cf push flight-availability -p  target/flight-availability-0.0.1-SNAPSHOT.jar -i 2 --random-route` is:

```
---
applications:
- name: flight-availability
  instances: 2
  path: target/flight-availability-0.0.1-SNAPSHOT.jar
  random-route: true
```

The `flight-availability` comes with a default `app-manifest.yml` in the root folder. This manifest is not ready to use as it is, it must be pre-processed by Maven to produce a `target/app-manifest.yml`. When we run `mvn install` Maven produces this `target/app-manifest.yml`.

To use this manifest we do the following:
`cf -f target/app-manifest.yml`


*Things we can do with the manifest.yml file* (more details [here](https://docs.pivotal.io/pivotalcf/1-11/devguide/deploy-apps/manifest.html))
- [ ] simplify push command with manifest files (`-f <manifest>`, `-no-manifest`)
- [ ] register applications with DNS (`domain`, `domains`, `host`, `hosts`, `no-hostname`, `random-route`, `routes`). We can register http and tcp endpoints.
- [ ] deploy applications without registering with DNS (`no-route`) (for instance, a messaging based server which does not listen on any port)
- [ ] specify compute resources : memory size, disk size and number of instances!! (Use manifest to store the 'default' number of instances ) (`instances`, `disk_quota`, `memory`)
- [ ] specify environment variables the application needs (`env`)
- [ ] as far as CloudFoundry is concerned, it is important that application start (and shutdown) quickly. If we are application is too slow we can adjust the timeouts CloudFoundry uses before it deems an application as failed and it restarts it:
	- `timeout` (60sec) Time (in seconds) allowed to elapse between starting up an app and the first healthy response from the app
	- `env: CF_STAGING_TIMEOUT` (15min) Max wait time for buildpack staging, in minutes
	- `env: CF_STARTUP_TIMEOUT` (5min) Max wait time for app instance startup, in minutes
- [ ] CloudFoundry is able to determine the health status of an application and restart if it is not healthy. We can tell it not to check or to checking the port (80) is opened or whether the http endpoint returns a `200 OK` (`health-check-http-endpoint`, `health-check-type`)
- [ ] CloudFoundry builds images from our applications. It uses a set of scripts to build images called buildpacks. There are buildpacks for different type of applications. CloudFoundry will automatically detect the type of application however we can tell CloudFoundry which buildpack we want to use. (`buildpack`)
- [ ] specify services the application needs (`services`)

## Platform guarantees: 4 High-Availability levels

We have seen how we can scale our application (`cf scale -i #` or `cf push  ... -i #`). When we specify the number of instances, we create implicitly creating a contract with the platform. The platform will try its best to guarantee that the application has those instances. Ultimately the platform depends on the underlying infrastructure to provision new instances should some of them failed. If the infrastructure is not ready available, the platform wont be able to comply with the contract. Besides this edge case, the platform takes care of our application availability.

Let's try to simulate our application crashed. To do so we enable the `/shutdown` endpoint in the *actuator*.
`cf set-env flight-availability ENDPOINTS_SHUTDOWN_ENABLED true`

We restart the application: `cf restart flight-availability`

And we send the `/shutdown` request to one of the instances: `curl -X POST https://<url>/shutdown`

If we have +1 instances, we have zero-downtime because the other instances are available to receive requests while PCF creates a new one. If we had just one instance, we have downtime of a few seconds until PCF provisions another instance.



# Cloud Foundry services

## Load flights from a provisioned database

We want to load the flights from a relational database (mysql) provisioned by the platform not an in-memory database. We are implementing the `FlightService` interface so that we can load them from a `FlightRepository`. We need to convert `Flight` to a *JPA Entity*. We [added](https://github.com/nagelpat/java-pcf-workshops/blob/load-flights-from-db/apps/flight-availability/pom.xml#L41-L49) **hsqldb** a *runtime dependency* so that we can run it locally.

1. `git checkout load-flights-from-db` (execute it from root folder of the cloned repo)
2. `cd apps/flight-availability`
3. Run the app  
  `mvn spring-boot:run`
4. Test it  
  `curl 'localhost:8080?origin=MAD&destination=FRA'` shall return `[{"id":2,"origin":"MAD","destination":"FRA"}]`
5. Before we deploy our application to PCF we need to provision a mysql database. If we tried to push the application without creating the service we get:
	```
	...
	FAILED
	Could not find service flight-repository to bind to mr-fa
	```

  `cf marketplace`  Check out what services are available

  `cf marketplace -s p-mysql pre-existing-plan ...`  Check out the service details like available plans

  `cf create-service ...`   Create a service instance with the name `flight-repository`

  `cf service ...`  Check out the service instance. Is it ready to use?

6. Push the application using the manifest. See the manifest and observe we have declared a service:

  ```
  applications:
  - name: flight-availability
    instances: 1
    memory: 1024M
    path: @project.build.finalName@.@project.packaging@
    random-route: true
    services:
    - flight-repository

  ```

7. Check out the database credentials the application is using:  
  `cf env flight-availability`

8. Test the application. Whats the url?

9. We did not include any jdbc drivers with the application. How could that work?


## Load flights fares from an external application

We want to load the flights from a relational database and the prices from an external application. For the sake of this exercise, we are going to mock up the external application in cloud foundry.

This diagram illustrates the architecture:
```
	---->[ flight-availability ] ---> [ fare-service ]
               |
               \/
    ( flight-repository db )

```

In the next sections we are going to describe the code changes we have done to transform the current version in the branch `load-flights-from-db` into the branch `load-fares-from-external-app-with-cups`.

First of all, we are going to checkout the new branch `load-fares-from-external-app-with-cups`.
1. `git checkout load-fares-from-external-app-with-cups`

### Create the external fare-service application
Let's have a look at the `fare-service`. It is a pretty basic REST app configured with basic auth (Note: *We could have simply relied on the spring security default configuration properties*):
```
server.port: 8081

fare:
  credentials:
    user: user
    password: password

```
And it simply returns a random fare for each requested flight:
```
@RestController
public class FareController {

	private Random random = new Random(System.currentTimeMillis());

	@PostMapping("/")
	public String[] applyFares(@RequestBody Flight[] flights) {
		return Arrays.stream(flights).map(f -> Double.toString(random.nextDouble())).toArray(String[]::new);
	}
}
```

Push the `fare-service` application to *Cloud Foundry*. This project comes with a convenient `manifest.yml`.
1. would you push the application with this command invoked from the `fare-service` folder ? `cf push`
2. did it work? why not?
3. Maybe you should have pushed it with `cf push -f target/manifest.yml`
4. Is there another way? `cd target`, `cf push` ?


### Make flight-availability call fare-service application
Let's have a look at how the `flight-availability` calls the `fare-service`. First of all, the implementation of the `FareService` interface uses `RestTemplate` to call the Rest endpoint.
```
@Service
public class FareServiceImpl implements FareService {

	private final RestTemplate restTemplate;

	public FareServiceImpl(@Qualifier("fareService") RestTemplate restTemplate) {
		this.restTemplate = restTemplate;
	}

	@Override
	public String[] fares(Flight[] flights) {

		 return restTemplate.postForObject("/", flights, String[].class);

	}

}
```
And we build the `RestTemplate` specific for the `FareService` (within `FlightAvailabilityApplication.java`). See how we setup the RestTemplate with basic auth and the root uri for any requests to the `fare-service` endpoint:
```
@Configuration
@ConfigurationProperties(prefix = "fare-service")
class FareServiceConfig {
	String uri;
	String username;
	String password;
	public String getUri() {
		return uri;
	}
	public void setUri(String uri) {
		this.uri = uri;
	}
	public String getUsername() {
		return username;
	}
	public void setUsername(String username) {
		this.username = username;
	}
	public String getPassword() {
		return password;
	}
	public void setPassword(String password) {
		this.password = password;
	}

	@Bean(name = "fareService")
	public RestTemplate fareService(RestTemplateBuilder builder, FareServiceConfig fareService) {
		return builder.basicAuthorization(getUsername(), getPassword()).rootUri(getUri()).build();
	}
}
```

### Configure flight-availability with fare-services's credentials

In traditional Java development we configured the credentials (i.e. url, username, password, etc) via a `Properties` file or `XML` file.
For instance, the `application.yml` would look like this:
```
fare-service:
  uri: http://localhost:8081
  username: user
  password: password

```

The issue with this approach is that when we push java applications to *Cloud Foundry* we push a single artifact, a `jar` or a `war`. Which forces us to bundle the properties file within the artifact, not great.

Another approach, more cloud-native, is to provide those credentials thru environment variables for instance via the `manifest.yml`.

	```
	env:
        FARE_SERVICE_URI: http://<fare-service-uri>
        FARE_SERVICE_USERNAME: user
        FARE_SERVICE_PASSWORD: password
	```

This approach has one inconvenience which is manifested when this `fare-service` is shared by more than one application. Why? We would have to set the same environment variables to all the applications.

Another approach, the recommended one, is to use *User Provided Service* in *Cloud Foundry*. This is what we are going to use in this lab.


## Load flights fares from an external application using User Provided Services

**Reference documentation**:
- [Spring Cloud Connectors](http://cloud.spring.io/spring-cloud-connectors/spring-cloud-connectors.html)
- [Extending Spring Cloud Connectors](http://cloud.spring.io/spring-cloud-connectors/spring-cloud-connectors.html#_extending_spring_cloud_connectors)
- [Configuring Service Connections for Spring applications in Cloud Foundry](https://docs.cloudfoundry.org/buildpacks/java/spring-service-bindings.html)

The following steps describe the code changes we had to make to consume credentials from a **User Provided Service**. We don't need to write any code because it is already provided in the java project `cloud-services`. However, we are going to walk you thru the code changes.

1. Create a brand new project called `cloud-services` where we extend the *Spring Cloud Connectors*. This project is able to parse `VCAP_SERVICES` and extract the credentials of standard services like relational database, RabbitMQ, Redis, etc. However we can extend it so that it can parse our custom service, `fare-service`. This project can work with any cloud, not only CloudFoundry. However, given that we are working with Cloud Foundry we will add the implementation for Cloud Foundry:
	```
    <dependency>
        	<groupId>org.springframework.cloud</groupId>
        	<artifactId>spring-cloud-cloudfoundry-connector</artifactId>
        	<version>1.2.3.RELEASE</version>
    </dependency>  

	```

2. Create a *ServiceInfo* class that holds the credentials to access the `fare-service`. We are going to create a generic [WebServiceInfo](https://github.com/MarcialRosales/java-pcf-workshops/blob/load-fares-from-external-app-with-cups/apps/cloud-services/src/main/java/io/pivotal/demo/cups/cloud/WebServiceInfo.java) class that we can use to call any other web service.  

3. Create a *ServiceInfoCreator* class that creates an instance of *ServiceInfo* and populates it with the credentials exposed in `VCAP_SERVICES`. Our generic [WebServiceInfoCreator](https://github.com/MarcialRosales/java-pcf-workshops/blob/load-fares-from-external-app-with-cups/apps/cloud-services/src/main/java/io/pivotal/demo/cups/cloud/cf/WebServiceInfoCreator.java). We are extending a class which provides most of the implementation. However, we cannot use it as is due to some limitations with the *User Provided Services* which does not allow us to tag our services. Instead, we need to set the tag within the credentials attribute. Another implementation could be to extend from `CloudFoundryServiceInfoCreator` and rely on the name of the service starting with a prefix like "ws-" for instance "ws-fare-service".

4. Register our *ServiceInfoCreator* to the *Spring Cloud Connectors* framework by adding a file called [org.springframework.cloud.cloudfoundry.CloudFoundryServiceInfoCreator](https://github.com/MarcialRosales/java-pcf-workshops/blob/load-fares-from-external-app-with-cups/apps/cloud-services/src/main/resources/META-INF/services/org.springframework.cloud.cloudfoundry.CloudFoundryServiceInfoCreator) with this content:
	```
	io.pivotal.demo.cups.cloud.cf.WebServiceInfoCreator
	```

5. Provide 2 types of *Configuration* objects, one for *Cloud* and one for non-cloud (i.e. when running it locally). The *Cloud* one uses *Spring Cloud Connectors* to retrieve the `WebServiceInfo` object. First of all, we build a *Cloud* object and from this object we look up the *WebServiceInfo* and from it we build the *RestTemplate*.
	```
	@Configuration
	@Profile({"cloud"})
	class CloudConfig  {

		@Bean
		Cloud cloud() {
			return new CloudFactory().getCloud();
		}

	    @Bean
	    public WebServiceInfo fareServiceInfo(Cloud cloud) {
	        ServiceInfo info = cloud.getServiceInfo("fare-service");
	        if (info instanceof WebServiceInfo) {
	        	return (WebServiceInfo)info;
	        }else {
	        	throw new IllegalStateException("fare-service is not of type WebServiceInfo. Did you miss the tag attribute?");
	        }
	    }

	    @Bean(name = "fareService")
		public RestTemplate fareService(RestTemplateBuilder builder, WebServiceInfo fareService) {
			return builder.basicAuthorization(fareService.getUserName(), fareService.getPassword()).rootUri(fareService.getUri()).build();

		}
	}
	```

6. We have the code and before we push the application we are going to create a *User Provided Service* that gives us the credentials to call `fare-service`:  
	`cf cups fare-service -p '{"uri": "https://user:password@<your-fare-service-uri>" }'`  

9. Declare `fare-service` as a service to the `flight-availability` manifest.yml
		```
		  ...
			services:
			- flight-repository
			- fare-service
		```
		When we push the `flight-availability`, PCF will inject the `fare-service` credentials to the `VCAP_SERVICES` environment variable.   

10. Build and push the `flight-availability` service

11. Test it `curl 'https://<my flight availability app>/fares?origin=MAD&destination=FRA'`
12. Does it work? Check the logs `cf logs flight-availability --recent`
13. It looks like our WebServiceInfoCreator has not being called or has not created a WebServiceInfo. Why? Maybe because it could not find any declaration which had the tag = "WebService".. Lets modify the service.
`cf uups fare-service -p '{"uri": "https://user:password@<your fare service uri>", "tag": "WebService" }'`  


Advanced lab (if time permits):

Note in the logs the following statement: `No suitable service info creator found for service fare-service Did you forget to add a ServiceInfoCreator?`. *Spring Cloud Connectors* can go one step further and create the ultimate application's service instance rather than only the *ServiceInfo*.
We leave to the attendee to modify the application so that it does not need to build a *FareService* Bean instead it is built via the *Spring Cloud Connectors* library.

	- Create a FareServiceCreator class that extends from `AbstractServiceConnectorCreator<FareService, WebServiceInfo>`
	- Register the FareServiceCreator in the file `org.springframework.cloud.service.ServiceConnectorCreator` under the `src/main/resources/META-INF/services` folder. Put the fully qualified name of your class in the file. e.g:
		```
		com.example.web.FareServiceCreator
		```
	- We don't need now the *Cloud* configuration class because the *Spring Cloud Connectors* will automatically create an instance of *FareService*.

### Consume services using a purely declarative approach

Reference documentation:
 - https://spring.io/blog/2015/04/27/binding-to-data-services-with-spring-boot-in-cloud-foundry
 - https://spring.io/blog/2015/01/27/12-factor-app-style-backing-services-with-spring-and-cloud-foundry


The Cloud Foundry **Java build pack** does auto reconfiguration for you. From the docs:

>	Auto-reconfiguration consists of three parts. First, it adds the cloud profile to Spring’s list of active profiles. Second it exposes all of the properties contributed by Cloud Foundry as a PropertySource in the ApplicationContext. Finally it re-writes the bean definitions of various types to connect automatically with services bound to the application.

If you prefer not to write Java code to read credentials from `VCAP_SERVICES` via the Spring Cloud Connectors, you can use this other mechanism which extracts the same credentials but from **Java Properties**. For instance, the `fare-service` configuration might look like this:
```
fare-service:
  url: ${vcap.services.fare-service.credentials.url}
  username: ${vcap.services.fare-service.credentials.username}
	password: ${vcap.services.fare-service.credentials.password}
```

The **Java build pack** (and in particular, the java auto-configuration module) exposes all the services declared in the `VCAP_SERVICES` as environment properties.


## Let external application access a platform provided service

Most likely, all the applications will run within the platform. However, if we ever had an external application access a service provided by the platform, say a database, there is a way to do it.

1. Create a service instance
2. Create a service key
	`cf create-service-key <serviceInstanceName> <ourServiceKeyName>`
3. Get the credentials `cf service-key <serviceInstanceName> <ourServiceKeyName>`. Share the credentials with the external application.

Creating a service-key is equivalent to binding an application to a service instance. The service broker creates a set of credentials for the application.


# Routes and Domains

**Reference documentation**:
- http://docs.cloudfoundry.org/devguide/deploy-apps/routes-domains.html


## Private and Public routes/domains (internal vs external facing applications)

What domains exists in our organization? try `cf domains`.  Anyone is private? and what does it mean private?  Private domain is that domain which is registered with the internal IP address of the load balancer. And additionally, this private domain is not registered with any public  DNS name. In other words, there wont be any DNS server able to resolve the private domain.

The lab consists in leveraging private domains so that only internal applications are accessible within the platform. Lets use the `fare-service` as an internal application.

There are various ways to implement this lab. One way is to actually declare the private domain in the application's manifest and redeploy it. Another way is to play directly with the route commands (`create-route`, and `delete-route`, `map-route`, or `unmap-route`).


## Blue-Green deployment

Reference documentation:
 - https://docs.cloudfoundry.org/devguide/deploy-apps/blue-green.html

Use the demo application to demonstrate how we can do blue-green deployments using what you have learnt so far with regards routes.

How would you do it? Say Blue is the current version which is running and green is the new version.

Key command: `cf map-route` and `cf unmap-route`


## Routing Services (intercept every request to decide whether to accept it or enrich it or track it)

**Reference documentation**:
- https://docs.pivotal.io/pivotalcf/services/route-services.html
- https://docs.pivotal.io/pivotalcf/devguide/services/route-binding.html

The purpose of the lab is to take any application and add a proxy layer that only accepts requests which carry a JWT Token else it fails with it a `401 Unauthorized`.
*Reminder: Routing service is a mechanism that allows us to filter requests never to alter the original endpoint. We can reject the request or pass it on as it is or modified, e.g adding extra headers.*

### Create the Proxy (or router service)
**Lab**: The code is already provided in the `routes` branch however we are going to walk thru the code below:

1. Create a Spring Boot application with a `web` dependency
  ```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
  ```
3. Create a @Controller class :
	```java
	@RestController
	class RouteService {

		static final String FORWARDED_URL = "X-CF-Forwarded-Url";

		static final String PROXY_METADATA = "X-CF-Proxy-Metadata";

		static final String PROXY_SIGNATURE = "X-CF-Proxy-Signature";

		private final Logger logger = LoggerFactory.getLogger(this.getClass());

		private final RestOperations restOperations;

		@Autowired
		RouteService(RestOperations restOperations) {
			this.restOperations = restOperations;
		}

	}
	```
2. Add a single request handler that receives all requests:
  ```java
	@RequestMapping(headers = { FORWARDED_URL, PROXY_METADATA, PROXY_SIGNATURE })
	ResponseEntity<?> service(RequestEntity<byte[]> incoming, @RequestHeader(name = "Authorization", required = false) String jwtToken ) {
		if (jwtToken == null) {
			this.logger.error("Incoming Request missing JWT Token: {}", incoming);
			return badRequest();
		} else if (!isValid(jwtToken)) {
			this.logger.error("Incoming Request missing or not valid JWT Token: {}", incoming);
			return notAuthorized();
		}

		RequestEntity<?> outgoing = getOutgoingRequest(incoming);
		this.logger.debug("Outgoing Request: {}", outgoing);

		return this.restOperations.exchange(outgoing,  byte[].class);
	}

  }
  ```
3. Validate JWT token header by simply checking that it starts with "Bearer". If it is not valid and/or it is missing, log it as an error.
	```java
	private static boolean isValid(String jwtToken) {
		return jwtToken.contains("Bearer"); // TODO add JWT Validation
	}
	```
4. Forward request to the uri in `X-CF-Forwarded-Url` along with the other 2 headers `X-CF-Proxy-Metadata` and `X-CF-Proxy-Signature`. We remove the `Authorization` header as it is longer needed:
	```java
	private static RequestEntity<?> getOutgoingRequest(RequestEntity<?> incoming) {
		HttpHeaders headers = new HttpHeaders();
		headers.putAll(incoming.getHeaders());

		URI uri = headers.remove(FORWARDED_URL).stream().findFirst().map(URI::create)
				.orElseThrow(() -> new IllegalStateException(String.format("No %s header present", FORWARDED_URL)));
		headers.remove("Authorization");

		return new RequestEntity<>(incoming.getBody(), headers, incoming.getMethod(), uri);
	}
	```
5. Build the app `mvn install`


### Test the proxy locally

To test it locally we proceed as follow:
1. Run the previous flight-availability app (assume that it is running on 8080)
3. Run this route-service app on port 8888 (or any other that you prefer) on a separate terminal : `mvn spring-boot:run -Dserver.port=8888`
7. Simulate request coming from a client via **CF Router** for url `http://localhost:8080` without any JWT token:
  ```
   curl -v -H "X-CF-Forwarded-Url: http://localhost:8080/" -H "X-CF-Proxy-Metadata: some" -H "X-CF-Proxy-Signature: signature "  localhost:8888/
  ```
  We should get a 400 Bad Request
8. Simulate request coming from a client via **CF Router** for url `http://localhost:8080` with invalid JWT token:
  ```
   curl -v -H "X-CF-Forwarded-Url: http://localhost:8080" -H "X-CF-Proxy-Metadata: some" -H "X-CF-Proxy-Signature: signature " -H "Authorization: hello" localhost:8888/
  ```
  We should get a 401 Unauthorized
9. Simulate request coming from a client via **CF Router** for url `http://localhost:8080` with valid JWT token:
  ```
   curl -v -H "X-CF-Forwarded-Url: http://localhost:8080" -H "X-CF-Proxy-Metadata: some" -H "X-CF-Proxy-Signature: signature " -H "Authorization: Bearer hello" localhost:8888/
  ```
  We should get a 200 OK and the body `hello`

### Test the proxy in Cloud Foundry using Router Service functionality
Let's deploy it to Cloud Foundry.

1. `cf push -f target/manifest.yml`
2. Create a user provided service that points to the url of our deployed `route-service`.
  ```
  cf cups route-service -r https://<route-service url>
  ```
3. Deploy the flight-availability app if it is not already deployed:

4. Configure Cloud Foundry to intercept all requests for `flight-availability` with the router service `route-service`:
  ```
  cf bind-route-service <application_domain> route-service --hostname <app_hostname>
  ```
  If you are not sure about the application_domain or app_hostname run: `cf app flight-availability | grep routes`. It will be <app_hostname>.<application_domain>    
5. Check that `flight-availability` is bound to `route-service`: `cf routes`
  ```
  space         host                                 domain                          port   path   type   apps            		service
  development   route-service-circulable-mistletoe   apps-dev.chdc20-cf.xxxxxx.com                        route-service
  development   app1-sliceable-jerbil                apps-dev.chdc20-cf.xxxxxx.com                        flight-availability route-service
  ```
5. Run in a terminal `cf logs route-service` to watch its logs
6. Try a url which has no JWT token:
  ```
  curl -v https://<app_hostname>.<application_domain>
  ```
  We should get back a 400 Bad Request
7. Try a url which has an invalid JWT token:
```
curl -v -H "Authorization: hello" https://<app_hostname>.<application_domain>
```
We should get back a 401 Unauthorized
8. Finally, try a url which has a valid JWT Token:
```
curl -v -H "Authorization: Bearer hello" https://<app_hostname>.<application_domain>
```
We should get back a 200 OK and the outcome from the `/` endpoint which is `hello`.
