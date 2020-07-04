---
layout: post
title: "Mocking HTTP calls with Ktor MockEngine"
date: 2020-03-06
categories: [KMP, Mobile App Development, Kotlin, Unit Tests]
---
In this post I'm going to illustrate how we can leverage Ktor's [`MockEngine`](https://api.ktor.io/1.3.1/io.ktor.client.engine.mock/-mock-engine/index.html) to create unit tests for HTTP calls. In order to do that, I'm going to build up on the JSONPlaceholder wrapper code from my [previous post](link!).


# Mocking JSONPlaceholder API

As stated in Ktor's online documentation, `MockEngine` provides an [`HttpClientEngine`](https://api.ktor.io/1.3.1/io.ktor.client.engine/-http-client-engine/index.html) for writing tests without network. Basically, it allows us to intercept HTTP calls and return mock responses. This is very useful as it makes writing unit tests for HTTP calls really easy!


## Creating mock responses

The JSONPlaceholder API provides the following resources:
* [/posts](https://jsonplaceholder.typicode.com/posts)
* [/comments](https://jsonplaceholder.typicode.com/comments)
* [/albums](https://jsonplaceholder.typicode.com/albums)
* [/photos](https://jsonplaceholder.typicode.com/photos)
* [/todos](https://jsonplaceholder.typicode.com/todos)
* [/users](https://jsonplaceholder.typicode.com/users)

Therefore, in order to create unit tests for our HTTP calls, we are going to need a mock response for each resource. To simplify the code I'm going to create the mock responses as static objects with the following general structure:

~~~
object {Model}MockResponse {
    operator fun invoke(): String =
        "..." // This contains the mock JSON response for the specific resource.
}
~~~

`{Model}` will be the name of the top level model class that represents the JSON object returned by a specific resource. This means we are going to need the following objects:

~~~ kotlin
object PostsMockResponse { ... }
object CommentsMockResponse { ... }
object AlbumsMockResponse { ... }
object PhotosMockResponse { ... }
object TodosMockResponse { ... }
object UsersMockResponse { ... }
~~~

In order to limit the size of our mock response objects we can set a predefined number of JSON objects that will be returned. For the sake of this post I decided to return only ten objects for each mock response.


## JSONPlaceholder API mock engine

Now that we have our mock responses in place it's time to create a `MockEngine` for JSONPlaceholder. Following the example provided on [Ktor's online documentation](https://ktor.io/clients/http-client/testing.html) we can implement our mock engine for JSONPlaceholder as follows:

~~~ kotlin
class ApiMockEngine {
    fun get() = client.engine

    private val responseHeaders = headersOf("Content-Type" to listOf(ContentType.Application.Json.toString()))
    private val client = HttpClient(MockEngine) {
        engine {
            addHandler { request ->
                if (request.url.encodedPath == "/posts") {
                    respond(PostsMockResponse(), HttpStatusCode.OK, responseHeaders)
                } else if (request.url.encodedPath == "/comments") {
                    respond(CommentsMockResponse(), HttpStatusCode.OK, responseHeaders)
                } else if (request.url.encodedPath == "/albums") {
                    respond(AlbumsMockResponse(), HttpStatusCode.OK, responseHeaders)
                } else if (request.url.encodedPath == "/photos") {
                    respond(PhotosMockResponse(), HttpStatusCode.OK, responseHeaders)
                } else if (request.url.encodedPath == "/todos") {
                    respond(TodosMockResponse(), HttpStatusCode.OK, responseHeaders)
                } else if (request.url.encodedPath == "/users") {
                    respond(UsersMockResponse(), HttpStatusCode.OK, responseHeaders)
                } else {
                    error("Unhandled ${request.url.encodedPath}")
                }
            }
        }
    }
}
~~~

The `get()` method is just a helper to easily access the `engine` property of an `ApiMockEngine` instance.

`responseHeaders` defines a standard `Content-Type: application/json` header to be returned by the mock engine.

The meaty portion is what happens inside the `addHandler` lambda. The general structure is:

~~~
if (request.url.encodedPath == "{resource}") {
    respond({Model}MockResponse(), HttpStatusCode.OK, responseHeaders)
}
~~~

Basically, when `ApiMockEngine` is used to make HTTP calls it will return the specific `{Model}MockResponse` object (as a JSON string) when the URL path corresponds to the associated `{resource}`. An HTTP OK (200) code and a standard `Content-Type: application/json` header will also be part of the response payload.


# Unit tests for HTTP calls

At this point we have everything we need to mock our HTTP calls. Now we just need to be able to leverage the mock engine to write unit tests for our HTTP calls.


## Dependency injection

In the [previous post](link!) we created a class that encapsulates the JSONPlaceholder API functionality. Such a class is instantiating a private instance of [`HttpClient`](https://api.ktor.io/1.3.2/io.ktor.client/-http-client/index.html), which implements a `HttpClientEngine` interface, to make HTTP calls:

~~~ kotlin
class Api() {
    [...]
    private val client = HttpClient {
        install(JsonFeature) {
            serializer = KotlinxSerializer()
        }
    }
    [...]
}
~~~

We can leverage dependency injection to be able to use different implementations of `HttpClientEngine` for specific purposes. In particular we can pass:
* An `HttpClient().engine` value for our production code to make (real) HTTP calls over the network.
* An `ApiMockEngine` instance for mocking HTTP calls in our unit tests.

We can therefore modify the `Api` class as follows:

~~~ kotlin
class Api(httpClientEngine: HttpClientEngine) {
    [...]
    private val client = HttpClient(httpClientEngine) {
        install(JsonFeature) {
            serializer = KotlinxSerializer()
        }
    }
    [...]
}
~~~

We can now instantiate `Api` with the desired engine:
* `Api(HttpClient().engine)` for production code.
* `Api(ApiMockEngine().get())` for unit tests.


## Unit tests in action

The full code for the HTTP calls unit tests is listed below:

~~~ kotlin
class ApiTests {
    private val apiMockEngine = ApiMockEngine()
    private val apiMock = Api(apiMockEngine.get())

    @Test
    fun `test posts`() = runBlockingTest {
        val posts = apiMock.posts()
        assertEquals(10, posts.count())
    }

    @Test
    fun `test albums`() = runBlockingTest {
        val albums = apiMock.albums()
        assertEquals(10, albums.count())
    }

    @Test
    fun `test comments`() = runBlockingTest {
        val comments = apiMock.comments()
        assertEquals(10, comments.count())
    }

    @Test
    fun `test photos`() = runBlockingTest {
        val photos = apiMock.photos()
        assertEquals(10, photos.count())
    }

    @Test
    fun `test todos`() = runBlockingTest {
        val todos = apiMock.todos()
        assertEquals(10, todos.count())
    }

    @Test
    fun `test users`() = runBlockingTest {
        val users = apiMock.users()
        assertEquals(10, users.count())
    }
}
~~~

In the above unit tests we are calling the specific method for each wrapped JSONPlaceholder resource. Since we injected the `ApiMockEngine` in the tests we are getting back the corresponding mock response we created earlier. This allows to write assertions to make sure the wrapping method is correctly retrieving and parsing the returned mock response.

For sake of brevity, in the above tests I'm only checking that the number of parsed objects corresponds to the number of objects defined in each mock response. More assertions can be added as needed to thoroughly check that the parsed objects correspond to the objects provided in the mock response.


# Conclusion

You can find the full code for wrapping JSONPlaceholder [here](https://github.com/andrea-prearo/JSONPlaceholderKotlin).

In this post we examined how it's possible to leverage Ktor's `MockEngine` to create unit tests for HTTP calls. In particular we saw how we can create our custom mock engine to mock HTTP calls and use `HttpClientEngine` to be able to inject it for unit testing.

In the next post we're going to take a look at how we can create an augmented Ktor exception and create unit tests for it.
