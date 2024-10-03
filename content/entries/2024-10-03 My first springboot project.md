---
title: My First Spring Boot Web Application
---

<div align="center">
<img src="/assets/My-First-Spring-boot-project/spring-kotlin.png"/>
</div>

In this article, I will explain the development process of a Tic Tac Toe full-stack game using WebSocket, with Spring Boot as the backend and React.js as the frontend. **I will start with the backend, which powers the frontend, but first, what is Spring Boot?**

Spring Boot is a server-side framework that makes it easy to create stand-alone, production-grade Spring-based applications that you can "just run." It takes an opinionated view of the Spring platform and third-party libraries, allowing you to get started with minimal fuss. Most Spring Boot applications require minimal Spring configuration.

## Project Goal

The goal of this project is to explore the Spring Boot ecosystem and its developer experience by implementing a simple Tic Tac Toe game using the Kotlin programming language.

## Development Environment

- IDE: [IntelliJ IDEA Community Edition](https://www.jetbrains.com/idea/download)
- JDK: [Amazon Corretto 21](https://docs.aws.amazon.com/corretto/latest/corretto-21-ug/downloads-list.html)
- Database: [PostgreSQL](https://www.postgresql.org/download/)
- Build Tool: Gradle

## Initialize Springboot Project

1. Go to [spring initializr](https://start.spring.io/)

2. Click on Add dependencies
3. Add these dependencies:
    - Spring Web
    - Spring Data R2DBC
    - PostgreSQL Driver
    - WebSocket

    it should look like this:
    <img src="/assets/My-First-Spring-boot-project/spring-initializr.png"/>

4. Click on Generate

it will start downloading a zip file, after the the download finished extract it then open the project using the IDE (IntelliJ IDEA Community Edition).
After opening the project the IDE will start downloading the dependencies.

## Project Architecture

<img src="/assets/My-First-Spring-boot-project/app-flow.png"/>

1. Client send a message.
2. The message will be converted to `WsCommand` then by the action from the parsed message, the message will be handled by a service.
3. The service will talk with repository to retrieve data.
4. The repository will talk with the db to retrieve data.
5. The db will execute the query from the repository.
6. The repository will convert the result to data class and then send it to the service.
7. The service return the result to the session service which will convert the result to JSON.
8. Send the the result to the client.

## Project Directories Structure

To make the development easier and fast to adapt we need to manage the project structure.

```bash
/config              # Will contain the configuration classes that spring boot relay on to configure the project requirements
/models              # Will contain the models (classes) that will be used for mapping the postgresql quires responses
/repo                # Will contain the interfaces that Spring Data R2DBC relay on to generate SQL quires by declaring functions which their names represent the SQL query
/responses           # Will contain the responses that will be sent to the client over websocket
/services            # Will contain the services that will handle websocket messages
/utils               # Helper functions used in transforming the classes to JSON and vice versa 
Application.kt       # The project entrypoint
WebSocketHandler.kt  # Will handle client websocket messages and send it to the corresponded services
```

## Let's Begin

### WebSocket configuration

We need to configure springboot to use WebSocket.

First we need to create a class called `WebSocketConfig` and annotate it with `@Configuration` to indicates that it is a configuration class, meaning the spring framework will use it to configure websocket functionality in `/config` directory,
Next add `@EnableWebSocket` annotation to enable websocket functionality, then we need to implement the `WebSocketConfigurer` interface to override `registerWebSocketHandlers` method to register the websocket endpoint:

```kt title="/config/WebSocketConfig.kt"
@Configuration
@EnableWebSocket
class WebSocketConfig() : WebSocketConfigurer {
    override fun registerWebSocketHandlers(registry: WebSocketHandlerRegistry) {
        registry
            .addHandler(
                WebSocketHandler(), // websocket handler: will explain it later in the article
                "/ws", // the entrypoint for the websocket protocol
            ).addInterceptors(HttpSessionHandshakeInterceptor()) // HttpSessionHandshakeInterceptor is an interceptors which implement the server-client handshake functionality
            .setAllowedOrigins("*") // here we allow any one to access to the endpoint, we can specify an origin but for now let it as it's
    }
}
```

### WebSocketHandler

The `WebSocketHandler` is the entry point to our WebSocket endpoint. It manages client WebSocket sessions, handling connection establishment, disconnection, and the reception of messages. By implementing the `TextWebSocketHandler` class, we define how the server handles WebSocket events and messages.

First, create a class called `WebSocketHandler` that implements the `TextWebSocketHandler` class:

```kt title="WebSocketHandler.kt"
class WebSocketHandler: TextWebSocketHandler() {
    override fun afterConnectionEstablished(session: WebSocketSession) {
           // Handle new connection
    }
    override fun handleTextMessage(
        session: WebSocketSession,
        message: TextMessage,
    ) {
        // Handle incoming message
    }
    override fun afterConnectionClosed(
        session: WebSocketSession,
        status: CloseStatus,
    ) {
        // Handle connection closed
    }

}
```

### Message structure

In a REST API, HTTP methods like GET, POST, PUT, and DELETE are defined in the request headers to indicate the type of action the client wants to perform on an endpoint. Similarly, in our WebSocket implementation, we will use a concept called commands.

A command indicates the action the client wants to perform, and the server handles it accordingly. Since WebSocket messages are sent as a byte array, Spring Boot transforms these messages into a string.<br/>
We will expect the command is in JSON format with the following primary keys:

- `clientId`: The client identifier that will be sent when client is connected to the server, will be UUID.
- `action`: Action type which indicates to which service the command should be handled with.

The action will be one of these:

- `CREATE_GAME`: Client want to create game.<br/>
- `GET_AVAILABLE_GAMES`: Client want the games that has no secund player or finished. <br/>
- `JOIN_GAME`: Client want to join a game.
- `QUIT_GAME`: Client want to quit a game
- `UPDATE_GAME`: Client want to preform an update to the game he joined in like updating the board<br/>

An example command would look like this:

```json
{
    "clientId":"303c055a-e0e3-4dc1-bc93-a13081668023",
    "action":"CREATE_GAME"
}
```

So to implement this command structure we need first create a kotlin file named as `WsCommand` in `/models` directory to define the action as an enum and will call it `ActionType`:

```kt title="/models/WsCommand.kt"
enum class ActionType {
    CREATE_GAME,
    JOIN_GAME,
    UPDATE_GAME,
    GET_AVAILABLE_GAMES,
    QUIT_GAME,
}
```

Next, define the command class, which will be used to map the incoming JSON data to a WsCommand object:

```kt title="/models/WsCommand.kt"
data class WsCommand(
    val action: ActionType,
    val gameId: UUID?,
    val clientId: UUID?,
    val board: Array<Array<String>>?,
    val isGamePrivate: Boolean?,
    val row: Int?,
    val column: Int?,
    val requestId: String,
)
```

The properties: `gameId`, `board`, `isGamePrivate`, `row`, `column`, `requestId` are optional and they will be explained later in the article.

### Handle Messages

First in `/utils` directory we need to create a kotlin file named `WsUtil`.<br/>
Next create a function called `messageToWsCommand` that will take a message (WebSocket message payload) as `String` and objectMapper as `ObjectMapper` from `jackson` which will transform the string message to JSON then mapping it to `WsCommand`.<br/>
`messageToWsCommand` will return `WsCommand` if the mapping success or `null` in case of failure:

```kt title="/utils/messageToWsCommand.kt"
fun messageToWsCommand(message: String, objectMapper: ObjectMapper): WsCommand? {
    return try {
        val jsonObject = objectMapper.readValue(message, WsCommand::class.java)
        jsonObject
    } catch (e: JsonParseException) {
        null
    }
}
```

Next, we need to use `messageToWsCommand` in `handleTextMessage` in `WebSocketHandler` class.<br/>
`handleTextMessage` provides `session` that will be used later for managing player session as well as message that will be converted to `WsCommand` using `messageToWsCommand`:

```kt title="WebSocketHandler.kt"
    override fun handleTextMessage(session: WebSocketSession,message: TextMessage) {
        val messageText = message.payload // the actual message (command)
        val command = messageToWsCommand(messageText, objectMapper) // convert messageText to WsCommand
        if (command == null) {
            //TODO: Send error message to client
            return
        }
            //TODO: handle command according to the action
    }
```

### Session Service

Session service will be responsible to send messages, manage client session...to be continue
