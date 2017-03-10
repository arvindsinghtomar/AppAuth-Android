# AppAuth Android

## Overview
AppAuth for Android is a client SDK for communicating with [OAuth 2.0]
(https://tools.ietf.org/html/rfc6749) and [OpenID Connect]
(http://openid.net/specs/openid-connect-core-1_0.html) providers. It 
strives to directly map the requests and responses of those specifications, 
while following the idiomatic style of the implementation language. In 
addition to mapping the raw protocol flows, convenience methods are 
available to assist with common tasks like performing an action with 
fresh tokens.

The library follows the best practices set out in [OAuth 2.0 for Native Apps]
(https://tools.ietf.org/html/draft-ietf-oauth-native-apps)
including using
[Custom Tabs](http://developer.android.com/tools/support-library/features.html#custom-tabs)
for the auth request. For this reason,
`WebView` is explicitly *not* supported due to usability and security 
reasons.

The library also supports the [PKCE](https://tools.ietf.org/html/rfc7636)
extension to OAuth which was created to secure authorization codes in 
public clients when custom URI scheme redirects are used. The library is 
friendly to other extensions (standard or otherwise) with the ability to 
handle additional parameters in all protocol requests and responses.

## Specification
### Supported Android Versions
AppAuth supports Android API 16 (Jellybean) and above.

When a Custom Tabs implementation is provided by a browser on 
the device (for example by [Chrome](https://developer.chrome.com/multidevice/android/customtabs)), 
Custom Tabs are used for authorization requests. Otherwise, 
the default browser is used as a fallback.

### Authorization Server Support

Both Custom URI Schemes (all supported versions of Android) and App Links
(API 23+) can be used with the library.

In general, AppAuth can work with any Authorization Server (AS) that supports
[native apps](https://tools.ietf.org/html/draft-ietf-oauth-native-apps),
either through custom URI scheme redirects, or App Links.
AS's that assume all clients are web-based or require clients to maintain
confidentiality of the client secrets may not work well.

## Configure the Demo App
Authorization services can be added to the demo app by defining
instances of `IdentityProvider`. Assuming a service named 
`myprovider`, the following steps need to be taken:
 
1. The name of the service should be defined in `myprovider_name` 
in `idp_configs_optional.xml`.

2. If the service supports OpenID Connect, 
`myprovider_discovery_uri` would be defined in 
`idp_configs_optional.xml` and set to the discovery URI for the 
service (e.g. `https://www.myprovider.com/.well-known/openid-configuration`).
    
    Otherwise, `myprovider_auth_endpoint_uri` and 
    `myprovider_token_endpoint_uri` would be defined in 
    `idp_configs_optional.xml` and set to the authorization and 
    token endpoint URIs respectively. 

3. The default scope string, `myprovider_scope_string`, should be 
defined in `idp_configs_optional.xml`.

4. A placeholder for the client ID, `myprovider_client_id`, should 
be defined in `idp_configs.xml`.

5. The redirect URI, `myprovider_redirect_uri`, can be defined in 
`idp_configs.xml`.

6. An on-off toggle, `myprovider_enabled`, should be defined in 
`idp_configs.xml` and set to false by default.

7. Button resources representing the IDP should be imported into 
the relevant directories under res.

This may result in an addition to `idp_configs.xml` that looks 
like

```
<bool name="myprovider_enabled">true</bool>
<string name="myprovider_client_id" translatable="false">YOUR_CLIENT_ID</string>
<string name="myprovider_redirect_uri" translatable="false">https://demo.myprovider.com/callback</string>
```

And an addition to `idp_configs_optional.xml` that looks like:

```
<string name="myprovider_name">MyProvider</string>
<string name="myprovider_scope_string">profile payment location</string>
<string name="myprovider_auth_endpoint_uri">https://www.myprovider.com/auth</string>
<string name="myprovider_token_endpoint_uri">https://www.myprovider.com/token</string>
<string name="myprovider_discovery_uri">https://www.myprovider.com/.well-known/openid-configuration</string>
```

With these properties defined, a new instance of IdentityProvider 
can be defined in `IdentityProvider`:

```
public static final IdentityProvider MYPROVIDER = new IdentityProvider(
    "MyProvider", // name of the provider, for debug strings
    R.bool.myprovider_enabled,
    NOT_SPECIFIED, // discovery document not provided
    R.string.myprovider_auth_endpoint_uri,
    R.string.myprovider_token_endpoint_uri,
    NOT_SPECIFIED, // dynamic registration not supported
    R.string.myprovider_client_id,
    R.string.myprovider_auth_redirect_uri,
    R.string.myprovider_scope_string,
    R.drawable.btn_myprovider, // your button image asset
    R.string.myprovider_name,
    android.R.color.black // text color on the button
);
```

And add to `IdentityProvider`'s static list of IDPs, e.g.:

```
public static final List<IdentityProvider> PROVIDERS = Arrays.asList(MYPROVIDER)
```

Finally you need to add a new intent-filter to the 
`net.openid.appauth.RedirectUriReceiverActivity` activity section 
of the `AndroidManifest.xml`

```
<!-- Callback from authentication screen -->
<activity android:name="net.openid.appauth.RedirectUriReceiverActivity">
   
!-- redirect URI for your new IDP -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <category android:name="android.intent.category.BROWSABLE"/>
        <data android:scheme="@string/your_idp_auth_redirect_scheme"/>
    </intent-filter>
</activity>
```

Make sure you've set `myauth_enabled` to true in the config, 
and your new IdP should show up in the list.

### Use of state param to send custom data
You can send custom data in Authorization request by 
performing following changes in `MainActivity.java` class of demo app.

```java
AuthorizationRequest authRequest = new AuthorizationRequest.Builder(
                serviceConfig,
                idp.getClientId(),
                ResponseTypeValues.CODE,
                idp.getRedirectUri(), state)
                .setScope(idp.getScope())
                .setLoginHint(loginHint)
                .build();
```

here `state` variable contains custom data which you need to send.

For example, I have generated random string to be send as custom value 
 
```java
String state = generateRandomState();
``` 

Here is the [MainActivity.java](https://github.com/arvindsinghtomar/AppAuth-Android/blob/master/app/java/net/openid/appauthdemo/MainActivity.java) file which 
you should over-ride in your app to use state param to send custom data.


## Auth Flow

AppAuth supports both manual interaction with the Authorization 
Server where you need to perform your own token exchanges, as well
as convenience methods that perform some of this logic for you. 
This example uses the convenience method which returns 
either an `AuthState` object, or an error.

### Tracking authorization state

`AuthState` is a class that keeps track of the authorization and 
token requests and responses, and provides a convenience method to
call an API with fresh tokens. This is the only object that you 
need to serialize to retain the authorization state of the 
session.Typically, one would do this by storing the authorization 
state in SharedPreferences or some other persistent store private 
to the app:

```java
@NonNull public AuthState readAuthState() {
  SharedPreferences authPrefs = getSharedPreferences("auth", MODE_PRIVATE);
  String stateJson = authPrefs.getString("stateJson");
  AuthState state;
  if (stateStr != null) {
    return AuthState.fromJsonString(stateJson);
  } else {
    return new AuthState();
  }
}

public void writeAuthState(@NonNull AuthState state) {
  SharedPreferences authPrefs = getSharedPreferences("auth", MODE_PRIVATE);
  authPrefs.edit()
      .putString("stateJson", state.toJsonString())
      .apply();
}
```

### Configuration

You can configure communication with your chosen authorization 
server by specifying the endpoints directly:

```java
AuthorizationServiceConfiguration config =
        new AuthorizationServiceConfiguration(name, mAuthEndpoint, mTokenEndpoint);

// perform the auth request...
```

Or through discovery:

```java
final Uri issuerUri = Uri.parse("https://accounts.google.com");
AuthorizationServiceConfiguration config;

AuthorizationServiceConfiguration.fetchFromIssuer(
    issuerUri,
    new RetrieveConfigurationCallback() {
      @Override public void onFetchConfigurationCompleted(
          @Nullable AuthorizationServiceConfiguration serviceConfiguration,
          @Nullable AuthorizationException ex) {
        if (ex != null) {
            Log.w(TAG, "Failed to retrieve configuration for " + issuerUri, ex);
        } else {
            // service configuration retrieved, proceed to authorization...
        }
      }
  });
```

Additionally, AppAuth provides additional client configuration, 
in order to control the browsers that can be used for the 
authorization flow, or to control the HttpURLConnection 
implementation that is used for token exchange. This can be done 
by creating an `AppAuthConfiguration` object:

```java
AppAuthConfiguration appAuthConfig = new AppAuthConfiguration.Builder()
    .setBrowserMatcher(new BrowserWhitelist(
        VersionedBrowserMatcher.CHROME_CUSTOM_TAB,
        VersionedBrowserMatcher.SAMSUNG_CUSTOM_TAB))
    .setConnectionBuilder(new ConnectionBuilder() {
      public HttpURLConnection openConnect(Uri uri) throws IOException {
        URL url = new URL(uri.toString());
        HttpURLConnection connection =
            (HttpURLConnection) url.openConnection();
        if (connection instanceof HttpsUrlConnection) {
          HttpsURLConnection connection = (HttpsURLConnection) connection;
          connection.setSSLSocketFactory(MySocketFactory.getInstance());
        }
      }
    })
    .build();
```

This configuration will only permit Chrome or Samsung Browser to 
be used as a custom tab for authorization flows, and changes the 
SSL socket factory to a hypothetical custom version 
`MySocketFactory`. The modification of the socket factory is 
useful for certificate pinning, or adding some additional
certificates to the trusted set (such as for testing).

This configuration is provided to `AuthorizationService` upon 
construction:

```java
new AuthorizationService(context, appAuthConfig);
```

### Dynamic client registration(Optional)

If the server supports dynamic client registration, a new registration
request can be constructed for dispatch:

```java
RegistrationRequest registrationRequest = new RegistrationRequest.Builder(
    serviceConfig,
    Arrays.asList(redirectUri))
    .build();
```

Requests are dispatched with the help of `AuthorizationService`. As this
request is asynchronous the response is passed to a callback:

```java
service.performRegistrationRequest(
    registrationRequest,
    new AuthorizationService.RegistrationResponseCallback() {
        @Override public void onRegistrationRequestCompleted(
            @Nullable RegistrationResponse resp,
            @Nullable AuthorizationException ex) {
            if (resp != null) {
                // registration succeeded, store the registration response
                AuthState state = new AuthState(resp);
                //proceed to authorization...
            } else {
              // registration failed, check ex for more details
            }
         }
    });
```

### Handling the Redirect
We are using a custom scheme to send the OAuth redirect back to 
app. The library configures the `RedirectUriReceiverActivity` to 
handle a custom scheme and need to declare this activity into 
your `AndroidManifest.xml` file by adding following:

```xml
    <activity android:name="net.openid.appauth.RedirectUriReceiverActivity">
        <intent-filter>
            <action android:name="android.intent.action.VIEW"/>
            <category android:name="android.intent.category.DEFAULT"/>
            <category android:name="android.intent.category.BROWSABLE"/>
            <data android:scheme="YOUR_CUSTOM_SCHEME"
                android:host="YOUR_REDIRECT_HOST"/>
        </intent-filter>
    </activity>
```

After completing authorization in custom tab, above custom scheme 
will redirect back to app.

When the response is captured by `RedirectUriReceiverActivity`, 
it is ultimately forwarded to the activity specified in your 
completion intent, and can be extracted from the intent data:

```java
public void onCreate(Bundle b) {
  // ...
  AuthorizationResponse resp = AuthorizationResponse.fromIntent(getIntent());
  AuthorizationException ex = AuthorizationException.fromIntent(getIntent());
  if (resp != null) {
    // authorization succeeded
  } else {
    // authorization failed, check ex for more details
  }
  // ...
}
```

The full redirect URI is also provided in your completion intent:

```java
public void onCreate(Bundle b) {
  // ...
  Uri redirectUri = getIntent().getData();
  // ...
}
```

Given the auth response, a token request can be created to 
exchange the authorization code:

```java
service.performTokenRequest(
    resp.createTokenExchangeRequest(),
    new AuthorizationService.TokenResponseCallback() {
      @Override public void onTokenRequestCompleted(
            TokenResponse resp, AuthorizationException ex) {
          if (resp != null) {
            // exchange succeeded
          } else {
            // authorization failed, check ex for more details
          }
        }
    });
```

If a confidential client is created through dynamic registration, 
and the server expects the client to authenticate at the token 
endpoint, the necessary client authentication must be supplied 
in the token request. This can be simplified by making sure to 
store the registration response in the `AuthState` instance, 
then `AuthState::getClientAuthentication` can construct the 
necessary client authentication:

```java
service.performTokenRequest(
    resp.createTokenExchangeRequest(),
    state.getClientAuthentication(),
    new AuthorizationService.TokenResponseCallback() {
      @Override public void onTokenRequestCompleted(
            TokenResponse resp, AuthorizationException ex) {
          if (resp != null) {
            // exchange succeeded
          } else {
            // authorization failed, check ex for more details
          }
        }
    });
```
### Making API Calls

With an updated AuthState based on the token exchange, it is then 
possible to make requests using guaranteed fresh tokens at any 
future point:

```java
AuthState state = readAuthState();
state.performActionWithFreshTokens(service, new AuthStateAction() {
  @Override public void execute(
      String accessToken,
      String idToken,
      AuthorizationException ex) {
    if (ex != null) {
      // negotiation for fresh tokens failed, check ex for more details
      return;
    }

    // use the access token to do something ...
  }
});
```