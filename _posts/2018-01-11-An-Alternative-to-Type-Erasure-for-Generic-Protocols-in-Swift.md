---
layout: post
title: "An Alternative to Type Erasure for Generic Protocols in Swift"
date: 2018-01-11
categories: [iOS, Mobile App Development, Swift]
---
*I would like to extend a huge thank you to [Aqeel](https://www.linkedin.com/in/aqeel-gunja-8297647) [Gunja](https://twitter.com/aqgunja) for contributing many of the ideas illustrated in this post.*

# Introduction #

Within the flagship Capital One iOS app, we work with many different types of account — card, bank, investment, etc. There are situations where we want to be able to collect all the information about a customer’s accounts. A typical example would be to calculate the total net worth.

In order to complete tasks similar to the above, I have been working quite a bit with *generic protocols* (also known as *protocol with associated types*). A common denominator among the different scenarios for which I used generic protocols, was the need to iterate over a number of objects conforming to the same generic protocol. Unfortunately, as the protocol of interest had an associated type, the task was not as straightforward as I hoped.

This situation is not unique to banking apps. You may run into the same kind of issues by working on any kind of app, any time you want to retrieve data through a common abstraction (protocol) while, at the same time, requiring to enforce type safety on such data through strong typing. Therefore, I thought I would lay out my learnings from the work we are doing at Capital One so other developers could take advantage of them in their apps.

# Use Case Scenario: A Travel Booking App #

To illustrate the issues I experienced by means of a simple use case scenario, let’s suppose we are building a travel booking app that allows a user to book flights, hotels and car rentals. The starting point for this exploration is a protocol with associated type:

~~~ swift
protocol Fetchable {
    associatedtype DataType
    func fetch(completionBlock: @escaping ([DataType]?) -> Void)
}
~~~

The above `Fetchable` protocol abstracts how we retrieve data (network response, managed object collection, etc.). The associated type DataType defines the generic type that will be returned by the fetch method.

The next step is to create a protocol to abstract the common properties of a booking:

~~~ swift
protocol Bookable {
    var identifier: String { get set }
    var startDate: Date { get set }
    var endDate: Date { get set }
}
~~~

The above protocol defines the minimum information required to fully define a booking:

* `identifier`: The unique identifier for the booking.

* `startDate`: Start date for the booking.

* `endDate`: End date for the booking.

Now we should conform to `Bookable` when creating a specific struct for each different type of booking:

~~~ swift
struct FlightBooking: Bookable, Codable {
    // MARK: - Bookable
    var identifier: String
    var startDate: Date
    var endDate: Date

    let flightNumber: String
    let from: String
    let to: String
    let isRoundTrip: Bool

    init(identifier: String, startDate: Date, endDate: Date, flightNumber: String, from: String, to: String, isRoundTrip: Bool) {
        self.identifier = identifier
        self.startDate = startDate
        self.endDate = endDate
        self.flightNumber = flightNumber
        self.from = from
        self.to = to
        self.isRoundTrip = isRoundTrip
    }
}

struct HotelBooking: Bookable, Codable {
    // MARK: - Bookable
    var identifier: String
    var startDate: Date
    var endDate: Date

    let roomNumber: Int

    init(identifier: String, startDate: Date, endDate: Date, roomNumber: Int) {
        self.identifier = identifier
        self.startDate = startDate
        self.endDate = endDate
        self.roomNumber = roomNumber
    }
}

struct RentalBooking: Bookable, Codable {
    // MARK: - Bookable
    var identifier: String
    var startDate: Date
    var endDate: Date

    let model: String
    let make: String

    init(identifier: String, startDate: Date, endDate: Date, model: String, make: String) {
        self.identifier = identifier
        self.startDate = startDate
        self.endDate = endDate
        self.model = model
        self.make = make
    }
}
~~~

For each type of booking we will define an appropriate *fetcher*, responsible for retrieving the required data. In particular, each *fetcher* defines the DataType required for the specific booking type (`FlightBooking`, `HotelBooking`, `RentalBooking`) it is responsible for.

Since it is not relevant to the following discussion, I’ll skip over the data retrieval code and make each *fetcher* just return some mock data.

~~~ swift
struct FlightBookingFetcher: Fetchable {
    typealias DataType = FlightBooking

    func fetch(completionBlock: @escaping ([FlightBooking]?) -> Void) {
        completionBlock([
            FlightBooking(identifier: "VX-XUJURM",
                          startDate: Date.bookingDate(from: "2017-11-12T10:30:00+0000"),
                          endDate: Date.bookingDate(from: "2017-11-16T09:00:00+0000"),
                          flightNumber: "VX-1511",
                          from: "SFO",
                          to: "SEA",
                          isRoundTrip: true)
            ])
    }
}

struct HotelBookingFetcher: Fetchable {
    typealias DataType = HotelBooking

    func fetch(completionBlock: @escaping ([HotelBooking]?) -> Void) {
        completionBlock([
            HotelBooking(identifier: "MC-83027626",
                         startDate: Date.bookingDate(from: "2017-11-12T00:00:00+0000"),
                         endDate: Date.bookingDate(from: "2017-11-16T00:00:00+0000"),
                         roomNumber: 304)
            ])
    }
}

struct RentalBookingFetcher: Fetchable {
    typealias DataType = RentalBooking

    func fetch(completionBlock: @escaping ([RentalBooking]?) -> Void) {
        completionBlock([
            RentalBooking(identifier: "ENT-2856847",
                          startDate: Date.bookingDate(from: "2017-11-12T00:00:00+0000"),
                          endDate: Date.bookingDate(from: "2017-11-16T00:00:00+0000"),
                          model: "Fiesta",
                          make: "Ford")
            ])
    }
}

extension Date {
    static func bookingDate(from string: String) -> Date {
        let dateFormatter = ISO8601DateFormatter()
        return dateFormatter.date(from: string) ?? Date()
    }
}
~~~

<em>
**NOTE**: For the sake of brevity, the `bookingDate` extension method on `Date` is provided to make sure that we always return a non nil value. In production code you should handle a possible nil value according to the specific requirements of your app.
</em>

Now we have all the elements we need to create a `BookingCoordinator` responsible for retrieving all the bookings information:

~~~ swift
enum BookingType {
    case flight
    case hotel
    case rental
}

struct BookingCoordinator {
    public func fetch() {
      // TODO: Retrieve all bookings information.
    }
}

let bookingCoordinator = BookingCoordinator()
bookingCoordinator.fetch()
~~~

# Do Protocols with Associated Types Support Iteration Out-Of-The-Box? #

What do you think a good way to retrieve all the booking information would be? Now, for the sake of this article I’ll pretend I never had to deal with some of the intricacies of protocols with associated types in Swift. Instead, I’ll approach the above task from a point of view that should be applicable to any high level programming language.

Personally, I think a good way to perform the required task would be as follows:

* Create an instance of each booking *fetcher*.

* Add each instance to an array.

* Iterate over each array item to call the generic `fetch` method.

This would allow the code to be clean and easy to understand.

# Attempt #1 #

By structuring our code as described above, we could end up with something like this:

~~~ swift
let fetchers = [FlightBookingFetcher(), HotelBookingFetcher(), RentalBookingFetcher()]
for fetcher in fetchers {
    fetcher.fetch { (bookings) in
        guard let bookings = bookings else {
            return
        }
        print(bookings)
    }
}
~~~

Now, the compiler throws the following error:

~~~ shell
Heterogeneous collection literal could only be inferred to '[Any]'; add explicit type annotation if this is intentional
~~~

# Attempt #2 #

Alright, I guess the compiler doesn’t recognize that all the items in the array conform to the `Fetchable` protocol. Let’s specify the type for the array:

~~~ swift
let fetchers: [Fetchable] = [FlightBookingFetcher(), HotelBookingFetcher(), RentalBookingFetcher()]
~~~

After the above changes, the compiler is now throwing:

~~~ shell
Protocol 'Fetchable' can only be used as a generic constraint because it has Self or associated type requirements
~~~

This error may be familiar to many Swift developers that used [protocol with associated types](http://www.russbishop.net/swift-associated-types). Without delving too much into details, in this particular situation, the error means that the Swift compiler is not able to manage an array of items conforming to the same protocol with associated type. Since the items of the array are of different types, even if each one of them conforms to the same protocol with associated type, the compiler can’t guarantee that we will be calling the right implementation of the generic `fetch` method.

# Attempt #3 #

Since we can’t specify a type that satisfies the compiler, let’s try to work around this by using the `Any` type, which is a valid type for any Swift `class` or `struct`:

~~~ swift
let fetchers: [Any] = [FlightBookingFetcher(), HotelBookingFetcher(), RentalBookingFetcher()]
~~~

This, in a way, works like a simple version of type erasure and the compiler doesn’t complain about the array of *fetcher* items anymore. But this doesn’t take us very far as the compiler is now unable to find a definition for the fetch method:

~~~ shell
Value of type 'Any' has no member 'fetch'
~~~

# Attempt #4 #

Oops! We erased the info about the type and now the compiler can’t find a definition for the `fetch` method. We could instruct the compiler to force cast each array item to the `Fetchable` protocol to enable it to find the definition for the `fetch` method:

~~~ swift
for fetcher in fetchers {
    (fetcher as! Fetchable).fetch { (bookings) in
      [...]
    }
}
~~~

But this doesn’t bring us very far since the compiler is still not able to guarantee that we will be calling the right implementation of the generic `fetch` method:

~~~ shell
Member 'fetch' cannot be used on value of protocol type 'Fetchable'; use a generic constraint instead

Protocol 'Fetchable' can only be used as a generic constraint because it has Self or associated type requirements
~~~

# Attempt #5 #

We could now be tempted to switch from an `Array` to a `Dictionary` to see what happens:

~~~ swift
let fetchers: [BookingType: Fetchable] = [
    .flight: FlightBookingFetcher(),
    .hotel: HotelBookingFetcher(),
    .rental: RentalBookingFetcher()
]
~~~

Unfortunately, this attempt doesn’t help us much as the compiler now throws the same error we saw previously when we tried to use an array of `Fetchable` items:

~~~ shell
Protocol 'Fetchable' can only be used as a generic constraint because it has Self or associated type requirements
~~~

# No, Protocols with Associated Types Don’t Support Iteration Out-Of-The-Box #

At this point, it looks like the only feasible way to fetch all the bookings is to invoke each *fetcher* individually:

~~~ swift
struct BookingCoordinator {
    public func fetch() {
        FlightBookingFetcher().fetch { (flightBookings) in
            guard let flightBookings = flightBookings else {
                return
            }
            print(flightBookings)
        }

        HotelBookingFetcher().fetch { (hotelBookings) in
            guard let hotelBookings = hotelBookings else {
                return
            }
            print(hotelBookings)
        }

        RentalBookingFetcher().fetch { (rentalBookings) in
            guard let rentalBookings = rentalBookings else {
                return
            }
            print(rentalBookings)
        }
    }
}
~~~

This works but:

* It doesn’t scale well.

*It requires a lot of code duplication.

# Can Type Erasure Help Out? #

One common way to work around the issues related to protocol with associated types is [*type erasure*](https://www.bignerdranch.com/blog/breaking-down-type-erasures-in-swift/). I’m not going to explain the technique here as the topic deserves its own post. I’m also not going to illustrate the details of my experimentation with type erasure for this particular scenario. What I am just going to say is that my attempts to leverage type erasure to achieve my initial goal (i.e.: *iterate over an array of items conforming to the same protocol with associated type*) weren’t successful.

# An Alternative Approach to Support Iteration for Strong Typed Protocols: Type Wrapping #

After discussing the above issues with some of my coworkers, we found a way to work around the limitations of protocol with associated types as related to iteration. In the rest of this post, I am going to illustrate how we modified the original code to make it support iteration.

I’ll start by stating that the solution we came up with is not a silver bullet but works rather well for our specific purpose. The proposed solution doesn’t use protocol with associated types; it is instead focused on obtaining a similar result by separating the protocol from its associated type using an approach we called `Type Wrapping`. This approach still manages to provide things like:

* Guarantees on the type association.

* Type safety through strong typing.

Let’s start examining the building blocks of our approach:

~~~ swift
protocol FetchableType {}

protocol Fetchable {
    func fetch(completionBlock: @escaping (FetchableType) -> Void)
}
~~~

As you can see, our main protocol (`Fetchable`) has no associated type anymore. Instead, the `fetch` method has now become generic and requires a completion block that will receive an instance of a type conforming to the new `FetchableType` protocol. In this context, `FetchableType` is used as a placeholder for the type.

In general, `FetchableType` could be any type that we want to be returned by the `fetch` method. In this particular scenario, `FetchableType` will basically wrap the array of items we want to return for each booking type.

The `FlightBooking`, `HotelBooking`, `RentalBooking` classes are unchanged. The *"magic"* happens in the *fetcher* struct. First of all, for each specific fetcher we are going to create a wrapper struct that conforms to the `FetchableType`. This will have the sole purpose of wrapping the array of items we want to return. Here are the wrappers that we will use for each specific fetcher:

~~~ swift
struct FlightBookingsWrapper: FetchableType {
    let bookings: [FlightBooking]?
}

struct HotelBookingsWrapper: FetchableType {
    let bookings: [HotelBooking]?
}

struct RentalBookingsWrapper: FetchableType {
    let bookings: [RentalBooking]?
}
~~~

In the above wrappers, we named the wrapped array `bookings` to provide a uniform name. This simplifies the abstraction, even if it is not strictly needed (we could have named the wrapped array any way we liked). Now, each fetcher can safely return the specific booking information wrapped inside the appropriate struct:

~~~ swift
struct FlightBookingFetcher: Fetchable {
    func fetch(completionBlock: @escaping (FetchableType) -> Void) {
        completionBlock(
            FlightBookingsWrapper(bookings: [
                FlightBooking(identifier: "VX-XUJURM",
                              startDate: Date.bookingDate(from: "2017-11-12T10:30:00+0000"),
                              endDate: Date.bookingDate(from: "2017-11-16T09:00:00+0000"),
                              flightNumber: "VX-1511",
                              from: "SFO",
                              to: "SEA",
                              isRoundTrip: true)
                ]))
    }
}

struct HotelBookingFetcher: Fetchable {
    func fetch(completionBlock: @escaping (FetchableType) -> Void) {
        completionBlock(
            HotelBookingsWrapper(bookings: [
                HotelBooking(identifier: "MC-83027626",
                             startDate: Date.bookingDate(from: "2017-11-12T00:00:00+0000"),
                             endDate: Date.bookingDate(from: "2017-11-16T00:00:00+0000"),
                             roomNumber: 304)
                ]))
    }
}

struct RentalBookingFetcher: Fetchable {
    func fetch(completionBlock: @escaping (FetchableType) -> Void) {
        completionBlock(
            RentalBookingsWrapper(bookings: [
                RentalBooking(identifier: "ENT-2856847",
                              startDate: Date.bookingDate(from: "2017-11-12T00:00:00+0000"),
                              endDate: Date.bookingDate(from: "2017-11-16T00:00:00+0000"),
                              model: "Fiesta",
                              make: "Ford")
                ]))
    }
}
~~~

After wrapping each booking type through the `FetchableType` protocol, our efforts are successful. We are finally able to iterate over the booking fetcher items, which are added to an array of `Fetchable`, and invoke the generic `fetch` method as desired:

~~~ swift
struct BookingCoordinator {
    public func fetch() {
        let fetchers: [Fetchable] = [FlightBookingFetcher(), HotelBookingFetcher(), RentalBookingFetcher()]
        for fetcher in fetchers {
            fetcher.fetch { (bookings) in
                print(bookings)
            }
        }
    }
}
~~~

## Conclusion ##

In this post, I described my personal experience with some of the limitations of generic protocols. In particular, I focused on the issues with iterating over a number of objects conforming to the same protocol with associated type. Then, I illustrated a technique we successfully applied at Capital One to work around such limitations. We called this technique `Type Wrapping` and hope it could be useful for anyone who ever experience the same issues.

You can find code samples that illustrate the issues and the solution discussed in this post on [GitHub](https://github.com/andrea-prearo/SwiftExamples/tree/master/GenericProtocols/TypeWrapping).
