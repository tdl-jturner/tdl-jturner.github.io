+++
draft = false
date = 2019-09-11T10:03:58-07:00
title = "Flutter - Writing The Code"
+++

This is a continuation of the initial [Flutter Design]({{< ref "flutter-design" >}}). In this post, I'll talk about writing the actual code and how it developed from an idea to a running project. When I started laying out Flutter, I wanted to make it so that each microservice was simple and self contained yet also consistent designed for modularity. I also wanted to start with a framework that I knew well and was very accessible for me. Thus I selected [Java](https://www.java.com) with [Spring](https://spring.io) and {{% gh Cassandra "apache/cassandra" %}} with {{% gh Casquatch "tmobile/casquatch" %}}.

## Service Registry
To start off this project, I needed to select a service registry. For the sake of this project, a service registry is simply a central place that all the microservices register with on startup and provides an API to round robin over them. After a bit of exploration, I selected {{% gh "Netflix Eureka" "Netflix/eureka" %}}. Eureka offers very easy integration though {{% gh "Spring Cloud Netflix" "spring-cloud/spring-cloud-netflix" %}}.

### Eureka Server
First step to getting going with Eureka is to spin up a server itself. This was done with a trivial spring project by adding the dependency in the pom
{{< highlight xml >}}
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
{{< /highlight >}}
Then simply adding the annotation to the base Application class
{{< highlight java >}}
@EnableEurekaServer
{{< /highlight >}}
Finally adding some basic properties
{{< highlight ini >}}
server.port=8761
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
{{< /highlight >}}
Once this was started up there was a simple console running at http://localhost:8761. See the full code at {{% gh Service-Registry "tdl-jturner/flutter/blob/master/java/service-registry" %}}.

### Eureka Client
Adding a Eureka Client wasn't much more complicated when using Spring as the dependency add does the heavy lifting for you.

First, I added the dependency
{{< highlight xml >}}
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
{{< /highlight >}}

Added a few properties to bootstrap.properties
{{< highlight ini >}}
spring.application.name=Client-Name
server.port=0
server.servlet.context-path=/${spring.application.name}
{{< /highlight >}}

Then added a few more to the application.properties. I did end up moving these to common module but that is optional:
{{< highlight ini >}}
eureka.client.service-url.defaultZone=http://localhost:8761/eureka
eureka.instance.preferIpAddress=true
{{< /highlight >}}
With this, the clients will automatically register with Eureka on startup. Occasionally this can take some time, so patience and checking the Eureka Console is going to be helpful.
Example of this can be seen in {{% gh Follow-Dao "tdl-jturner/flutter/blob/master/java/follow-dao" %}}.

### Client Usage
Once the client is wired up, querying Eureka is done through the EurekaClient object.
First, we auto wire that up with:
{{< highlight java >}}
@Autowired
private EurekaClient eurekaClient;
{{< /highlight >}}
Find the next client with
{{< highlight java >}}
eurekaClient.getNextServerFromEureka("ServiceName",false);
{{< /highlight >}}
or if you want https then use
{{< highlight java >}}
eurekaClient.getNextServerFromEureka("ServiceName",true);
{{< /highlight >}}
Now you have a string like https://127.0.0.1:51235 referencing the host and ip of the registered service.
Example of this can be seen in {{% gh FollowServiceController "tdl-jturner/flutter/blob/master/java/follow-service/src/main/java/net/thedigitallink/flutter/service/follow/FollowServiceController.java#L35" %}}.

## DAO
The DAOs all follow the same general model and thus I'll focus on just the Follow DAO.

First there is a model which mimics the table in Cassandra as a simple mapping of two user names in this example. (See [Casquatch Manual](https://tmobile.github.io/casquatch/overview/concepts/#casquatch-entity) for more information on this.)
{{< highlight java >}}
@CasquatchEntity
@Getter @Setter @NoArgsConstructor
public class Follow extends AbstractCasquatchEntity {
    @PartitionKey
    private String follower;
    private String author;
}
{{< /highlight >}}

The REST services exist in a class annotated with @RestController from Spring and contain simple API Wrappers from {{% gh Casquatch "tmobile/casquatch" %}}. I started using the [auto generated rest api](https://tmobile.github.io/casquatch/features/restapi/) then stripped out what I did not need.
An API would look something like this
{{< highlight java >}}
@RequestMapping(value = "/save", method= RequestMethod.POST)
public Response<Void> save(@RequestBody Request<Follow> request) {
    log.trace("POST | /save | {}",request.toString());
    return new Response<>(casquatchDao.save(Follow.class,request.getPayload(),request.getQueryOptions()), Response.Status.SUCCESS);
}
{{< /highlight >}}
To use this API, you would submit a POST to /save with JSON request similar to
{{< highlight json >}}
{"payload": {"follower":"MyUsername", "author":"OtherUsername"}}
{{< /highlight >}}
Example of this can be seen in {{% gh FollowDaoService "tdl-jturner/flutter/blob/master/java/follow-dao/src/main/java/net/thedigitallink/flutter/dao/follow/FollowDaoService.java#L19" %}}.

## Service
While the DAOs were simple wrappers due to {{% gh Casquatch "tmobile/casquatch" %}} sitting beneath them. The Service layers were designed to do a little more logic as they needed to interact with each other.

As they are going to be passing around JSON objects, they needed a set of models to parse the JSON Requests / Responses. I started by having these within the microservices but this ended up being too much duplicated code and thus I rolled them in to a {{% gh "common module" "tdl-jturner/flutter/tree/master/java/common/src/main/java/net/thedigitallink/flutter/service/models" %}}. I did use a few Abstract base classes to deduplicate the code but the models are little more than a class to parse the JSON.

The URIs for each API are derived from a simple utility function as follows:
{{< highlight java >}}
private URI getUri(String service,String api) {
    InstanceInfo instanceInfo = eurekaClient.getNextServerFromEureka(service.toUpperCase(),false);
    return URI.create(String.format("http://%s:%s/%s%s",instanceInfo.getIPAddr(),instanceInfo.getPort(),service.toLowerCase(),api));
}
{{< /highlight >}}
This function queries Eureka to get the instance information and combines that with the service name to build a URI like http://localhost:51235/user-dao/get for the api call.

A service API then looks something like the following:
{{< highlight java >}}
@RequestMapping(value = "/get/{id}", method=RequestMethod.GET)
public ResponseEntity<Message> getMessage(@PathVariable String id) {
   log.trace("GET | /get/{}",id);
   try {
       ResponseEntity<MessageResponse> entity = restTemplate.postForEntity(getUri("message-dao","/get"),new HttpEntity<>(Message.builder().id(UUID.fromString(id)).build().toRequestString(),httpHeaders), MessageResponse.class);
       return new ResponseEntity<>(entity.getBody().getPayload().get(0),entity.getStatusCode());
   }
   catch (Exception e) {
       log.trace("ERROR",e);
       return new ResponseEntity<>(HttpStatus.INTERNAL_SERVER_ERROR);
   }
}
{{< /highlight >}}
Here the code is accepting a simple GET API for something like  message-service/get/12345 to pull a message. It then builds a new Message Object using the provided id number, wraps it in an HttpEntity, and posts this created object to the DAO api which would be message-dao/get. The result is finally processed and an appropriate status code is returned.

## Testing
For testing, I wanted to make it usable on additional microservices regardless of them being part of the same project so I made it a completely independent module that was a EurekaClient of its own. This required two main utility functions.The first is is a random object generator  and the second the same getUri from above that queries Eureka to get the server for the URI.

The random object generator is built using {{% gh podam "mtedone/podam" %}} to populate POJOs with random data and then saves it through the relevant DAO microservice. This looks something like:
{{< highlight java >}}
private User random() {
    User user = podamFactory.manufacturePojoWithFullData(User.class);
    restTemplate.postForEntity(getUri("user-dao","/save"),new HttpEntity<>(user.toRequestString(),httpHeaders),UserResponse.class);
    return user;
}
{{< /highlight >}}

Now that the POJO is created and saved, a basic service test would look something like:
{{< highlight java >}}
@Test
public void testGet() {
    User user = random();
    ResponseEntity<UserResponse> entity = restTemplate.postForEntity(getUri("user-dao","/get"),new HttpEntity<>(user.toRequestString(),httpHeaders), UserResponse.class);
    assert(entity.getStatusCode().is2xxSuccessful());
    assertEquals(user.getUsername(), entity.getBody().getPayload().get(0).getUsername());
}
{{< /highlight >}}
In this test, the object is posted against the user-dao service and the returning payload is compared.
Full examples can be seen in  {{% gh "Integration-Tests" "tdl-jturner/flutter/blob/master/java/integration-tests/src/test/java/net/thedigitallink/flutter/tests" %}}.

## User Interface
The User Interface is extremely bare bones but serves as a starting point for testing. My goal would be to write this in a more advanced format as an exploration of a UI technology. For now it is simply SpringMVC using Freemarker templates.

### Login
Login was done insecurely and at the most simple way for the sake of demonstration. In order to login as a user there is a /login/{username} api which is linked to from /UserList. /UserList provides a list of all known users including message and follow counts for you to select from. The logged in username is stored in plain text in a cookie.

### Timeline
The timeline is the main part of the UI as it is the primary interface for the application. A user may access their own timeline from /timeline or another user's timeline with /timeline/{username}. Either method calls /timeline-service/get/{username} to retrieve the relevant messages. This API in turn calls /timeline-servie/refresh/{username} which compiles a list of all messages for the user and all followed users which have not yet been stored in the timeline for that user. It then inserts them and returns the results.

The difference between the two timeline interfaces is that if accessing one's own timeline they see an option to post a message while accessing another user's timeline provides a follow or unfollow link as appropriate.

## Updated Architecture
A few adjustments to the architecture were needed as the project was fleshed out. In addition the SearchService and NotificationService have not yet been developed.
{{<mermaid>}}
graph TD
    UI-->ServiceRegistry
    ServiceRegistry-->MessageService
    ServiceRegistry-->TimelineService
    ServiceRegistry-->FollowService
    ServiceRegistry-->UserService
    TimelineService-->MessageService
    TimelineService-->FollowService
    TimelineService-->TimelineDAO
    MessageService-->MessageDAO
    UserService-->UserDAO
    FollowService-->FollowDAO
    TimelineDAO-->DataStore
    MessageDAO-->DataStore
    UserDAO-->DataStore
    FollowDAO-->DataStore
{{< /mermaid >}}

## Next Steps
In the next post of this series, I plan to break down how the load generation tool functions to similar customer traffic. I'll come back to the SearchService and NotificationService as these are going to require additional technologies. Stay tuned!
