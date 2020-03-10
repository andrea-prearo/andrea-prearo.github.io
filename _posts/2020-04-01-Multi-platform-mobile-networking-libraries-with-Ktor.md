---
layout: post
title: "Multi-platform mobile networking libraries with Ktor"
date: 2020-03-05
categories: [KMP, Mobile App Development, Kotlin, Unit Tests]
---
In this post I'm going to illustrate how it's possible to leverage [Ktor](https://ktor.io/) to create a shared mobile library that wraps a REST API. The code I'm going to present here can be hosted inside a Kotlin Multi Platform (a.k.a KMP) project and consumed by any Android and iOS app. For the sake of this post I'm going to target a simple and publicly available API: [JSONPlaceholder](https://jsonplaceholder.typicode.com/).


# JSONPlaceholder API Overview

The JSONPlaceholder API is rather simple and provides online fake data. Here are the available resources:
* [/posts](https://jsonplaceholder.typicode.com/posts)
* [/comments](https://jsonplaceholder.typicode.com/comments)
* [/albums](https://jsonplaceholder.typicode.com/albums)
* [/photos](https://jsonplaceholder.typicode.com/photos)
* [/todos](https://jsonplaceholder.typicode.com/todos)
* [/users](https://jsonplaceholder.typicode.com/users)

Each one of the above resources returns a predefined number of JSON objects containing fake data representing an entity corresponding to the resource name (i.e.: posts, comments, ...).


# Wrapping the `/users` resource

To limit the scope of this posts I'm going to show how it's possible to create a wrapper for the `/users` resource. Such a resource returns the most complex data, which uses nested JSON objects, among the resources provided by JSONPlaceholder. All remaining resources can be wrapped the same way and with less effort since they don't return nested objects.


## Creating the `User` model

The `/users` resource returns an array of JSON objects representing fake users. Each user object has the structure shown in the following snippet:

~~~ json
{
    "id": 1,
    "name": "Leanne Graham",
    "username": "Bret",
    "email": "Sincere@april.biz",
    "address": {
        "street": "Kulas Light",
        "suite": "Apt. 556",
        "city": "Gwenborough",
        "zipcode": "92998-3874",
        "geo": {
        "lat": "-37.3159",
        "lng": "81.1496"
        }
    },
    "phone": "1-770-736-8031 x56442",
    "website": "hildegard.org",
    "company": {
        "name": "Romaguera-Crona",
        "catchPhrase": "Multi-layered client-server neural-net",
        "bs": "harness real-time e-markets"
    }
}
~~~

We can take advantage of Kotlin's `@Serializable` annotation to create classes to model each user object returned in the JSON. In order to achieve that we are going to need a top level `User` class and a few nested classes (`Address`, `Company` and `Geolocation`):

~~~ kotlin
@Serializable
data class User(
    val id: Int,
    val name: String,
    val username: String,
    val email: String,
    val address: Address,
    val phone: String,
    val website: String,
    val company: Company
)

@Serializable
data class Address(
    val street: String,
    val suite: String,
    val city: String,
    val zipcode: String,
    val geo: Geolocation
)

@Serializable
data class Geolocation(
    val lat: String,
    val lng: String
)

@Serializable
data class Company(
    val name: String,
    val catchPhrase: String,
    val bs: String
)
~~~


## Retrieving and parsing JSON data

Now that we have our serializable models in place we will create a class that encapsulates the JSONPlaceholder API functionality. The full code is listed below:

~~~ kotlin
enum class Endpoint(val path: String) {
    Users("/users")
}

class Api() {
    private val baseUrl = "https://jsonplaceholder.typicode.com"
    private val client = HttpClient {
        install(JsonFeature) {
            serializer = KotlinxSerializer()
        }
    }

    suspend fun users(): List<User> {
        return client.get {
            setupCall(Endpoint.Users)
        }
    }

    private fun HttpRequestBuilder.setupCall(endpoint: Endpoint) {
        url {
            takeFrom(urlString = baseUrl)
            encodedPath += endpoint.path
        }
    }
}
~~~

The `/users` resource is modeled using an `enum class` to encapsulate its details.

Our workhorse is the `Api` class: It uses Ktor's default [`HttpClient`](https://api.ktor.io/1.3.1/io.ktor.client/-http-client/index.html) which will allows us to make HTTP requests. `HttpClient` is highly configurable: In this particular case we are instructing it to use `KotlinxSerializer` to deserialize any HTTP response:

~~~ kotlin
private val client = HttpClient {
    install(JsonFeature) {
        serializer = KotlinxSerializer()
    }
}
~~~

The `setupCall` helper method allows us to easily build an `url` instance given the desired endpoint:

~~~ kotlin
private fun HttpRequestBuilder.setupCall(endpoint: Endpoint) {
    url {
        takeFrom(urlString = baseUrl)
        encodedPath += endpoint.path
    }
}
~~~

The created `url` instance will be used by Ktor to make an HTTP call:

~~~ kotlin
suspend fun users(): List<User> {
    return client.get {
        setupCall(Endpoint.Users)
    }
}
~~~

Ktor's `HttpClient` will infer `User` as the class for deserializing the HTTP response and will try to parse the JSON response and return the result as a `List<User>` instance.

Each remaining JSONPlaceholder resource (i.e.: posts, comments, ...) can be wrapped using the same approach illustrated above.


# Conclusion

You can find the full code for wrapping JSONPlaceholder [here]().

In this post we examined how it's possible to leverage Ktor to nicely wrap a REST API. In particular we saw how we can use Ktor's `HttpClient` to make HTTP requests and `KotlinxSerializer` to deserialize HTTP responses into Kotlin serializable classes.

In the next post we're going to take a look at how we can create unit tests for HTTP calls.
