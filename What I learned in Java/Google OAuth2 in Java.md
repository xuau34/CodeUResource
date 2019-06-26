# OAuth2

There are many good introductions on the internet.  
> [Chinese one] (https://blog.yorkxin.org/2013/09/30/oauth2-1-introduction.html)

1. User access the authorization `code` from the data website through thier own accounts
2. User then transfer `code` into the requesting server/website by GET
3. Requester, client in the intro, use `code` to get an `access token`

	* Owing to `code` is transfered by GET, there would be CSRF, cross-site request forgery, and the solution is to add another lock, called `state`. Key is created by the client and sent to the data server, then the data server will send the same key back to client to make sure it's a valid `access token`.
	* There're two types of `access token`: one is called `access token`, and the other is `refresh token`. The former one could be used once, while the later one could be refreshed to get new access everytime. However, `refresh token` can only be sent by the first time the user authorized the accessibility.

# Google OAuth2 in Java

><b>[Authorization Code Flow](https://developers.google.com/api-client-library/java/google-oauth-java-client/oauth2#authorization_code_flow)</b>

Instead of the AuthorizationCodeFlow this website refers to, there's another one called GoogleAuthorizationCodeFlow inherited AuthorizationCodeFlow.

Because our final project needs data from Google Calendar users, and because Google one is wrapped better and forcus only on Google's, I decided to use Google one.



