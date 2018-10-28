# 依赖注入

When building services in Lagom, your code will have dependencies on Lagom APIs and other services that need to be satisfied by concrete implementations at runtime. Often, the specific implementations will vary between development, test and production environments, so it's important not to couple your code tightly to a concrete implementation class. A class can declare constructor parameters with the abstract types of its dependencies --- usually represented by traits in Scala --- allowing the concrete implementations to be provided when the class is constructed. This pattern is called "dependency injection" and is fundamental to the way Lagom applications are assembled. Read "Dependency Injection in Scala using MacWire" for more background on this pattern and its benefits.

在 Lagom 中构建服务时，您的代码将依赖于 Lagom APIs 和其他需要在运行时由具体实现满足的服务。通常，特定的实现会因开发、测试和生产环境的不同而不同，所以重要的是不要将代码与具体的实现类紧密地结合在一起。类可以使用依赖项的抽象类型声明构造函数参数——通常用 Scala 中的特征（traits）表示——允许在构造类时提供具体的实现。这种模式称为“依赖注入”，是 Lagom 应用程序组装方式的基础。请阅读“使用 MacWire 在 Scala 中的依赖注入”以了解此模式及其优点的更多背景知识。

Your service's dependencies will have their own dependencies on other APIs in turn. Taken all together, these form a dependency graph that must be constructed when your application starts up. This process is called "wiring" your application, and is performed by creating a subclass of LagomApplication that contains initialization code.

你的服务的依赖关系将依次依赖于其他 APIs。总之，它们形成了一个依赖关系图，在应用程序启动时必须构造这个依赖关系图。这个过程称为 “wiring” 你的应用程序，通过创建包含初始化代码的 LagomApplication 子类来执行。

It is common to have clusters of interdependent classes that together form a larger logical component. It can be useful to modularize wiring code in a way that reflects these groupings. You can do so in Scala by defining a "component" trait that includes a combination of lazy val declarations that instantiate concrete implementations of interfaces that the component provides. Some of their constructor parameters may be declared as abstract methods, indicating a dependency that must be provided by another component or your application itself. Your application can then extend multiple components to mix them together into a fully assembled dependency graph. If you don't fulfill all of the declared requirements for the components you include in your service, it will not compile.

相互依赖的类集群在一起形成更大的逻辑组件是很常见的。对于模块化组织代码进而反映这些分组的方式可能很有用。在 Scala 中，可以通过定义一个“组件”特性来实现，该特性包括一些惰性 val 声明的组合，这些声明实例化了组件提供的接口的具体实现。它们的一些构造函数参数可以声明为抽象方法，表示必须由另一个组件或应用程序本身提供的依赖项。然后，您的应用程序可以扩展多个组件，将它们组合到一个完全组装的依赖关系图中。如果您没有完成服务中包含的组件的所有声明的需求，它将无法编译。

In "[Dependency Injection in Scala: guide](https://di-in-scala.github.io/#modules)" this method of mixing components together to form an application is referred to as the "thin cake pattern". The guide also introduces [Macwire](https://di-in-scala.github.io/#macwire) which we'll use in the next steps.

在“[Scala中的依赖注入:指南](https://di-in-scala.github.io/#modules])”中，这种将组件混合在一起形成应用程序的方法被称为 “thin cake pattern”。本指南还介绍了[Macwire](https://di-in-scala.github.io/#macwire)，我们将在接下来的步骤中使用它。

## Wiring together a Lagom application

> 译者注：本文是理解 Lagom 应用启动逻辑的关键步骤

In section [[Service descriptors|ServiceDescriptors]] and [[Implementing services|ServiceImplementation]] we created a Service and its implementation, we now want to bind them and create a Lagom server with them. Lagom's Scala API is built on top of Play Framework, and uses Play's [compile time dependency injection support](https://www.playframework.com/documentation/2.6.x/ScalaCompileTimeDependencyInjection) to wire together a Lagom application. Play's compile time dependency injection support is based on the thin cake pattern we just described.

在 [[Service descriptors|ServiceDescriptors]] 和 [[Implementing services|ServiceImplementation]] 中，我们创建了一个服务及其实现，现在我们想要将它们绑定起来，并使用它们创建一个 Lagom 服务器。Lagom 的 Scala API 构建在 Play 框架之上，使用 Play 的[编译时依赖注入支持](https://www.playframework.com/documentation/2.6.x/ScalaCompileTimeDependencyInjection)来连接 Lagom 应用程序。Play 的编译时依赖注入支持基于我们刚刚描述的 thin cake pattern。

Once you are creating your Application you will use Components provided by Lagom and obtain your dependencies from them (e.g., an Akka Actor System or a `CassandraSession` for your read-side processor to access the database). Although it's not strictly necessary, we recommend that you use [Macwire](https://github.com/adamw/macwire) to assist in wiring dependencies into your code. Macwire provides some very lightweight macros that locate dependencies for the components you wish to create so that you don't have to manually wire them together yourself. Macwire can be added to your service by adding the following to your service implementations dependencies:

创建应用程序之后，您将使用 Lagom 提供的组件并从它们获得依赖关系(例如，Akka Actor System 或用于访问数据库的读端处理器的 `CassandraSession`)。虽然这不是严格必要的，但我们建议您使用 [Macwire](https://github.com/adamw/macwire) 来帮助将依赖关系连接到代码中。Macwire 提供了一些非常轻量级的宏，这些宏定位您希望创建的组件的依赖项，这样您就不必自己手动将它们连接在一起。Macwire 可通过向您的服务实现依赖项添加以下内容添加到您的服务中:

@[macwire](code/macwire.sbt)

The simplest way to build the Application cake and then wire your code inside it is by creating an abstract class that extends [`LagomApplication`](api/com/lightbend/lagom/scaladsl/server/LagomApplication.html):

构建应用程序 “cake” 并将代码连接到其中的最简单方法是创建一个抽象类来扩展[`LagomApplication`](api/com/lightbend/lagom/scaladsl/server/LagomApplication.html):

@[lagom-application](code/ServiceImplementation.scala)

In this sample, the `HelloApplication` gets [[AhcWSComponents|ScalaComponents#Third-party-Components]] mixed in using the cake pattern and implements the `lazy val lagomService` using Macwire. The important method here is the `lagomServer` method. Lagom will use this to discover your service bindings and create a Play router for handling your service calls. You can see that we've bound one service descriptor, the `HelloService`, to our `HelloServiceImpl` implementation. The name of the Service Descriptor you bind will be used as the Service name, that is used in cross-service communication to identify the client making a request.

在这个示例中，`HelloApplication` 使用 cake 模式混合了 [[AhcWSComponents|ScalaComponents#Third-party-Components]，并使用 Macwire 实现了 `lazy val lagomService`。这里的重要方法是 `lagomServer` 方法。Lagom 将使用它来发现您的服务绑定，并创建一个 Play 路由器来处理您的服务调用。您可以看到，我们已经将一个服务描述符`HelloService` 绑定到我们的 `HelloServiceImpl` 实现。您绑定的 Service Descriptor 的名称将用作服务名称，在跨服务通信中用于标识发出请求的客户端。

We've used Macwire's `wire` macro to wire the dependencies into `HelloServiceImpl` - at the moment our service actually has no dependencies so we could just construct it manually ourselves, but it's not likely that a real service implementation would have no dependencies.

我们使用 Macwire 的 `wire` 宏将依赖项连接到“ `HelloServiceImpl` — 目前我们的服务实际上没有依赖项，所以我们可以自己手工构造它，但是一个真正的服务实现不可能没有依赖项。

The `HelloApplication` is an abstract class, the reason for this is that there is still one method that hasn't been implemented, the `serviceLocator` method. The `HelloApplication`  extends [`LagomApplication`](api/com/lightbend/lagom/scaladsl/server/LagomApplication.html) which needs a `serviceLocator: ServiceLocator`, a `lagomServer: LagomServer` and a `wcClient: WSClient`. We provide the `wcClient: WSClient` mixing in `AhcWSComponents` and we provide `lagomServer: LagomServer` programmatically. A typical application will use different service locators in different environments, in development, it will use the service locator provided by the Lagom development environment, while in production it will use whatever is appropriate for your production environment, such as the service locator implementation provided by [Lightbend Orchestration](https://developer.lightbend.com/docs/lightbend-orchestration/current/features/service-location.html). So our main application cake leaves this method abstract so that it can mix in the right one depending on which mode it is in when the application gets loaded.

`HelloApplication` 是一个抽象类，其原因是仍然有一个尚未实现的方法，即 `serviceLocator` 方法。`HelloApplication` 扩展了 [`LagomApplication`](api/com/lightbend/lagom/scaladsl/server/LagomApplication.html)，它需要一个 `serviceLocator: ServiceLocator`，一个 `lagomServer: LagomServer` 和一个 `wcClient: WSClient`。我们提供 `wcClient: WSClient` 混合 `AhcWSComponents` ，我们通过编程提供  `lagomServer: LagomServer` 。一个典型的应用程序将在不同的环境中使用不同的服务定位器，在开发中，它将使用 Lagom 开发环境提供的服务定位器，而在生产中，它将使用任何适合您的生产环境的服务定位器实现，例如[Lightbend Orchestration](https://developer.lightbend.com/docs/lightbend-orchestration/current/features/service-location.html)。因此，我们的主应用程序 cake 留下了这个方法的抽象，以便它可以根据应用程序加载时的模式混合到正确的模式中。

Having created our application cake, we can now write an application loader. Play's mechanism for loading an application is for the application to provide an application loader. Play will pass some context information to this loader, such as a classloader, the running mode, and any extra configuration, so that the application can bootstrap itself. Lagom provides a convenient mechanism for implementing this, the [`LagomApplicationLoader`](api/com/lightbend/lagom/scaladsl/server/LagomApplicationLoader.html):

创建了应用程序蛋糕之后，我们现在可以编写应用程序加载器了。Play 加载应用程序的机制是让应用程序提供应用程序加载器。Play 会将一些上下文信息传递给这个加载器，比如类加载器、运行模式和任何额外的配置，这样应用程序就可以自我引导了。Lagom 提供了一个方便的机制来实现它，[`LagomApplicationLoader`](api/com/lightbend/lagom/scaladsl/server/LagomApplicationLoader.html):

@[lagom-loader](code/ServiceImplementation.scala)

The loader has two methods that must be implemented, `load` and `loadDevMode`. You can see that we've mixed in different service locators for each method, we've mixed in [`LagomDevModeComponents`](api/com/lightbend/lagom/scaladsl/devmode/LagomDevModeComponents.html) that provides the dev mode service locator and registers the services with it in dev mode, and in prod mode, for now, we've simply provided [`NoServiceLocator`](api/com/lightbend/lagom/scaladsl/api/ServiceLocator$$NoServiceLocator$.html) as the service locator - this is a service locator that will return nothing for every lookup. We'll see in the [[deploying to production|ProductionOverview]] documentation how to select the right service locator for production.

加载器有两个必须实现的方法，`load` 和 `loadDevMode`。你可以看到，我们在不同的服务定位器中混合了每个方法，我们混合了 [`LagomDevModeComponents`](api/com/lightbend/lagom/scaladsl/devmode/LagomDevModeComponents.html)，它提供了 dev 模式的服务定位器并在 dev 模式和 prod 模式下注册服务，我们简单地提供了 [`NoServiceLocator`](api/com/lightbend/lagom/scaladsl/api/ServiceLocator$$NoServiceLocator$.html) 作为服务定位器—这是一个服务定位器，它不会为每次查找返回任何内容。我们将在[[deploying to production|ProductionOverview]]文档中看到如何为生产选择正确的服务定位器。

A third method, `describeService`, is optional, but may be used by tooling, for example by [Lightbend Orchestration](https://developer.lightbend.com/docs/lightbend-orchestration/current/features/endpoint-detection.html), to discover what service APIs are offered by this service. The metadata read from here may in turn be used to configure service gateways and other components.

第三个方法是可选的，但是可以由工具使用，例如 [Lightbend Orchestration](https://developer.lightbend.com/docs/lightbend-orchestration/current/features/endpoint-detection.html)，以发现这个服务提供了哪些服务 APIs。从这里读取的元数据可以反过来用于配置服务网关和其他组件。

Finally, we need to tell Play about our application loader. We can do that by adding the following configuration to `application.conf`:

最后，我们需要告诉 Play 我们的应用程序加载器。我们可以通过在 `application.conf` 中添加以下配置来实现:

    play.application.loader = com.example.HelloApplicationLoader


## Defining your own components

To create a complex Application (one that uses Persistence, Clustering, the Broker API, etc...) you will need to mix in many Components. It is a good choice to create small custom traits that mix in some of those Components and build the Application by mixing in your small custom traits. That will let you test parts of the complete Application in isolation.

要创建一个复杂的应用程序(使用持久性、集群、代理API等)，您需要混合许多组件。创建混合了这些组件的小型定制特性，并通过混合您的小型定制特性来构建应用程序，这是一个不错的选择。这将使您能够独立地测试整个应用程序的各个部分。

Imagine your Service consumes messages from a Broker topic where `Orders` are notified. Then your service stores that info into a database and does some processing with a final step of invoking a third party endpoint. If you only wanted to test the consuming of messages and proper storage you could create an `OrderConsumingComponent` trait and mix in `LagomServiceClientComponents` and `CassandraPersistenceComponents` so that you could consume the messages and store them. On your test you could extend your `OrderConsumingComponent` with `TestTopicComponents` that provides a mocked up broker so you didn't need to start a broker to run the tests. Finally, on your Application you would mix in the tested `OrderConsumingComponent` and `LagomKafkaClientComponents`.

假设您的服务使用来自通知 `Orders` 的代理主题的消息。然后，您的服务将这些信息存储到数据库中，并执行一些处理，最后一步是调用第三方端点。如果您只想测试消息的使用和适当的存储，那么您可以创建一个 `OrderConsumingComponent` 特性，并混合使用 `LagomServiceClientComponents` 和 `CassandraPersistenceComponents` ，这样您就可以使用消息并存储它们。在您的测试中，您可以使用 `TestTopicComponents` 来扩展您的 `OrderConsumingComponent` ，它提供了一个模拟代理，因此您不需要启动代理来运行测试。最后，在应用程序中混合使用经过测试的 `OrderConsumingComponent` 和 `LagomKafkaClientComponents`。

Lagom provides several Components. For this reference guide we grouped the components by feature:

Lagom 提供了几个组件。为了这个参考指南，我们将组件按特性分组:

 * [[Service Components|ScalaComponents#Service-Components]]
 * [[Persistence and Cluster Components|ScalaComponents#Persistence-and-Cluster-Components]]
 * [[Broker API Components|ScalaComponents#Broker-API-Components]]
 * [[Service Locator Components|ScalaComponents#Service-Locator-Components]]
 * [[Third party Components|ScalaComponents#Third-party-Components]]
