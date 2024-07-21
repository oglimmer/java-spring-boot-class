# Chapter 04

Web and CLI Clients

# Goal

Developing and testing a REST API via curl, postman and a (very simple) Vue web application.

# Context and Knowledge

* Having an efficient development workflow is key for successful software devlopment, thus being able to test and iterate on your REST APIs needs to be easy
* Always try to automate your workflows, curl and postman enables that
* Professional development uses Test Driven Development (TDD) and we will look at automated tests in chapter 05

# Follow up from Chapter 03

After completing the `@GameService` and integrating it into `@GameController` we have bascially completed the Kniffel game REST API.

This is how a `GameService` could look like:

```java
@Service
public class GameService {

    // fake database backend
    private final Map<String, KniffelGame> games = new HashMap<>();

    public KniffelGame createGame(String[] playerNames) {
        // using Java's streaming API to convert string[] into KniffelPlayer and create a List
        List<KniffelPlayer> players = Arrays.stream(playerNames).map(KniffelPlayer::new).collect(Collectors.toList());
        KniffelGame kniffelGame = new KniffelGame(players);
        games.put(kniffelGame.getGameId(), kniffelGame);
        return kniffelGame;
    }

    public KniffelGame getGameInfo(String gameId) {
        return games.get(gameId);
    }

    public void roll(KniffelGame kniffelGame, int[] diceToKeep) {
        kniffelGame.reRollDice(diceToKeep);
    }

    public void bookRoll(KniffelGame kniffelGame, BookingType enumBookingType) {
        kniffelGame.bookDiceRoll(enumBookingType);
    }
}
```

This is how a `GameController` could look like:

```java
@RestController
@RequestMapping("/api/v1/game")
@CrossOrigin
public class GameController {

    @Autowired
    private GameService gameService;

    @Autowired
    private ModelMapper modelMapper;

    @PostMapping("/")
    public @NonNull GameResponse createGame(@RequestBody CreateGameRequest createGameRequest) {
        KniffelGame game = gameService.createGame(createGameRequest.getPlayerNames());
        return mapGameResponse(game);
    }

    @GetMapping("/{gameId}")
    public @NonNull GameResponse getGameInfo(@PathVariable String gameId) {
        KniffelGame game = gameService.getGameInfo(gameId);
        return mapGameResponse(game);
    }

    @PostMapping("/{gameId}/roll")
    public @NonNull GameResponse roll(@PathVariable String gameId, @RequestBody DiceRollRequest diceRollRequest) {
        KniffelGame kniffelGame = gameService.getGameInfo(gameId);
        gameService.roll(kniffelGame, diceRollRequest.getDiceToKeep());
        return mapGameResponse(kniffelGame);
    }

    @PostMapping("/{gameId}/book")
    public @NonNull GameResponse book(@PathVariable String gameId, @RequestBody BookRollRequest bookRollRequest) {
        KniffelGame kniffelGame = gameService.getGameInfo(gameId);
        BookingType enumBookingType = BookingType.valueOf(bookRollRequest.getBookingType());
        gameService.bookRoll(kniffelGame, enumBookingType);
        return mapGameResponse(kniffelGame);
    }
    // method mapGameResponse ...
}
```

## Why do we seperate between `GameController` and `GameService`?

One could argue that the class `GameService` is not needed and you could easily put all the code directly into `GameController`. While it might not be obvious in this small example, in larger applications it is very important to seperate the REST / web / http aspects of the code from the business / game logic.

The two main reasons you need to have the code separated into Controller and Service are:

* DB transactions
* Re-usable business / game logic

This tutorial doesn't want to go into the details, but I wanted to mention it here, so you have at least heard why people insist on seperating the code.

## What you need to keep in mind

* `@RestController` classes
    * handles mapping REST DTOs into logic classes and vice versa
    * handles http status codes, it might map a Java exception to http status code - we will see how this works in chapter 5
* `@Service` classes
    * connects the REST layer to business/game logic classes and the database layer - in our case the `games = new HashMap()` acts as a in-memory database
    * does not know anything about REST / http, that means it does not work with http status code or REST DTOs

## Game logic

To implement the game logic, I have used a library from https://mvn.oglimmer.com/. This library provides the class KniffelGame, KniffelPlayer and BookingType. As said earlier you can either use those classes from the library or implement your own game logic.

We should now have a working REST API for Kniffel.

Let's test it.

# Step 1 - Testing in Postman

Postman is a REST API testing tool. You can down it here: https://www.postman.com/downloads/

Unfortunately you need an account to use Postman properly and on top of that, I find Postman the wrong tool. It doesn't integrate into CI/CD pipelines, so I automate things with curl. But many people use Postman, so I want to show it here as well. 

After starting Postman and logging into your account you can import the OpenAPI / Swagger definition into Postman. Under "Collections" click the "Import" button. Use your OpenAPI defintion URL into the field `http://localhost:8080/v3/api-docs`. You can import it either way.

There are many tutorials and videos on the internet about Postman. You can play the game via Postman, if you like.

# Step 2 - Playing the game via command line with curl+jq

We can write a "real" client for the command line to play Kniffel with `curl` and `jq`.

## Prerequisites for Linux and macOS

On Linux and macOS nstall the package `curl` and `jq` with your package manager.

## Prerequisites for Windows

On Windows curl is installed by default and you can install jq via `winget install jqlang.jq`. Don't use "PowerShell", you have to use "CMD" as `curl` has a completely different syntax in PowerShell.

## Creating a came

Execute this on the terminal:

```bash
curl "http://localhost:8080/api/v1/game/" -d "{\"playerNames\": [\"oli\",\"mike\"]}" -H "Content-Type: application/json"
```

This should return some JSON, most importantly you need to find the Game ID. This will also show you the dice this player has initially rolled.

A result could look like this:

```JSON
{"gameId":"11c5481a96c244b3a23317969370c284","playerData":[{"name":"mike","score":0},{"name":"oli","score":0}],"currentPlayerName":"oli","state":"ROLL","usedBookingTypes":[],"availableBookingTypes":["ONES","TWOS","THREES","FOURS","FIVES","SIXES","THREE_OF_A_KIND","FOUR_OF_A_KIND","FULL_HOUSE","SMALL_STRAIGHT","LARGE_STRAIGHT","KNIFFEL","CHANCE"],"diceRolls":[1,1,2,5,5],"rollRound":1}
```

Now the user has to call the re-roll endpoint twice, also passing the dice to keep as a parameter.

## calling /roll

```bash
# replace 11c5481a96c244b3a23317969370c284 with your game id
# replace 5,5 with the dice values you want to keep - thus not re-roll
curl "http://localhost:8080/api/v1/game/11c5481a96c244b3a23317969370c284/roll" -X POST -d "{\"diceToKeep\": [5,5]}" -H "Content-Type: application/json"
```

Now you should see again JSON. The "dice rolls" should have changed, except for the dice you wanted to keep. Call this endpoint a second time.

With the last re-roll you will see that the field state has changed to "BOOK":

```JSON
{"gameId":"11c5481a96c244b3a23317969370c284","playerData":[{"name":"mike","score":0},{"name":"oli","score":0}],"currentPlayerName":"oli","state":"BOOK","usedBookingTypes":[],"availableBookingTypes":["ONES","TWOS","THREES","FOURS","FIVES","SIXES","THREE_OF_A_KIND","FOUR_OF_A_KIND","FULL_HOUSE","SMALL_STRAIGHT","LARGE_STRAIGHT","KNIFFEL","CHANCE"],"diceRolls":[1,3,5,5,5],"rollRound":3}
```

this means that the next call needs to use the .../book endpoint.

## calling /book

To call the book endpoint we use another curl command like this:

```bash
# make sure you use a bookingType to score the max points, as I rolled 3x a 5, I go for the category "FIVES" 
curl "http://localhost:8080/api/v1/game/11c5481a96c244b3a23317969370c284/book" -X POST -d "{\"bookingType\": \"FIVES\"}" -H "Content-Type: application/json"
```

The result is JSON again and you will see that the currentPlayer has changed. So we need to re-roll dice again.

## Making an easy to use script on Linux and macOS

We can put everything togehter and make it playable.

Save this as "play.sh", give it execution permissions `chmod +x play.sh` and run it:

```bash
#!/bin/sh

GAME_CREATE=$(curl "http://localhost:8080/api/v1/game/" -d '{"playerNames": ["oli","mike"]}' -H "Content-Type: application/json" -s | jq)

echo "$GAME_CREATE"

GAME_ID=$(echo "$GAME_CREATE" | jq -r '.gameId')

RUNNING=true

while "$RUNNING" = "true"; do
    ROLL_ROUND=0
    while [ $ROLL_ROUND -ne 3 ]; do
      curl "http://localhost:8080/api/v1/game/$GAME_ID" -s | jq

      echo "Enter dice to keep: (comma separated) "
      read -r data

      ROLL_RESPONSE=$(curl "http://localhost:8080/api/v1/game/$GAME_ID/roll" -X POST -d "{\"diceToKeep\": [$data]}" -H "Content-Type: application/json" -s)
      ROLL_ROUND=$(echo "$ROLL_RESPONSE" | jq -r '.rollRound')
    done

    curl "http://localhost:8080/api/v1/game/$GAME_ID" -s | jq

    echo "Enter booking type: "
    read -r data

    curl "http://localhost:8080/api/v1/game/$GAME_ID/book" -X POST -d "{\"bookingType\": \"$data\"}" -H "Content-Type: application/json" -s | jq

done
```

Playing Kniffel via your REST API on the terminal ;)

## Making an easy to use script on Windows

Create a file `play.bat` and put this content into it:

```bat
@echo off
setlocal enabledelayedexpansion

for /f %%i in ('curl http://localhost:8080/api/v1/game/ -d "{\"playerNames\": [\"oli\",\"mike\"]}" -H "Content-Type: application/json"') do set GAME_CREATE=%%i

echo %GAME_CREATE%

for /f %%i in ('"echo %GAME_CREATE%" ^| jq -r .gameId') do set "GAME_ID=%%i"

set RUNNING=true

:while_loop
if "!RUNNING!"=="true" (
    set ROLL_ROUND=0
    :roll_round
    if !ROLL_ROUND! neq 3 (
        curl http://localhost:8080/api/v1/game/%GAME_ID% -s | jq

        set /p DATA="Enter dice to keep: (comma separated)"

        for /f %%i in ('curl http://localhost:8080/api/v1/game/%GAME_ID%/roll -X POST -d "{\"diceToKeep\": [!DATA!]}" -H "Content-Type: application/json" -s ^| jq -r ".rollRound"') do set ROLL_ROUND=%%i

        goto :roll_round
    )

    curl http://localhost:8080/api/v1/game/%GAME_ID% -s | jq

    set /p DATA="Enter booking type:"

    curl http://localhost:8080/api/v1/game/%GAME_ID%/book -X POST -d "{\"bookingType\": \"!DATA!\"}" -H "Content-Type: application/json" -s | jq

    goto :while_loop
)
```

## Why curl + jq matter?

While writing a terminal client looks a bit superfluous or without a real world use-case, it is the testing and automation capabilities of curl, jq and other command line tools which makes them invaluable for software development.

# Step 3 - Writing a real html/css/javascript client

Let's do a crash course on Vue.

This requires that you have nodejs installed. See [their webpage](https://nodejs.org/en/download/package-manager) how to download and install nodejs.

We start by creating a "vue" project via:

```bash
npm create vue@latest
```

You will be asked a couple of questions, feel free to use my answers:

```shell
Vue.js - The Progressive JavaScript Framework

✔ Project name: … kniffel-client
✔ Add TypeScript? … Yes
✔ Add JSX Support? … No
✔ Add Vue Router for Single Page Application development? … No
✔ Add Pinia for state management? … No
✔ Add Vitest for Unit Testing? … No
✔ Add an End-to-End Testing Solution? › No
✔ Add ESLint for code quality? … Yes
✔ Add Prettier for code formatting? … Yes
```

You should be able to 

```bash
cd kniffel-client # step into the newly created directory
npm install # this installs the dependencies for this project
npm run dev # this starts vue for development with a test webserver running on 5173
```

Access the project at http://localhost:5173 - you should see a template project using Vue.

## Creating the http helper classes

As we have a Swagger/OpenAPI specification for our REST API, we don't have to write the TypeScript code for that manually. We can generate this TypeScript code based on the Swagger/OpenAPI specification.

We need to install a generator tool called "openapi-typescript" and its runtime "openapi-fetch":

```bash
npm i openapi-fetch
npm i --save-dev openapi-typescript
```

Now we can generate the code. 

*IMPORTANT*: 
First we have to make sure that our REST API - the Spring application - is started.

Now we download the OpenAPI specification as json and save it to `api.json`. Then we run the code generator to create TypeScript code based on the api.json file.

```bash
curl "http://localhost:8080/v3/api-docs" > api.json
npx openapi-typescript api.json -o ./src/api/v1.d.ts
```

If you look into `./src/api/v1.d.ts` you will find all types and all code to call the http endpoints.

## Create game

We are not looking all details of a Vue / npm / nodejs programming, as this needs its own tutorial. Let's focus on the chances we need to do, to make this project a Kniffel REST API client.

Open the file `src/App.vue`, remove it's content.

Let's start with "create game" funtionality:

```vue
<script setup lang="ts">
// import a function from the vue framework and the openapi-fetch runtime plus generated classes
import { ref } from 'vue'
import type { Ref } from 'vue'
import createClient from 'openapi-fetch';
import type { components, paths } from '@/api/v1';

// this creates an http client for our REST api. If you don't use generated code to access the
// REST API, you would use fetch API (https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
// for this tutorial - and I would recommend it anyway - we use this client instead of fetch
const client = createClient<paths>({ baseUrl: "http://localhost:8080" });

// define the interface to hold the player name
interface PlayerInformation {
    index: number; // position
    name: string; // name
}
// ref() defines a reactive variable
// this means whenever we update the variable in JavaScript the UI gets updated and
// whenever the browser updates the UI our variable gets updated as well - both
// directions automatically
const names = ref<PlayerInformation[]>([
    {index: 0, name: ''}, 
    {index: 1, name: ''}
]);
// a function to call the REST API using the generated code
async function createGame() {
    const { data } = await client.POST("/api/v1/game/", {
        body: {
            playerNames: names.value.map(n => n.name)
        }
    });
    alert(data);
}
</script>

<template>
    <h1>Player Names</h1>
    <form>
        <ul>
            <li v-for="ply in names" :key="ply.index">
                Player {{ ply.index+1 }}'s name: <input type="text" v-model="ply.name" />
            </li>
        </ul>
    </form>
    <button @click="names.push({index: names.length, name: ''})">Add Name</button> &nbsp;
    <button @click="createGame">Create Game</button>
</template>
```

You should be able to create a game at http://localhost:5173. After pressing the "Create Game" button, you should see response JSON.

### Show the game information

We need to define more data interfaces to use the response of the create game REST call and show it on the screen.

```vue
<script setup lang="ts">
// keep everything else in <script>

// this defines a variable to hold the loaded GameData using a type from the generated code
// while the data coming from the REST API cannot be undefined, the variable gameData is
// undefined right after initialization and before the REST call comes back
const gameData : Ref<components["schemas"]["GameResponse"] | undefined> = ref();

// look for this method and change it
async function createGame() {
    const { data } = await client.POST("/api/v1/game/", {
        body: {
            playerNames: names.value.map(n => n.name)
        }
    });
    // after loading the data, we store it
    gameData.value = data;
}
</script>
<!-- replace everything from here on -->
<template>
    <div v-if="!gameData?.gameId">
        <h1>Player Names</h1>
        <form>
            <ul>
                <li v-for="ply in names" :key="ply.index">
                    Player {{ ply.index+1 }}'s name: <input type="text" v-model="ply.name" />
                </li>
            </ul>
        </form>
        <button @click="names.push({index: names.length, name: ''})">Add Name</button> &nbsp;
        <button @click="createGame">Create Game</button>
    </div>
    <div v-if="gameData?.gameId">
        <h1>Game Scores</h1>
        <ul>
            <li v-for="ply in gameData.playerData" :key="ply.name">
                Player {{ ply.name }} - Score: {{ ply.score }}
            </li>
        </ul>
        <h1 class="mt-20">
            Current player: {{ gameData.currentPlayerName }}
        </h1>
    </div>
    <div v-if="gameData?.state === 'ROLL'">
        <h3>Roll round: {{ gameData.rollRound }}</h3>
        <h3 style="margin-top: 30px;">Select the dice to keep:</h3>
          <ul>
              <li v-for="(die, idx) in gameData.diceRolls" :key="idx">
                  {{ die }}
              </li>
          </ul>
    </div>
</template>
```

Now we can see the game information after the game's creation.

## Re-roll the dice

Let's add checkboxes for each dice (to keep it) and a button to do the re-roll.

```vue
<script setup lang="ts">
// .. keep everyting here

// this will store the checkbox selection 
const rerollSelection = ref([false, false, false, false, false]);

// add at the end of script
async function reroll() {
    // we need to pass the dice values to the REST API, but we store only the keep/reroll boolean
    // information per dice. so this converts the yes/no to dice values to keep
    const diceToKeep : number[] = [];
    if (gameData.value) {
        for (let i = 0; i < gameData.value.diceRolls.length; i++) {
            // simple logic: if the user checked the box for "keep it",
            // we have to push the value to an array
            if (rerollSelection.value[i]) {
                diceToKeep.push(gameData.value.diceRolls[i]);
            }
        }
        const { data } = await client.POST("/api/v1/game/{gameId}/roll", {
            params: {
                path: {
                    gameId: gameData.value.gameId
                }
            },
            body: {
                diceToKeep
            }
        });
        gameData.value = data;
        // the result contains only the dice values, but we need to keep the checkboxes on
        // the dice values the user kept in the last round, so we have to look for
        // dice values the users checked and re-check it
        if (gameData.value) {
            for (let i = 0; i < gameData.value.diceRolls.length; i++) {
                const idxToKeep = diceToKeep.indexOf(gameData.value.diceRolls[i]);
                if (idxToKeep === -1) {
                    rerollSelection.value[i] = false;
                } else {
                    rerollSelection.value[i] = true;
                    diceToKeep.splice(idxToKeep, 1);
                }
            }
        }
    }
}
</script>
<!-- replace everything from here on -->
<template>
    <div v-if="!gameData?.gameId">
        <h1>Player Names</h1>
        <form>
            <ul>
                <li v-for="ply in names" :key="ply.index">
                    Player {{ ply.index+1 }}'s name: <input type="text" v-model="ply.name" />
                </li>
            </ul>
        </form>
        <button @click="names.push({index: names.length, name: ''})">Add Name</button> &nbsp;
        <button @click="createGame">Create Game</button>
    </div>
    <div v-if="gameData?.gameId">
        <h1>Game Scores</h1>
        <ul>
            <li v-for="ply in gameData.playerData" :key="ply.name">
                Player {{ ply.name }} - Score: {{ ply.score }}
            </li>
        </ul>
        <h1 class="mt-20">
            Current player: {{ gameData.currentPlayerName }}
        </h1>
    </div>
    <div v-if="gameData?.state === 'ROLL'">
        <h3>Roll round: {{ gameData.rollRound }}</h3>
        <div> These types are still available:
            {{ gameData.availableBookingTypes }}
        </div>
        <h3 style="margin-top: 30px;">Select the dice to keep:</h3>
          <ul>
              <li v-for="(die, idx) in gameData.diceRolls" :key="idx">
                  {{ die }} <input type="checkbox" v-model="rerollSelection[idx]" />
              </li>
          </ul>
        <button @click="reroll">Roll</button>
    </div>
</template>
```

## Select booking type 

Now we need to add dropdown box to select the booking type and a button to call the REST API to send it.

You can replace everything with this content.

```vue
<script setup lang="ts">
import { ref } from 'vue'
import type { Ref } from 'vue'
import createClient from 'openapi-fetch';
import type { components, paths } from '@/api/v1';

const client = createClient<paths>({ baseUrl: "http://localhost:8080" });

interface PlayerInformation {
    index: number;
    name: string;
}

const names : Ref<PlayerInformation[]> = ref([{index: 0, name: ''}, {index: 1, name: ''}]);
const gameData : Ref<components["schemas"]["GameResponse"]|undefined> = ref();
const rerollSelection = ref([false, false, false, false, false]);
// the booking type selected in the dropdown box
const selectedBookingType : Ref<components["schemas"]["GameResponse"]["usedBookingTypes"]|undefined> = ref();

async function createGame() {
    const { data } = await client.POST("/api/v1/game/", {
        body: {
            playerNames: names.value.map(n => n.name)
        }
    });
    gameData.value = data;
}

async function reroll() {
    const diceToKeep : number[] = [];
    if (gameData.value) {
        for (let i = 0; i < gameData.value.diceRolls.length; i++) {
            if (rerollSelection.value[i]) {
                diceToKeep.push(gameData.value.diceRolls[i]);
            }
        }
        const { data } = await client.POST("/api/v1/game/{gameId}/roll", {
            params: {
                path: {
                    gameId: gameData.value.gameId
                }
            },
            body: {
                diceToKeep
            }
        });
        gameData.value = data;
        selectedBookingType.value = undefined;
        if (gameData.value) {
            for (let i = 0; i < gameData.value.diceRolls.length; i++) {
                const idxToKeep = diceToKeep.indexOf(gameData.value.diceRolls[i]);
                if (idxToKeep === -1) {
                    rerollSelection.value[i] = false;
                } else {
                    rerollSelection.value[i] = true;
                    diceToKeep.splice(idxToKeep, 1);
                }
            }
        }
    }
}

// simple REST API call to send the booking type
async function book() {
  if (gameData.value) {
    const { data } = await client.POST("/api/v1/game/{gameId}/book", {
      params: {
        path: {
          gameId: gameData.value.gameId
        }
      },
      body: {
        bookingType: selectedBookingType.value
      }
    });
    gameData.value = data;
    rerollSelection.value = [false, false, false, false, false];
  }
}

</script>

<template>
    <div v-if="!gameData?.gameId">
        <h1>Player Names</h1>
        <form>
            <ul>
                <li v-for="ply in names" :key="ply.index">
                    Player {{ ply.index+1 }}'s name: <input type="text" v-model="ply.name" />
                </li>
            </ul>
        </form>
        <button @click="names.push({index: names.length, name: ''})">Add Name</button> &nbsp;
        <button @click="createGame">Create Game</button>
    </div>
    <div v-if="gameData?.gameId">
        <h1>Game Scores</h1>
        <ul>
            <li v-for="ply in gameData?.playerData" :key="ply.name">
                Player {{ ply.name }} - Score: {{ ply.score }}
            </li>
        </ul>
        <h1 class="mt-20">
            Current player: {{ gameData.currentPlayerName }}
        </h1>
    </div>
    <div v-if="gameData?.state === 'ROLL'">
        <h3>Roll round: {{ gameData?.rollRound }}</h3>
        <div> These types are still available:
            {{ gameData?.availableBookingTypes }}
        </div>
        <h3 style="margin-top: 30px;">Select the dice to keep:</h3>
          <ul>
              <li v-for="(die, idx) in gameData.diceRolls" :key="idx">
                  {{ die }} <input type="checkbox" v-model="rerollSelection[idx]" />
              </li>
          </ul>
        <button @click="reroll">Roll</button>
    </div>
    <div v-if="gameData?.state === 'BOOK'">
        <h1>Final dice rolls: {{  gameData.diceRolls }}</h1>
        <div class="mt-20">
          Select the booking type:
        </div>
        <select v-model="selectedBookingType">
            <option v-for="cat in gameData.availableBookingTypes" :key="cat" :value="cat">{{ cat }}</option>
        </select>
        <button @click="book">Book</button>
    </div>
</template>

<style scoped>
button,input {
    margin: 10px;
}
.mt-20 {
    margin-top: 20px;
}
</style>
```

How you can play Kniffel with your REST API on the browser in a 'couch co-op' style.

# What we've learnt

* Postman as REST API testing tool
* curl and jq can be used to debug, test or automate http calls and thus any REST API
* How to build a very simple Vue application

# Extras if you time

* A long list of useful curl commands https://curl.se/docs/tutorial.html
* Read more about Vue https://vuejs.org/
