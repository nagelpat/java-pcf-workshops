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
- [Cloud Foundry services](#cloud-foundry-services)
  - [Lab - Load flights from an in-memory database](#load-flights-from-an-in-memory-database)
  - [Lab - Load flights from a database](#load-flights-from-a-provisioned-database)  
  - [Lab - Load flights' fares from a 3rd-party application](#load-flights-fares-from-an-external-application)
  - [Lab - Load flights fares from an external application using User Provided Services](#load-flights-fares-from-an-external-application-using-user-provided-services)
  - [Lab - Let external application access a platform provided service](#let-external-application-access-a-platform-provided-service)
- [Routes and Domains](#routes-and-domains)
  - [Lab - Organizing application routes](#organizing-application-routes)
  - [Lab - Private and Public routes/domains](#private-and-public-routesdomains)
  - [Lab - Blue-Green deployment](#blue-green-deployment)
  - [Lab - Routing Services](#routing-services)
- [Build packs](#build-packs)
  - [Lab - Adding functionality](#adding-functionality)
  - [Lab - Changing functionality](#changing-functionality)
- [Make applications resilient](#make-applications-resilient)


<!-- /TOC -->
# Introduction

`git clone https://github.com/MarcialRosales/java-pcf-workshops.git`

# Pivotal Cloud Foundry Technical Overview

Reference documentation:
- https://docs.pivotal.io
- [Elastic Runtime concepts](http://docs.pivotal.io/pivotalcf/concepts/index.html)


## Run Spring boot app
We have a spring boot application which provides a list of available flights based on some origin and destination.

You can use the existing code or
1. `git checkout master`
2. `cd java-pcf-workshops/apps/flight-availability`
3. `mvn spring-boot:run`
4. `curl 'localhost:8080?origin=MAD&destination=FRA'`

We would like to make this application available to our clients. How would you do it today?

## Run web site
We also want to deploy the Maven site associated to our flight-availability application so that the team can check the latest java unit reports and/or the javadocs.

1. `git checkout master`
2. `cd java-pcf-workshops/apps/flight-availability`
3. `mvn site:run`
4. Go to your browser, and check out this url `http://localhost:8080`

We would like to make this application available only within our organization, i.e. not publicly available to our clients. How would you do it today?

# Deploying simple apps

Reference documentation:
- [Using Apps Manager](http://docs.pivotal.io/pivotalcf/1-9/console/index.html)
- [Using cf CLI](http://docs.pivotal.io/pivotalcf/1-9/cf-cli/index.html)
- [Deploying Applications](http://docs.pivotal.io/pivotalcf/1-9/devguide/deploy-apps/deploy-app.html)
- [Deploying with manifests](http://docs.pivotal.io/pivotalcf/1-9/devguide/deploy-apps/manifest.html)

## Deploy Spring boot app
Deploy flight availability and make it publicly available on a given public domain

1. `git checkout master`
2. `cd java-pcf-workshops/apps/flight-availability`
3. Build the app  
  `mvn install`
4. Deploy the app  
  `cf push flight-availability -p target/flight-availability-0.0.1-SNAPSHOT.jar --random-route`
5. Try to deploy the application using a manifest
6. Check out application's details, whats the url?  
  `cf app flight-availability`  
7. Check out the health of the application ([actuator](https://github.com/MarcialRosales/java-pcf-workshops/blob/master/apps/flight-availability/pom.xml#L37-L40)):  
  `curl <url>/health`

## Deploy web site
Deploy Maven site associated to the flight availability and make it internally available on a given private domain

1. `git checkout master`
2. `cd java-pcf-workshops/apps/flight-availability`
3. Build the site  
  `mvn site`
4. Deploy the app  
  `cf push flight-availability-site -p target/site --random-route`
5. Check out application's details, whats the url?  
  `cf app flight-availability-site`  

# Cloud Foundry services

## Load flights from an in-memory database

We want to load the flights from a relational database. We are implementing the `FlightService` interface so that we can load them from a `FlightRepository`. We need to convert `Flight` to a *JPA Entity*. We [added](https://github.com/MarcialRosales/java-pcf-workshops/blob/load-flights-from-db/apps/flight-availability/pom.xml#L41-L49) **hsqldb** a *runtime dependency* so that we can run it locally.

1. `git checkout load-flights-from-in-memory-db`
2. `cd apps/flight-availability`
3. Run the app  
  `mvn spring-boot:run`
4. Test it  
  `curl 'localhost:8080?origin=MAD&destination=FRA'` shall return `[{"id":2,"origin":"MAD","destination":"FRA"}]`

Can we deploy this application directly to PCF?

## Load flights from a provisioned database

We want to load the flights from a relational database (mysql) provisioned by the platform not an in-memory database.

1. `git checkout load-flights-from-db`
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

1. `git checkout load-fares-from-external-app`
2. `cd apps/flight-availability` (on terminal 1)
3. Run the flight-availability app
  `mvn spring-boot:run`
4. `cd apps/fare-service` (on terminal 2)
5. Run the fare-service apps  
  `mvn spring-boot:run`
4. Test it  (on terminal 3)  
  `curl 'localhost:8080/fares/origin=MAD&destination=FRA'` shall return something like this `[{"fare":"0.016063185475725605","origin":"MAD","destination":"FRA","id":"2"}]`

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

Let's have a look at how the `flight-availability` talks to the `fare-service`. First of all, the implementation of the `FareService` interface uses `RestTemplate` to call the Rest endpoint.
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

And we provide the credentials for the `fare-service` in the `application.yml`:
```
fare-service:
  uri: http://localhost:8081
  username: user
  password: password

```

We tested it that it works locally. Now let's deploy to PCF. First we need to deploy `fare-service` to PCF. Then we deploy  `flight-availability` service. Do we need to make any changes? We do need to configure the credentials to our fare-service.

We have several ways to configure the credentials for the `fare-service` in `flight-availability`.

1. Set credentials in application.yml, build the flight-availability app (`mvn install`) and push it (`cf push <myapp> -f target/manifest.yml`).
	```
	fare-service:
		uri: <copy the url of the fare-service in PCF>
	```
2. Set credentials as environment variables in the manifest. Thanks to Spring boot configuration we can do something like this:
	```
	env:
	  FARE_SERVICE_URI: http://<fare-service-uri>
		FARE_SERVICE_USERNAME: user
		FARE_SERVICE_PASSWORD: password
	```
	Rather than modifying the manifest again lets simly verify that this method works. Lets simply set a wrong username via command-line:
	```
	cf set-env <myapp> FARE_SERVICE_USERNAME "bob"
	cf env <myapp> 	(dont mind the cf restage warning message)
	cf restart <myapp>
	```
	And now test it,
	`curl 'https://mr-fa-cronk-iodism.apps-dev.chdc20-cf.solera.com/fares?origin=MAD&destination=FRA'`
	should return  
	`{"timestamp":1490776955527,"status":500,"error":"Internal Server Error","exception":"org.springframework.web.client.HttpClientErrorException","message":"401 Unauthorized","path":"/fares"``


3. Inject credentials using a User Provided Service.
We are going to tackle this step in a separate lab.


## Load flights fares from an external application using User Provided Services

**Reference documentation**:
- [Spring Cloud Connectors](http://cloud.spring.io/spring-cloud-connectors/spring-cloud-connectors.html)
- [Extending Spring Cloud Connectors](http://cloud.spring.io/spring-cloud-connectors/spring-cloud-connectors.html#_extending_spring_cloud_connectors)
- [Configuring Service Connections for Spring applications in Cloud Foundry](https://docs.cloudfoundry.org/buildpacks/java/spring-service-bindings.html)


1. Create a User Provided Service which encapsulates the credentials we need to call the `fare-service`:  
 	`cf uups fare-service -p '{"uri": "https://user:password@<your-fare-service-uri>" }'`  
2. Add `fare-service` as a service to the `flight-availability` manifest.yml
	```
	  ...
		services:
		- flight-repository
		- fare-service
	```
	When we push the `flight-availability`, PCF will inject the `fare-service` credentials to the `VCAP_SERVICES` environment variable.   

3. Create a brand new project called `cloud-services` where we extend the *Spring Cloud Connectors*. This project is able to parse `VCAP_SERVICES` and extract the credentials of standard services like relational database, RabbitMQ, Redis, etc. However we can extend it so that it can parse our custom service, `fare-service`. This project can work with any cloud, not only CloudFoundry. However, given that we are working with Cloud Foundry we will add the implementation for Cloud Foundry:
	```
		<dependency>
        	<groupId>org.springframework.cloud</groupId>
        	<artifactId>spring-cloud-cloudfoundry-connector</artifactId>
        	<version>1.2.3.RELEASE</version>
    </dependency>  

	```

4. Create a *ServiceInfo* class that holds the credentials to access the `fare-service`. We are going to create a generic [WebServiceInfo](https://github.com/MarcialRosales/java-pcf-workshops/blob/load-fares-from-external-app-with-cups/apps/cloud-services/src/main/java/io/pivotal/demo/cups/cloud/WebServiceInfo.java) class that we can use to call any other web service.  
5. Create a *ServiceInfoCreator* class that creates an instance of *ServiceInfo* and populates it with the credentials exposed in `VCAP_SERVICES`. Our generic [WebServiceInfoCreator](https://github.com/MarcialRosales/java-pcf-workshops/blob/load-fares-from-external-app-with-cups/apps/cloud-services/src/main/java/io/pivotal/demo/cups/cloud/cf/WebServiceInfoCreator.java). We are extending a class which provides most of the implementation. However, we cannot use it as is due to some limitations with the *User Provided Services* which does not allow us to tag our services. Instead, we need to set the tag within the credentials attribute. Another implementation could be to extend from `CloudFoundryServiceInfoCreator` and rely on the name of the service starting with a prefix like "ws-" for instance "ws-fare-service".
6. Register our *ServiceInfoCreator* to the *Spring Cloud Connectors* framework by adding a file called [org.springframework.cloud.cloudfoundry.CloudFoundryServiceInfoCreator](https://github.com/MarcialRosales/java-pcf-workshops/blob/load-fares-from-external-app-with-cups/apps/cloud-services/src/main/resources/META-INF/services/org.springframework.cloud.cloudfoundry.CloudFoundryServiceInfoCreator) with this content:
	```
	io.pivotal.demo.cups.cloud.cf.WebServiceInfoCreator
	```
7. Provide 2 types of *Configuration* objects, one for *Cloud* and one for non-cloud (i.e. when running it locally). The *Cloud* one uses *Spring Cloud Connectors* to retrieve the `WebServiceInfo` object. First of all, we build a *Cloud* object and from this object we look up the *WebServiceInfo* and from it we build the *RestTemplate*.
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

8. Build and push the `flight-availability` service
9. Test it `curl 'https://<my flight availability app>/fares?origin=MAD&destination=FRA'`
10. Maybe it fails ...
11. Maybe we had to declare the service like this: `cf uups fare-service -p '{"uri": "https://user:password@<your fare service uri>", "tag": "WebService" }'`  


Note in the logs the following statement: `No suitable service info creator found for service fare-service Did you forget to add a ServiceInfoCreator?`. *Spring Cloud Connectors* can go one step further and create the ultimate application's service instance rather than only the *ServiceInfo*.
We leave to the attendee to modify the application so that it does not need to build a *FareService* Bean instead it is built via the *Spring Cloud Connectors* library.

	- Create a FareServiceCreator class that extends from `AbstractServiceConnectorCreator<FareService, WebServiceInfo>`
	- Register the FareServiceCreator in the file `org.springframework.cloud.service.ServiceConnectorCreator` under the `src/main/resources/META-INF/services` folder. Put the fully qualified name of your class in the file. e.g:
		```
		com.example.web.FareServiceCreator
		```
	- We don't need now the *Cloud* configuration class because the *Spring Cloud Connectors* will automatically create an instance of *FareService*.

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


## Organizing application routes

To work on this lab, we are going to checkout the branch `routes`.

We have the following apps: `app1`, `app2`, `app3` pushed to PCF using this [manifest](https://github.com/MarcialRosales/java-pcf-workshops/blob/routes/apps/lab2-manifest.yml). And we want to expose them the following way:

<public_domain>/v1/tasks -> app1
<public_domain>/v1/vin -> app2
<public_domain>/v1 -> app3

0. `git checkout routes`,
1. `cd apps\demo`,
2. build the demo app `mvn install`,
3. `cd ..``
1. Push 3 apps : `cf push -f labs-manifest.yml`
2. Create subdomain: `cf create-domain <org> mr.apps-dev.chdc20-cf.solera.com` (not absolutely necessary)
3. Create first route : `cf create-route development mr.apps-dev.chdc20-cf.solera.com --path v1/tasks `. This step is only necessary if we want to reserve the route. Say we dont have the applications ready to be deployed but we want to reserve their routes.
4. Check the routes: `cf routes` returns at least this line:
  `development                                     mr.apps-dev.chdc20-cf.solera.com          /v1/tasks`
5. Map the route to app1 :  `cf map-route app1 mr.apps-dev.chdc20-cf.solera.com --path v1/tasks`  
6. Test it: `curl https://mr.apps-dev.chdc20-cf.solera.com/v1/tasks/env | jq .vcap | grep vcap.application.name ` shall return `"vcap.application.name": "app1"`
7. Note: The applications, in this case app1, receives the full url, i.e. `/v1/tasks/*`. See lab2-manifest.yml for exact details.
8. Create 2nd route: `cf map-route app2 mr.apps-dev.chdc20-cf.solera.com --path v1/vin `
9. Test it: `curl https://mr.apps-dev.chdc20-cf.solera.com/v1/vin/env | jq .vcap | grep vcap.application.name ` shall return `"vcap.application.name": "app2"`
10. Create 3rd route: `cf map-route app3 mr.apps-dev.chdc20-cf.solera.com --path v1 `
11. Test it: `curl https://mr.apps-dev.chdc20-cf.solera.com/v1/env | jq .vcap | grep vcap.application.name ` shall return `"vcap.application.name": "app3"`


As you can see it is a pretty basic proxy with limited capability to do any fancy stuff like url rewrites and/or define routes per operation (GET, etc.). But at least, with this lab we can an idea of the kind of things we can build.

## Private and Public routes/domains

What domains exists in our organization? try `cf domains`.  Anyone is private? and what does it mean private?  Private domain is that domain which is registered with the internal IP address of the load balancer. And additionally, this private domain is not part of the public wildcard DNS name used to name public servers. In other words, there wont be any DNS server able to resolve the private domain.

The lab consists in leveraging private domains so that only internal applications are accessible within the platform. Lets use the `fare-service` as an internal application.

There are various ways to implement this lab. One way is to actually declare the private domain in the application's manifest and redeploy it. Another way is to play directly with the route commands (`create-route`, and `delete-route`, `map-route`, or `unmap-route`).


## Blue-Green deployment

Use the demo application to demonstrate how we can do blue-green deployments.

## Routing Services

**Reference documentation**:
- https://docs.pivotal.io/pivotalcf/services/route-services.html
- https://docs.pivotal.io/pivotalcf/devguide/services/route-binding.html

The purpose of the lab is to take any application and add a proxy layer that only accepts requests which carry a JWT Token else it fails with it a `401 Unauthorized`.
*Reminder: Routing service is a mechanism that allows us to filter requests never to alter the original endpoint. We can reject the request or pass it on as it is or modified, e.g adding extra headers.*

**Lab**:

1. Create a Spring Boot application with a `web` dependency
  ```
    <dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
  ```
2. Add a `@Controller` class that expects 3 headers:
  ```
  @Controller
  class Proxy {

  ...
    @RequestMapping(headers = { FORWARDED_URL, PROXY_METADATA, PROXY_SIGNATURE })
  	ResponseEntity<?> service(RequestEntity<byte[]> incoming) {
      if (jwtToken == null || !isValid(jwtToken)) {
        this.logger.error("Incoming Request missing or not valid JWT Token: {}", incoming);
        return notAuthorized();
      }

      RequestEntity<?> outgoing = getOutgoingRequest(incoming);
      this.logger.debug("Outgoing Request: {}", outgoing);

      return this.restOperations.exchange(outgoing, byte[].class);
  	}
    ...
  }
  ```
3. Validate JWT token header by simply checking that it starts with "Bearer". If it is not valid and/or it is missing, log it as an error.
4. Forward request to the uri in `X-CF-Forwarded-Url` along with the other 2 headers `X-CF-Proxy-Metadata` and `X-CF-Proxy-Signature`. We remove the `X-CF-Forwarded-Url` header as it is longer needed.
5. Build the app `mvn install`

To test it locally we proceed as follow:
1. Run the previous demo app
2. `cd demo`
3. Run it on port 8081 : `mvn spring-boot:run -Dserver.port=8081` (on  terminal 1)
4. Run the route-service app
5. `cd route-service`
6. `mvn spring-boot:run` (on terminal 2)
7. Simulate request coming from a client via **CF Router** for url `http://localhost:8081` without any JWT token:
  ```
   curl -v -H "X-CF-Forwarded-Url: http://localhost:8081/" -H "X-CF-Proxy-Metadata: some" -H "X-CF-Proxy-Signature: signature "  localhost:8080/
  ```
  We should get a 400 Bad Request
8. Simulate request coming from a client via **CF Router** for url `http://localhost:8081` with invalid JWT token:
  ```
   curl -v -H "X-CF-Forwarded-Url: http://localhost:8081" -H "X-CF-Proxy-Metadata: some" -H "X-CF-Proxy-Signature: signature " -H "Authorization: hello" localhost:8080/
  ```
  We should get a 401 Unauthorized
9. Simulate request coming from a client via **CF Router** for url `http://localhost:8081` with valid JWT token:
  ```
   curl -v -H "X-CF-Forwarded-Url: http://localhost:8081" -H "X-CF-Proxy-Metadata: some" -H "X-CF-Proxy-Signature: signature " -H "Authorization: Bearer hello" localhost:8080/
  ```
  We should get a 200 OK and the body `hello`

Let's deploy it to Cloud Foundry.

1. `cf push -f target/manifest.yml`
2. Create a user provided service that points to the url of our deployed `route-service`.
  ```
  cf cups route-service -r https://<route-service url>
  ```
3. Deploy a sample application:
  - `cd ..`
  - `cf push -f lab3-manifest.yml` (it will push `app1` )
4. Configure Cloud Foundry to intercept all requests for `app1` with the router service `route-service`:
  ```
  cf bind-route-service <application_domain> route-service --hostname <app_hostname>
  ```
  If you are not sure about the application_domain or app_hostname run: `cf app app1 | grep urls`. It will be <app_hostname>.<application_domain>    
5. Check that `app1` is bound to `route-service`: `cf routes`
  ```
  space         host                                 domain                          port   path   type   apps            service
  development   route-service-circulable-mistletoe   apps-dev.chdc20-cf.xxxxxx.com                        route-service
  development   app1-sliceable-jerbil                apps-dev.chdc20-cf.xxxxxx.com                        app1            route-service
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


# Build packs

**Reference documentation**:
- [Custom Buildpacks](http://docs.pivotal.io/pivotalcf/buildpacks/custom.html)

Only administrators are allowed to manage build packs. This means adding new ones, update them, change the order, and delete them. We can check what build packs are available by running `cf buildpacks`.

However, developers can specify the URI to a git repository where it resides a custom build pack. Although administrators can disable this option too.


## Adding functionality

We are going to customize the Java Build pack so that it declares a Java system property `staging.timestamp` with the timestamp when the application was staged.


1. Fork the Cloud Foundry Java buildpack from github
2. Clone your fork
3. Open the buildpack in your preferred editor
4. We add a *framework* component that will set the Java system property. To do that, first create `java-buildpack/lib/java_buildpack/framework/staging_timestamp.rb` and add the following contents:
	```
	require 'java_buildpack/framework'

	module JavaBuildpack::Framework

	  # Adds a system property containing a timestamp of when the application was staged.
	  class StagingTimestamp < JavaBuildpack::Component::BaseComponent
	    def initialize(context)
	      super(context)
	    end

	    def detect
	      'staging-timestamp'
	    end

	    def compile
	    end

	    def release
	      @droplet.java_opts.add_system_property('staging.timestamp', "'#{Time.now}'")
	    end
	  end
	end
	```
5. Next we activate our new component by adding it to `java-buildpack/config/components.yml` as seen here:
	```
	frameworks:
	  - "JavaBuildpack::Framework::AppDynamicsAgent"
	  - "JavaBuildpack::Framework::JavaOpts"
	  - "JavaBuildpack::Framework::MariaDbJDBC"
	  - "JavaBuildpack::Framework::NewRelicAgent"
	  - "JavaBuildpack::Framework::PlayFrameworkAutoReconfiguration"
	  - "JavaBuildpack::Framework::PlayFrameworkJPAPlugin"
	  - "JavaBuildpack::Framework::PostgresqlJDBC"
	  - "JavaBuildpack::Framework::SpringAutoReconfiguration"
	  - "JavaBuildpack::Framework::SpringInsight"
	  - "JavaBuildpack::Framework::StagingTimestamp" #Here's the bit you need to add!
	```
6. Commit your changes and push it to your repo
7. Push an application that uses your build pack and test that the buildpack did its job.
 	- `git checkout load-flights-from-in-memory-db`
 	- `cd apps/flight-availability`
 	- `cf push -f target/manifest.yml -b https://github.com/MarcialRosales/java-buildpack`
 	- `curl https://<your_app_uri/env | jq . | grep "staging.timestamp"`


## Changing functionality

The Java buildpack in particular is highly customizable and there is an environment setting for almost every aspect of the buildpack. However, for those rare occasions, we are going to demonstrate how we can change some functionality such as fixing the JRE version. By default, the Java build pack downloads the latest JRE patch version, i.e. 1.8.0_x.

We will update our build pack to utilize java 1.8.0_25 rather than simply the latest 1.8.

1. Change `java-buildpack/config/open_jdk_jre.yml` as shown:
	```
	repository_root: "{default.repository.root}/openjdk/{platform}/{architecture}"
	version: 1.8.0_+ # becomes 1.8.0_25
	memory_sizes:
	  metaspace: 64m.. # permgen becomes metaspace
	memory_heuristics:
	  heap: 85
	  metaspace: 10 # permgen becomes metaspace
	  stack: 5
	  native: 10
	```
2. Commit and push
3. Push the application again with this build pack and check in the staging logs that we are using JRE 1.8.0_25.

# Make applications resilient

In a distributed environment, failure of any given service is inevitable. It is impossible to prevent failures so it is better to embrace failure and assume failure will happen. The applications we have seen in the previous labs interacts with other services via a set of adaptors/libraries such as *Spring RestTemplate*. We need to be able to control the interaction with those libraries to provide greater tolerance of latency and failure. *Hystrix* does this by isolating points of access between the application and the services, stopping cascading failures across them, and providing fallback options, all of which improve the system's overall resiliency.

Principles of resiliency: (influenced by [Release It! Design and Deploy Production-Ready Software](http://pragprog.com/book/mnee/release-it))
- A failure in a service dependency should not break the upstream services
- The API should automatically take corrective action when one of its service dependencies fails
- The API should be able to show us what’s happening right now

A Hystrix's circuit breaker opens when:
- A request to the remote service times out. *Protects our service should the downstream service be down*
- The thread pool and bounded task queue used to interact with a service dependency are at 100% capacity. *Protects our service from overloading the downstream service and/or our service itself*
- The client library used to interact with a service dependency throws an exception. *Protects our service in case of a faulty downstream service*

It is important to understand that the circuit breaker does only open when the error rate exceeds certain threshold and not when the first failure occurs. The whole idea of the circuit breaker is to immediately fallback using of these 3 approaches:
- **custom fallback** - which returns a stubbed response read from a cache, etc.
- **fail silent** - which returns an empty response provided the caller expects an empty response
- **fail fast** - return back a failure when it makes no sense to return a fallback response. `503 Service not available` could be a sensible error that communicates the caller that it was not possible to attend the request

The 3rd principle is to have some visibility on the circuit breaker state to help us troubleshoot issues. The circuit breaker tracks requests and errors over a 10 second (default) rolling window. Requests and/or errors older than 10 seconds are discarded; only the results of requests over the last 10 seconds matter.

Here is an excerpt that shows what a Hystrix Dashboard looks like. It shows one circuit for one application:
- The grey 0% in the upper right shows the overall error rate for the circuit. It returns red when the rate is greater than 0 %.
- The green “Closed” word show that the circuit it healthy. If it is not healthy it shows "Open" in red.
- The green count (200,545) is the number of successfully handled requesta
- The blue count (0) is the number of short-circuited requests, i.e. Hystrix returned a custom/empty fallback or failed fast.
- The orange count (19) is the number of timed out requests
- The purple count (94) is the number of rejected requests because the exceeded the maximum concurrency level
- The red count (0) is the number of failed requests to the downstream service
- Each circuit has a circle to the left that encodes call volume (size of the circle - bigger means more traffic) and health (color of the circle - green is healthy and red indicates a service that’s having problems)
- The sparkline indicates call volume over a 2 minute rolling window
- The stats below the Circuit state corresponds to the average, median and percentiles response times.

![Sample Circuit Dashboard](assets/dashboard-0.png)

## Labs

We are not going to use the applications from the previous labs instead we are going to work on 2 brand new applications, *client* and *service-a*, because they will allow us to simulate failures in the downstream service, *service-a*. These applications are in the `add-resiliency` branch. Check it out `git checkout add-resiliency`.

First we are going to start with a non-resilient version of these two applications. Then we test the lack of resiliency using 3 scenarios and assess the performance and availability impact on the *client* application. Once we understand the implications of a non-resilient application, we make the corresponding code changes to make it resilient. Before we test the resiliency, we explore various ways exist to monitor whats happening (remember this is the 3rd principle of resiliency). Finally, we proceed to test various failure scenarios and identify the problems thru the Hystrix dashboard.

- [Start with non-resilient client-server](hystrix-README.md#introduction)
- [Test lack of resiliency](hystrix-README.md#Test-lack-of-resiliency)
	- [*service-a* goes down]()
	- [*service-a* is unexpectedly too slow to respond]()
	- [*service-a* is constantly failing]()
- [Make application resilient](hystrix-README.md#Make-application-resilient)
	- [Monitor the circuits using Actuator]()
	- [Monitor the circuits using Hystrix dashboard]()
	- [*service-a* goes down]()
	- [*service-a* becomes very slow to respond]()
	- [*service-a* is constantly failing]()
	- [Prevent overloading *service-a*]()
- [How to configure Hystrix](hystrix-README.md#How-to-configure-Hystrix)
