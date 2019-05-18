---
layout: post
title: "JSON parsing in Swift — Part II: a functional approach to JSON parsing"
date: 2016-06-01
categories: [iOS, Mobile App Development, Swift, Functional Programming, JSON, Parsing]
---
In this second part of the article we will continue the discussion started in [Part I](./ios/mobile%20app%20development/swift/protocol%20oriented/json/parsing/2016/05/17/JSON-parsing-in-Swift.html) and examine the remaining steps to implement the proposed approach to JSON parsing:
* Define the mapping between JSON keys and container properties [3.].
* Apply functional concepts to to simplify the parsing syntax [4.].

Let’s get started!

# Define the mapping between JSON keys and container properties #

In order to make the container code cleaner and easier to read, the required method

~~~ swift
static func decode(json: JSON) -> DecodableType?
~~~

will be implemented in a protocol extension.

Because the mapping between JSON keys and container properties [3.] and the parsing syntax [4.] are tightly coupled, we are going to examine the details for both, at the same time, in each of the following Parsing Syntax Steps.

# JSON Parsing Syntax Step 1: parse<Type> functions #

Before diving into our parsing code, let’s introduce a couple of functions that will help us to improve code readability:

~~~ swift
infix operator >>>= {}

public func >>>= <T,U>(optional : T?, f : T -> U?) -> U? {
    return flatten(optional.map(f))
}

public func flatten<T>(x: T??) -> T? {
    if let y = x { return y }
    return nil
}
~~~

The ``>>>=`` operator takes an optional value and applies a function only if the optional is not `nil`. It uses the flatten function which flattens a nested optional into a single one. We will use this operator to simplify the syntax of our parsing methods.

The basic types we can parse from JSON are:
* `String`
* `Bool`
* `Int`
* `Float`
* `Double`

We can take advantage of the `>>>=` operator to easily implement a set of functions to parse each one of the above types:

~~~ swift
public func parseString(input: JSON, key: String) -> String? {
    return input[key] >>>= { $0 as? String }
}

func parseNumber(input: JSON, key: String) -> NSNumber? {
    return input[key] >>>= { $0 as? NSNumber }
}

public func parseBool(input: JSON, key: String) -> Bool? {
    return parseNumber(input, key: key).map { $0.boolValue }
}

public func parseInt(input: JSON, key: String) -> Int? {
    return parseNumber(input, key: key).map { $0.integerValue }
}

public func parseFloat(input: JSON, key: String) -> Float? {
    return parseNumber(input, key: key).map { $0.floatValue }
}

public func parseDouble(input: JSON, key: String) -> Double? {
    return parseNumber(input, key: key).map { $0.doubleValue }
}
~~~

The `parseNumber` function is used internally to parse numerical types (`Bool`, `Int`, `Float`, `Double`) and, consequently, doesn’t need to be made public. Each of the above functions carries out the same basic task: extracting the value from a JSON key and perform an optional cast to the required type.

With the above functions in place, we can now implement the mapping, and parse the content, as follows:

~~~ swift
extension Location: JSONDecodable {
    static func decode(json: JSON) -> Location? {
        let label = parseString(json, key: "label")
        let data = LocationData.decode(json["data"] as? JSON)
        return Location(label: label,
            data: data)
    }
}
~~~

~~~ swift
extension LocationData: JSONDecodable {
    static func decode(json: JSON) -> LocationData? {
        let address = parseString(json, key: "address")
        let city = parseString(json, key: "city")
        let state = parseString(json, key: "state")
        let country = parseString(json, key: "country")
        let zipCode = parseString(json, key: "zipCode")
        return LocationData(address: address,
            city: city,
            state: state,
            country: country,
            zipCode: zipCode)
    }
}
~~~

In this first step we encapsulated both the type casting and the parsing logic by means a set of specific functions. The code is cleaner than the explicit cast, but we can do better. Let’s make another step towards a cleaner syntax.

# JSON type simplification #

Before we dive into the details of the next parsing step, let’s start with a way to simplify working with JSON. We are going declare a `typealias` to avoid having to cast the parsed JSON content to `[String: AnyObject]` over and over:

~~~ swift
typealias JSON = AnyObject
~~~

This type definition will allow us to extract data from parsed JSON content, as a dictionary, as follows (notice we don’t need the cast to `[String: AnyObject]` anymore):

~~~ swift
let locations = json["locations"]
~~~

# JSON Parsing Syntax Step 2: `JSONParse<T>` binding #

As in the previous step, before diving into our parsing code, let’s introduce a function that will help us to improve code readability:

~~~ swift
infix operator >>> { associativity left precedence 150 }

public func >>><T,U>(a: T?, f: T -> U?) -> U? {
    if let x = a {
        return f(x)
    } else {
        return .None
    }

}
~~~

The `>>>` operator performs two actions sequentially, by passing the result of the first into the second.

> In other languages, the `>>>` operator is also known as **binding operator**.

Now, what if instead of having to invoke a specific parsing function we could let the compiler make the right choice for us, based on the type of the property that will store the parsed value? This would make our code even cleaner!

Well, this is an easy improvement since Swift supports [type inference](https://en.wikipedia.org/wiki/Type_inference). We can achieve this simplification by providing the following functions:

~~~ swift
public func JSONParse<T>(object: JSON?) -> T? {
 return object as? T
}

public func JSONArray(object: JSON?) -> [JSON]? {
 return object as? [JSON]
}
~~~

The first is a generic function that performs an optional cast based on the parameter type. Assuming we declared our container properties correctly (matching the type of the value of the corresponding key in the JSON content), the `JSONParse` function will take care of all the details for us. We just need to pass the specific JSON key as a parameter and it will return the extract the value.

The second function allows us to simplify the parsing of arrays, from the JSON content, by making the downcast transparent.

With the above operator and functions available, we can update the parsing code for our containers as follows:

~~~ swift
extension Location: JSONDecodable {
    static func decode(json: JSON) -> Location? {
        return Location(
            label: json["label"] >>> JSONParse,
            data: json["data"] >>> LocationData.decode)
    }
 }
~~~

~~~ swift
extension LocationData: JSONDecodable {
static func decode(json: JSON) -> LocationData? {
        return LocationData(
            address: json["address"] >>> JSONParse,
            city: json["city"] >>> JSONParse,
            state: json["state"] >>> JSONParse,
            country: json["country"] >>> JSONParse,
            zipCode: json["zipCode"] >>> JSONParse)
    }
}
~~~

In this second step we let the compiler take care of both the type casting and the parsing logic by means of a generic function and a functional operator. The resulting code looks much cleaner than before. But we can do even better!

# JSON Parsing Syntax Step 3: `<|` operator #

In this last step we are going to introduce another couple of functional operators that will make the parsing code even cleaner. Here they are:

~~~ swift
infix operator <| { associativity left precedence 150 }
infix operator <|| { associativity left precedence 150 }

public func <|<T>(json: JSON, key: String) -> T? {
    return json[key] >>> JSONParse
}

public func <||<T>(json: JSON, key: String) -> [T]? {
    return json <| key
}
~~~

The `<|` operator extracts the value for a specific key from a JSON object (both passed as parameters) and passes it to the JSONParse function (using the `>>>` operator) to be optionally casted to the required type.

The `<||` operator simply applies `<|` to extract an array object from a JSON key.

By taking advantage of all the functions and operators we previously defined, the final parsing code for our containers will be:

~~~ swift
extension Location: JSONDecodable {
static func decode(json: JSON) -> Location? {
        return Location(
            label: json <| "label",
            data: json <| "data" >>> LocationData.decode)
    }
}
~~~

~~~ swift
extension LocationData: JSONDecodable {
    static func decode(json: JSON) -> LocationData? {
        return LocationData(
            address: json <| "address",
            city: json <| "city",
            state: json <| "state",
            country: json <| "country",
            zipCode: json <| "zipCode")
    }
}
~~~

At this point, the syntax required to parse JSON is much cleaner than it was in the initial implementation. It is true that it may appear more cryptic and maybe a little *“magical”*. But, in my opinion, this is an acceptable trade-off.

# Conclusion #

Most of what I’ve described above has already been discussed in older posts (see **References**). But, nevertheless, I found it very interesting to implement the steps to improve the parsing syntax myself. My main takeaways from this experiment are:

* A better understanding of Swift compiler limitations
* A deeper knowledge of:
    * Generic methods/functions
    * Operators
    * Generic protocols with associated types

I’ve created a small library, [LiteJSONConvertible](https://github.com/andrea-prearo/LiteJSONConvertible), to make the full code for my exploration of JSON parsing with Swift easily available.

# References #

[Efficient JSON in Swift with Functional Concepts and Generics](https://robots.thoughtbot.com/efficient-json-in-swift-with-functional-concepts-and-generics)
[Real World JSON Parsing with Swift](https://robots.thoughtbot.com/real-world-json-parsing-with-swift)
[Parsing JSON in Swift](http://chris.eidhof.nl/posts/json-parsing-in-swift.html)
