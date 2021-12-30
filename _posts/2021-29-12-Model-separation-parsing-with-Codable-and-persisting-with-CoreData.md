---
layout: post
title: "Model separation: parsing with Codable and persisting with Core Data"
date: 2021-12-29
categories: [iOS, Mobile App Development, Swift, Core Data, Codable]
---
It's been a while since my [previous post on Core Data](https://andrea-prearo.github.io/ios/mobile%20app%20development/swift/core%20data/codable/2018/05/07/Working-with-Codable-and-Core-Data.html). And, as often happens, my approach to the problem evolved (hopefully for the better). So, I decided to write about what I changed and why.

## Separating the *domain model* from the *persistence model*

The main change I made over the past two years has been to separate the *domain model* from the *persistence model*. There are a few reasons behind this choice.

First, this helps with having a better separation of concerns. The *domain model* is defined by the API and representend by means of JSON. This means that we can just use barebone structs to represent the entities returned by the API. The *persistence model*, instead, is heavily dependent on the particular persistent storage of choice. In our case the chosen persistent storage is Core Data which, for the purposes of this post, we can consider as an [ORM](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping) layer on top of a SQLite database. Of course, we want to make sure we have a 1:1 relationship between the *domain model* and the *persistence model* representing the same entity.

Among other things, this separation allows us, to some degree, to change each representation independently. For instance, we could change the details of one of entities persisted in Core Data without having to touch the corresponding *domain model* representation. This could be useful, for instance, if we wanted to use ad hoc types other than the primitive ones from JSON (`bool`, `number`, `string`) for our *persistence model*. On the other hand, though, the model representation is usually changed because of the need to comply with some API update.

A second important advantage of model separation is better testability. Keeping the *domain model* separated from the *persistence model* allow us to test the parsing logic independently from the persistence logic.

Let's see how we can apply model separation to the [`User` class](https://gist.github.com/andrea-prearo/5d9aa5c59ed80ef454d121825f4e17d0#file-coredatacodable-9-swift) from my [previous post](https://andrea-prearo.github.io/ios/mobile%20app%20development/swift/core%20data/codable/2018/05/07/Working-with-Codable-and-Core-Data.html).

## *Domain model* handling: *User* and *Codable* ##

Since the *domain model* will be only used to represent API entities, we can implement it as a simple `Codable struct`:

~~~ swift
struct User: Codable, Equatable {
    enum CodingKeys: String, CodingKey {
        case avatarUrl = "avatar"
        case username
        case role
    }

    let avatarUrl: String?
    let username: String?
    let role: String?
}
~~~

## *Persistence model* handling: *UserManagedObject* and *Core Data* ##

Since we'll be using Core Data, the *persistence model* needs to inherit from `NSManagedObject`:

~~~ swift
class UserManagedObject: NSManagedObject {
    @NSManaged var avatarUrl: String?
    @NSManaged var username: String?
    @NSManaged var role: String
}
~~~

Now, since there are only a few valid roles, we could make it easier to handle them by encapsulating them into a `UserRole` enum:

~~~ swift
enum UserRole: String {
    case unknown = "Unknown"
    case admin = "Admin"
    case owner = "Owner"
    case user = "User"

    static func fromString(_ value: String?) -> UserRole {
        guard let value = value else { return .unknown }
        return UserRole(rawValue: value) ?? .unknown
    }
}

class UserManagedObject: NSManagedObject {
    @NSManaged var avatarUrl: String?
    @NSManaged var username: String?
    @NSManaged private var roleValue: String

    var role: UserRole {
        get {
            return UserRole.fromString(roleValue)
        }
        set {
            roleValue = newValue.rawValue
        }
    }
}
~~~

> **NOTE:** There may be different ways to achieve this using Core Data specific functionality. But I personally prefer a more programmatic approach to transforming values.

## Conversion across *domain* and *persistence* models

There are mainly two situations where we need to convert across *domain* and *persistence* models:
* **Retrieving data from the API and persisting it in Core Data**: In this case, we will be creating *domain* objects, as we parse the JSON response content, and then converting them to their corresponding *persistence* objects (`User` -> `UserManagedObject`)
* **Uploading Core Data updates to the backend**: In case we have updated *persistence* objects, we likely need to communicate such updates to the backend through the API (usually by means of a `PUT` call). This requires converting one or more existing *persistence* objects to their corresponding *domain* objects that can be handled by the API (`UserManagedObject` -> `User`).

Since I always find useful to express requirements through an interface, I decided to create a couple of protocols to serve this purpose.

### Converting from *domain* to *persistence* model: *ManagedObjectConvertible*

In order to express the need to be able to convert from the *domain model* to the *persistence model*, we can rely on the *ManagedObjectConvertible* protocol:

~~~ swift
/// Protocol to provide functionality for Core Data managed object conversion.
protocol ManagedObjectConvertible {
    associatedtype ManagedObject

    /// Converts a conforming instance to a managed object instance.
    ///
    /// - Parameter context: The managed object context to use.
    /// - Returns: The converted managed object instance.
    func toManagedObject(in context: NSManagedObjectContext) -> ManagedObject?
}
~~~

Now, to convert a `User` object to its corresponding `UserManagedObject` one, we just need to conform to the `ManagedObjectConvertible` protocol:

~~~ swift
extension User: ManagedObjectConvertible {
    func toManagedObject(in context: NSManagedObjectContext) -> UserManagedObject? {
        let entityName = UserManagedObject.entityName
        guard let entityDescription = NSEntityDescription.entity(forEntityName: entityName, in: context) else {
            NSLog("Can't create entity \(entityName)")
            return nil
        }
        let object = UserManagedObject.init(entity: entityDescription, insertInto: context)
        object.avatarUrl = avatarUrl
        object.username = username
        object.role = UserRole.fromString(role)
        return object
    }
}
~~~

The above code is just creating the appropriate type of Core Data entity and assigning its property values from the current `User` object. Just a quick note about the `context` (`NSManagedObjectContext`) parameter: Since the method is creating a new Core Data object, the `context` we use when calling it should be the appropriate context to be able to write to Core Data: In particular it must support writing to Core Data main context from a background thread.

### Converting from *persistence* to *domain* model: *ModelConvertible*

Similarly to what we saw for the previous conversion, to express the need to be able to convert from the *persistence model* to the *domain model*, we can rely on the *ModelConvertible* protocol:

~~~ swift
/// Protocol to provide functionality for data model conversion.
protocol ModelConvertible {
    associatedtype Model

    /// Converts a conforming instance to a data model instance.
    ///
    /// - Returns: The converted data model instance.
    func toModel() -> Model?
}
~~~

To convert a `UserManagedObject` object to its corresponding `User` one, we just need to conform to the `ModelConvertible` protocol:

~~~ swift
extension UserManagedObject: ModelConvertible {
    // MARK: - ModelConvertible
    /// Converts a UserManagedObject instance to a User instance.
    ///
    /// - Returns: The converted User instance.
    func toModel() -> User? {
        return User(avatarUrl: avatarUrl,
                    username: username,
                    role: role.rawValue)
    }
}
~~~

## Parsing the JSON response and storing users in Core Data

Now that we have fully separated the *domain model* from the *persistence model*, let's see how we can refactor the existing [`UserController`](https://gist.github.com/andrea-prearo/c5bfc26cb927f0e7ad8c46e1cd7bda7b#file-coredatacodable-2-swift) (which is responsible for retriving users data from the API and persisiting it in Core Data). The required changes are rather simple:

~~~ swift
func parse(_ jsonData: Data) -> Bool {
    do {
        [...]

        // Parse JSON data
        let users = try JSONDecoder().decode([User].self, from: jsonData)

        // Update Core Data
        let managedObjectContext = persistentContainer.viewContext
        _ = users.map { $0.toManagedObject(in: managedObjectContext) }
        try managedObjectContext.save()

        return true
    } catch let error {
        print(error)
        return false
    }
}

~~~

As mentioned earlier, we just need to make sure we're using an appropriate context (one that supports writing to Core Data main context from a background thread) when calling the `toManagedObject` method.

## Conclusion ##

In this post, I illustrated my most recent approach in regards to separating the *domain model* from the *persistence model* when working with Core Data. This should allow better separation of concerns and enhanced testability.

The updated code is available on [GitHub](https://github.com/andrea-prearo/SwiftExamples/tree/master/CoreDataCodable).

As briefly touched upon throughout this post, working with Core Data can be tricky. Especially when trying to leverage background threads/queues to interact with it. In the next post I'm going to describe some of the approaches I found useful to deal with some of Core Data rough edges.
