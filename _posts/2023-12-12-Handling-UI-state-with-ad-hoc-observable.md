---
layout: post
title: "Handling UI state with an ad hoc observable"
date: 2023-12-12
categories: [iOS, Mobile App Development, Swift]
---
In the previous posts I illustrated how it's possible to handle UI state using, respectively, a [Finite-state Machine](https://andrea-prearo.github.io/ios/mobile%20app%20development/swift/2023/10/13/Handling-UI-state-with-finite-state-machines.html) and a [Redux state container](https://andrea-prearo.github.io/ios/mobile%20app%20development/swift/2023/11/25/Handling-UI-state-with-redux.html).

Leveraging either one of the above approaches can help make sure we manage state in a consistent way and, ideally, make the code easier to test and reason about as well. But sometimes the very prescriptive nature of such approaches, or the the extra complexity they introduce, may feel unnecessary.

In this post I'm going to illustrate how we can manage state by means of an ad hoc `Combine` observable. For easy comparison, I'll be presenting the same use case of the previous posts: A basic login screen implemented in Swift.


## Converting finite states to properties

The first step in simplifying how we manage state is to convert our finite states for the login to properties. Representing the login system as a collection of properties, instead of a series of finite states, should allow us to eliminate the complexity of handling state transitions and just focus on the information the state should convey.

These are the finite states we used for the previous implementations:

~~~ swift
case idle
case validatingCredentials
case validCredentials
case authenticating
case authenticated
case failure(LoginError)
~~~

We'll take a look at their purpose and determine how we can convert a finite state into a property that conveys the same information or whether we can get rid of a finite state altogether.

`idle` is the initial finite state. We don't need to convert it since we'll be relying on ad hoc properties to represent the state of the login system as a whole. Similarly, we don't need to convert `validatingCredentials` since its only purpose was to signal that the user started interacting with the login screen.

The `validCredentials` finite state tells us whether the credentials entered by the user are valid, so we should convert it to a property that conveys the same information: A `var hasValidCredentials: Bool` property should suffice.

The purpose of `authenticating` was to signal that the app was trying to authenticate the user. We can conveys the same information using a `var isBusy: Bool` property.

Finally, we want to convert the finite states for success and error: `authenticated` and `failure(LoginError)`. We can easily convert them to: `var isAuthenticated: Bool` and `var error: LoginError?`. Of course, in order to behave properly, the state management code will need to enforce these properties to be mutually exclusive, since having a successful authentication and an error at the same time would be likely incorrect.

To summarize, here are the properties that will convey the same information of the finite states for the login system:

~~~ swift
var hasValidCredentials: Bool
var isBusy: Bool
var isAuthenticated: Bool
var error: LoginError?
~~~

Now let's take a look at how we can represent and manage the state by leveraging the above properties:

~~~ swift
struct LoginState {
    var hasValidCredentials: Bool {
        guard error == nil else {
            return false
        }
        return hasValidUsername && hasValidPassword
    }
    var isBusy = false
    var isAuthenticated = false
    var error: LoginError?

    var username = "" {
        willSet {
            hasValidUsername = validateUsername(newValue)
        }
    }
    var password = "" {
        willSet {
            hasValidPassword = validatePassword(newValue)
        }
    }

    private var hasValidUsername = false
    private var hasValidPassword = false

    private func validateUsername(_ value: String) -> Bool {
        // Way too simple `username` validation
        // for illustration purposes only
        return value.count >= 8
    }

    private func validatePassword(_ value: String) -> Bool {
        // Way too simple `password` validation
        // for illustration purposes only
        return value.count >= 8
    }
}
~~~

~~~ swift
enum LoginError: Error {
    case invalidCredentials
    case networkError(Error)
}
~~~

We already discussed the purpose and behavior of `hasValidCredentials`, `isBusy`, `isAuthenticated` and `error` earlier.

The `username` and `password` properties will be bound to the UI and used to process the user input and trigger the proper validation code, as shown in the code above.


## Leveraging the ad hoc state in the login logic

Now that we've fully defined and implemented our ad hoc state (`LoginState`), the next step is to wrap it inside a view model, which will publish the state as an observable that the screen (view) will leverage to drive the UI behavior.

Here's the full view model code:

~~~ swift
class LoginViewModel: ObservableObject {
    @Published var state = LoginState()

    func authenticate() {
        guard state.hasValidCredentials else { return }
        state.isBusy = true
        // Simulate network call
        // since we're not using a real
        // authentication service
        DispatchQueue.global(qos: .background).asyncAfter(deadline: .now() + 2.0) { [weak self] in
            guard let self else { return }
            DispatchQueue.main.async {
                self.state.isAuthenticated = true
                self.state.isBusy = false
            }
        }
    }

    func ackError() {
        state.error = nil
    }
}
~~~

As you can notice, the view model is pretty straightforward:
* it publishes the ad hoc `state` as a `Combine` observable
* it provides the `authenticate()` and `ackError()` methods to interact with the `state`.

The last step is to wire up the login screen (a SwiftUI view) to the view model.

Here's the full view code:

~~~ swift
struct LoginView: View {
    @ObservedObject private var viewModel: LoginViewModel
    @State private var shouldShowError: Bool = false

    init(viewModel: LoginViewModel) {
        self.viewModel = viewModel
    }

    private func isSignInButtonDisabled() -> Bool {
        guard !viewModel.state.isBusy else { return true }
        return !viewModel.state.hasValidCredentials
    }

    var body: some View {
        VStack {
            GroupBox {
                TextField("Username", text: $viewModel.state.username)
                    .textFieldStyle(.roundedBorder)
                SecureField("Password", text: $viewModel.state.password)
                    .textFieldStyle(.roundedBorder)
                Button("Sign In") {
                    viewModel.authenticate()
                }
                .buttonStyle(DefaultPrimaryButtonStyle(disabled: isSignInButtonDisabled()))
            }
            .padding(.horizontal)
            .fullScreenCover(isPresented: $viewModel.state.isBusy) {
                ProgressView()
                    .background(BackgroundBlurView())
            }
        }
        .alert(isPresented: $shouldShowError) {
            Alert(
                title: Text("Error"),
                message: Text(viewModel.state.error?.localizedDescription ?? "Unknown error"),
                dismissButton: .default(Text("OK")) {
                    viewModel.ackError()
                }
            )
        }
        .onChange(of: viewModel.state.error) { error in
            // Make sure the full screen spinner is
            // dismissed before showing the alert.
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.5) {
                shouldShowError = (error != nil) && !viewModel.state.isBusy
            }
        }
        .padding()
    }
}
~~~

As shown above, the view code is pretty much the same we used in the previous implementations.


## Conclusion

In this post, I illustrated how it's possible to manage state by means of a `Combine` based ad hoc observable. If the complexity of a Redux or FSM based implementation is not your cup of tea, this approach to state management could be a reasonably good alternative.
