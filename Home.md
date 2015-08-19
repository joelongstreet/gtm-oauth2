# Using the OAuth 2 Controllers #


## Introduction ##

[OAuth 2](http://wiki.oauth.net/w/page/25236487/OAuth-2) is a protocol allowing
your application to obtain authorization to read or modify a user’s files or
data on an external server.

The server generates a web page for the user to sign in with her name and
password, including a button explicitly granting access to some of her data.
Upon successful authentication, the server gives tokens to your application
representing the user's authorization.

With the Objective-C OAuth 2 controllers, the web page can be presented as an
iOS view or a Mac sheet within your application. The controllers also provide an
authentication object that simplifies your application's future requests for the
user's data.

There is also a [gtm-oauth](https://github.com/google/gtm-oauth) library for the
older OAuth 1 protocol. Most applications should use the OAuth 2 library for
services that support OAuth 2.

## Using the iOS and Mac OS X OAuth 2 Controllers ##

The OAuth 2 controllers are useful for authenticating both to Google servers and
to other services.

There are example iOS and Mac applications using the OAuth 2 controllers in the
library's
[Examples directory](https://github.com/google/gtm-oauth2/tree/master/Examples).

Controller methods with "Google" in the name are for use only with Google
services.

## Adding the Controllers to Your Project ##

The project has targets for building a static library for iOS and a framework
for Mac OS X. Alternatively, the source files and xibs may be dragged directly
into your project file and compiled with your application.

The source files required are:

| _iOS and Mac OS X_ | _iOS_ | _Mac OS X_ |
| ------------------ | ----- | ---------- |
| GTMOAuth2Authentication.h/m GTMOAuth2SignIn.h/m GTMHTTPFetcher.h/m | GTMOAuth2ViewControllerTouch.h/m GTMOAuth2ViewTouch.xib | GTMHTTPFetchHistory.h/m GTMOAuth2WindowController.h/m GTMOAuth2Window.xib |

Add the
[standard frameworks](https://developer.apple.com/library/ios/#recipes/xcode_help-project_editor/Articles/AddingaLibrarytoaTarget.html')
`Security.framework` and `SystemConfiguration.framework` to your project.

When linking against a static library build of the controller, specify the
[-ObjC build option](http://developer.apple.com/mac/library/qa/qa2006/qa1490.html')
for the application target's "Other Linker Flags".

### ARC Compatibility ###

When the controller source files are compiled directly into a project that has
ARC enabled, then ARC must be disabled specifically for the controller source
files.

To disable ARC for source files in Xcode 4, select the project and the target in
Xcode. Under the target "Build Phases" tab, expand the Compile Sources build
phase, select the library source files, then press Enter to open an edit field,
and type `-fno-objc-arc`  as the compiler flag for those files.

### System Requirements ###

The Mac controller is compatible with Mac OS X 10.5 and later. The iOS
controller is compatible with iPhone OS 3 and later.

The sample applications use Objective-C blocks, so are intended for iOS 4 and
Mac OS X 10.6.

### Threading and Queues ###

Sign-in calls should be made on the main thread. Authorizing a request is best
done on the main thread as well.

Authorizing a request may be done on a background thread if the thread has a
spinning run loop. Alternatively, a `GTMOAuth2Authentication` object may be
assigned a `GTMHTTPFetcherService` object, where the fetcher service object has
a delegateQueue property indicating the queue to be used for calling back to the
application. Note that delegateQueue support requires iOS 6 or Mac OS X 10.7.

## Signing In to Google Services ##

### Registering Your Application ###

To sign in to Google services with the Objective-C OAuth 2 controllers, first
obtain a client ID and secret from the [API Console](https://console.developers.google.com/).

In the console, create a project, then click "API Access" in the left column.
Create a new Client ID for an **Installed Application** (not a web application)
type **other** (not iOS).  This will provide Client ID and Secret strings to be
used with the controller.

Click "Services" in the left column, and check the list of APIs to verify that
the needed APIs are turned on for the project.  Any Google APIs not listed in
the console do not need to be turned on individually.

### Displaying the Controllers ###

**iOS:** To display a sign-in view, your iOS application makes these calls to
push the view:

```Objective-C
#import "GTMOAuth2ViewControllerTouch.h"

static NSString *const kKeychainItemName = @"OAuth2 Sample: Google+";

NSString *kMyClientID = @"abcd";     // pre-assigned by service
NSString *kMyClientSecret = @"efgh"; // pre-assigned by service

NSString *scope = @"https://www.googleapis.com/auth/plus.me"; // scope for Google+ API

GTMOAuth2ViewControllerTouch *viewController;
viewController = [[[GTMOAuth2ViewControllerTouch alloc] initWithScope:scope
             clientID:kMyClientID
         clientSecret:kMyClientSecret
     keychainItemName:kKeychainItemName
             delegate:self
     finishedSelector:@selector(viewController:finishedWithAuth:error:)] autorelease];

[[self navigationController] pushViewController:controller
                                       animated:YES];
```

Each service will provide developers with **Client ID** and **Client Secret**
strings intended to be hardcoded as constants into applications.  Google
services issue the strings via the [Google API Console](https://console.developers.google.com/).

**Mac:** A Mac application would display sign-in as a sheet on the current
window, like this:

```Objective-C
#import "GTMOAuthWindowController.h"

GTMOAuth2WindowController *windowController;
windowController = [[[GTMOAuth2WindowController alloc] initWithScope:scope
              clientID:kMyClientID
          clientSecret:kMylientSecret
      keychainItemName:kKeychainItemName
        resourceBundle:nil] autorelease];

[controller signInSheetModalForWindow:currentWindow
                             delegate:self
                     finishedSelector:@selector(windowController:finishedWithAuth:error:)];
```

The **scope** is a string identifying what access is being requested. For access
to more than one scope, separate the scopes with a space, or use the convenience
method `scopeWithStrings:` in the class  `GTMOAuth2Authentication`.

The **keychain item name** is used to save the token on the user’s keychain, and
should identify both your application name and the service name(s). If
keychainItemName is nil, the token will not be saved, and the user will have to
sign in again the next time the application is run.

When the user signs in successfully or cancels signing in, the view or window
controller will invoke your finishedSelector’s method:

```Objective-C
- (void)viewController:(GTMOAuth2ViewControllerTouch *)viewController
      finishedWithAuth:(GTMOAuth2Authentication *)auth
                 error:(NSError *)error {
 if (error != nil) {
   // Authentication failed
 } else {
   // Authentication succeeded
 }
}
```

If `[error code]` is `kGTMOAuthErrorWindowClosed` (-1000), then the user closed
the sign-in view before completing authentication. Otherwise, any error reflects
the server response in validating the user's access.

The controllers also support Objective-C block completion handlers as
alternatives to the delegate and finished selectors.

### Using the Authentication Tokens ###

If authentication succeeds, your application should retain the authentication
object. It can be used directly to authorize NSMutableURLRequest objects:

```Objective-C
[auth authorizeRequest:myNSURLMutableRequest
              delegate:self
     didFinishSelector:@selector(authentication:request:finishedWithError:)];
```

The authentication object may need to fetch a refreshed access token before
authorizing the request, so the authorization call is asynchronous, though it
will often authorize the request and call the callback immediately.

If the error passed to the callback is nil, then the request was authorized:

```Objective-C
- (void)authentication:(GTMOAuth2Authentication *)auth
               request:(NSMutableURLRequest *)request
     finishedWithError:(NSError *)error {
 if (error != nil) {
   // Authorization failed
 } else {
   // Authorization succeeded
 }
}
```

Authorization may also be done with a block callback:

```Objective-C
[auth authorizeRequest:myNSURLMutableRequest
     completionHandler:^(NSError *error) {
  if (error == nil) {
    // the request has been authorized
  }
}];
```

With the [GTM HTTP Fetcher](https://github.com/google/gtm-http-fetcher), the
authentication object can be attached to the fetcher, making the authorization
step transparent:

```Objective-C
[fetcher setAuthorizer:auth];
```

Similarly, in the
[Google APIs Client library](https://github.com/google/google-api-objectivec-client)
and the [Google Data APIs library](https://github.com/google/gdata-objectivec-client)
the authentication object can be stored in a library service object to authorize
future API requests:

```Objective-C
[[self contactService] setAuthorizer:auth];
```

**Note:** To avoid sending authorization tokens insecurely, authorization should
be added only to requests with the https scheme. To authorize a request that has
any other URL scheme, the authentication object property
`shouldAuthorizeAllRequests` must first be set to `YES` to override the https
requirement and avoid the error `kGTMOAuth2ErrorUnauthorizableRequest`.

### Retrieving Authorization from the Keychain ###

If your application saves the authorization to the keychain (by setting the
controller's keychainItemName), it can be retrieved the next time the
application launches:

```Objective-C
- (void)awakeFromNib {
  // Get the saved authentication, if any, from the keychain.
  GTMOAuth2Authentication *auth;
  auth = [GTMOAuth2ViewControllerTouch authForGoogleFromKeychainForName:kKeychainItemName
                                                               clientID:kMyClientID
                                                           clientSecret:kMyClientSecret];

  // Retain the authentication object, which holds the auth tokens
  //
  // We can determine later if the auth object contains an access token
  // by calling its -canAuthorize method
  [self setAuthentication:auth];
}
```

If no authorization was saved, then “auth” will still be a valid authorization
object but will be unable to authorize requests:

```Objective-C
BOOL isSignedIn = [auth canAuthorize]; // returns NO if auth cannot authorize requests
```

### Signing Out ###

To completely discard the user’s authorization, use the view or window
controller calls to remove the keychain entry and to ask the Google server to
revoke the token:

```Objective-C
 [GTMOAuth2ViewControllerTouch removeAuthFromKeychainForName:kKeychainItemName];

 [GTMOAuth2ViewControllerTouch revokeTokenForGoogleAuthentication:auth];
```

Finally, release the authorization object.

## Signing in to non-Google Services ##

### Registering Your Application ###

Check the API documentation for the service to determine how to obtain a client
ID and secret.

If the service supports installed applications with the redirect URI
"urn:ietf:wg:oauth:2.0:oob", then register your application as an installed
application.

Otherwise, register as a normal web application, and if the registration
requires that you also specify a redirect URI, supply some **invalid** URL at
your domain. The controllers will look for loading of the redirect URI as an
indication that sign-in has completed, but will not attempt to load the URL.

### Displaying the Controllers ###


To sign in to a non-Google service, you should consult the service's API
documentation to obtain client ID and client secret strings, to find the
required scope string for API operations, and to determine the API endpoints
(URLs) for authorization and for obtaining tokens.

To use the OAuth 2 controllers with services other than Google APIs, your
application should create an authentication object with the appropriate
constants, like this:

```Objective-C
NSString *kMyClientID = @"abcd";     // pre-assigned by service
NSString *kMyClientSecret = @"efgh"; // pre-assigned by service

- (GTMOAuth2Authentication *)myCustomAuth {

  NSURL *tokenURL = [NSURL URLWithString:@"https://api.example.com/oauth/token"];

  // We'll make up an arbitrary redirectURI.  The controller will watch for
  // the server to redirect the web view to this URI, but this URI will not be
  // loaded, so it need not be for any actual web page.
  NSString *redirectURI = @"http://www.google.com/OAuthCallback";

  GTMOAuth2Authentication *auth;
  auth = [GTMOAuth2Authentication authenticationWithServiceProvider:@"Custom Service"
                                                           tokenURL:tokenURL
                                                        redirectURI:redirectURI
                                                           clientID:kClientID
                                                       clientSecret:kClientSecret];
  return auth;
}
```

Displaying the sign-in view with a custom auth also requires providing the OAuth
2 endpoints (URLs) and scope string to the controller, as shown here:

```Objective-C
- (void)signInToCustomService {
  [self signOut];

  GTMOAuth2Authentication *auth = [self authForCustomService];

  // Specify the appropriate scope string, if any, according to the service's API documentation
  auth.scope = @"read";

  NSURL *authURL = [NSURL URLWithString:@"https://api.example.com/oauth/authorize"];

  // Display the authentication view
  GTMOAuth2ViewControllerTouch *viewController;
  viewController = [[[GTMOAuth2ViewControllerTouch alloc] initWithAuthentication:auth
        authorizationURL:authURL
        keychainItemName:kKeychainItemName
                delegate:self
        finishedSelector:@selector(viewController:finishedWithAuth:error:)] autorelease];

  // Now push our sign-in view
  [[self navigationController] pushViewController:viewController animated:YES];
}
```

The keychain item name is used to save the token on the user’s keychain, and
should identify both your application name and the service name(s). If
keychainItemName is nil, the token will not be saved, and the user will have to
sign in again the next time the application is run.

When the user signs in successfully or cancels signing in, the view or window
controller will invoke your finishedSelector’s method:

```Objective-C
- (void)viewController:(GTMOAuthViewControllerTouch *)viewController
      finishedWithAuth:(GTMOAuthAuthentication *)auth
                 error:(NSError *)error {
 if (error != nil) {
   // Sign-in failed
 } else {
   // Sign-in succeeded
 }
}
```

If `[error code]` is `kGTMOAuth2ErrorWindowClosed` (-1000), then the user closed
the sign-in view before completing authorization. Otherwise, any error reflects
the server response in validating the user's access.

The controllers also support Objective-C block completion handlers as
alternatives to the delegate and finished selectors.

### Using the Authentication Tokens ###

If authentication succeeds, your application should retain the authentication
object. It can be used directly to authorize NSMutableURLRequest objects:

```Objective-C
[auth authorizeRequest:myNSURLMutableRequest
              delegate:self
     didFinishSelector:@selector(authentication:request:finishedWithError:)];
```

The authentication object may need to fetch a refreshed access token, so the
authorization call is asynchronous, though it will often authorize the request
and call the callback immediately.

If the error passed to the callback is nil, then the request was authorized:
```Objective-C
- (void)authentication:(GTMOAuth2Authentication *)auth
               request:(NSMutableURLRequest *)request
     finishedWithError:(NSError *)error {
 if (error != nil) {
   // Authorization failed
 } else {
   // Authorization succeeded
 }
}
```

Authorization may also be done with a block callback:
```Objective-C
[auth authorizeRequest:myNSURLMutableRequest
     completionHandler:^(NSError *error) {
  if (error == nil) {
    // the request has been authorized
  }
}];
```

With the [GTM HTTP Fetcher](https://github.com/google/gtm-http-fetcher), the
authentication object can be attached to the fetcher, making the authorization
step transparent:

```Objective-C
[fetcher setAuthorizer:auth];
```

**Note:** To avoid sending authorization tokens insecurely, authorization should
be added only to requests with the https scheme. To authorize a request that has
any other URL scheme, the authentication object property
`shouldAuthorizeAllRequests` must first be set to `YES` to override the https
requirement and avoid the error `kGTMOAuth2ErrorUnauthorizableRequest`.

### Retrieving Authorization from the Keychain ###

If your application saves the authorization to the keychain (by setting the
controller's keychainItemName), it can be retrieved the next time the
application launches:

```Objective-C
- (void)awakeFromNib {
  // Get the saved authentication, if any, from the keychain.
  GTMOAuth2Authentication *auth = [self myCustomAuth];
  if (auth) {
    BOOL didAuth = [GTMOAuth2ViewControllerTouch authorizeFromKeychainForName:@"My App: Custom Service"
                                                               authentication:auth];
  }

  // Retain the authentication object, which holds the auth tokens
  //
  // We can determine later if the auth object contains an access token
  // by calling its -canAuthorize method
  [self setAuthentication:auth];
}
```

If no authorization was saved, then “auth” will still be a valid authorization
object but will be unable to authorize requests:

```Objective-C
BOOL isSignedIn = [auth canAuthorize]; // returns NO if auth cannot authorize requests
```

### Signing Out ###

To completely discard the user’s authorization, use the view or window
controller calls to remove the keychain entry:

```Objective-C
[GTMOAuthViewControllerTouch removeAuthFromKeychainForName:kKeychainItemName];
```

Finally, release the authorization object.

# Design Notes #

The library's classes are designed in three layers.

> *Window/View Controller* :: user interface & application API, including keychain<br>
> *Sign-In* :: OAuth sign-in sequence & WebView handling<br>
> *Authentication* :: data handling, token fetches

Classes are written to be independent of the layers above them.

The window and view controllers are retained only during the user's sign-in
interaction.

The sign-in object is typically not used directly by the client application.

The authentication object must be retained by the client app to authorize
NSMutableURLRequests. It is also used to save authentication to and read
authentication from the keychain.

# Questions and Comments #

You can learn more about the OAuth 2 protocol for desktop and mobile
applications at
[Google's documentation](http://code.google.com/apis/accounts/docs/OAuth2.html).

Additional documentation for the controllers is available in the header files.

If you have any questions or comments about the library or this documentation,
please join the [discussion group](http://groups.google.com/group/gtm-oauth2).
