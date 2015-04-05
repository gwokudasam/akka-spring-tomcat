currency-fair-test
========================

Introduction
------------

Application is implemented in Java and runs in Tomcat.
Implementation is based on usage of **[Akka](http://akka.io)** toolkit, to enable highly concurrent processing of transaction messages.
Spring is used for dependency injection and orchestration of components. Endpoints are built using `spring-mvc` and its REST support.
Frontend is simple single page application which uses "traditional" ajax polling to update reports related to transactions.

Architecture overview
--------------

Akka'a `ActorSystem` is created on application startup and destroyed on application shutdown through dedicated listener defined in `web.xml`.

Actors are created via Spring extension of `ActorSystem` in order to enable injection of properties into actors via standard Spring's features like `@Autowired`.

JSON messages are consumed by `spring-mvc` controller. If received message's structure is valid, controller starts an actor, called `TransactionProcessor`, for message handling. This actor also serves as supervisor for other 2 used actors.

Upon reception of message, `TransactionProcessor` tries to transform it into domain object and persist it to database, so it can be later queried and analyzed.
If transformation succeeds, `TransactionProcessor` is spawning child actor caller `Transferer`. `Transferer`'s sole role is to perform transfer of bought amount to user's account. Depending on success of transfer, it returns corresponding message to `TransactionProcessor`, which then acts accordingly, and shuts down `Transferer` actor.

Another child actor supervised by `TransactionProcessor` is called `ErrorHandler` and it logs errors into database, so they can be reported and analyzed.
`ErrorHandler` is called when `Transferer` fails to transfer funds or when `TransactionProcessor` fails to transform message into domain object. `ErrorHandler` shuts itself down upon completed task.

Finally, `TransactionProcessor` is terminating itself after 3 minutes, which is conservatively-set period of time sufficient to process single message.

Persistence layer is implemented with JPA/Hibernate and data is stored in MySQL database.

Frontend is html page + jquery for asynchronous calling of various controller methods that are returning JSON response. It is possible to query and generate reports on date basis, since every `Transaction` and `Error` have corresponding timestamps.
