# Chapter 04

Database backend

# Goal

We want to use a relational database - Postgres - as the storage backend.

# Context and Knowledge

* There are many different ways to integrate a database (relational or non-relational) into a Spring application, we will use "Spring Data JPA" which is an Object-Relational-Mapper (ORM) framework based on the Java Persistence (JPA) and Hibernate
* JPA - Java Persistence - is an API specification to unify access for Java enterprise applications to relational databases. If your application uses JPA you can easily change the underlaying database form MariaDB to Postgres to Oracle to MS SQL Server and so on. As JPA is only an API specificiation you can also change the implementation without the need to change your code. The most popular implementation is Hibernate, other implementations are EclipseLink or OpenJPA.
* Object-Relational-Mapper (ORM) are frameworks to map database table rows into Java objects and vice versa, it makes working with databases from code much easier (but sometimes also more dangerous)
* Logging information to the console or a file is an important part of every enterprise server application. It is recommended to use a logging framework
* An easy way to do logging properly is use Lombok's @Slf4j annotation
* You can control what is being logged to the console in the application.properities file

# Step 1 - Dependencies 

Adding database support to our application needs several steps and refactorings. Let's do them one by one.

We start by adding the needed dependencies to `pom.xml`:

```xml
<!-- dependency to add Spring Data JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<!-- database driver for Postgres -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
<!-- Postgres in Component tests -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

Of course you can go back to https://start.spring.io/ and find those dependencies by yourself there:

* Spring Data JPA
* PostgreSQL Driver
* Testcontainers

The reason why we need "Spring Data JPA" is clear, that's the Spring module adding Spring Data JPA support.

Of course you need to pick a Database server: the most two popular options for our kind of application is Postgres or MariaDB - both are open source and free of charge. Postgres is more powerful, while MariaDB is often simpler or less complex to use.

Testcontainers provide (docker) containers for automated tests (unit, component, integration, etc). We need this because our tests will need a Postgres database, so we provide a temporary container based Postgres via Testcontainers to those tests.

# Step 2 - Postgres Server

We need to start a Postgres server (or have access to one running on another server).

The easiest way to start such a Postgres server for development on your local machine, is using docker:

```bash
docker run -d --name postgres -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres
```

For this tutorial you don't need to access the database server directly, but if you are interested you can use a client listed onhttps://wiki.postgresql.org/wiki/PostgreSQL_Clients. Downlaod it and use the following parameters to connect to your database:

```
Host: localhost
Port: 5432 (standard)
User: postgres
Passsword: postgres
```

# Step 3 - Spring application.properties

We have to tell Spring or JPA/Hibernate how to access the database server.

Open the file `application.properties` in /src/main/resources and put this content:

```ini
# Java always uses those "JDBC URL"
# Spring always the usage of variables with default values, so we can
# set the host from a environment variable
spring.datasource.url=jdbc:postgresql://${DB_HOST:localhost}/postgres
spring.datasource.username=postgres
spring.datasource.password=postgres
# Java JDBC always needs a driver class
spring.datasource.driver-class-name=org.postgresql.Driver
# hibernate will always create and update the tables defined by @Entity
spring.jpa.hibernate.ddl-auto=update

# log only INFO, WARN and ERROR for "all packages"
logging.level.root=INFO
# replace com.oglimmer with your base package name
# log DEBUG, INFO, WARN and ERROR for "com.oglimmer" packages
logging.level.com.oglimmer=DEBUG

# uncomment if you want to see the SQL statements
#logging.level.org.hibernate.SQL=DEBUG
#logging.level.org.hibernate.orm.jdbc.bind=TRACE
```

# Step 4 - JPA basics

JPA asks you to provide 2 types of classes

* `@Entity` - those classes define the structure of a table, each attribute in your class will be a column in the database table
* `@Repository` - those interfaces define the operations (insert, update, delete, find) for an `@Entity` (table)

To integrate our Kniffel service with a database, we have to replace the map below (this map acted as our "data storage"):

```java
@Service
public class GameService {

    private final Map<String, KniffelGame> games = new HashMap<>();

    // ... other methods
```

instead of the Map, we want to use a `@Repository` interface to store and load data from Postgres DB.

Let's look at the `GameService` method to get a `KniffelGame` by its id:

```java
public KniffelGame getGameInfo(@NonNull @lombok.NonNull String gameId) {
    // we simply used the get method on the map to "load" a game from the "data storage"
    KniffelGame game = games.get(gameId);
    if (game == null) {
        throw new GameNotFoundException();
    }
    return game;
}
```

we want to change this into using a find method on a `@Repository` interface. This should look like this:

```java
public KniffelGame getGameInfo(@NonNull @lombok.NonNull String gameId) {
    return kniffelGameRepository.findByGameId(gameId).orElseThrow(GameNotFoundException::new);
}
```

So we have to create this interface `KniffelGameRepository`. Let's create a file `KniffelGameRepository.java`:

```java
import com.oglimmer.kniffel.db.entity.KniffelGame;
import org.springframework.data.repository.CrudRepository;
import java.util.Optional;

public interface KniffelGameRepository extends CrudRepository<KniffelGame, Long> {

    Optional<KniffelGame> findByGameId(String gameId);

}
```

This needs some explanations!

## JPA repository interfaces

What do we see here:

* an interface instead of a class - an interface does not have method implementations or attribures
* by extending our interface from `CrudRepository` we get the following functionality
    * findAll
    * save
    * count
    * findById (primary key)
    * existsById
    * deleteById
* the generic parameter on CrudRepository<KniffelGame, Long> tells Spring which @Entity this class supports and the type of primary key
* we define a custom "findBy..." method : `Optional<KniffelGame> findByGameId(String gameId)` - it will find a record by gamdId

All those methods - coming from CrudRepository or our custom method - execute SQL against the database. Let's look at some examples:

| method | sql |
|----|----|
| findAll | `select * from kniffel_game` | 
| save | `insert into kniffel_game values (?,?,?,?)` <br> or <br> `update kniffel_game set rollRound=?, currentPlayer=?, state=?, gameId=? where id = ?` |
| count | `select count(*) from kniffel_game` | 
| findById (primary key) | `select * from kniffel_game where id = ?` | 
| deleteById | `delete from from kniffel_game where id = ?` | 
| findByGameId | `select * from kniffel_game where gameId = ?` | 

The implementation is provided by the Spring Data JPA framework, so the only the we have to do, is to define the interface.

We also have to provide the @Entity class, to define the table and column structure.

## JPA @Entity

Let's recap how the attributes in `KniffelGame` and `KniffelPlayer` looked:

```java
@Getter
@Setter
public class KniffelGame {
    private List<KniffelPlayer> players;
    private String gameId;
    private int rollRound;
    private KniffelPlayer currentPlayer;
    private GameState state;
    private List<Integer> diceRolls;
}

@Getter
@Setter
public class KniffelPlayer {
    private String name;
    private int score;
    private List<BookingType> usedBookingTypes;
}

// BookingType and GameState are enums defing those constants
```

This is the data those classes need to fullfil their purpose. This data also needs to be stored in the database, so we turn those two classes into @Entity objects.

We need JPA annotations to tell Spring/JPA/Hibernate how to map the Java attributes into database columns.

```java
@Entity
@Getter
@Setter
public class KniffelGame {

    @Id
    @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "game")
    private List<KniffelPlayer> players;

    private String gameId;

    private int rollRound;

    @OneToOne
    private KniffelPlayer currentPlayer;

    @Enumerated(EnumType.STRING)
    private GameState state;

    @ElementCollection
    private List<Integer> diceRolls;
}
```

As you see we need to add a new attribute `Long id` this is the primary key for the table. In relational databases, each table should have a primary key to identify the row in the table. In theory you could use the `gameId` for this, but it is always good practice to have an extra, dedicated attribute for this.

Let's discuss the annotations:

* `@Entity` - all classes with this annotation will get their own database table. The name of the class will be the database table. In this case the database table is called "kniffel_game".

* `@Id` - this makes this attribute the primary key

* `@GeneratedValue` - tells JPA to create the database column in a way that the database auto-generates the values for this primary key

* `@OneToMany` - we can make child objects available. JPA will create the database schema, so we know which players are in a game and which game belongs to a player

* `@OneToOne` - this column self-references the table, in the database table this column is an ID of its self

* `@Enumerated` - tells JPA to store the enums name, not a number in the database. This makes the database easier to read

* `@ElementCollection` - tells JPA to create another table in the database to store the content of list.

This is how the `KniffelGame` will look like. 

```java
@Entity
@Getter
@Setter
public class KniffelPlayer {

    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private int score;

    @ElementCollection(targetClass = BookingType.class)
    @Enumerated(EnumType.STRING)
    private List<BookingType> usedBookingTypes;

    @ManyToOne
    private KniffelGame game;
}
```

This defines another table called "kniffel_player", we see the counter part for a reference: `@ManyToOne` this annotation make child objects available from the perspective of the child objects to the parent object.

## Create database records

We looked at the method `getGameInfo` to fetch (or load) data from the database, we also need to change the method where we create this data.

We insert data into the database by using the `save` method on a `Repository` interface:

```java
public KniffelGame createGame(@NonNull @lombok.NonNull String[] playerNames) {
    List<KniffelPlayer> players = Arrays
            .stream(playerNames)
            .map(KniffelPlayer::new)
            .collect(Collectors.toList());
    kniffelPlayerRepository.saveAll(players);
    KniffelGame kniffelGame = new KniffelGame(players);
    kniffelGameRepository.save(kniffelGame);
    return kniffelGame;
}
```

As you see, you need to save the references data objects - in our case the KniffelPlayers - first.

* we create for each participating player an object of `KniffelPlayer`
* we use `kniffelPlayerRepository` to save all of those objects to the database
* we create the object for `KniffelGame`
* we store `kniffelGame` also to the database

## References to Repositories from a Service

We also have to add `KniffelGameRepository` and  `KniffelPlayerRepository` to our `GameService` class. As all `Repository` interfaces are Spring managed objects as well, we can simply use `@Autowired` to use dependency injection.

```java
@Service
@Transactional
public class GameService {

    @Autowired
    private KniffelGameRepository kniffelGameRepository;

    @Autowired
    private KniffelPlayerRepository kniffelPlayerRepository;

    // rest of the file....
```

As you see we also added `@Transactional` to the class itself. 

## Database transaction

Back to `createGame` - what happens if the creation of the KniffelGame fails (maybe because the network fails)? We already created the KniffelPlayers to the database!

We leave the database data in an inconsistent state. Half the data is created, the other half is not.

While for our game it could be ok, to just not care about this, in a financial banking system you need to make sure that:

> either the entire set of database changes are done OR none of them

this is called "database transaction" - ALL or NONE - never just "some changes".

Adding `@Transactional` to the class tells Spring/JPA to use database transaction on all methods. So if an exception occurs somewhere in a method, all changes to the database inside this method will be reverted.

# Step 5 - Using a database in Component test

Let's look at our class `GameEndpointTest` where we tested the REST endpoints.

We will use "Testcontainers" to run a temporary database while running the tests.

This temporary database is a real Postgres Server running as a docker container.

We already added the maven dependencies in a previous step, so we now need to add some annotations to our test class:

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
public class GameEndpointTest {

    @LocalServerPort
    private int port;

    @DynamicPropertySource
    private static void overrideProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgreSQLContainer::getJdbcUrl);
        registry.add("spring.datasource.username", postgreSQLContainer::getUsername);
        registry.add("spring.datasource.password", postgreSQLContainer::getPassword);
        registry.add("spring.datasource.driver-class-name", postgreSQLContainer::getDriverClassName);
    }

    @Container
    private static PostgreSQLContainer postgreSQLContainer = new PostgreSQLContainer("postgres")
            .withDatabaseName("fake-db-name")
            .withUsername("fake-user")
            .withPassword("fake-password");

    // we can leave the rest as it was    
```

We don't go into the details, but those annotations and the setup methods, will start a docker container with a Postgres database and configure Spring to use it.

# Putting it all togehter

We needed to make many changes to add the Postgres database backend to our REST API.

To focus on the important conceptional changes, I haven't shown here the changes to move the logic from https://github.com/oglimmer/kniffel-rules-lib into this project. By taking a look at the kniffel-rules-lib you should still be able to reproduce this chapter and complete the gaps by yourself.

But if you struggle to assemple everything together and make it work, you find the final result here https://github.com/oglimmer/java-spring-boot-class-sources/tree/main/chapter06/kniffel-server

# Step 6 - Logging

Any server application needs to log information. The reason can be errors, inform about warnings, for information purposes or just to support debugging.

While you could just write a string to stdout with

```java
System.out.println("Error!");
```

it is recommended to use a logging framework.

This give you configurable control on what you want to log and where to log it.

As we are using Lombok we can use the annotation `@Slf4j` on our classes to provide a logging variable

```java
@Slf4j
public class KniffelGame {

    // attributes and other methods

    public void roll(int[] diceToKeep) {
        // each {} tells the logging framework to expect a variable
        log.debug("Game {} roll dice. current dice {}, to keep {}", gameId, diceRolls, diceToKeep);
        // other code in this method
    }

    // rest of the class
}
````

With the annotation we have a variable `log` available. This variable has 5 methods to log strings with its parameters to different "log level":

* void trace(string)
* void debug(string)
* void info(string)
* void warn(string)
* void error(string)

Let's look at the configuration. This is done in `application.properties`:

```ini
# log only INFO, WARN and ERROR for "all packages"
logging.level.root=INFO
# log DEBUG, INFO, WARN and ERROR for "com.oglimmer" packages
logging.level.com.oglimmer=DEBUG
```

Another advantage of logging frameworks is, that they offer this configurable behavior for libraries as well. If we want to enable full logging for Hibernate's SQL statements we can add this to `application.properties` to see all SQL and their parameters being executed:

```init
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE
```

# What we've learnt

* What is an ORM and how to use it
* How to add Postgres to a Spring application
* How to use logging and what logging levels are in Spring + Lombok
* Spring / JPA annotiation
    * @Entity
    * @Id
    * @GeneratedValue
    * @Enumerated
    * @ElementCollection
    * @OneToOne
    * @ManyToOne
    * @OneToMany
    * @Transactional
    * @Slf4j
* Testing annotiation
    * @Testcontainers
    * @Container
    * @DynamicPropertySource

# Extras if you time

* Read the Spring docu about Data JPA https://spring.io/projects/spring-data/
* Read more about JPA and Hibernate: https://en.wikipedia.org/wiki/Jakarta_Persistence and https://hibernate.org/
* Read more about logging with Slf4j https://www.slf4j.org/manual.html
