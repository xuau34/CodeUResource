# OAuth2

OAuth2 is the OAuth protocol or framework that allows third-party application to grant limited access to an HTTP service, either on behalf of a resource owner or by allowing the third-party application to obtain access on its own behalf. - [Understanding OAuth2](http://www.bubblecode.net/en/2016/01/22/understanding-oauth2/)


There are many good introductions on the internet.  
> [Chinese one] (https://blog.yorkxin.org/2013/09/30/oauth2-1-introduction.html)

There will be three roles in this scenario - Our own website, called `client`, the `user`, and the website from which we're going to get *authorization token*, called `data server`.

The **authorization token** is the key that `client` can access users' data in `data server`.

The overall flow of how a `client` can get to token -

1. `User` get the *authorization code* from the `data server` through thier own accounts. In other words, `user`'s browser should be redirected to a site of `data server` in order to request the *code* in their own account, which is under the `data server`.
	* Meanwhile, the request should contain `redirectURI` for `data server` to redirect `user` and return the *code*.
	* As well as `state`, which is explained belowe.
2. `data server` return the `code` and the `state` by GET method into the `redirectURI`. Make sure the `state` is the same.
3. Use `code` to get an `access token`, namely *authorization token*
4. Use `code` to get data from `data server`

* Some explanation -

	* Owing to `code` is transfered by GET, there would be CSRF, cross-site request forgery, and the solution is to add another lock, called `state`. Key is created by the client and sent to the data server, then the data server will send the same key back to client to make sure it's a valid `access token`.
	* There're two types of `access token`: one is called `access token`, and the other is `refresh token`. The former one could be used once, while the later one could be refreshed to get new access everytime. However, `refresh token` can only be sent by the first time the user authorized the accessibility.  

# Google OAuth2 in Java

>**[Authorization Code Flow](https://developers.google.com/api-client-library/java/google-oauth-java-client/oauth2#authorization_code_flow)**

Instead of the `AuthorizationCodeFlow` this website refers to, there's another one called `GoogleAuthorizationCodeFlow` which inherits `AuthorizationCodeFlow`.

Below is the explanation of how I used `GoogleAuthorizationCodeFlow` to get `refresh token`.

>[Full code link](https://github.com/joycenerd/team-49-codeu/blob/master/src/main/java/com/google/codeu/servlets/CredentialServlet.java)

### `GoogleAuthorizationCodeFlow`

This class have methods to get the `code` and *authorization token*, and it maintains each user's token for further use.

> [documentation](https://developers.google.com/api-client-library/java/google-api-java-client/reference/1.19.1/com/google/api/client/googleapis/auth/oauth2/GoogleAuthorizationCodeFlow)

### constructor

#### required variables

* [HttpTransport](https://developers.google.com/api-client-library/java/google-http-java-client/reference/1.20.0/com/google/api/client/http/HttpTransport) - for http transport, thread-safe. Use the globle one for efficiency - One should use `new UrlFetchTransport()` if it's on appengine.

```java  
HttpTransport HTTP_TRANSPORT = new UrlFetchTransport();
```

* [JsonFactory](https://developers.google.com/api-client-library/java/google-http-java-client/reference/1.20.0/com/google/api/client/json/JsonFactory) - this class is used to parse json

```java  
JsonFactory JSON_FACTORY = JacksonFactory.getDefaultInstance();
```

* [GoogleClientSecrets](https://developers.google.com/api-client-library/java/google-api-java-client/reference/1.19.1/com/google/api/client/googleapis/auth/oauth2/GoogleClientSecrets) - Stores the data from `client_secrets.json`   
	* I put the credential file into `src/main/resources`, so that I can get the file from path like `/file.json`  

```java
InputStream in = CredentialServlet.class.getResourceAsStream(CREDENTIALS_FILE_PATH);
GoogleClientSecrets clientSecrets = GoogleClientSecrets.load(JSON_FACTORY, new InputStreamReader(in));
```  

* `List<String>` SCOPES - which scopes like calendar or map and what kind of access rights like read-only.
	* I found there's a class for calendar's scopes - [CalendarScopes](https://developers.google.com/resources/api-libraries/documentation/calendar/v3/java/latest/com/google/api/services/calendar/CalendarScopes.html)

```java
List<String> SCOPES = Collections.singletonList(CalendarScopes.CALENDAR);
```

* [AppEngineDataStoreFactory](https://developers.google.com/api-client-library/java/google-http-java-client/reference/1.20.0/com/google/api/client/extensions/appengine/datastore/AppEngineDataStoreFactory) - to maintain data manually, specificly for tokens.


```java
AppEngineDataStoreFactory DATA_STORE_FACTORY = AppEngineDataStoreFactory.getDefaultInstance();
```

#### code

```java
GoogleAuthorizationCodeFlow flow = new GoogleAuthorizationCodeFlow.Builder( HTTP_TRANSPORT, JSON_FACTORY, clientSecrets, SCOPES)
										.setDataStoreFactory( DATA_STORE_FACTORY )
										.build();;
```

* If the type of token that you need is refresh one, you need to `flow.setAccessType("offline")` before `build()`.

#### Other stuff

* `.setCredentialCreatedListener()` - a callback function when a credential is created. (Credential part will be explained later)
* `.addRefreshListener` - a callback if a `refresh token` is refreshed to get new `access token`

### Get authorization code

To avoid *CSRF*, we need to set `state` first, and restore it with this user, which I chose to store in the session.

```java
String state = new BigInteger(130, new SecureRandom()).toString(32);
request.getSession().setAttribute("OAuthState", state);
```

Then use `flow` to get the url that we'll redirect user to.
Remember to set the `state` and the `redirectURI`.

Another thing is that Google OAuth2 requires to register the valid URI in *OAuth2 client IDs* setting in GCP.


```java
String OAuth2Url = flow.newAuthorizationUrl()
          .setRedirectUri(request.getRequestURL().toString())
          .setState( state ) //against cross-site request forgery
          .build(); //build for string. otherwise, it'd just be object.
```

Redirect user page to `OAuth2Url`.

```java
response.sendRedirect(OAuth2Url);
```

Hence, the user page will be redirected again by `data server` to the `redirectURI` with `code` and `state`.

### Get Access/Refresh Token

Before using the `code`, we need to make sure that this request is truly from `data server`. So we need to check if the `state` is the same with the one stored under this user.

```java
Object state =  request.getSession().getAttribute("OAuthState");
request.getSession().removeAttribute("OAuthState");
if( state == null || state.toString().equals(request.getParameter("state")) == false ){
    //CSRF occurs
    return;
}
```

Use `flow` and `code` to get `access token` and `refresh token`

I'm not really sure about the meaning of setRedirectUri here.

If some one could tell me, that would help a lot!

```java
try {
  TokenResponse tokenResponse = flow.newTokenRequest( code )
    .setRedirectUri( RedirectUri )
    .execute();
}catch (TokenResponseException e) {
  //Deal with failed authorization
}
```

### Register token under this user

This part is more for `refresh token`, since `access token` can only be used once.

The same, by the help of `flow`~

```java
Credential credential = flow.createAndStoreCredential(tokenResponse, userId);
```

`Credential` is a class that contains a user's `access token` and `refresh token`, and will be used when you request to `data server`.

### Retrieve a `Credential`

```
Credential credential = flow.loadCredential(userId)

```


### Check if authorized / Use registed refresh token to get new access token

Load credential of this user first.
If this user hasn't aughorized yet, the return will be `null`.

Then refresh the credential. The return is boolean indicating whether the `refresh token` in it is valid or not.

If succeed, it will return the new access with new life span.

```java
Credential credential = flow.loadCredential(userId);
if( credential != null ){
	try{
		if( credential.refreshToken() == false ){
			//Deal with invalid authorization
		}
	}catch(TokenResponseException e)
		//This would invoke onTokenErrorResponse
	}
}
```

### If the refresh token is no longer valid


#### Possible Reasons

One possible reason is from cleaned or broken datastore.

The other reason is because this user deleted your authorization.

Or it may because there're two datastore using the same `client_id`. For example, in the local version and the GCP one.

#### Solution

Re-run the whole authorization procedure.
