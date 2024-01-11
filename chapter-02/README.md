# Chapter 02

REST endpoints

# Goal

Basic knowledge of REST endpoints and your REST API exposes some.

# Context and Knowledge

* We want to design our Kniffel REST API as simple as possible, this decision drives some design aspects
    * our use-case is a "couch co-op" scenario - all players use the same client UI
    * we do NOT separate endpoints for game, player/account, dice
    * our API isn't concerned about any authentication as all players sit next to each other
* We need the following functionality as REST endpoints (so we need 4 endpoints):
    1. create a game, specifying the players name
    1. retrieve the current game information, statistics, options
    1. decide which dice to re-roll
    1. decide which category to score the current dice roll
* What's important to know about "REST endpoints"?
    * REST is not protocol, not a specification, it's a philosophie
    * still REST uses the http protocol and JSON as input/output data
    * so each **endpoint** is the same as an **URL** - like /api/game or /api/game/37475/roll-dice
    * http (not *REST*) defines several "methods", GET, POST, PATCH, PUT, DELETE - but REST uses this idea to distingish between create, read, update, delete. it's important to understand, that only the combination of an URL and a method defines an endpoint
    * once a REST API endpoint is offered to the public, you should not change it, therefore we always want to use a version number in an endpoint - if we want to change something, we increase the version number in the URL
* http methods - conventions for REST:
    * GET - data retrieval or any kind or form
    * POST - creation of a data record / resource or invokation of an action
    * PUT - updating a data record / resource (all fields)
    * PATCH - updating some fields of a data record / resource
    * DELETE - deleting a data record / resource
* Lombok is a Java library to type less code. With simple annotations it will generate code for you

## Step 1 - Our first endpoint

Let's create a new Java class (in Java all code must live in classes) which serves a REST endpoint.

### Create a Java Classes

Next to the KniffelApplication.java file, create a file "ServerStatusController.java" like this:

```java
// Every java file needs a package at the top.
// The package needs to match the folder structure after ./src/main/java,
// just with . instead of / or \
package com.oglimmer.kniffel;

// The file name ( + .java) needs to match the class name!
public class ServerStatusController { 
}
```

That's a java class without any attributes or methods. It should compile without errors.

### Define a RestController

Now we need to make this class a RestController using a Java annotation. Spring works a lot with annotations. Annotations always start with `@`

```java
// If you want to use any other class you have to import it first.
// With * at the end, you import all classes in this package
import org.springframework.web.bind.annotation.*;

// this annotation enables this class for http REST processing
@RestController 
public class ServerStatusController {
}
```

Keep in mind, that you don't see the code which actually does the RestController processing. @RestController is just a hint for the Spring runtime code, that Spring should look into this class for endpoints methods.

### Add an actual endpoint 🎉

We need to add a method and let's Spring know the URL and method (see context above!)

```java
import org.springframework.web.bind.annotation.*;
import java.lang.management.*;

@RestController
// this is the base URL for all endpoint methods in this class
@RequestMapping("/api/v1/server")
public class ServerStatusController {

    // method = POST
    // url = /api/v1/server/thread-dump
    @PostMapping("/thread-dump")
    public void threadDump() {
        // this code writes the StackTrace of all threads to stdout
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        for (ThreadInfo threadInfo : threadMXBean.dumpAllThreads(true, true)) {
            System.out.println(threadInfo);
        }
    }

}
```

Start the application and execute the http request

```bash
curl "http://localhost:8080/api/v1/server/thread-dump" --request POST --verbose
```

As this is a POST request you cannot just put it into the browser address bar (these are always GET requests). You need to use JavaScript. This html page uses the fetch API to execute the http request:

```html
<!DOCTYPE html>
<html lang="en">
<body>

<button id="execButton">Execute Request</button>

<script>
document.getElementById('execButton').addEventListener('click', async () => {
    try {
        const response = await fetch("http://localhost:8080/api/v1/server/thread-dump", { method: "POST" });
        alert(`Request successful! Return code = ${response.status}. See dev console for more.`);
    } catch (error) {
        console.error('Fetch error:', error);
        alert('Error occurred! See dev console for more.');
    }
});
</script>

</body>
</html>
```

Save this html as `test.html` and run `docker run --rm -v $PWD:/usr/share/nginx/html -p 80:80 nginx` in the same directory. Access the page with a browser at http://localhost/test.html

Does it work? No? Why?

When you check the dev console (what you should always do in those cases), you see the error message "CORS Missing Allow Origin" in the network tab.

## Step 2 - Understanding CORS

* CORS stands for "Cross-Origin Resource Sharing" - but we need start a step earlier
* Browsers have a "Same-Origin-Policy", which means JavaScript can only do http requests to the **same host and the same port** as your website. In our above's example we call from localhost:80 to localhost:8080 - this is forbidden by the Same-Origin-Policy.
* So CORS provides a possibility to overwrite the Same-Origin-Policy
* What's CORS technically? CORS are http response headers send by the REST API. Our REST API needs to send a header like this:

```
Access-Control-Allow-Origin: *
```

This will tell the browser:

> All websites are allowed to call this REST API.

If you want to be more secure, you can only allow certain websites to call your REST API:

```
Access-Control-Allow-Origin: http://localhost
```

This will tell the browser:

> The website running on http://localhost is allowed to call this REST API (which as we know runs on http://localhost:8080)

Adding CORS headers to our resonses in Java Spring is easy:

```java
@RestController
@RequestMapping("/api/v1/server")
@CrossOrigin // this will allow *
// @CrossOrigin(origins = "http://localhost")
public class ServerStatusController {
    // ... no changes here
}
```

Test the browser page again!

Before we continue, it should be said that CORS has many more aspects, like you need to allow headers and methods individually, also CORS has the concept of preflight requests. But our Kniffel REST API does not require attention to these topics.

## Step 3 - Returning JSON

* REST uses JSON as the data format for input and ouput of data
* Let's add a "status" endpoint which returns status information via JSON.

We add a method and a @GetMapping annotation with the url to our class:

```java
public class ServerStatusController {
    // GET because this is pure data retrieval
    @GetMapping("/status")
    public ServerStatus getServerStatus() {
        // in Java you have to create all objects with: new <Class name>()
        ServerStatus serverStatus = new ServerStatus();
        // At this point we have a new object of class ServerStatus and we hold a
        // reference in the variable serverStatus.
        // This object lives on the Heap and is memory managed by Java (a Garbage Collector
        // will delete it if no references are pointing to it anymore)
        serverStatus.setServerTime(System.currentTimeMillis());
        serverStatus.setServerVersion("1.0.0");
        serverStatus.setServerName("Kniffel Server");
        serverStatus.setServerStatus("OK");
        return serverStatus;
    }
}
```

To return JSON we only need to return a Java object. Spring will convert this to JSON by default.

Let's create this class. Create a file ServerStatus.java like this:

```java
import lombok.*;

@Getter // this will generate getters for all attributes
@Setter // this will generate setter for all attributes
public class ServerStatus {
    private long serverTime;
    private String serverVersion;
    private String serverName;
    private String serverStatus;
}
```

Such classes are often call POJOs (plain old java object) or DTOs (data transfer objects), which means it just holds some data, but does not have any methods.

Test the endpoint with a browser http://localhost:8080/api/v1/server/status or `curl http://localhost:8080/api/v1/server/status --verbose`

## Step 4 - Passing JSON into the REST API

For our Kniffel REST API we want to see how to pass an array of strings - the players names - into a POST call - the create game endpoint.

Let's create a file GameController.java, making it a RestController with one Post mapping for /api/v1/game/.

This takes the http request body as JSON and just prints it's content to stdout:

```java
@RestController
@RequestMapping("/api/v1/game")
@CrossOrigin
public class GameController {

    @PostMapping("/")
    public void createGame(
        // spring needs to know where this Java method parameter is coming from
        // @RequestBody means it's the http request body
        @RequestBody CreateGameRequest createGameRequest) {
            System.out.println("createGameRequest = " + createGameRequest);
    }
}
```

Now we need to create a new file CreateGameRequest.java for the parameter object:

```java
@Getter
@Setter
// this creates a method "toString()" which returns all attributes of the class
// we need this to make System.out.println work on this class
@ToString
public class CreateGameRequest {
    private String[] playerNames;
}
```

You can test this via terminal:

```bash
curl http://localhost:8080/api/v1/game/ --request POST --data-ascii '{"playerNames": ["oli", "ilo"]}' --header "Content-Type: application/json" --verbose
```

As we send a request body (`--data-ascii` defines the request body), we need to let Spring know what data format is used in the body, we do this via the header `Content-Type: application/json`.

## Step 5 - Extend the html page to call the "create game" endpoint

Extend the html page from above (test.html) to call our new endpoint. You have to do 3 changes:

* change the URL, the http method should be good
* provide the header for the content-type = application/json
* pass the request body (either as a fix string or get it dymanically from text field)

You can find the some help at https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch or ask chatGPT.

## Step 6 - Path variables

We have to look at one last build block for our REST endpoints: path variables.

We already saw the endpoint to create a game : POST to /api/v1/game/

To retrieve the data for an existing game the url should look like this: `/api/v1/game/<ID>` and the method should be a GET request.

```java
// ... from above
public class GameController {

    // ... from above

    // see that gameId is inside {} to make it a path variable
    @GetMapping("/{gameId}")
    public void getGameInfo(
        // the annotation @PathVariable tells spring to take the parameter data from the URL
        @PathVariable String gameId) {
            System.out.println("gameId = " + gameId);
    }
}
```

Call this with curl (like `curl http://localhost:8080/api/v1/game/37646`) or a web browser. 

## Step 7 - Designing our 4 Kniffel API endpoints

With the knowledge we have learnt from this chapter you should be able to complete or create all 4 endpoints:

1. a POST endpoint for "create game" with a request body parameter object containing String[] for player names. This needs to return at least the game id, but maybe it can also return the same data as our GET endpoint
1. a GET endpoint to retrieve an existing game with a path variable containing the ID of a game. This should return an object containing all relevant game information, like player names, their score, the thrown dice and the options to proceed and whatever you think the client needs to know.
1. a POST endpoint to re-roll the dice the player hasn't liked. This needs to have a path variable for the game ID and a request body parameter for the dice which should be kept (maybe int[]). This should also return the same information as the GET endpoint.
1. a POST endpoint to "book" the current dice roll into one of the Kniffel's categories. Again we need a path variable for the game ID and a request body parameter for the "booking type" (int or string). This should also return the same information as the GET endpoint.

Come up with an suggestions for your REST endpoint methods. **It doesn't matter if the result is not perfect - the goal is to develop a proposal.**

# What we've learnt

* annotations are key for Spring and JEE
* Java Spring annotations
    * @RestController
    * @RequestMapping(_baseurl_)
    * @CrossOrigin
    * @GetMapping(_url_postfix_)
    * @PostMapping(_url_postfix_)
    * @RequestBody
    * @PathVariable
* Lombok annotations
    * @Getter
    * @Setter
    * @ToString
* What Same-Origin-Policy or CORS is and how to use it
* How to return JSON (response body is JSON) and how to pass JSON into the REST API (request body is JSON)
* How to use path variables - a variable inside an URL

# Extras if you have time

* Read more about Browser security https://en.wikipedia.org/wiki/Same-origin_policy and https://en.wikipedia.org/wiki/Cross-origin_resource_sharing
* Read more about Lombok and it's annotations https://projectlombok.org/features/
