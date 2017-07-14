# Using AppAuth with [Gluu](https://gluu.org/docs/)


The Gluu Server is a free open source identity and access management (IAM) platform. The Gluu Server is a container distribution composed of software written by Gluu and incorporated from other open source projects. configuration is quick and simple. 

If you do not already have a Gluu Server, you can [read the docs](http://gluu.org/docs/ce) to learn how to download and deploy the software for free.
## Creating OpenID Client on Gluu server

First of all install and login to your gluu server than follow steps below:-

  1. After login, navigate to https://{{Your_gluu_server_domain}}/identity/client/inventory and select **Add Client**
  1. Entre required values to fields( minimum required fileds are :- Client Name,Client Secret,Application Type,Pre-Authorization,Persist Client Authorizations,Logout Session Required) 
  1. Choose **Native** as the Application Type.
  1. Scroll to bottom and you will find  **"Add Grand type"** button click on it and select **Authorization Code**  as Grant type in opened popup. click ok button in popup to save settings.
  1. Redirect URIs can be like this (**"appscheme://client.example.com"**)
  1. Populate your new OpenID Connect application with values similar to:
  1. Copy the **Client ID**, as it will be needed for the client configuration.
  1. Click **Finish** to redirect back to the *General Settings* of your application.
  
  
**Note:-** You can also create OpenID Clients by using gluu's oxAuth-rp client
  Link for oxAuth-Rp will be https://{{Your_gluu_server_domain}}/oxauth-rp/home.htm 



### Clone the project
https://github.com/openid/AppAuth-Android.git

 clone project from give repo and do following changes is code.

 Finally, within your application update ``res/raw/auth_config.json`` with your settings. You will get a warning if it is incomplete or invalid. Here is an example JSON configuration: 

```json
{
  "client_id": "{{YourClientID}}",
  "redirect_uri": "appscheme://client.example.com",
  "authorization_scope": "openid email profile",
  "discovery_uri": "https://{{your_idp_domain}}/.well-known/openid-configuration",
  "authorization_endpoint_uri": "",
  "token_endpoint_uri": "",
  "registration_endpoint_uri": "",
  "https_required": true
}
```



# Gluu AppAuth Dynamic Registration.

### changes in xml file: 

> If we keep  **client_id**  blank string in  ``res/raw/auth_config.json`` application will automatically register new client to dynamic registration end point. 


We need to add header in library in request in the method RegistrationRequestTask in the file called AuthorizationService.java
Path to the file is /library/java/net/openid/appauth/AuthorizationService.java.

```java
conn.setRequestProperty("Content-Type", "application/json");
```

---


And also change the manifest file and change the activity tag to


```xml
<activity android:name="net.openid.appauth.RedirectUriReceiverActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="http" />
        <data android:scheme="https" />
        <data android:host="example.gluu.org" />
    </intent-filter>
</activity>
```



So that all the URIs that matches "example.gluu.org" are opened in the app itself
