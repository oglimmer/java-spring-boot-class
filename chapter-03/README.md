# Chapter 03

Swagger, Application layers and game logic

# Goal

Adding OpenAPI documentation, understanding dependency injection and implementing a GameService.

# Context and Knowledge

* For a REST API application you usually want to separate you code into at least 3 layers:
    * **presentation** layer, also called http, web, REST layer
    * **business** layer, also called logic, services layer
    * **persistence** layer, also called database,  data access layer
* The `@RestController` itself and the DTOs used in the `@RestController` and other classes used to map or manage the `@RestController` input/output live in the presentation layer of the application.
* The business layer contains the business logic or in our case the game logic and also the services to handle them.
* The persistence layer has the code to access the database, there should be no business logic there.

## DB transaction control

If you don't know what DB transactions are, just skip this.

One reason to use different application layers is, to have control over the DB transactions.

* presentation layer = all methods here never participate in transactions
* business layer = each method controls its transactions
* persistence layer = each method either participates in an existing transaction or starts a new one if non existed

Spring offers an easy to use annotation @Transactional to control DB transaction behavior. But we don't need that for our REST API.

## Follow up from Chapter 02 - Review the 4 endpoints

Before we continue we want to review the "homework" for the last step in Chapter 02.

This is a possible GameController.java (without package, imports):


```java
@RestController
@RequestMapping("/api/v1/game")
@CrossOrigin
public class GameController {

    @PostMapping("/")
    public GameResponse createGame(@RequestBody CreateGameRequest createGameRequest) {
        return new GameResponse();
    }

    @GetMapping("/{gameId}")
    public GameResponse getGameInfo(@PathVariable String gameId) {
        return new GameResponse();
    }

    @PostMapping("/{gameId}/roll")
    public GameResponse roll(@PathVariable String gameId, @RequestBody DiceRollRequest diceRollRequest) {
        return new GameResponse();
    }

    @PostMapping("/{gameId}/book")
    public GameResponse book(@PathVariable String gameId, @RequestBody BookRollRequest bookRollRequest) {
        return new GameResponse();
    }
}
```

Each of those classes below must go into their own .java file. Let's look at the DTOs:

```java
@Getter @Setter @ToString
public class GameResponse {
    private String gameId; // secret ID to access the game
    private PlayerData[] playerData; // returns the score per player
    private String currentPlayerName;
    
    // either BOOK or ROLL
    // for BOOK the /book endpoint needs to be called
    // for ROLL the /roll endpoint needs to be called
    private String state; 
    private String[] usedBookingTypes; // returns the booking types used by the current player

    // returns the available booking types for the current player
    // this string should be used for the /book endpoint
    private String[] availableBookingTypes;
    private int[] diceRolls; // dice values the player rolled (1...6). array size = 5
    private int rollRound; // for state==ROLL, returns the round 1,2,3
}

@Getter @Setter @ToString
public class PlayerData {
    private String name;
    private int score; // total score so far
}

@Getter @Setter @ToString
public class CreateGameRequest {
    private String[] playerNames;
}

@Getter @Setter @ToString
public class DiceRollRequest {
    // dice values ranging from 1 to 6
    // must not exceed 5 elements
    // only dice values returned by diceRolls are allowed
    private int[] diceToKeep;
}

@Getter @Setter @ToString
public class BookRollRequest {
    // one string from availableBookingTypes
    private String bookingType;
}
```

If your classes look a bit different, you can either keep them as they are or change them to my suggestion. Using my API defintion gives you the chance to use my Web UI later on.

## Step 1 - Swagger / OpenAPI

REST APIs are often provided by team A and used by team B - even outside the company, so good documentation is very important.

There is a standard for documenting REST APIs : [OpenAPI](https://spec.openapis.org/oas/latest.html). A company called SmartBear provides a software called Swagger, a web ui for OpenAPI documentation.

To enable OpenAPI and the Swagger UI we simple have to add these dependencies to `pom.xml`

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

Depending on your IDE you might have to reload the `pom.xml` after editing it. Start the server and access http://localhost:8080/swagger-ui/index.html and you should see all your endpoints and you can even try them out in the browser (click the "try it out" button).

### Understading the difference between OpenAPI and Swagger-UI

OpenAPI is JSON file defining your REST API. It is autogenerated by Spring and you can look at it via http://localhost:8080/v3/api-docs (or download it with `curl http://localhost:8080/v3/api-docs`).

Swagger-UI is generic(!) web UI visualizing your OpenAPI JSON from http://localhost:8080/v3/api-docs.

### More configuration for OpenAPI / Swagger

We can configure our OpenAPI JSON to show more details and information of our API.

Create a new file Beans.java:

```java
// add your package

import io.swagger.v3.oas.models.*;
import io.swagger.v3.oas.models.info.*;
import org.springframework.context.annotation.*;

// this annotation tells Spring to look for @Bean inside this class
@Configuration
public class Beans {
    // that's the key
    @Bean
    public OpenAPI springShopOpenAPI() {
        return new OpenAPI()
                .info(new Info().title("Kniffel Game API")
                        .description("Kniffel as a service - KaaS")
                        .version("v0.0.1")
                        .license(new License().name("Apache 2.0").url("https://www.apache.org/licenses/LICENSE-2.0")))
                .externalDocs(new ExternalDocumentation()
                        .description("Kniffel Regeln")
                        .url("https://www.schmidtspiele.de/files/Produkte/4/49030%20-%20Kniffel/49203_49030_Kniffel_DE.pdf"));
    }
}
```
_(working code)_

Here we see the usage of an important concept "Dependency Injection".

## Step 2 - Dependency Injection - Part I: hide the "new" operator

The idea of "Dependency Injection" (DI) is to de-couple places where code needs an object and the code which provides this object.

We don't use the code from this step, all code in this section is for demonstration purposes only.

In our case from above the Swagger code needs an object for [`OpenAPI`](https://github.com/swagger-api/swagger-core/blob/master/modules/swagger-models/src/main/java/io/swagger/v3/oas/models/OpenAPI.java).

The internal Swagger code use this code to create an `OpenAPI` object:

```java
OpenAPI api = new OpenAPI();
// using api to get all the info stored there
```
_(snipped)_

This works, but now it's impossible that you as a user of the Swagger package provide your own object (instance) of class OpenAPI.

So we want to give the user of the Swager package the option to do `new OpenAPI()`.

To achieve this Spring offers Dependency Injection:

```java
//  you could find this code somewhere inside of Swagger
public class SomeClassInSwaggerPackages
    
    @Autowired
    private OpenAPI api; // a consuming reference
    
    // more methods and attributes ...
}
```
_(snipped)_

The attribute `api` is of type `OpenAPI`, but it is not defined how it is created. What happens is that Spring will provide an instance via Dependency Injection. Spring knows how to do that, because of `@Autowired` which means: here put an instance from "Dependency Injection".

So back to your code in `Beans.java` :

```java
@Bean
public OpenAPI springShopOpenAPI() {
    return new OpenAPI();
}
```
_(snipped)_

This is the counterpart of the Dependency Injection. The `@Bean` annotation tells Spring that if it ever needs an instance for `OpenAPI` (because there is `@Autowired` at a field of such type), then use this method to create this instance.

##### Conslusion:

We decoupled `return new OpenAPI();` from the declaration `private OpenAPI api;` of the attribute. Now everybody who uses the Swagger package can provide their own instance of `OpenAPI` without changing the Swagger code.

### Concept of Singleton

There is one important concept to clarify:

How many instances of `@Bean` will be created? Or in other words: when we use `@Autowired` 10-times, will Spring create 10 instances of this class or only one instance and return the same to all `@Autowired` places?

It is only 1 instance. It's the same object for all consumers of the `@Bean`.

This is called the [Singleton Design Pattern](https://en.wikipedia.org/wiki/Singleton_pattern).

Let's keep the consequences in mind:

* those classes are considered _not thread-safe_
* means our `@Bean` (or later `@Service`, `@Component`) classes shares all the attributes with all consumers
* therefore you must not put any object attributes in such a class

## Step 3 - More OpenAPI documentation

We can also provide information about our endpoints and their data structures to OpenAPI. We provide this information with annotations directly attached to the methods and classes.

This is the createGame method and its DTOs:

```java
@PostMapping("/")
public GameResponse createGame(@RequestBody CreateGameRequest createGameRequest) {
    return new GameResponse();
}
```

we can add the OpenAPI information with the following annotations:

```java
@Operation(
        summary = "Create a new game with a specific number of players",
        description = "The number of players must be at least 2. All player names must be different.", responses = {
        @ApiResponse(
                responseCode = "200", description = "Game created", content = @Content(
                mediaType = MediaType.APPLICATION_JSON_VALUE,
                schema = @Schema(implementation = GameResponse.class)))})
@PostMapping("/")
public @NotNull GameResponse createGame(@RequestBody CreateGameRequest createGameRequest) {
    return new GameResponse();
}
```

The @NotNull annotation comes from `import jakarta.validation.constraints.NotNull;`. It tells Swagger/OpenAPI that this response is never null.

We can also add more information on the DTO classes:

```java
@Schema(description = "Game creation request")
@Getter @Setter @ToString
public class CreateGameRequest {
    @Schema(description = "Defines the participating players. Each player name must be different and at least 2 players have to be provided",
            example = "[\"john doe\", \"jane doe\"]")
    @NotNull 
    private String[] playerNames;
}
```

We also have to add @NotNull to all Response classes. Here is a example for the `GameResponse` class:

```java
@Getter @Setter @ToString
public class GameResponse {
    @NotNull
    private String gameId; // secret ID to access the game
    @NotNull
    private PlayerData[] playerData; // returns the score per player
    @NotNull
    private String currentPlayerName;

    // either BOOK or ROLL
    // for BOOK the /book endpoint needs to be called
    // for ROLL the /roll endpoint needs to be called
    @NotNull
    private String state;
    @NotNull
    private String[] usedBookingTypes; // returns the booking types used by the current player

    // returns the available booking types for the current player
    // this string should be used for the /book endpoint
    @NotNull
    private String[] availableBookingTypes;
    @NotNull
    private int[] diceRolls; // dice values the player rolled (1...6). array size = 5
    @NotNull
    private int rollRound; // for state==ROLL, returns the round 1,2,3
}
```

If we miss this, our OpenAPI specification will say that all of those attributes can be null, which is not the case. While our clients could still work with this, it makes their job much harder, as a client needs to cover all of those "may be undefined/null" scenarios.


It's always good practice to add those annotiations to all of your REST controller methods and all DTOs. It helps your consumers to understand your API, but it also helps you to think about your API, its use-cases, validations and limitations. You should do it right when you write the REST controller and the DTOs, not after completing the development as an afterthought.

## Step 4 - Application Layers

Our REST API does everything in memory, so we don't have anything in the persistence layer.

It is usually good practise to have a business layer service class for our domain objects, for us that means we want to have a `GameService` class.

```java
// this enables this class for Dependency Injection
@Service
public class GameService {

    public KniffelGame createGame(String[] playerNames) {
        return null; // just a dummy for now
    }

}
```

This class provides functionality for all uses cases like "create a game", "load a game", "re-roll the dice for a game" or "book a dice roll for a game". Our method "createGame" takes an array of player name and will return an object of type `KniffelGame` which is our actual game class holding all the attributes and methods to play Kniffel. For now we return null, because we first want to connect `GameService` and `GameController` together.

So let us integrate this method to our `GameController`. We want to use the Dependency Injection between `GameController` and GameService.

We have used `@Bean` in the code before to provide an object for Dependency Injection, but for our own classes there is an even easier way: the `@Service` annotation. All of our own classes annotated with `@Service` (or `@Component`) will be available for `@Autowired`.

So we can change our `GameController` to use this:

```java
// some annotiations...
public class GameController {

    @Autowired
    private GameService gameService;

    @PostMapping("/")
    public @NotNull GameResponse createGame(@RequestBody CreateGameRequest createGameRequest) {
        KniffelGame game = gameService.createGame(createGameRequest.getPlayerNames());
        return new GameResponse(); // TODO map data of game into GameResponse
    }
    
    //... more
}
```

Now the only problem we have left, is a compiler error for `KniffelGame`.

## Step 5 - Game Logic - `class KniffelGame`

As this is a tutorial for Spring REST API and not for Java coding, so I leave you with three options for the actual game logic:

* [easy] Use my kniffel-rules library
* [moderate] Use my kniffel-rules library, but implement the logic to calculate the score by yourself
* [hard] Implement the game logic for Kniffel by yourself

### How to use my kniffel-rules library for option 1 or 2

If you want to use my library you have to change `pom.xml`.

First add at the end of `pom.xml` (just before \</project>) this new repository:

```xml
<repositories>
    <repository>
        <id>oglimmer-repository-releases</id>
        <name>oglimmer repository</name>
        <url>https://mvn.oglimmer.com/releases</url>
    </repository>
</repositories>
```

Then add this dependency:

```xml
<dependency>
    <groupId>com.oglimmer</groupId>
    <artifactId>kniffel-rules-lib</artifactId>
    <version>0.0.2</version>
</dependency>
```

You can always add dependencies from the default repository, but my kniffel rule lib is not available there and you need to add my custom repository.

Now we can use the class `KniffelGame`, which implements Kniffel, into our `GameService`. If you want to have a look at the source code, you can find it here https://github.com/oglimmer/kniffel-rules-lib/.

### Option 2 - implement your own score calculation (optional)

Create a file `KniffelRulesImplementation.java` with the following content:

```java
// your package

import java.util.List;
import com.oglimmer.kniffel.service.KniffelRules;

public class KniffelRulesImplementation extends KniffelRules {
    public int getScore1To6(List<Integer> diceRolls, int valueToScore) {
        // count all dice values with value == valueToScore
        return 1;
    }

    public int getScoreForXofAKind(List<Integer> diceRolls, int valueToScore) {
        // check if there are $valueToScore (3 or 4) dice with the same value
        // if yes: count the dice values, else: 0
        return 1;
    }

    public int getScoreFullHouse(List<Integer> diceRolls) {
        // check if there are 2 dice with the same value
        // and 3 dice with the same value return 25, else 0;
        return 1;
    }

    public int getScoreSmallStraight(List<Integer> diceRolls) {
        // check if 4 out 5 values are [1,2,3,4] or [2,3,4,5] or [3,4,5,6]
        // keep in mind that the order might be different
        // return 30, else 0;
        return 1;
    }

    public int getScoreLargeStraight(List<Integer> diceRolls) {
        // check if [1,2,3,4,5] or [2,3,4,5,6]
        // keep in mind that the order might be different
        // return 40, else 0;
        return 1;
    }

    public int getScoreKniffel(List<Integer> diceRolls) {
        // check if all dice have the same value, return 50, else 0
        return 1;
    }

    public int getScoreChance(List<Integer> diceRolls) {
        // sum up all dice values, no conditions
        return 1;
    }
}
```

To let the system use your own class add this to your `Beans.java`:

```java
@Bean
public KniffelRules kniffelRules() {
    return new KniffelRulesImplementation();
}
```

Now you need to implement the score calculation.

### Write your own Kniffel game logic (for the hard option)

Of course you can just come up with your own game logic class design, but you might want to create those 4 files:

* `enum BookingType` - an enum defining all booking types of Kniffel
* `enum GameState` - an enum defining the two states: "in re-rolling dice" (can be done twice by the player), "in selecting the booking type"
* `class KniffelPlayer` - a simple DTO / POJO to store the name, the score and the already used booking types.
* `class KniffelGame` - the actual game logic class, with at least
    * a constructor taking a `List<KniffelPlayer>` to create the game and initiatlize itå
    * `void reRollDice(int[] diceToKeep)` - to re-roll the dice
    * `void bookDiceRoll(BookingType bookingType)` - to book the current dice on a category and calculate the new score for this player

### Implementation of `createGame()` in class `GameService` (all options)

We have to return a newly created KniffelGame object. To do so, we also have to create the KniffelPlayer objects for each player. At the end we have to store the created object, so we can reuse it in later calls.

```java
@Service
public class GameService {

    // This attribute acts as our "database backend"!
    // we keep track of all created games and
    // make those accessible by their game ID.
    // - key = game ID
    // - value = the actual game object 
    private final Map<String, KniffelGame> games = new HashMap<>();

    public KniffelGame createGame(String[] playerNames) {
        // The constructor of KniffelGame needs a java.util.List of 
        // KniffelPlayer. KniffelPlayer is the game logic object for a Player.
        List<KniffelPlayer> players = 
            Arrays.stream(playerNames) // String[] to stream
                .map(name -> new KniffelPlayer(name)) // create 1 object for each name
                .collect(Collectors.toList()); // convert the stream to List

        // now we can create the game logic object
        KniffelGame kniffelGame = new KniffelGame(players);

        // we store in memory, so we can use it in later REST calls
        games.put(kniffelGame.getGameId(), kniffelGame);
        return kniffelGame;
    }
```

Before we have a working solution for "create game" we have to solve the last TODO. In our `GameController` we have this:

```java
@PostMapping("/")
public GameResponse createGame(@RequestBody CreateGameRequest createGameRequest) {
    KniffelGame game = gameService.createGame(createGameRequest.getPlayerNames());
    return new GameResponse(); // TODO map data of game into GameResponse
}
```

So we need to copy all attributes from KniffelGame into GameResponse. You can do this by yourself and just use all the getters on KniffelGame to call all the setters on GameResponse. But usually people use a "mapping library" and one popular library is "ModelMapper".

## Step 6 - Mapping data with lib `ModelMapper`

In theory you could use the internal logic classes and expose them via REST API endpoints, but that's a very bad idea. You would expose all of your data to clients, also any change inside your logic could change the data structure for clients. Therefore it is good practise to have dedicated DTOs for input/output of REST APIs and of course a mapping between them.

First we need to add the dependency for ModelMapper lib. In pom.xml add:

```xml
<dependency>
    <groupId>org.modelmapper</groupId>
    <artifactId>modelmapper</artifactId>
    <version>3.2.0</version>
</dependency>
```

Now we can add a new attribute into our `GameController`

```java
public class GameController {
    // ...
    @Autowired
    private ModelMapper modelMapper;
    // ...
}
```

With `@Autowired` we use Dependency Injection to get an instance of it. But we have to provide this instance by ourself, as it doesn't come with Spring - if cannot we just added the dependency.

Open our `Beans.java` file and add this `@Bean` defintion:

```java
    @Bean
    public ModelMapper modelMapper() {
        return new ModelMapper();
    }
```

Now we write a method to map a `KniffelGame` object into a `GameResponse` (and add this to `GameController`)

```java
    private GameResponse mapGameResponse(KniffelGame game) {
        // to copy all attributes with the same name we use modelMapper method
        GameResponse gameResponse = modelMapper.map(game, GameResponse.class);
        // we have to copy the PlayerData for each object
        gameResponse.setPlayerData(game.getPlayers().values().stream().map(p -> modelMapper.map(p, PlayerData.class)).toArray(PlayerData[]::new));
        // same for the already used booking types
        gameResponse.setUsedBookingTypes(game.getCurrentPlayer().getUsedBookingTypes().stream().map(Enum::toString).toArray(String[]::new));
        // and the not used bookings types
        gameResponse.setAvailableBookingTypes(Arrays.stream(BookingType.values()).filter(bt -> !game.getCurrentPlayer().getUsedBookingTypes().contains(bt)).map(Enum::toString).toArray(String[]::new));
        return gameResponse;
    }
```

Use this method to convert a KniffelGame object into a GameResponse object in `createGame()` and remove the TODO.

At this point we have a working REST API to create a game of Kniffel.

## Step 7 - Complete the other endpoints

The `GameService` class should look like this:

```java
@Service
public class GameService {

    private final Map<String, KniffelGame> games = new HashMap<>();

    public KniffelGame createGame(String[] playerNames) {
        // as discussed above
    }

    public KniffelGame getGameInfo(String gameId) {
        // ??
    }

    public void roll(KniffelGame kniffelGame, int[] diceToKeep) {
        // ??
    }

    public void bookRoll(KniffelGame kniffelGame, BookingType enumBookingType) {
        // ??
    }
}
```

Complete the code and integrate the usage inside of `GameController`. All of those methods can be implemented in less than 5 lines of code.

# What we've learnt

* What OpenAPI and Swagger is and how to add it to a Spring project
* The concept of Application layers
* What Dependency Injection is, how it works, why it is useful and how to use it in Spring
* What the Singleton Design Pattern is and what one must consider when using it
* How to write services to manage our business (in this case game) logic
* Writing, extending or using the Kniffel game logic
* How to add a new repository in maven
* Using the lib ModelMapper to copy data from one Java class into another
* Spring annotations
    * @Autowired
    * @Service / @Component
    * @Bean
    * @Configuration


# Extras if you time

* Read more for ModelMapper on https://modelmapper.org/
