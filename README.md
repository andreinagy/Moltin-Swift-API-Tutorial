# Moltin API Authentication

Moltin provides the most cutting edge payment processor to enable mobile checkout in your store.

The Moltin SDK is the easiest way to integrate payments in your iOS app. Visit  [Swift](https://moltin.com/getting-started/swift) or [Objective C](https://moltin.com/getting-started/objective-c) to get started. Both are open source and you can inspect the source code at any time.

This tutorial describes the authentication using direct connections to the Moltin API. Use this when you don't want to create 3rd party dependencies to your project.
On this premise, no other libraries have been chosen for this tutorial and example project.

Recreating the example project and reviewing all the code is the best way to understand the implementation. However, if at any time you get stuck on the implementation or you feel confident in the basics feel free to download the [example project](#example).

## Create Your Xcode Project

Once you obtained your *client_id* and *client_secret* from Moltin, you can start work on your project:

1. Open **Xcode 8**.
2. Choose **Single view application**
3. Fill in the product name and make sure the language is **Swift**.
4. In the project navigator, choose the target and add the **Security.framework** in the *Linked Frameworks and Libraries* section. This will be required for the keychain token storage later.

### Preview of the View Controller Actions

The application UI will consist of the following buttons:

![Moltin View Controller](moltin-view-controller.png)

* **Auth** will be used to test out the happy path on the authentication code. It will call the server with the correct *client_id* and *client_secret*.
* **Bad Auth** will perform the authentication request with bad credentials. This will be used to demonstrate a failed authentication.
* **Refresh** Token will be an exercise at the end of this tutorial.
* **Delete** Token will delete the stored token after a successful authentication.
* **Products** will perform a products request, so the result of requests can be observed while authenticated or not.

## Overview of the Moltin API

Moltin uses a RESTful API for all services.
The Authorization endpoints use [OAuth 2.0](https://oauth.net/2/) and rely on the **client_id** and **client_secret** pair.

### Obtaining a new token

The auth call should be a POST request made to `https://api.molt.in/oauth/access_token` with the following parameters:
* `grant_type` that can be *implicit*, *client_credentials*, *refresh_token* and *authorization_code*. For iOS apps, the **implicit** value should be used.
* **`client_id`** is the user ID you received when you created your store on the Moltin servers.
* **`client_secret`** is the password/secret that you received when you created your store on the Moltin servers.

![Moltin Auth](moltin-auth-success.gif)

### Refreshing a token

The refresh token call should be a POST request made to `https://api.molt.in/oauth/access_token` with the following parameters:
* `grant_type` **refresh_token**.
* `client_id` is the user ID you received when you created your store on the Moltin servers.
* `client_secret` is the password/secret that you received when you created your store on the Moltin servers.
* `refresh_token` should have the value of the previously expired token.

### Using a valid token

For all future requests, the authorization token should be included in the http headers:

```js
Authorization: Bearer followed_by_the_valid_access_token_XXX
```

To exemplify this concept, we'll use the *products* endpoint.
The list of products can be accessed using a GET request made to ` https://api.molt.in/v1/products`.
This tutorial aims to show that a *400* response is received when the token is not available and a *200* response is received when the header is properly formed.

![Products Success](moltin-products-success.gif)

### Using an expired token

When the token expires, any following API call will receive a result similar to:

```js
{
  "status":false,
  "error":"Access token is not valid"
}
```

![Products Failed](moltin-products-failed.gif)

## Project overview

In the new project, code will be added to help with Swift requests making and response parsing.
Eventually, the essential components of the app should work like in the figure below:

![Application class diagram](moltin-class-diagram.png)

* **`Moltin.swift`** will host the Swift code that provides the API interface. All networking and response handling code should be contained here.
* **`MoltinTokenStorage.swift`** should contain token storage and expiration logic. It's useful to add response parsing to check that the request response is a token expired message.
* **`KeychainWrapper.swift`** will be an iOS keychain wrapper to use for storing the token.
* **`String+PercentEscapeString.swift`** will contain a helper extension to percent escape request parameters.
* **`Dictionary+StringOutput.swift`** will be used to turn parameter dictionaries into a payload string.
* **`ViewController.swift`** is the default project's view controller and will provide the UI interface to `Moltin.swift`

## Add Helper Code

To reduce repetition of code, first create a few helper methods.

### Percent Escaping String

1. Create a new file in Xcode and chose a **Swift File** template.
2. Save it in the project directory as **String+PercentEscapeString**.
3. Fill in the code below or a similar implementation:

```swift
extension String {
    
    /* Percent encode string for HTTP request
    */
    func percentEscapedString() -> String {
        var string = self
        string = string.addingPercentEncoding(withAllowedCharacters: .urlHostAllowed)!
        return string
    }
}
```

The above method ensures the URL encoding of GET requests. This is necessary to replace certain reserved characters with specific string starting with a percent sign. For example:
* the **space** character is replaced by **%20**.
* the **/** character is replaced by **%2F**.
* the **&** character is replaced by **%26**.

### String Output Dictionary

1. Create a new file in Xcode and chose a **Swift File** template.
2. Save it in the project directory as **Dictionary+StringOutput**.
3. Fill in the code below or a similar implementation:

```swift
extension Dictionary {
    
    /* Convenience function to provide http body
    */
    func encodedString() -> String {
        
        let string = self.map {
            if let key = $0 as? String,
                let value = $1 as? String {
                return "\(key.percentEscapedString())=\(value.percentEscapedString())"
            }
            return ""
            }
            .joined(separator: "&")
        
        return string
    }
}
```

The above method works on an ordinary swift dictionary such as `["key1": "value1", "key2": "value2"]` and turns it into a string `"key1=value1&key2=value2"`.

### Keychain Wrapper

I feel it's important to promote the best security practices when the user's data is concerned.
The safest way to store values on iOS is to use the built in encryption systems accessible via the keychain.
For tutorial purposes one could have stored the token in an instance variable only or `UserDefaults`. The solution below is extensible and can be used as a starting point for a custom implementation.

1. Create a new file in Xcode and chose a **Cocoa Touch Class** template.
2. Save it in the project directory as **KeychainWrapper**.
3. Fill in the code below or a similar implementation:

```swift
/* Sample keychain wrapper to store the access token in a secure way on iOS.
 */
class KeychainWrapper {

    static let standard = KeychainWrapper()
    fileprivate static let serviceName = "MoltinKeychainService"
    
    func set(value: String, key: String) -> Bool {
        if self.valueExists(key: key) {
            return self.update(value: value, key: key)
        } else {
            return self.create(value: value, key: key)
        }
    }
    
    func value(key: String) -> String? {
        guard let data = self.data(key: key) else {
            return nil
        }
        return String(data: data, encoding: String.Encoding.utf8)
    }
    
    func removeValue(key:String) -> Bool {
        return self.deleteValue(key: key)
    }
    
    func reset() -> Bool {
        let searchDictionary = self.basicDictionary()
        let status = SecItemDelete(searchDictionary as CFDictionary)
        return status == errSecSuccess
    }
    
    func valueExists(key: String) -> Bool {
        return self.data(key: key) != nil
    }
}

/* Private accessors
 */
extension KeychainWrapper {
    
    fileprivate func create(value: String, key: String) -> Bool {
        var dictionary = self.searchDictionary(key: key)
        
        dictionary[kSecValueData as String] = value.data(using: String.Encoding.utf8, allowLossyConversion: false) as AnyObject?
        
        let status = SecItemAdd(dictionary as CFDictionary, nil)
        return status == errSecSuccess
    }
    
    fileprivate func update(value: String, key: String) -> Bool {
        
        let searchDictionary = self.searchDictionary(key: key)
        var updateDictionary = [String: AnyObject]()
        
        updateDictionary[kSecValueData as String] = value.data(using: String.Encoding.utf8, allowLossyConversion: false) as AnyObject?
        
        let status = SecItemUpdate(searchDictionary as CFDictionary, updateDictionary as CFDictionary)
        
        return status == errSecSuccess
    }
    
    fileprivate func deleteValue(key: String) -> Bool {
        let searchDictionary = self.searchDictionary(key: key)
        let status = SecItemDelete(searchDictionary as CFDictionary)
        
        return status == errSecSuccess
    }
    
    fileprivate func data(key: String) -> Data?  {
        
        var searchDictionary = self.searchDictionary(key: key)
        
        searchDictionary[kSecMatchLimit as String] = kSecMatchLimitOne
        searchDictionary[kSecReturnData as String] = kCFBooleanTrue
        
        var retrievedData: AnyObject?
        let status = SecItemCopyMatching(searchDictionary as CFDictionary, &retrievedData)
        
        var data: Data?
        if status == errSecSuccess {
            data = retrievedData as? Data
        }
        
        return data
    }
    
    fileprivate func searchDictionary(key: String) -> [String: AnyObject] {
        let encodedIdentifier = key.data(using: String.Encoding.utf8, allowLossyConversion: false)
        
        var searchDictionary = self.basicDictionary()
        searchDictionary[kSecAttrGeneric as String] = encodedIdentifier as AnyObject?
        searchDictionary[kSecAttrAccount as String] = encodedIdentifier as AnyObject?
        
        return searchDictionary
    }
    
    fileprivate func basicDictionary() -> [String: AnyObject] {
        return [kSecClass as String : kSecClassGenericPassword, kSecAttrService as String : KeychainWrapper.serviceName as AnyObject]
    }
}
```

The purpose of this class is to create a **MoltinKeychainService** and provide easy access to the iOS keychain API. It isn't concerned with the actual token, and can be used to store any string in the keychain.

## Token Storage

The minimal token storage shown below saves the access token for later use. 
When the token expires, as described in the documentation, an error response is received to any request:

```js
{
  "status":false,
  "error":"Access token is not valid"
}
```

This class is capable of evaluating request responses to determine if the current token is expired.

1. Create a new file in Xcode and chose a **Cocoa Touch Class** template.
2. Save it in the project directory as **MoltinTokenStorage**.
3. Fill in the code below or a similar implementation:

```swift
import UIKit

/* Auth request response JSON keys
 */
enum AuthResponseKeys: String {
    case AccessToken = "access_token"
    case TokenType = "token_type"
    case Identifier = "identifier"
    case ExpiresEpochTime = "expires"
    case ExpiresInterval = "expires_in"
}

/* Any response to any call when the accessToken is not valid anymore
 */
enum TokenInvalidResponseKeys: String {
    case Status = "status"
    case Error = "error"
    case Errors = "errors"
}

class MoltinTokenStorage: NSObject {
    
    static let kAccessTokenKey = "access_token_key"
    
    let keychainWrapper = KeychainWrapper()
    
    /* Property only set through func update(data)
    */
    private(set) var accessToken: String? = nil {
        didSet {
            if let t = self.accessToken {
                let error = self.keychainWrapper.set(value: t, key: MoltinTokenStorage.kAccessTokenKey)
                print("saving access token: \(t)")
                if error {
                    print("error saving access token to the keychain")
                }
            } else {
                let error = self.keychainWrapper.removeValue(key: MoltinTokenStorage.kAccessTokenKey)
                if error {
                    print("error removing access token to the keychain")
                }
            }
        }
    }
    
    override init() {
        self.accessToken = self.keychainWrapper.value(key: MoltinTokenStorage.kAccessTokenKey)
    }
    
    func update(data: Data?) throws {
        if let d = data {
            do {
                if let json = try JSONSerialization.jsonObject(with: d) as? [String: AnyObject] {
                    if let t = json[AuthResponseKeys.AccessToken.rawValue] as? String {
                        self.accessToken = t
                    }
                }
            } catch {
                throw error
            }
        }
    }
    
    func deleteToken() {
        self.accessToken = nil
    }
    
    /* Return the Moltin error message if the token is expired.
     *  An empty message means that the token is valid.
    */
    func authErrorMessage(data: Data?) -> String? {
        var result: String? = nil
        if let d = data {
            do {
                if let json = try JSONSerialization.jsonObject(with: d) as? [String: AnyObject] {
                    if let bool = json[TokenInvalidResponseKeys.Status.rawValue] as? Bool {
                        if bool == false {
                            if let error = json[TokenInvalidResponseKeys.Error.rawValue] as? String {
                                result = error
                            } else if let error = json[TokenInvalidResponseKeys.Errors.rawValue] as? String {
                                result = error
                            } else {
                                result = "Unknown Error"
                            }
                        }
                    }
                }
            } catch {
                result = error.localizedDescription
            }
        }
        return result
    }
}
```

The `TokenStorage` class leverages on the `KeychainWrapper` and adds additional functionality.
The method `func tokenExpiredMessage(data: Data?)` uses `JSONSerialization` to check the response for an authentication error. Tokens are valid for an hour. If the expiry time is reached, an auth or refresh request should be made before any other request.
The largest missing functionality in the above code is the token expiration time handling.

## Moltin API interface

The networking code to interface with the Moltin API can be added to a dedicated class to keep concerns separated.
The *Moltin.swift* file can contain parameter keys and REST paths as constants.
A rundown of methods and properties:
* **`shared`** is the singleton accessor to prevent any racing problems.
* **`tokenStorage`** is the reference to the token storage object previously created.
* **`func auth(...)`** takes a *client_id* and *client_secret* parameter and runs a closure when the operation is completed.
* **`func refreshToken()`** is a placeholder for the exercise.
* **`func getProducts(...)`** uses the stored token if it exists and runs the completion block when the call is completed.

The requests are made using `URLSession` which is the modern way to make HTTP requests on iOS.

1. Create a new file in Xcode and chose a **Cocoa Touch Class** template.
2. Save it in the project directory as **Moltin**
3. Fill in the code below or a similar implementation:

```swift
/* Convenience for HTTP methods
 */
enum HTTPMethodType: String {
    case GET = "GET"
    case POST = "POST"
    case PUT = "PUT"
    case DELETE = "DELETE"
}

/* Access token types
 */
enum GrantType: String {
    case Implicit = "implicit"
    case ClientCredentials = "client_credentials"
    case RefreshToken = "refresh_token"
}

/* Auth parameter keys
 */
enum AuthParameter: String {
    case GrantType = "grant_type"
    case ClientId = "client_id"
    case ClientSecret = "client_secret"
    case RefreshToken = "refresh_token"
}

/* Moltin endpoints
 */
enum MoltinEndpoint: String {
    case Auth = "oauth/access_token"
    case Products = "v1/products"
}

class Moltin: NSObject {
    
    static let kMoltinAPIURLString = "https://api.molt.in/"
    
    static let shared = Moltin()
    
    let tokenStorage = MoltinTokenStorage()
    
    /* Auth call to obtain an access token.
     * On production apps, this should be called if there isn't any saved access token
    */
    func auth(clientId: String,
              clientSecret: String,
              completion: @escaping (_ errorMessage: String?) -> ()) {
        
        let dict = [
            AuthParameter.ClientId.rawValue: clientId,
            AuthParameter.ClientSecret.rawValue: clientSecret,
            AuthParameter.GrantType.rawValue: GrantType.ClientCredentials.rawValue,
            ]
        
        if let url = URL(string: Moltin.kMoltinAPIURLString + MoltinEndpoint.Auth.rawValue) {
            var request = URLRequest(url: url)
            request.httpMethod = HTTPMethodType.POST.rawValue
            
            request.httpBody = dict.encodedString().data(using: String.Encoding.utf8)
            
            let session = URLSession.shared
            
            let task = session.dataTask(with: request) {
                [weak self]
                (data, response, error) in
                print("Auth received response: \(response) \n error: \(error)")
                
                var errorMessage: String? = nil
                
                if let strongSelf = self {
                    
                    /* Try to extract the token from the response.
                    */
                    do {
                        try strongSelf.tokenStorage.update(data: data)
                    } catch {
                        errorMessage = error.localizedDescription
                    }
                    
                    /* Catch authentication errors.
                     */
                    errorMessage = strongSelf.tokenStorage.authErrorMessage(data: data)
                }
                completion(errorMessage)
            }
            
            task.resume()
        }
    }
    
    /* Refresh an existing token
    */
    func refreshToken() {
        print("refreshToken() not implemented")
    }
    
    func deleteToken() {
        self.tokenStorage.deleteToken()
    }
    
    func getProducts(completion: @escaping (_ errorMessage: String?, _ result: [String: AnyObject]?) -> ()) {
        
        let token = self.tokenStorage.accessToken ?? ""
        let dict = [
            "Authorization": "Bearer \(token)",
        ]
        
        if let url = URL(string: Moltin.kMoltinAPIURLString + MoltinEndpoint.Products.rawValue) {
            var request = URLRequest(url: url)
            request.httpMethod = HTTPMethodType.GET.rawValue
            
            for key in dict.keys {
                request.setValue(dict[key], forHTTPHeaderField: key)
            }
            
            let session = URLSession.shared
            
            let task = session.dataTask(with: request) {
                [weak self]
                (data, response, error) in
                print("Products received response: \(response) \n error: \(error)")
                
                var errorMessage: String? = nil
                var result: [String: AnyObject]? = nil
                
                if let strongSelf = self {
                    errorMessage = strongSelf.tokenStorage.authErrorMessage(data: data)
                }
                
                /* Intentionally allowing deserialization to see contents
                */
                do {
                    if let json = try JSONSerialization.jsonObject(with: data!) as? [String: AnyObject] {
                        result = json
                        
                    }
                } catch {
                    errorMessage = error.localizedDescription
                }
                completion(errorMessage, result)
            }
            
            task.resume()
        }
    }
}
```

## View Controller

Now that the API interface is in place, integration in the view controller can be accomplished.
The buttons need to access the Moltin API and check the result of each call in the completion block to see if an `errorMessage` string is present.
Helper methods can be used to make an `UIAlertController` code less duplicated.

1. Open the **ViewController.swift** in the editor
2. Fill in the code below or a similar implementation:

	```swift
class ViewController: UIViewController {
    
    let kClientId = "your_client_id_here"
    let kClientSecret = "your_client_secret_here"
    
    @IBAction func auth(_ sender: Any) {
        Moltin.shared.auth(clientId: kClientId, clientSecret: kClientSecret) {
            [weak self]
            (errorMessage) in
            
            DispatchQueue.main.async {
                if let strongSelf = self {
                    strongSelf.showAlert(errorMessage: errorMessage,
                                         successString: "Authentication suceeded")
                }
            }
        }
    }
    
    @IBAction func badAuth(_ sender: Any) {
        Moltin.shared.auth(clientId: "wrongClientId", clientSecret: kClientSecret) {
            [weak self]
            (errorMessage) in
            
            DispatchQueue.main.async {
                if let strongSelf = self {
                    strongSelf.showAlert(errorMessage: errorMessage,
                                         successString: "Authentication suceeded")
                }
            }
        }
    }
    
    @IBAction func refreshToken(_ sender: Any) {
        
        Moltin.shared.refreshToken()
    }
    
    @IBAction func deleteToken(_ sender: Any) {
        Moltin.shared.deleteToken()
    }
    
    @IBAction func getProducts(_ sender: Any) {
        Moltin.shared.getProducts {
            [weak self]
            (errorMessage, dict) in
            
            DispatchQueue.main.async {
                if let strongSelf = self {
                    strongSelf.showAlert(errorMessage: errorMessage,
                                         successString: "Products list received")
                }
            }
        }
    }
    
    /* Helper to show the error message if exists or a preset success string.
     */
    func showAlert(errorMessage: String?, successString: String) {
        if let string = errorMessage {
            self.showAlert(title: "Error", message: string)
        } else {
            self.showAlert(title: "Success", message: successString)
        }
    }
    
    /* Show an UIAlertViewController
     */
    func showAlert(title: String, message: String) {
        let alert = UIAlertController(title: title, message: message, preferredStyle: .alert)
        let action = UIAlertAction(title: "OK", style: .default, handler: nil)
        alert.addAction(action)
        self.present(alert, animated: true)
    }
}
```
	Replace kClientId with your **client_id** and kClientSecret with your **client_secret**.

3. Open **Interface Builder** and place the buttons: Auth, Bad Auth, Refresh Token, Delete Token, Products.
4. Connect the **`@IBAction`** members to the *Touch Up Inside* events on the buttons. To do this, select the *Connections Inspector* in the *Utilities* panel while the View Controller Scene is selected.

![Connect IBActions](moltin-ibactions.gif)

## Run the project

The project should compile and build without errors.
Run it in the simulator or real device and check the Xcode console for output.

![Moltin Run App](moltin-run-app.gif)

To review the current flows:
* **Auth** makes a call to obtain a token.
* **Bad Auth** makes a call with wrong credentials to demonstrate the failure to obtain a token case.
* **Delete Token** deletes the current token from the keychain.
* **Products** tries to retrieve the list of products using the current token. If the token is invalid, the call fails.

## Take a look at our example project<a name="example"></a>

1. Download the finished example project [MoltinAuth.zip](MoltinAuth.zip)
2. Unpack and open the **MoltinAuth.xcodeproj** in Xcode.
3. Replace your **`kClientId`** and **`kClientSecret`** in `ViewController.swift`.
4. Run the project in the Simulator or real device.

## Exercises
* Implement Refresh Token
* Add token expiration code

## Conclusion
This tutorial demonstrates the interaction with the Moltin API using iOS standard SDK.
Apart from the token refreshing code, this is an implementation that can be used as a merchant app template.
More endpoints can be added straight in the `Moltin.swift` file following the existing authentication example to obtain a complete implementation.

## References
* [Moltin Authentication Reference](https://docs.moltin.com/authenticate/)
* [Apple URLSession reference](https://developer.apple.com/reference/foundation/urlsession)
* [Apple Keychain Services Programming Guide](https://developer.apple.com/library/prerelease/content/documentation/Security/Conceptual/keychainServConcepts/01introduction/introduction.html)
* [OAuth 2 Reference](https://oauth.net/2/)