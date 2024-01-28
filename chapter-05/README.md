# Chapter 05

Testing, http status codes and Defensive code

# Goal

We want to use appropriate http status codes and validate input parameters properly. This chapter shows you what 'Java' offers for Null safety and gives an introduction into unit and component testing.

# Context and Knowledge

* REST APIs should use the http status codes where possible. Here is a list of selected, useful return codes:
    * 200, OK -> the operation succeeded
    * 201, CREATED -> the operation (usually a POST) created something
    * 204, NO CONTENT -> while the operation succeeded, there is not content in the response
    * 400, BAD REQUEST -> the parameter provided where wrong, missing or cannot be decoded
    * 401, UNAUTHORIZED -> this method need authentication and this failed
    * 403, FORBIDDEN -> the user is authenticated, but not authorized, so the user doesn't have the permissions to access this operation
    * 404, NOT FOUND -> the resource / data behind this endpoint does not exist or the user does not have the permission to see it
    * 500, INTERNAL SERVER ERROR ->  something went wrong during processing
* Java was released in 1996, a time where Null safety wasn't a thing yet. That means any Java object can be null at any point in time and there is no way to find this out until you use it at runtime. In a perfect world you would check every access to every object for "not null" before using it, but in reality that means `NullPointerException` is a common problem in Java. We will see how to address this problem in 2024.
* Automated testing is a very important part of professional software development. It ensures that you or other people can change code at any point in time and easily check if the change breaks something.

# Step 1 - Http status code

An endpoint which creates an entity should return 201 not 200. Let's open `GameController` and change the status code for createGame to 201. We can do this with a new annotiation. We should also change the OpenAPI documentation accordingly:

```java
@Operation(
        summary = "Create a new game with a specific number of players",
        description = "The number of players must be at least 2. All player names must be different.", responses = {
        @ApiResponse(
                responseCode = "201", description = "Created", content = @Content(
                mediaType = MediaType.APPLICATION_JSON_VALUE,
                schema = @Schema(implementation = GameResponse.class)))})
@ResponseStatus(HttpStatus.CREATED)
@PostMapping("/")
public GameResponse createGame(@RequestBody CreateGameRequest createGameRequest) {
    KniffelGame game = gameService.createGame(createGameRequest.getPlayerNames());
    return mapGameResponse(game);
}
```

You can see the changed return code by using: 

```bash
# the -v is important to see the response headers!
curl "http://localhost:8080/api/v1/game/" -d "{\"playerNames\": [\"oli\",\"mike\"]}" -H "Content-Type: application/json" -v
```

## Validations and Error handling

At the description already says, the user needs to provide at least 2 names and they must all be different. So our API should check that and return http status codes in those cases.

To return an different http status we can simple throw a `ResponseStatusException`.

Checking for the 2 conditions could look like this:

```java
@Operation(
        summary = "Create a new game with a specific number of players",
        description = "The number of players must be at least 2. All player names must be different.", responses = {
        @ApiResponse(
                responseCode = "201", description = "Created", content = @Content(
                mediaType = MediaType.APPLICATION_JSON_VALUE,
                schema = @Schema(implementation = GameResponse.class)))})
@ResponseStatus(HttpStatus.CREATED)
@PostMapping("/")
public GameResponse createGame(@RequestBody CreateGameRequest createGameRequest) {
    if (createGameRequest == null || createGameRequest.getPlayerNames() == null || createGameRequest.getPlayerNames().length < 2) {
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "At least 2 players have to be provided");
    }
    if (Arrays.stream(createGameRequest.getPlayerNames()).distinct().count() != createGameRequest.getPlayerNames().length) {
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "All player names must be different");
    }
    KniffelGame game = gameService.createGame(createGameRequest.getPlayerNames());
    return mapGameResponse(game);
}
```

You can test this Postman or curl easily:

```bash
# the -v is important to see the response headers!
curl "http://localhost:8080/api/v1/game/" -d "{\"playerNames\": [\"oli\",\"oli\"]}" -H "Content-Type: application/json" -v
```


# Step 2 - Error handling from a different application layer

Our business / game logic could throw an exceptions as well. If we don't handle those, Spring will return a 500 - internal server error. And while this is not completely wrong we can do a better job and map those exceptions.

Let's assume we want to make sure that re-rolling can only be done, when the game is in state ROLL and booking a roll can only be done while in state BOOK.

We create a new file `IllegalGameStateException.java` and define a new class IllegalGameStateException there:

```java
// to make it an exception in java you need to extend it from Exception (or RuntimeException)
// using RuntimeException means it's an "unchecked" exception, so we don't need to declare
// its usage on all methods with the "throws" keyword
public class IllegalGameStateException extends RuntimeException {
    IllegalGameStateException(String reason) {
        super(reason);
    }
}
```

Now we can check the state in our business logic. Open `GameService` and change the two methods with a validation and throwing an `IllegalGameStateException` in case the state is illegal.

```java
    public void roll(KniffelGame kniffelGame, int[] diceToKeep) {
        if (kniffelGame.getState() != GameState.ROLL) {
            throw new IllegalGameStateException("Game is not in roll state");
        }
        kniffelGame.reRollDice(diceToKeep);
    }

    public void bookRoll(KniffelGame kniffelGame, BookingType enumBookingType) {
        if (kniffelGame.getState() != GameState.BOOK) {
            throw new IllegalGameStateException("Game is not in book state");
        }
        kniffelGame.bookDiceRoll(enumBookingType);
    }
```

Finally we open `GameController` and add this exception mapping. With this method and its annotation Spring knows that every `IllegalGameStateException` in this REST controller should be mapped into a ResponseStatusException with a 400 and a simple error message in the body (if you want to have an even better REST API you should return a JSON objects for errors as well, but for the sake of simplicity we just return a plain string here).

```java
@ExceptionHandler(IllegalGameStateException.class)
public void handleException(IllegalGameStateException e) {
    throw new ResponseStatusException(HttpStatus.BAD_REQUEST, e.getMessage());
}
````

At this point it is a good exercise to add handle invalid gameId values passed to our endpoints. What you have to do:

* create a new class `GameNotFoundException` (keep in mind that this needs to inherit from another class)
* inside class `GameService` and method `getGameInfo` check if the result of `games.get(gameId)` is null and if so, throw a `GameNotFoundException`
* add an `@ExceptionHandler(GameNotFoundException.class)` to class `GameController` and "convert" it into an appropriate ResponseStatusException

# Step 3 - `null` checks

Java was released in 1996, a time where Null safety wasn't a thing yet. That means any Java object can be null at any point in time and there is no way to find this out until you use it at runtime. In a perfect world you would check every access to every object for "not null" before using it, but in reality that means `NullPointerException` is a common problem in Java. We will see how to address this problem in 2024.

As written in the introduction, Java does not provie Null safety mechanism, but we can annotations from Spring and Lombok to improve the situation.

## Coding time helper

Let's look at our `GameService`:

```java
public KniffelGame createGame(String[] playerNames) {
    List<KniffelPlayer> players = Arrays.stream(playerNames).map(KniffelPlayer::new).collect(Collectors.toList());
    KniffelGame kniffelGame = new KniffelGame(players);
    games.put(kniffelGame.getGameId(), kniffelGame);
    return kniffelGame;
}
```

What happens when the method parameter `playerNames` is passed as `null`? Arrays.stream will throw a `NullPointerException` - at runtime.

Spring provides an annotiation `@NonNull` to mark parameters that they must not be null. Your IDE (both IntelliJ and VS code) will evalutate this annotiation at coding time and let you know when you possibly put a null value into this.

This is how the method looks with this coding time type information added:

```java
public KniffelGame createGame(@NonNull String[] playerNames) {
    List<KniffelPlayer> players = Arrays.stream(playerNames).map(KniffelPlayer::new).collect(Collectors.toList());
    KniffelGame kniffelGame = new KniffelGame(players);
    games.put(kniffelGame.getGameId(), kniffelGame);
    return kniffelGame;
}
```

Keep in mind that this will not change any program behavior at runtime! It will simply help you while coding to spot a possible NullPointerException.

## Runtime checks

There is also an annotation for runtime not null checks. It comes from lombok and looks like this `@lombok.NonNull`.

Adding this to our `craeteGame` method:

```java
public KniffelGame createGame(@NonNull @lombok.NonNull String[] playerNames) {
    List<KniffelPlayer> players = Arrays.stream(playerNames).map(KniffelPlayer::new).collect(Collectors.toList());
    KniffelGame kniffelGame = new KniffelGame(players);
    games.put(kniffelGame.getGameId(), kniffelGame);
    return kniffelGame;
}
```

At compile time Lombok will add code to this method and at runtime it will look like this:

```java
public KniffelGame createGame(@NonNull @lombok.NonNull String[] playerNames) {
    if (playerNames == null) {
        throw new NullPointerException("playerNames is marked non-null but is null");
    }
    List<KniffelPlayer> players = Arrays.stream(playerNames).map(KniffelPlayer::new).collect(Collectors.toList());
    KniffelGame kniffelGame = new KniffelGame(players);
    games.put(kniffelGame.getGameId(), kniffelGame);
    return kniffelGame;
}
```

While in this simple case, this additional check might not be very useful, image a method where access to playerNames happens only under some conditiations, so the NullPointerException might not be raised all the time. In such cases a dedicated not null check helps to spot those problems early in development.

You might want to go through our application and add both annotiations to all parameters which must not be null.

# Step 4 - Unit tests

The different types of automated tests, their purpose and usage are complex and you can write entire books about it. I will oversimplify things, but this should give you a good starting point.

> Unit tests check the correctness of a single method, mostly algorithms or calculations

Let's assume you have written your own `KniffelRulesImplementation`:

```java
public class KniffelRulesImplementation extends KniffelRules {

    // here are all other methods

    public int getScoreForXofAKind(List<Integer> diceRolls, int xOfAKind) {
        return diceRolls.stream()
            // group dice by their value, after this we have a map:
            // key: dice value 
            // value: list of dice values (even those elements always have the same integer, 
            //                but the size of the list says how often this value was thrown)
            .collect(Collectors.groupingBy(d -> d))
            // proceed with lists only
            .values()
            .stream()
            // remove all lists with size < xOfAKind
            .filter(d -> d.size() >= xOfAKind)
            // we either have 1 or 0 of those cases
            .findFirst()
            // calc the score by adding the dice values, but only count xOfAKind times
            .map(d -> d.stream().limit(xOfAKind).mapToInt(i -> i).sum())
            // if we didn't have a list with size >= xOfAKind, score 0
            .orElse(0);
    }
}
```

This logic is already quite complicated and it makes sense to write a unit test to validate the algorithm.

Create a file `RuleTest.java` under `/src/test/java` (!!!) and put this content there:

```java
import org.junit.jupiter.api.Assertions;
import org.junit.jupiter.api.Test;

import java.util.Arrays;
import java.util.List;

public class RuleTest {

    // an instance to test
    private KniffelRulesImplementation kniffelRules = new KniffelRulesImplementation();

    @Test
    public void test_threeOfKind_success() {
        List<Integer> diceRolls = Arrays.asList(1, 3, 6, 6, 6);
        int res = kniffelRules.getScoreForXofAKind(diceRolls, 3);
        Assertions.assertEquals(18, res);
    }

    @Test
    public void test_threeOfKind_success4Elements() {
        List<Integer> diceRolls = Arrays.asList(1, 1, 1, 1, 2);
        int res = kniffelRules.getScoreForXofAKind(diceRolls, 3);
        Assertions.assertEquals(3, res);
    }

    @Test
    public void test_threeOfKind_noPoints() {
        List<Integer> diceRolls = Arrays.asList(1, 3, 5, 6, 6);
        int res = kniffelRules.getScoreForXofAKind(diceRolls, 3);
        Assertions.assertEquals(0, res);
    }
}
```

You can run this inside your IDE or with

```bash
./mvnw test
```

This should result in output like this:

```bash
[INFO] --- surefire:3.1.2:test (default-test) @ kniffel ---
[INFO] Using auto detected provider org.apache.maven.surefire.junitplatform.JUnitPlatformProvider
[INFO]
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.oglimmer.kniffel.web.RuleTest
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.075 s -- in com.oglimmer.kniffel.web.RuleTest
[INFO]
[INFO] Results:
[INFO]
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  4.688 s
[INFO] Finished at: 2024-01-21T19:29:23+01:00
[INFO] ------------------------------------------------------------------------
```

As you see 3 tests have been executed and no errors occurred.

# Step 5 - Component tests

Before we start, let me say that some people call this type an integration test. I prefer component test, as it only tests this endpoint, this one component. I use integration test, if your test code tests any sort of integration, what means more than one component.

Create a file `GameEndpointTest.java` and put this content into it:

```java
import com.oglimmer.kniffel.web.dto.*;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.http.ResponseEntity;
import java.util.Arrays;
import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class GameEndpointTest {

    @LocalServerPort
    private int port;

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private GameService gameService;

    @Test
    void game_create_ok() {
        CreateGameRequest createGameRequest = new CreateGameRequest();
        createGameRequest.setPlayerNames(new String[]{"oli", "ilo"});
        ResponseEntity<GameResponse> gameResponse = this.restTemplate.postForEntity(
                "http://localhost:" + port + "/api/v1/game/", createGameRequest, GameResponse.class);
        assertThat(gameResponse.getStatusCode().is2xxSuccessful()).isTrue();
        assertThat(gameResponse.getBody()).isNotNull();
        assertThat(gameResponse.getBody().getGameId()).isNotBlank();
        String gameId = gameResponse.getBody().getGameId();
        assertThat(gameId).isEqualTo(gameService.getGameInfo(gameId).getGameId());
    }

    @Test
    void game_create_failDuplicate() {
        CreateGameRequest createGameRequest = new CreateGameRequest();
        createGameRequest.setPlayerNames(new String[]{"oli", "oli"});

        ResponseEntity<GameResponse> gameResponse = this.restTemplate.postForEntity(
                "http://localhost:" + port + "/api/v1/game/", createGameRequest, GameResponse.class);

        assertThat(gameResponse.getStatusCode().is4xxClientError()).isTrue();
    }

    @Test
    void game_create_failOnly1Name() {
        CreateGameRequest createGameRequest = new CreateGameRequest();
        createGameRequest.setPlayerNames(new String[]{"oli"});

        ResponseEntity<GameResponse> gameResponse = this.restTemplate.postForEntity(
                "http://localhost:" + port + "/api/v1/game/", createGameRequest, GameResponse.class);

        assertThat(gameResponse.getStatusCode().is4xxClientError()).isTrue();
    }
}
```

Spring will start up the entire REST runtime when it sees the annotation `@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)`. Spring will also use a random port (and not 8080), so we can always run this time, even if we have already started the application.

As Spring starts all `@RestController`, `@Configuration`, `@Service`, `@Component` we can also use `@Autowired` to get an instance of `TestRestTemplate` and `GameService` injected. `TestRestTemplate` will help us to execute REST calls from a client perspective.

In each `@Test` method the first 2 lines create a `CreateGameRequest` object and sets the player names up for our test.

```java
CreateGameRequest createGameRequest = new CreateGameRequest();
createGameRequest.setPlayerNames(new String[]{"oli", "ilo"});
```

Now we can execute a real HTTP REST call against our API. The `restTemplate` class provided by Spring is very powerful but easy to use. We simply pass the URL to the endpoint, the DTO we just created and we let Spring know what class will be returned.

```java
ResponseEntity<GameResponse> gameResponse = this.restTemplate.postForEntity(
        "http://localhost:" + port + "/api/v1/game/", createGameRequest, GameResponse.class);
```

We can now use the Assertions API to check if the returned data is what we expect.

```java
// the status code must be 2xx (we could also check for a 201)
assertThat(gameResponse.getStatusCode().is2xxSuccessful()).isTrue();
// the body must have something
assertThat(gameResponse.getBody()).isNotNull();
// the game id is not empty
assertThat(gameResponse.getBody().getGameId()).isNotBlank();
// finally we do the ultimate check: check if the returned game id can be retrieved
// on the gameService.getGameInfo() method
String gameId = gameResponse.getBody().getGameId();
assertThat(gameId).isEqualTo(gameService.getGameInfo(gameId).getGameId());
```

You can run the test again

```bash
./mvnw test
```

This will always run all tests, so it runs our unit tests for the rules and it runs the new component tests for the create game endpoint. This is also integrated into the maven lifecycle. So whenever we create a production package via `./mvnw package` the tests are executed as well.

# Step 6 - Containerization of our backend and frontend

## Writing a Dockerfile for Spring

A Dockerfile defines how to build a image. Then an image can be used to start a container.

Inside the spring application directory, create a file `Dockerfile` (without an extension) and put this into it:

```Dockerfile
# we have to start with "FROM" this defines the base filesystem.
# as we want to build a java system, we use a popular java distribution as the base system
FROM eclipse-temurin:21-jdk as builder

WORKDIR /opt/build

# copy the "context root" into /opt/build
# for us "context root" is the base of this project
COPY . .

# execute mvn package
RUN ./mvnw package

# we start over again, as we only want to run java, we don't need a
# jdk anymore, use a tiny jre instead
FROM eclipse-temurin:21-jre-alpine

WORKDIR /opt/app

# copy the jar from the first build stage into this image
COPY --from=builder /opt/build/target/kniffel-0.0.1-SNAPSHOT.jar /opt/app/

# define how the container should run
CMD ["java", "-jar", "kniffel-0.0.1-SNAPSHOT.jar"]
```

Before we can start the container we have to build the image:

```bash
# the . (dot) at the end defines which "context root" directory 
docker build --tag kniffel-backend .
```

We now possess an image incorporating both a Java runtime environment and our Kniffel REST API jar file. It is time to run this image:

```bash
# if you want to expose TCP ports to the outside of the container
# you have to explicitly map the port from outside:inside
docker run --rm -p 8080:8080 kniffel-backend
```

You can access the server on port 8080, e.g. http://localhost:8080/swagger-ui/index.html or any client we have written so far.

## A Dockerfile for our Vue-client

Similar to what we have done for the backend, we create a file `Dockerfile` inside the vue directory:

```Dockerfile
# again we start with a base image matching our needs. in this case nodejs
FROM node:18 as builder

# copy "context root" into /opt/frontend
COPY . /opt/frontend

# install the npm dependencies and build static html/css/js files
RUN cd /opt/frontend && \
    npm ci && \
    npm run build

# start over again, this time use a nginx tiny image to serve our static files
FROM nginx:stable-alpine

# copy the output from the first stage
COPY --from=builder /opt/frontend/dist /usr/share/nginx/html

# this is informative only, it tells docker which port this image exposes
EXPOSE 80
```

Again we can test this Dockerfile by building the image:

```bash
# the . (dot) at the end defines which "context root" directory 
docker build --tag kniffel-frontend .
```

This built an image with nginx and our static files. Run it via:

```bash
docker run --rm -p 80:80 kniffel-frontend
```

If you still have the backend running, you can use the frontend via http://localhost

## Puting all togeher into a docker-compose.yml

To start (and stop) backend and frontend with one command, we can write a docker compose file.

First make sure you have stopped both containers from the previous steps.

Create a file `docker-compose.yml` in the parent directory of your backend and frontend directories.

```yml
version: '3'

services:
  backend:
    # change kniffel-server to match your directory name of the spring REST api
    build: ./kniffel-server
    ports:
      - 8080:8080
  frontend:
    # change kniffel-client to match your directory name of the vue client
    build: ./kniffel-client
    ports:
      - 80:80
```

Now you can start both containers with a simple command:

```bash
docker compose up --build
```

To run it as a daemon you can add `-d` like this:

```bash
docker compose up --build -d

# stop and remove containers with
docker compose down
```

# What we've learnt

* How to improve the null safety situation
* The difference between unit and component tests and how to write and execute them
* Understanding that unit tests should test all of our algorithms and calculations
* Knowing how to write Spring component tests to run tests against REST endpoints or general Spring services and componenets
* Spring annotiation
    * @ResponseStatus
    * @ExceptionHandler
    * @SpringBootTest
    * @LocalServerPort
* JUnit 5 annotiation
    * @Test
* How to write a Dockerfile for Spring and Vue applications and how to build and start images and containers

# Extras if you time

* read about all http status codes https://developer.mozilla.org/en-US/docs/Web/HTTP/Status
* read about Null safety for Kotlin https://kotlinlang.org/docs/null-safety.html
