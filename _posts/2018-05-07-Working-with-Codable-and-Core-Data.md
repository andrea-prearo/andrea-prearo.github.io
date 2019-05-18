---
layout: post
title: "Working with Codable and Core Data"
date: 2018-05-07
categories: [iOS, Mobile App Development, Swift, Core Data, Codable]
---
Recently, I have been working on implementing a caching mechanism for an iOS app. In order to achieve that, I set the following goals:

* Leverage the [`Codable`](https://developer.apple.com/documentation/swift/codable) protocol to easily parse the JSON response from the web service and create the appropriate model instance.

* Store the required model instances in Core Data.

This task has been an interesting learning experience. So, I decided to go back to [one of my sample apps](https://github.com/andrea-prearo/SwiftExamples/tree/master/SmoothScrolling/Client) to illustrate how it is possible to make data models support `Codable` and work with Core Data.

## The model: *NSManagedObject* and *Codable* ##

The sample app I started from has only one simple model, `User`, illustrated below:

~~~ swift
enum Role: String {
    case unknown = "Unknown"
    case user = "User"
    case owner = "Owner"
    case admin = "Admin"

    static func get(from: String) -> Role {
        if from == user.rawValue {
            return .user
        } else if from == owner.rawValue {
            return .owner
        } else if from == admin.rawValue {
            return .admin
        }
        return .unknown
    }
}

struct User {
    let avatarUrl: String
    let username: String
    let role: Role

    init(avatarUrl: String, username: String, role: Role) {
        self.avatarUrl = avatarUrl
        self.username = username
        self.role = role
    }
}
~~~

In order to be able to store instance of `User` in Core Data, a few changes are required.

# Convert from *struct* to *class* #

The first step to make the User model work with Core Data is to make it inherit from [`NSMangedObject`](https://developer.apple.com/documentation/coredata/nsmanagedobject). This requires declaring `User` as a class, instead of a struct, because `NSManagedObject` inherits from `NSObject` (which is the ancestor of most Objective-C classes).

# Use primitive data types #

Using primitive data types makes it easier for model properties to be stored in Core Data. For sake of simplicity, then, we will change `role` to be a `String` type instead of an `enum`.

# Declare properties as *@NSManaged var* #

We need to declare all properties that will be stored in Core Data as `@NSManaged var`. This is required to allow Core Data to correctly access such properties.

# Allow seamless encoding/decoding with Core Data #

To make `Codable` work with Core Data we need to conform to both `Encodable` and `Decodable` in a way that allows to correctly interact with the app [persistent container](https://developer.apple.com/documentation/coredata/nspersistentcontainer). Basically, we need to appropriately implement `Encodable`’s [`encode(to:)`](https://developer.apple.com/documentation/swift/encodable/2893603-encode) and `Decodable`’s [`init(from:)`](https://developer.apple.com/documentation/swift/decodable/2894081-init). In particular, we need to be able to access the persistent *managed object context* and correctly insert each entity (NSManagedObject) representing a User into Core Data (more on this in the `Parsing the JSON response and storing users in Core Data` section below).

The resulting updated code for the `User` model is as follows:

~~~ swift
class User: NSManagedObject, Codable {
    enum CodingKeys: String, CodingKey {
        case avatarUrl = "avatar"
        case username
        case role
    }

    // MARK: - Core Data Managed Object
    @NSManaged var avatarUrl: String?
    @NSManaged var username: String?
    @NSManaged var role: String?

    // MARK: - Decodable
    required convenience init(from decoder: Decoder) throws {
        guard let codingUserInfoKeyManagedObjectContext = CodingUserInfoKey.managedObjectContext,
            let managedObjectContext = decoder.userInfo[codingUserInfoKeyManagedObjectContext] as? NSManagedObjectContext,
            let entity = NSEntityDescription.entity(forEntityName: "User", in: managedObjectContext) else {
            fatalError("Failed to decode User")
        }

        self.init(entity: entity, insertInto: managedObjectContext)

        let container = try decoder.container(keyedBy: CodingKeys.self)
        self.avatarUrl = try container.decodeIfPresent(String.self, forKey: .avatarUrl)
        self.username = try container.decodeIfPresent(String.self, forKey: .username)
        self.role = try container.decodeIfPresent(String.self, forKey: .role)
    }

    // MARK: - Encodable
    public func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(avatarUrl, forKey: .avatarUrl)
        try container.encode(username, forKey: .username)
        try container.encode(role, forKey: .role)
    }
}
~~~

## The controller: Parsing JSON responses ##

Now that our updated `User` model is ready, let’s look into how we can parse the JSON response from the web service.

# Accessing the *managed object context* #

In order to use Core Data to store our `User` instances we need to be able to access the persistent container and, in particular, the *managed object context* ([`viewContext`](https://developer.apple.com/documentation/coredata/nspersistentcontainer/1640622-viewcontext)) from the `Decodable` initializer implementation inside the model:

~~~ swift
// MARK: - Decodable
required convenience init(from decoder: Decoder) throws {
    // We need to access the managed object context (viewContext) of the persistence container here!
}
~~~

Since we can’t change the signature of the above method, and explicitly pass the required *managed object context* as a parameter, we have to find an alternative way to make the context available.

One way to achieve that is to store the context in the custom dictionary [`userInfo`](https://developer.apple.com/documentation/swift/decoder/2892930-userinfo) property of the `Decoder` instance. The `userInfo` requires a key of [`CodingUserInfoKey`](https://developer.apple.com/documentation/swift/codinguserinfokey) type to store the contextual information. To make things easier we will provide a `CodingUserInfoKey` extension that conveniently wraps the key name:

~~~ swift
public extension CodingUserInfoKey {
    // Helper property to retrieve the context
    static let managedObjectContext = CodingUserInfoKey(rawValue: "managedObjectContext")
}
~~~

Now, we can easily refer to the key reserved to store the *managed object context* as `CodingUserInfoKey.managedObjectContext`. Because the `CodingUserInfoKey` initializer returns an optional, though, we should always make sure to access our `CodingUserInfoKey.managedObjectContext` extension in a safe way and avoid using forced unwrapping:

~~~ swift
guard let codingUserInfoKeyManagedObjectContext = CodingUserInfoKey.managedObjectContext else {
    fatalError("Failed to retrieve managed object context")
}
~~~

# Parsing the JSON response and storing users in Core Data #

The JSON parsing method is part of a controller, `UserController`, that will take care of all the logic required for fetching the data representing our users from both the network and Core Data. Here’s the relevant parsing code:

~~~ swift
class UserController: UserControllerProtocol {
    [...]

    private let persistentContainer: NSPersistentContainer

    [...]

    init(persistentContainer: NSPersistentContainer) {
        self.persistentContainer = persistentContainer
    }

    [...]
}

private extension UserController {
    func parse(_ jsonData: Data) -> Bool {
        do {
            guard let codingUserInfoKeyManagedObjectContext = CodingUserInfoKey.managedObjectContext else {
                fatalError("Failed to retrieve context")
            }

            // Clear storage and save managed object instances
            if currentPage == 0 {
                clearStorage()
            }

            // Parse JSON data
            let managedObjectContext = persistentContainer.viewContext
            let decoder = JSONDecoder()
            decoder.userInfo[codingUserInfoKeyManagedObjectContext] = managedObjectContext
            _ = try decoder.decode([User].self, from: jsonData)
            try managedObjectContext.save()

            return true
        } catch let error {
            print(error)
            return false
        }
    }

    [...]
}
~~~

Let’s step through the salient points of the above code.

The `UserController` requires a `NSPersistentContainer` instance to be initialized. This will ensure we can access Core Data persistent container and its *managed object context* when needed.

As usual, when using `Codable`, we create a `JSONDecoder` instance inside the `parse` method to parse the JSON response. Before the actual parsing, though, we store the *managed object context* in the decoder `userInfo` dictionary using `CodingUserInfoKey.managedObjectContext` as the key:

~~~ swift
let managedObjectContext = persistentContainer.viewContext
let decoder = JSONDecoder()
decoder.userInfo[codingUserInfoKeyManagedObjectContext] = managedObjectContext
~~~

The *managed object context* instance we just stored in `userInfo` will be used by the `User` class while performing its decoding task (as described in the `Allow seamless encoding/decoding with Core Data` section above). This is the first main difference in having to deal with `Codable` and `NSManagedObject`, compared to what we usually do when working with `Codable` alone.

Next, we proceed to parse the JSON response to retrieve our `User` instances as usual:

~~~ swift
decoder.decode([User].self, from: jsonData)
~~~

While decoding the JSON response, the `User` class will take care of correctly initializing the values of its `@NSManaged var` properties and make sure that the new `User` instances are correctly inserted in Core Data:

~~~ swift
class User: NSManagedObject, Codable {
    [...]

    // MARK: - Core Data Managed Object
    @NSManaged var avatarUrl: String?
    @NSManaged var username: String?
    @NSManaged var role: String?

    // MARK: - Decodable
    required convenience init(from decoder: Decoder) throws {
        guard let codingUserInfoKeyManagedObjectContext = CodingUserInfoKey.managedObjectContext,
            let managedObjectContext = decoder.userInfo[codingUserInfoKeyManagedObjectContext] as? NSManagedObjectContext,
            let entity = NSEntityDescription.entity(forEntityName: "User", in: managedObjectContext) else {
            fatalError("Failed to decode User")
        }

        self.init(entity: entity, insertInto: managedObjectContext)

        let container = try decoder.container(keyedBy: CodingKeys.self)
        self.avatarUrl = try container.decodeIfPresent(String.self, forKey: .avatarUrl)
        self.username = try container.decodeIfPresent(String.self, forKey: .username)
        self.role = try container.decodeIfPresent(String.self, forKey: .role)
    }

    [...]
}
~~~

Now, to finalize our parsing task, we need to make sure that our newly inserted `User` instances are saved into the *managed object context* to be persisted:

~~~ swift
try managedObjectContext.save()
~~~

This is the second, and last, main difference compared to what we usually do when working with `Codable` alone. Once the `parse` method is successfully executed all the `User` instances retrieved from the JSON response will have been saved and will be accessible in our Core Data persistent storage.

## Core Data as a caching mechanism ##

This new sample app requires only one model, which makes the database structure embarrassingly simple:

![](/assets/2018-05-07-Working-with-Codable-and-Core-Data/Image1.png)

![](/assets/2018-05-07-Working-with-Codable-and-Core-Data/Image2.png)

The use case for Core Data is rather simple: To allow the app to be used offline (i.e.: when network connectivity is not available). The simplest way to achieve this is to delete, and re-create, Core Data database every time the app has network connection. This is why in the `parse(…)` method we call `clearStorage()` before actually parsing the JSON response: We want to clear the storage (database) before we start adding the parsed `User` instances. The code required to clear the storage is rather simple, as we just need to delete one table:

~~~ swift
private extension UserController {
    [...]

    func clearStorage() {
        let managedObjectContext = persistentContainer.viewContext
        let fetchRequest = NSFetchRequest<NSFetchRequestResult>(entityName: UserController.entityName)
        let batchDeleteRequest = NSBatchDeleteRequest(fetchRequest: fetchRequest)
        do {
            try managedObjectContext.execute(batchDeleteRequest)
        } catch let error as NSError {
            print(error)
        }
    }

    [...]
}
~~~

In order to retrieve the stored `User` instances, the `UserController` provides the `fetchFromStorage()` method:

~~~ swift
private extension UserController {
    [...]

    func fetchFromStorage() -> [User]? {
        let managedObjectContext = persistentContainer.viewContext
        let fetchRequest = NSFetchRequest<User>(entityName: UserController.entityName)
        let sortDescriptor1 = NSSortDescriptor(key: "role", ascending: true)
        let sortDescriptor2 = NSSortDescriptor(key: "username", ascending: true)
        fetchRequest.sortDescriptors = [sortDescriptor1, sortDescriptor2]
        do {
            let users = try managedObjectContext.fetch(fetchRequest)
            return users
        } catch let error {
            print(error)
            return nil
        }
    }

    [...]
}
~~~

Both methods perform their respective task by means of a [`NSFetchRequest`](https://developer.apple.com/documentation/coredata/nsfetchrequest). With the two above methods implemented, we now have everything we need to successfully interact with Core Data and our User instances.

## Conclusion ##

In this post, I described my personal experience working with `Codable` and Core Data. In particular, I focused on how to seamlessly parse JSON responses and store the resulting models in the appropriate database table in Core Data.

The code for the sample app illustrated in this post is available on [GitHub](https://github.com/andrea-prearo/SwiftExamples/tree/master/CoreDataCodable).
