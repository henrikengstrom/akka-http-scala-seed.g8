Server logic
----------------

The main class, `QuickstartServer`, is runnable because it extends `App`, as shown in the following snippet. We will discuss the trait `JsonSupport` later.

@@snip [QuickstartServer.scala]($g8src$/scala/com/lightbend/akka/http/sample/QuickstartServer.scala) { #main-class }

Next, we'll examine the code in this class that:

* Binds a `Route` to endpoints and HTTP methods
* Binds a server that will handle all requests to an IP and a port
* Adds error handling for when something goes wrong

Under the hood, Akka HTTP uses [Akka Streams](http://doc.akka.io/docs/akka/current/scala/stream/index.html). You can think of a `Route` as a flow of in- and outbound data, which is a perfect fit for Akka Streams. (Technically the `Route` type is `RequestContext ⇒ Future[RouteResult]` but there is no need to worry about what that means now.) We don't have time to cover Akka Streams here, but if you are interested, you should take a look at the Hello World sample application for Akka Streams.

## Binding endpoints
Each Akka HTTP `Route` contains one or more `akka.http.scaladsl.server.Directives`, such as: `path`, `get`, `post`, `complete`, etc. For the user registry service, the example needs to support the actions listed below. For each, we can identify a path, the HTTP directive, and return value:

Comments: maybe there is a better term to use here than "action" or "functionality", but those are all I could come up with?

|Functionality       | Path       | HTTP directive  | Returns              |
|--------------------|------------|-----------------|----------------------|
| Create a user      | /user      | POST            | Confirmation message |
| Retrieve a user    | /user/$ID  | GET             | JSON payload         |
| Remove a user      | /user/$ID  | DELETE          | Confirmation message |
| Retrieve all users | /users     | GET             | JSON payload         |

In the `QuickstartServer` source file, the definition of the `Route` begins with the line:
`lazy val routes: Route =`. Let's look at the pieces of the example `Route` that bind the enpoints, HTTP methods, and message or payload for each action.

### Creating a new user

The definition of the endpoint to create a new user looks like the following:

@@snip [QuickstartServer.scala]($g8src$/scala/com/lightbend/akka/http/sample/QuickstartServer.scala) { #users-get-post }

Note the following building blocks from the snippet:

* `path` : matches against the incoming URI, in this case, all requests appended with `/user`.
* `post` : matches against the incoming HTTP directive, in this case, `POST`.
* `entity(as[User])` : automatically converts the incoming payload--in this case, we expect JSON--into an entity. We will look more at this functionality in the @ref:[JSON](json.md) section.
* `! createUser()` :  send a message to the actor `userRegistryActor` to have it create a new user. We will look at the actor implementation later.
* `complete` : used to reply back to the request. The `StatusCodes.Created` is translated to Http response code 201. On success, we also send a string confirmation message back to the caller.    

### Retrieving and removing a user

Next, the example defines how to retrieve and remove a user. In this case, the URI must include the user's id in the form: `/user/$ID`. See if you can identify the code that handles that in the following snippet. This part of the route includes logic for both the GET and the DELETE methods.

@@snip [QuickstartServer.scala]($g8src$/scala/com/lightbend/akka/http/sample/QuickstartServer.scala) { #users-get-delete }

This part of the `Route` contains the following:

* `path("user" / Segment) { name => user` : this bit of code matches URIs of the exact format `/user/$ID`, where the ID is a name and `Segment` automatically extracts the name into the `user` variable. For example the URI `/user/Bruce` will populate the `user` variable with the value "Bruce."
* `get` : matches against the incoming HTTP directive and includes the business logic
* `val userInfo` uses the Akka [ask](http://doc.akka.io/docs/akka/current/scala/actors.html#send-messages) pattern, it:
    * Sends a message asynchronously to the actor and return a `Future`, which represents a _possible_ reply.
    * The reply maps to the type `UserInfo`. When the `Future` completes, it will use the second part of the code to evaluate to either `Success`, with or without a result, or a `Failure`.
    * Returns something to the requester, regardless of the outcome, using the `complete` directive with an appropriate response code and value.
* `~` : fuses routes together - this will become more apparent when you look at the complete `Route` definition below.
* `delete` : matches against the HTTP `DELETE` method. The business logic for deleting a user is straight forward. It sends instructions to remove a user to the user registry actor and returns a status code to the client, which in this case is `StatusCodes.OK` (HTTP status code 200).

### Retrieving all users

The last part of the `Route` gets all registered users:

@@snip [QuickstartServer.scala]($g8src$/scala/com/lightbend/akka/http/sample/QuickstartServer.scala) { #users-get-post }

This part of the `Route` matches URIs where the path includes the value `/users`. The code is similar to that for retrieving one user. It sends a message to the user registry actor using an `ask` and passes the `Future` to the `complete` method. However, there is a difference in the way the results are handled.

Compare the simple `complete(users)` in this part of the `Route` and the more complex `onComplete(userInfo)` for retrieving a particular user that we looked at earlier. `GetUsers` will always return something; an empty list simply means that there are no registered users. However, when retrieving a single user, we need to distinguish between a failure and the case where the user does not exist. When sent a `GetUser(name)` the user registry actor sends back an `Option[UserInfo]` that can contain `Some(UserInfo)` or  `None`, which indicates that there was no match in the registry.

### The complete Route

Below is the complete `Route` definition from the sample application:

@@snip [QuickstartServer.scala]($g8src$/scala/com/lightbend/akka/http/sample/QuickstartServer.scala) { #all-routes }


## Binding the HTTP server

At the beginning of the `main` class, the example defines some implicit values that will be used by the Akka HTTP server:

@@snip [QuickstartServer.scala]($g8src$/scala/com/lightbend/akka/http/sample/QuickstartServer.scala) { #server-bootstrapping }

Akka Streams uses these values:

* `ActorSystem` : provides a context in which actors will run. What actors, you may wonder? Akka Streams uses actors under the hood, and the actor system defined in this `val` will be picked up and used by Streams.
* `ActorMaterializer` : allocates all the necessary resources the actors need to run.

Further down in `QuickstartServer.scala`, you will find the code to instantiate the server:

@@snip [QuickstartServer.scala]($g8src$/scala/com/lightbend/akka/http/sample/QuickstartServer.scala) { #http-server }

The `bindAndhandle` method only takes three parameters; `routes`, the hostname, and the port. That's it! When this program runs--as you've seen--it starts an Akka HTTP server on localhost port 8080. Note that startup happens asynchronously and therefore the `bindAndHandle` method returns a `future`.

The code for stopping the server includes the `StdIn.readLine()` method that will wait until RETURN is pressed on the keyboard. When that happens, `flatMap` uses the `Future` returned when we started the server to get to the `unbind()` method. Unbinding is also an asynchronous function. When the `Future` returned by `unbind()` completes, the example code makes sure that the actor system is properly terminated.

## Error handling

Finally, let's look at how the example handles errors. As capable engineers as we are, we know that errors will happen. Our systems will be more robust if we build error handling in than if we leave it as an afterthought.

The example uses a very simple exception handler that catches all unexpected exceptions and responds back to the client with an `InternalServerError` (HTTP status code 500). The response includes an error message and the URI that encountered the error. We extract the URI by using the `extractUri` directive.

@@snip [QuickstartServer.scala]($g8src$/scala/com/lightbend/akka/http/sample/QuickstartServer.scala) { #exception-handler }

## The complete server code

Here is the complete server code used in the sample:

@@snip [QuickstartServer.scala]($g8src$/scala/com/lightbend/akka/http/sample/QuickstartServer.scala)

Let's move on to the actor that handles registration.
