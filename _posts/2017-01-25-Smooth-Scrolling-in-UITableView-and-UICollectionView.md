---
layout: post
title: "Smooth Scrolling in UITableView and UICollectionView"
date: 2017-01-25
categories: [iOS, Mobile App Development, Swift]
---
As most iOS developers know, displaying sets of data is a rather common task in building a mobile app. Apple’s SDK provides two components to help carry out such a task without having to implement everything from scratch: A table view ([`UITableView`](https://developer.apple.com/reference/uikit/uitableview)) and a collection view ([`UICollectionView`](https://developer.apple.com/reference/uikit/uicollectionview)).

Table views and collection views are both designed to support displaying sets of data that can be scrolled. However, when displaying a very large amount of data, it could be very tricky to achieve a perfectly smooth scrolling. This is not ideal because it negatively affects the user experience.

As a member of the iOS dev team for the Capital One Mobile app, I’ve had the chance to experiment with table views and collection views; this post reflects my personal experience in displaying large amounts of scrollable data. In it, we’ll review the most important tips to optimize the performance of the above mentioned SDK components. This step is paramount to achieving a very smooth scrolling experience. Note that most of the following points apply to both `UITableView` and `UICollectionView` as they share a good amount of their “under the hood” behavior. A few points are specific to `UICollectionView`, as this view puts additional layout details on the shoulders of the developer.

Let’s begin with a quick overview of the above mentioned components.
`UITableView` is optimized to show views as a sequence of rows. Since the layout is predefined, the SDK component takes care of most of the layout and provides delegates that are mostly focused on displaying cell content.
`UICollectionView`, on the other hand, provides maximum flexibility as the layout is fully customizable. However, flexibility in a collection view comes at the cost of having to take care of additional details regarding how the layout needs to be performed.

## Tips Common to both UITableView and UICollectionView ##

<em>
**NOTE**: I am going to use `UITableView` for my code snippets. But the same concepts apply to `UICollectionView` as well.
</em>

# Cells Rendering is a Critical Task #

The main interaction between `UITableView` and `UITableViewCell` can be described by the following events:

* The table view is requesting the cell that needs to be displayed ([`tableView(_:cellForRowAt:)`](https://developer.apple.com/reference/uikit/uitableviewdatasource/1614861-tableview)).

* The table view is about to display the cell ([`tableView(_:willDisplay:forRowAt:)`](https://developer.apple.com/reference/uikit/uitableviewdelegate/1614883-tableview)).

* The cell has been removed from the table view ([`tableView(_:didEndDisplaying:forRowAt:)`](https://developer.apple.com/reference/uikit/uitableviewdelegate/1614870-tableview)).

For all the above events, the table view is passing the index (row) for which the interaction is taking place. Here’s a visualization of the `UITableViewCell` lifecycle:

![](/assets/2017-01-25-Smooth-Scrolling-in-UITableView-and-UICollectionView/Image1.png)

First off, the [`tableView(_:cellForRowAt:)`](https://developer.apple.com/reference/uikit/uitableviewdatasource/1614861-tableview) method should be as fast as possible. This method is called every time a cell needs to be displayed. The faster it executes, the smoother scrolling the table view will be.

There are a few things we can do in order to make sure we render the cell as fast as possible. The following is the basic code to render a cell, taken from [Apple’s documentation](https://developer.apple.com/library/content/referencelibrary/GettingStarted/DevelopiOSAppsSwift/Lesson7.html#//apple_ref/doc/uid/TP40015214-CH8-SW1):

~~~ swift
override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    // Table view cells are reused and should be dequeued using a cell identifier.
    let cell = tableView.dequeueReusableCell(withIdentifier: "reuseIdentifier", for: indexPath)

    // Configure the cell ...

    return cell
}
~~~

After fetching the cell instance that is about to be reused ([`dequeueReusableCell(withIdentifier:for:)`](https://developer.apple.com/reference/uikit/uitableview/1614878-dequeuereusablecell)), we need to configure it by assigning the required values to its properties. Let’s take a look at how we can make our code execute quickly.

# Define the View Model for the Cells #

One way is to have all the properties we need to show be readily available and just assign those to the proper cell counterpart. In order to achieve this, we can take advantage of the [**MVVM**](https://www.objc.io/issues/13-architecture/mvvm/) pattern. Let’s assume we need to display a set of users in our table view. We could define the Model for the `User` as:

~~~ swift
enum Role: String {
    case Unknown = "Unknown"
    case User = "User"
    case Owner = "Owner"
    case Admin = "Admin"

    static func get(from: String) -> Role {
        if from == User.rawValue {
            return .User
        } else if from == Owner.rawValue {
            return .Owner
        } else if from == Admin.rawValue {
            return .Admin
        }
        return .Unknown
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

Defining a View Model for the `User` is straightforward:

~~~ swift
struct UserViewModel {
    let avatarUrl: String
    let username: String
    let role: Role
    let roleText: String

    init(user: User) {
        // Avatar
        avatarUrl = user.avatarUrl

        // Username
        username = user.username

        // Role
        role = user.role
        roleText = user.role.rawValue
    }
}
~~~

# Fetch Data Asynchronously and Cache View Models #

Now that we have defined our Model and View Model, let’s get them to work! We are going to fetch the data for the users through a web service. Of course, we want to implement the best user experience possible. Therefore, we will take care of the following:

* Avoid blocking the main thread while fetching data.

* Updating the table view right after we retrieve the data.

This means we will be fetching the data asynchronously. We will perform this task through a specific controller, in order to keep the fetching logic separated from both the Model and the View Model, as follows:

~~~ swift
class UserViewModelController {
    fileprivate var viewModels: [UserViewModel?] = []

    func retrieveUsers(_ completionBlock: @escaping (_ success: Bool, _ error: NSError?) -> ()) {
        let urlString = ... // Users Web Service URL
        let session = URLSession.shared

        guard let url = URL(string: urlString) else {
            completionBlock(false, nil)
            return
        }
        let task = session.dataTask(with: url) { [weak self] (data, response, error) in
            guard let strongSelf = self else { return }
            guard let data = data else {
                completionBlock(false, error as NSError?)
                return
            }
            let error = ... // Define a NSError for failed parsing
            if let jsonData = try? JSONSerialization.jsonObject(with: data, options: .allowFragments) as? [[String: AnyObject]] {
                guard let jsonData = jsonData else {
                    completionBlock(false,  error)
                    return
                }
                var users = [User?]()
                for json in jsonData {
                    if let user = UserViewModelController.parse(json) {
                        users.append(user)
                    }
                }

                strongSelf.viewModels = UserViewModelController.initViewModels(users)
                completionBlock(true, nil)
            } else {
                completionBlock(false, error)
            }
        }
        task.resume()
    }

    var viewModelsCount: Int {
        return viewModels.count
    }

    func viewModel(at index: Int) -> UserViewModel? {
        guard index >= 0 && index < viewModelsCount else { return nil }
        return viewModels[index]
    }

}

private extension UserViewModelController {
    static func parse(_ json: [String: AnyObject]) -> User? {
        let avatarUrl = json["avatar"] as? String ?? ""
        let username = json["username"] as? String ?? ""
        let role = json["role"] as? String ?? ""
        return User(avatarUrl: avatarUrl, username: username, role: Role.get(from: role))
    }

    static func initViewModels(_ users: [User?]) -> [UserViewModel?] {
        return users.map { user in
            if let user = user {
                return UserViewModel(user: user)
            } else {
                return nil
            }
        }
    }
}
~~~

Now we can retrieve the data and update the table view asynchronously as shown in the following code snippet:

~~~ swift
class MainViewController: UITableViewController {
    fileprivate let userViewModelController = UserViewModelController()

    override func viewDidLoad() {
        super.viewDidLoad()

        userViewModelController.retrieveUsers { [weak self] (success, error) in
            guard let strongSelf = self else { return }
            if !success {
                DispatchQueue.main.async {
                    let title = "Error"
                    if let error = error {
                        strongSelf.showError(title, message: error.localizedDescription)
                    } else {
                        strongSelf.showError(title, message: NSLocalizedString("Can't retrieve contacts.", comment: "The message displayed when contacts can’t be retrieved."))
                    }
                }
            } else {
                DispatchQueue.main.async {
                    strongSelf.tableView.reloadData()
                }
            }
        }
    }

    [...]
}
~~~

We can use the above snippet to fetch the users data in a few different ways:

* Only the when loading the table view the first time, by placing it in [`viewDidLoad()`](https://developer.apple.com/reference/uikit/uiviewcontroller/1621495-viewdidload).

* Every time the table view is displayed, by placing it in [`viewWillAppear(_:)`](https://developer.apple.com/reference/uikit/uiviewcontroller/1621510-viewwillappear).

* On user demand (for instance via a pull-down-to-refresh), by placing it in the method call that will take care of refreshing the data.

The choice depends on how often the data can be changing on the backend. If the data is mostly static or not changing often the first option is better. Otherwise, we should opt for the second one.

# Load Images Asynchronously and Cache Them #

It’s very common to have to load images for our cells. Since we’re trying to get the best scrolling performance possible, we definitely don’t want to block the main thread to fetch the images. A simple way to avoid that is to load images asynchronously by creating a simple wrapper around [`URLSession`](https://developer.apple.com/library/ios/documentation/Foundation/Reference/NSURLSession_class/):

~~~ swift
extension UIImageView {
    func downloadImageFromUrl(_ url: String, defaultImage: UIImage? = UIImageView.defaultAvatarImage()) {
        guard let url = URL(string: url) else { return }
        URLSession.shared.dataTask(with: url, completionHandler: { [weak self] (data, response, error) -> Void in
            guard let httpURLResponse = response as? NSHTTPURLResponse where httpURLResponse.statusCode == 200,
                let mimeType = response?.mimeType, mimeType.hasPrefix("image"),
                let data = data where error == nil,
                let image = UIImage(data: data)
            else {
                return
            }
        }).resume()
    }
}
~~~

This lets us fetch each image using a background thread and then update the UI once the required data is available. We can improve our performances even further by caching the images.

In case we don’t want - or can’t afford - to write custom asynchronous image downloading and caching ourselves, we can take advantage of libraries such as [SDWebImage](https://github.com/rs/SDWebImage) or [AlamofireImage](https://github.com/Alamofire/AlamofireImage). These libraries provide the functionality we’re looking for out-of-the-box.

# Customize the Cell #

In order to fully take advantage of the cached View Models, we can customize the `User` cell by subclassing it (from `UITableViewCell` for table views and from `UICollectionViewCell` for collection views). The basic approach is to create one outlet for each property of the Model that needs to be shown and initialize it from the View Model:

~~~ swift
class UserCell: UITableViewCell {
    @IBOutlet weak var avatar: UIImageView!
    @IBOutlet weak var username: UILabel!
    @IBOutlet weak var role: UILabel!

    func configure(_ viewModel: UserViewModel) {
        avatar.downloadImageFromUrl(viewModel.avatarUrl)
        username.text = viewModel.username
        role.text = viewModel.roleText
    }
}
~~~

# Use Opaque Layers and Avoid Gradients #

Since using a transparent layer or applying a gradient requires a good amount of computation, if possible, we should avoid using them to improve scrolling performance. In particular, we should avoid changing the *alpha* value and preferably use a standard RGB color (avoid `UIColor.clear`) for the cell and any image it contains:

~~~ swift
class UserCell: UITableViewCell {
    @IBOutlet weak var avatar: UIImageView!
    @IBOutlet weak var username: UILabel!
    @IBOutlet weak var role: UILabel!

    func configure(_ viewModel: UserViewModel) {
        setOpaqueBackground()

        [...]
    }
}

private extension UserCell {
    static let defaultBackgroundColor = UIColor.groupTableViewBackgroundColor

    func setOpaqueBackground() {
        alpha = 1.0
        backgroundColor = UserCell.defaultBackgroundColor
        avatar.alpha = 1.0
        avatar.backgroundColor = UserCell.defaultBackgroundColor
    }
}
~~~

# Putting Everything Together: Optimized Cell Rendering #

At this point, configuring the cell once it’s time to render it should be easy peasy and really fast because:

* We are using the cached View Model data.

* We are fetching the images asynchronously.

Here’s the updated code:

~~~ swift
override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell = tableView.dequeueReusableCell(withIdentifier: "UserCell", for: indexPath) as! UserCell

    if let viewModel = userViewModelController.viewModel(at: (indexPath as NSIndexPath).row) {
        cell.configure(viewModel)
    }

    return cell
}
~~~

## Tips Specific to UITableView ##

# Use Self-Sizing Cells for Cells of Variable Height #

In case the cells we want to display in our table view have variable height, we can use [self sizable cells](http://useyourloaf.com/blog/self-sizing-table-view-cells/). Basically, we should create appropriate Auto Layout constraints to make sure the UI components that have variable height will stretch correctly. Then we just need to initialize the [`estimatedRowHeight`](https://developer.apple.com/reference/uikit/uitableview/1614925-estimatedrowheight) and [`rowHeight`](https://developer.apple.com/reference/uikit/uitableview/1614852-rowheight) property:

~~~ swift
override func viewDidLoad() {
   [...]
   tableView.estimatedRowHeight = ... // Estimated default row height
   tableView.rowHeight = UITableViewAutomaticDimension
}
~~~

<em>
**NOTE**: In the unfortunate case we can’t use self-sizing cells (for instance, if support for iOS7 is still required) we’d have to implement [`tableView(_:heightForRowAt:)`](https://developer.apple.com/reference/uikit/uitableviewdelegate/1614998-tableview) to calculate each cell height. It is still possible, though, to improve scrolling performances by:
</em>

* <em>Pre-calculating all the row heights at once.</em>

* <em>Return the cached value when `tableView(_:heightForRowAt:)` is called.</em>

## Tips Specific to UICollectionView ##

We can easily customize most of our collection view by implementing the appropriate [`UICollectionViewFlowLayoutDelegate`](https://developer.apple.com/reference/uikit/uicollectionviewflowlayout) protocol method.

# Calculate your Cell Size #

We can customize our collection view cell size by implementing [`collectionView(_:layout:sizeForItemAt:)`](https://developer.apple.com/reference/uikit/uicollectionviewdelegateflowlayout/1617708-collectionview):

~~~ swift
@objc(collectionView:layout:sizeForItemAtIndexPath:)
func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, sizeForItemAt indexPath: IndexPath) -> CGSize
    // Calculate the appropriate cell size
    return CGSize(width: ..., height: ...)
}
~~~

# Handle Size Classes and Orientation Changes #

We should make sure to correctly refresh the collection view layout when:

* Transitioning to a different Size Class.

* Rotating the device.

This can be achieved by implementing [`viewWillTransition(to:with:)`](https://developer.apple.com/reference/uikit/uicontentcontainer/1621466-viewwilltransition):

~~~ swift
override func viewWillTransition(to size: CGSize, with coordinator: UIViewControllerTransitionCoordinator) {
    super.viewWillTransition(to: size, with: coordinator)
    collectionView?.collectionViewLayout.invalidateLayout()
}
~~~

# Dynamically Adjust Cell Layout #

In case we need to dynamically adjust the cell layout, we should take care of that by overriding [`apply(_:)`](https://developer.apple.com/reference/uikit/uicollectionreusableview/1620139-apply) in our custom collection view cell (which is a subclass of `UICollectionViewCell`):

~~~ swift
override func apply(_ layoutAttributes: UICollectionViewLayoutAttributes) {
    super.apply(layoutAttributes)

    // Customize the cell layout
    [...]
}
~~~

For instance, one of the common tasks usually performed inside this method is adjusting the maximum width of a multi-line `UILabel`, by programmatically setting its [`preferredMaxLayoutWidth`](https://developer.apple.com/reference/uikit/uilabel/1620534-preferredmaxlayoutwidth) property:

~~~ swift
override func apply(_ layoutAttributes: UICollectionViewLayoutAttributes) {
    super.apply(layoutAttributes)

    // Customize the cell layout
    let width = layoutAttributes.frame.width
    username.preferredMaxLayoutWidth = width - 16
}
~~~

## Conclusion ##

You can find a small sample with the proposed tips for `UITableView` and `UICollectionView` [here](https://github.com/andrea-prearo/SwiftExamples/tree/master/SmoothScrolling/Client).

In this post we examined some common tips to achieve smooth scrolling for both `UITableView` and `UICollectionView`. We also presented some specific tips that apply to each specific collection type. Depending on the specific UI requirements, there could be better or different ways to optimize your collection type. However, the basic principles described in this post still apply. And, as usual, the best way to find out which optimizations work best is to profile your app.
