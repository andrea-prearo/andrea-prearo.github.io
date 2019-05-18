---
layout: post
title: "JSON parsing in Swift — Part I: a generic protocol for JSON parsing"
date: 2016-05-17
categories: [iOS, Mobile App Development, Swift, Protocol Oriented, JSON, Parsing]
---
Parsing JSON is a very common task for iOS developers. But the functionality provided out-of-the-box by the Foundation framework is very basic. There are many Open Source libraries available that implement higher level functionality and promise to make this task easier and safer.

As a personal experiment I tried to implement my own JSON parsing library, modeled after a couple of approaches I really like. My implementation is intended to be very minimalistic and focused on a generic protocol based approach that should allow to parse JSON content, and store it in appropriate containers (`class` or `struct` instances), with a minimum amount of code. In order to keep complexity minimal, there will be no particular error handling: in the unfortunate scenario where parsing a specific JSON key fails the corresponding stored value will be nil.
The out-of-the-box solution

Let’s assume we need to interact with a Web Service that returns JSON content structured as follows:

~~~ json
{
    "locations": [{
         "label": "Home",
         "data": {
             "address": "6925 Felicity Coves",
             "city": "East Davin",
             "state": "Washington",
             "country": "USA",
             "zipCode": "22998-1456"
         }
    },{
        "label": "Work",
        "data": {
            "address": "0506 Gretchen River",
            "city": "Huntington Beach",
            "state": "Connecticut",
            "country": "USA",
            "zipCode": "61182-9561"
        }
    }]
}
~~~

This strategy works fine but it is a little tedious. In particular, you need to downcast the content (as optional) specifying the expected type every time you extract the value from a dictionary key.
A generic protocol for parsing JSON: JSONDecodable

Instead of having to manipulate the JSON content as above, I am going to illustrate a very simplistic way to use a more concise syntax to perform the same task. The proposed approach is based on the following steps:
1. Define a protocol to easily parse JSON content into properly designed containers (`class` or `struct` instances).
2. Define the containers that will store data from JSON.
3. Define the mapping between JSON keys and container properties.
4. Apply functional concepts to to simplify the parsing syntax.

The cornerstone of my approach is going to be a generic protocol. I will not go into the details of what a *generic protocol* is in Swift and what class of problems it could be used to solve (maybe in a next article…). Let’s just say that it provides a way to define a protocol that accepts a *generic type*. Or as [Russ Bishop](http://www.russbishop.net/) said in a very clear way:
> An associated type in a protocol says “I don’t know what exact type this is; some concrete class/struct/enum that adopts me will fill in the details”.
>
> Russ Bishop

Here’s the protocol that defines the methods we need to parse a generic JSON content [.1]:

~~~ swift
public protocol JSONDecodable {
    typealias DecodableType // (Swift 2.2)
    // associatedtype DecodableType (Swift 2.3)

    static func decode(json: JSON) -> DecodableType?
    static func decode(json: JSON?) -> DecodableType?
    static func decode(json: [JSON]) -> [DecodableType?]
    static func decode(json: [JSON]?) -> [DecodableType?]
}

public extension JSONDecodable {
    static func decode(json: JSON?) -> DecodableType? {
        guard let json = json else { return nil }
        return decode(json)
    }

    static func decode(json: [JSON]) -> [DecodableType?] {
        return json.map(decode)
    }

    static func decode(json: [JSON]?) -> [DecodableType?] {
        guard let json = json else { return [] }
        return decode(json)
    }
}
~~~

<em>
**NOTE**: as of Swift 2.2, the associated type in a protocol is declared using `typealias`. With Swift 2.3, this will be deprecated in favor of the `associatedtype` keyword.
</em>

The above protocol defines a set of methods that allow us to easily parse JSON content. Note that each decode method returns either an optional or an array of optionals of `DecodableType` type. This is because the parsing may fail (because of an error, a missing value or a misspelled key) and we want to return nil for that specific occurrence. `DecodableType` identifies a generic type (the *associated type*) required by the `JSONDecodable` protocol. It will be replaced, at compile time, by the specific concrete type of the container (`class` or `struct`) that implements the protocol and provides the mapping between the JSON keys and the container properties.

We should be able to parse any kind of JSON content by implementing the four protocol methods defined in the `JSONDecodable` protocol. Three of them have default implementations, provided as a `JSONDecodable` protocol extension. These methods handle standard parsing scenarios and will eventually call the specific `decode` method, provided by the container conforming to the `JSONDecodable` protocol, to parse the content to be stored in its properties:

~~~ swift
static func decode(json: JSON) -> DecodableType?
~~~

The implementation of this method depends on the specific JSON content we need to parse. Basically, each container (`class` or `struct`) will have to describe how to retrieve its property values from the JSON content by providing the mapping between its properties and the keys of the JSON object containing the required data. This will in turn make parsing JSON content a matter of defining some appropriate containers that map 1:1 to the objects represented in the JSON data. By implementing the `decode` method appropriately (as we will see in [Part II](./ios/mobile%20app%20development/swift/functional%20programming/json/parsing/2016/06/01/JSON-parsing-in-Swift.html) of this article), we will be able to instruct the top level container to start the parsing process and make sure that each nested container will in turn continue such process, by means of its specific `decode` method, until we have parsed the entire content.

Now that we have defined the protocol [1.] for parsing JSON content, let’s take a look at the next step: define the containers that will store the parsed data [2.].

# Define containers for storing JSON content #

This step depends on the specific JSON content we want to parse, as the container will have to be modeled appropriately. Keeping in mind the JSON sample shown at the beginning of this article, let’s take a l0ok at the definition of the containers we will need in order to correctly parse it:

~~~ swift
struct Location {
    let label: String?
    let data: LocationData?

    init(label: String?,
         data: LocationData?) {
        self.label = label
        self.data = data
    }
}
~~~

~~~ swift
struct LocationData {
    let address: String?
    let city: String?
    let state: String?
    let country: String?
    let zipCode: String?

    init(address: String?,
         city: String?,
         state: String?,
         country: String?,
         zipCode: String?) {
        self.address = address
        self.city = city
        self.state = state
        self.country = country
        self.zipCode = zipCode
    }
}
~~~

Each container defines appropriate properties for storing the values from the JSON content. Each property is optional since it is possible that the parsing could fail, in which case the stored value will be `nil`.

In order to make the container code cleaner and easier to read, the required method

~~~ swift
static func decode(json: JSON) -> DecodableType?
~~~

will be implemented in a protocol extension (as we’ll see in [Part II](./ios/mobile%20app%20development/swift/functional%20programming/json/parsing/2016/06/01/JSON-parsing-in-Swift.html)).

# Next #

In [Part II](./ios/mobile%20app%20development/swift/functional%20programming/json/parsing/2016/06/01/JSON-parsing-in-Swift.html) we will examine the remaining steps:
* Define the mapping between JSON keys and container properties [3.]
* Apply functional concepts to to simplify the parsing syntax [4.].

# References #

[The Swift Programming Language (Swift 2.2): Generics](https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Generics.html)
[Swift: Associated Types](http://www.russbishop.net/swift-associated-types)
