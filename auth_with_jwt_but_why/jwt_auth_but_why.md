# Auth with JWT: But Why?
A wise man once asked me why a developer would choose JWT (JSON Web Tokens) over a traditional session cookie, and I thought, what if I do a bunch of research and write a really long article about that! So here it goes, let’s do some brain chewing!

*Note: this article mostly focuses on browser-based sessions. There are little bits here and there about native app sessions, but not much, as the rabbit holes are many and incredibly deep.*

**Who should read this?**

Anyone who is new to JWT or authentication and authorization, or doesn’t know when to choose JWT over traditional session cookies.

**Prerequisite Knowledge**

State and the Web, basic understanding of HTTP request-response cycle

**What’s in this article?**

Auth with session cookies, auth with JWT, backend control of sessions, web storage vs cookie jar, CORS requests, a brief note on native apps, use cases, conclusion

**But I just want a tutorial!**

That’s not what this is. Move along buddy...or stay and enjoy some good ol’-fashioned thinking.

**I noticed a mistake in the article**

![You Ought to Know](./images/you-ought-to-know.jpg)

Tell me about it. Seriously. Very seriously.

**Can I have the short version?**

JWT is a good option (but not the only one) to consider when you need to:
* Authorize access across multiple domains / services
* Reduce database requests to speed up a site (if session authorization requests are causing the issue)
* Create an API that works with browser-based and native apps
* Scale an app horizontally (i.e. add more servers)

*Note that I’m not saying the above can’t be done with traditional sessions or other technologies.*

You might want to skip JWT if the app or API:
* Already works well with old-fashioned session cookies
* Is served only in a browser
* Doesn’t receive Reddit-level traffic :(
* Will never be “publicly” available (i.e. the API serves your site but no others)
* Transmits specific user information with every auth request (i.e. each request MUST result in a query of the users database where the session data is also stored - ignore this bullet if sessions are stored in a separate database)
* Requires fine-grained tracking and control of a user’s sessions from the backend

This is just a starting point for thinking about using server-side sessions vs. JWT. With time, experience, and research, I’m sure you’ll come up with many more. 

Also, asking “JWT or session cookie?” is probably the wrong question – a better question is to ask whether the application needs stateful sessions or stateless sessions, and then from there, decide on the proper implementation. This will make more sense if you read the rest of the article.

## A short(ish) analogy to keep in mind as you learn more
![Gandalf the Bouncer](./images/gandalf-bouncer.jpg)

Think of JWT like a state or national ID card. First, you have to go to a government office, show some form of identification to prove you are who you say you are, and then you’re issued with a shiny new ID card. Now, when you go to the club and the bouncer asks you to prove you’re over 18 years old, you just show your ID. The bouncer looks for specific markers on the ID, and if it looks genuine, they let you in, and you dance the night away to the smooth sounds of [Daybreak](https://www.youtube.com/watch?v=XXvuUp-KY5g).

Traditional cookie-based sessions are a little different. The bouncer doesn’t just look at your ID card, they also call the issuing government office to verify the details on your ID card. Once the government gives the thumbs up, the bouncer lets you enter the club and enjoy those saxy sounds.

Now let’s say you do something that really riles the government’s feathers, causing ‘The Man’ to invalidate your ID. However, you still have your ID card in your possession. In the JWT scenario, the bouncer looks at your ID card, sees that it’s not expired, and lets you in – you get a night filled with smooth jazz. In the cookie-based scenario, the bouncer calls a government office and finds out your ID card isn’t worthy – no jazz for you! (Note: there are ways to immediately take away your access with JWT – the JWT bouncer would have to act more like the session cookie bouncer, more on that later).

## Auth with session cookies
Before we dive into JWT, let’s make sure we understand the basic auth pattern with our good old-fashioned session cookies (just because they’re old, doesn’t mean they’re obsolete!). Keep in mind this is one way of handling it - there are others!
1. Jambaby visits catdanceparty.com and is presented with a login page.
2. Jambaby enters their credentials, in this case a username and password, correctly.
3. Jambaby’s credentials are securely transmitted to catdanceparty.com via a POST request.
4. The site’s server receives the request, and looks up Jambaby’s credentials in the appropriate database. There’s a match, so the server generates a unique session identifier (randomized url-safe string) and stores it in the database. (If there isn’t a match, a session identifier will not be generated and Jambaby will not receive a precious, tasty cookie. Poor, sad Jambaby.)
5. The server sends a cookie containing the unique session identifier to Jambaby’s browser (aka the client). The client is then redirected to the appropriate page.
6. Jambaby lands on a protected page. The browser sends the session cookie (from the previous step) to the server as part of the GET request.
7. The server parses out the unique session identifier from the cookie, looks up the identifier in the database, and finds a match. The protected page loads successfully. (If a valid identifier were not found, the protected page would not load.)

In summary, any time a user (the Jambaby of the story) wants to access a protected route on the site, their unique session identifier must be looked up in the appropriate database. In other words, more requests for access means more database queries.

![Request-response with session cookie](./images/cookie-cycle.png)

*The above illustration is only concerned with illustrating requests for authentication and authorization. For an SPA, the server would respond with JSON, and the frontend would control the routing.*

## Auth with JWT
One of the benefits of using JWT, or other tokens, for authentication is that we don’t have to store and access session data in a database. Instead, the server receiving the authorization request merely checks if the JWT is valid, similar to how the bouncer at the club looks at an ID card instead of calling a government office (aka the database).

**So what exactly is this JWT business?**

JWT refers to a specific data format – it does not refer to how or where that data should be stored (e.g. cookies or web storage) or accessed. So what does a JWT look like?
[eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJmbGF0aXJvbiIsIm5hbWUiOiJ0b2tlbnRpbWUiLCJpYXQiOjE1Nzk4MDkwMjIsImN1c3RvbSI6ImknbSBnbGFkIHlvdSBtYWRlIGl0IHRoaXMgZmFyIGluIHRoZSByZWFkaW5nLCBwYXQgeW91cnNlbGYgb24gdGhlIGJhY2sifQ.hHJa6miIHQ7ldf_effL4YrM0_EzpOYe9SggSqnPXiRk](https://jwt.io/#debugger-io?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJmbGF0aXJvbiIsIm5hbWUiOiJ0b2tlbnRpbWUiLCJpYXQiOjE1Nzk4MDkwMjIsImN1c3RvbSI6ImknbSBnbGFkIHlvdSBtYWRlIGl0IHRoaXMgZmFyIGluIHRoZSByZWFkaW5nLCBwYXQgeW91cnNlbGYgb24gdGhlIGJhY2sifQ.hHJa6miIHQ7ldf_effL4YrM0_EzpOYe9SggSqnPXiRk)

If you look carefully at that beautiful string above, you’ll see that it’s dot separated. Each dot denotes the beginning of a new part of the token. Every JWT has the following parts in the same order: Header.Payload.Signature

**Parts of a JWT**

```
{
  "alg": "HS256",
  "typ": "JWT"
}
```

The header typically consists of two parts – the signing algorithm and the type of token, which is ‘JWT’. The signing algorithm must be specified in order to validate the signature (this will make more sense soon, promise). 

```
{
  "sub": "1234567890",
  "name": "Blarney McDougalPants",
  "iat": 1516239022,
  "custom": "I am so special"
}
```

The payload contains information, called claims, that is specific to an application’s needs. For example, an application might need an expiration date for the token and the name of the issuer as well as other non-sensitive data. Since the payload can easily be [decoded](https://www.google.com/search?q=decode+base64&oq=decode+&aqs=chrome.2.69i57j0l6j69i60.3031j1j7&sourceid=chrome&ie=UTF-8), it’s important to not store sensitive data, such as any information that can be used to identify a person, in a JWT.

```
BgnTL5iqjdvsC-LSCD9YNlmpcn6ppU7_OZWYj3uHoPs
```

The signature is used to confirm that the token wasn’t tampered with before or during transmission. Once a server receives a JWT, it can confirm that the JWT is valid by running the encoded header and payload along with the secret through the hashing algorithm specified in the header to generate its signature. If the generated signature matches the received JWT’s signature, the JWT is valid. This will make more sense in a moment.

**But why does it look like nonsense?**
We write the header and payload as ordinary JSON, and then those parts are encoded using the base64url encoding scheme, which basically spits out strings that look like gibberish. After that, the signature needs to be created. To make the signature, the encoded header and payload are hashed with a secret (another nonsensical-looking string) using a hashing algorithm, which is specified in the header. The hashed string is then encoded in the same manner as the header and payload. The secret, provided it remains a secret, makes it difficult for malicious users to forge or tamper with a JWT, since they need the secret to generate a valid signature. If you’re familiar with password salting, you can think of the secret as being similar to the salt.

![JWT Signature Diagram](./images/signature.png)

**Basic Auth Flow with Stateless JWT**

This is *one possible way* JWT can be used for authenticating and authorizing a user:

1. Jambaby visits catdanceparty.com and is presented with a login page.
2. Jambaby enters their credentials, in this case a username and password, correctly.
3. Jambaby’s credentials are securely transmitted to catdanceparty.com via a POST request.
4. The site’s server receives the request, and looks up Jambaby’s credentials in the appropriate database. There’s a match, so the server generates a JWT with an expiration date inside of it.
5. The server sends the JWT to Jambaby’s browser (aka the client). The client stores the token and is then redirected to the appropriate page.
6. Jambaby lands on a protected page. The browser sends the JWT (from the previous step) to the server as part of the GET request.
7. The server checks if the JWT is valid by generating the signature for the given JWT and checking if it matches the signature on the JWT sent by the client. If there’s a match and the JWT has not yet expired, the protected page loads successfully. (If the signatures didn’t match or the JWT were expired, the protected page wouldn’t load).

![JWT Session Auth Flow](./images/jwt-session.png)

*The example above assumes there’s a database dedicated to storing only credentials. It does not illustrate the usage of any other databases that may be required to complete requests. For an SPA, the server would respond with JSON, and the frontend would control the routing.*

In summary, any time a user (the Jambaby of the story) wants to access a protected route on the site, the server simply checks if the JWT is valid. In other words, more requests for access does not necessarily mean more database queries.

## Backend control of traditional sessions vs. JWT sessions

![Hacker man](./images/hacker-man.png)

Depending on the application, we might need fine-grained control over every users’ sessions from the backend. Any web app that stores sensitive data will likely need backend control over individual sessions, e.g. automatically or manually logging a user out. Another way to think of this is to ask yourself, “How much damage can a hacker do to a user if the hacker hijacks a user’s session?” There are other reasons for needing backend tracking and control, but we’ll stick with Hackerman as our main reason.

**Cookie-based sessions vs. stateless JWT sessions**

One of the benefits of using stateless JWT sessions is that it has the potential to result in fewer database queries. With cookie-based sessions, every attempt to access a protected route results in a database query to check if the session is valid. JWT, on the other hand, only requires the server to validate the token’s signature – no lookups are necessary. However, there are some downsides to using stateless JWT sessions.

With a cookie-based session, it’s common to associate a user’s session with a unique session ID in a sessions database (or some other storage mechanism). On the backend, this means we can easily log out a single user by nullifying or changing their session ID so that it no longer matches the ID stored in the client’s session cookie. If we have reason to believe a specific user’s account has been compromised, or if the user’s permissions or password have changed (or maybe they forgot my birthday), we can simply update the session’s ID on the backend, and they’re logged out. It’s also possible to keep track of which sessions are active on different user devices.

Since stateless JWT means we aren’t storing session data server-side, we can’t easily nullify users’ sessions. If a user changes their password, we can’t log them out on any devices other than the one used to change the password. If we suspect a specific account has been hacked, we can’t nullify the session on the backend, because we aren’t storing any session data. Lastly, if we’re storing any data about a user’s permissions in a JWT and we then change those permissions on the backend, that user will have the old-level of access until their JWT expires. For the last example, we could check the user’s permissions in the database for any permission-protected requests, but that might negate the benefits of using stateless JWT for authorization, since it would result in additional database queries.

It’s equally easy to log out all users at once with either approach. For cookies, we can nullify all of the session data, or if we’re using encrypted cookies, we can update the secret used to decrypt the cookies. For JWT, just change the secret used to validate the tokens’ signatures.

**Cookie-based sessions vs. stateful JWT sessions**

But wait, I thought JWT was stateless?! Well, in a way, JWT can be anything we want it to be (except our best friend, because that would be weird), including stateful. There are a number of ways to make JWT stateful – two common methods include whitelisting and blacklisting tokens. Both of those methods require storing and accessing data server-side in order to validate tokens.

A JWT whitelist will include identifiers for all tokens that should be honored. A blacklist will include identifiers for all tokens that should be rejected. In either case, we’ll need to automate tasks that keep these lists up to date. For example, there’s no reason to keep a token on either list once it officially expires. Using a whitelist or blacklist gives us control over user accounts in the same way traditional sessions do, since we now have the power to revoke access on the backend by either adding a token to the blacklist or removing it from the whitelist. However, we now have to look up whether or not a token is valid, just like we have to look up session IDs, in addition to validating the signature.

**Summary**

Ultimately, the amount of control we need over a user’s sessions depends on the application and the type of user data we’re storing. If we don’t need to quickly and easily revoke access, we can consider using stateless JWT sessions. If we need fine-grained control, we’ll need stateful sessions.

## Web storage vs cookie jar

![Cookie monster thinking](./images/cookie-monster.jpg)

While traditional sessions are always stored in cookies, tokens can be stored in cookies or web storage. There are a lot of opinions on whether JWTs should be stored in cookies or web storage, but ultimately, it depends on the specifics of the web app. Let’s find out what differentiates cookies from web storage, and what that means for us.

**What is a cookie?**

A cookie is a small file containing data stored in key-value pairs. It can be set on the client by the web server, when sent in the response, or by client-side JavaScript. Cookies can store up to ~4 kilobytes of data, and they are automatically sent by the browser with every request to the domain where the cookie originated. Since they are automatically sent by the browser and increase the size of each request, they may reduce performance for users who do not have high-speed internet access.

A cookie can be persistent, i.e. remain in storage after the browser is closed (until they reach the expiration date or are deleted by the user), or session-based, i.e. deleted when the browser is closed. Browsers automatically store cookies in their own “cookie jar,” preventing one domain from accessing another domain’s cookies (i.e. catdanceparty.com only has access to its own cookies).

Depending on the flags used when a cookie is set, it can be accessed by JavaScript and HTTP requests or only by HTTP requests. We can prevent JavaScript from having access to cookies by setting them using the “HttpOnly” flag, which defends our cookies against [cross-site scripting (XSS) attacks](https://owasp.org/www-community/attacks/xss/). We can further protect cookies by setting the “secure” flag, which limits them to being sent in requests over HTTPS only. On the server-side, we’ll need to ensure all responses are sent over HTTPS, preventing the efficacy of [man-in-the-middle attacks](https://us.norton.com/internetsecurity-wifi-what-is-a-man-in-the-middle-attack.html). HttpOnly cookies, however, are vulnerable to [CSRF attacks](https://portswigger.net/web-security/csrf), which we defend against using [CSRF tokens](https://portswigger.net/web-security/csrf/tokens).

**What is web storage?**
Web storage allows sites to store data in key-value pairs to a client’s browser, similar to cookies. However, the storage limit is much higher (up to 5MB, but depends on the browser). Data stored in web storage is not automatically sent with each request, and it is stored and accessed using client-side JavaScript.

Data can be stored in web storage via local storage or session storage, both of which can be cleared with JavaScript. Data stored in local storage persists and has no expiration date. It is cleared when a user clears the browser’s cache. Anything stored in session storage is automatically deleted when the user closes their browser or the tab displaying the website. Similar to cookies, when a site stores information in web storage, only that site has access to that data.

Information stored in web storage is vulnerable to XSS attacks, since it can be accessed by JavaScript. Extra care must be taken when using third-party JavaScript to ensure it does not contain any malicious code. Unlike cookies, there is no built-in method or flag to ensure data stored in web storage is sent over HTTPS only when including it in requests. However, there is middleware that can do this for us or we can write our own scripts.

**Summary**
We can’t really say it’s better or safer to use web storage or cookies, since it all comes down to the specifics of the web app, but cookies do appear to have a slight edge in browser-based applications. HttpOnly cookies aren’t vulnerable to XSS attacks, which is important since it’s common practice to use many third-party scripts in a single web app. It’s also easier to ensure that cookies are sent with requests only over a secure connection (HTTPS) by using the “secure” flag. However, cookies are vulnerable to CSRF attacks, while data in web storage is not.

When creating browser-based apps, cookies have the added convenience of automatically being sent with each request. The same is not true for data stored in web storage. However, manually sending data typically does not require much additional code to be written on the client-side, and it may reduce the overall size of all combined requests coming from a single client, since the additional data is only included with the requests as needed (provided the app is coded that way). Conversely, [JWTs are typically larger than cookies](https://dzone.com/articles/cookies-vs-tokens-the-definitive-guide), since they typically store more data than session cookies, and this might actually increase the overall size of all combined requests.  ¯\_(ツ)_/¯

Lastly, some organizations advise against storing session identifiers and sensitive information in web storage at all due to the threat of XSS attacks, which they estimate is a larger threat than CSRF attacks.

## Cross-Origin Resource Sharing (CORS)

![Dancing cat](https://i.imgur.com/KGDOjbf.gif)

Web apps commonly request resources from other domains, e.g. catdanceparty.com might request images from catpartyimages.com. Or the frontend and backend might be hosted on two different servers. In either case, you’re going to end up with a good ol’ CORS request! You probably know them well, they’re those requests that fill your developer console with all kinds of fun errors, like this famous ditty: 
```
Cross-Origin Request Blocked: 
The Same Origin Policy disallows 
reading the remote resource at 
https://dancemagicdance.com/
```

Pure poetry!

Those CORS request errors occur when a request to one domain comes from a completely different domain. The server that receives the request has control over which other domains, if any, can access its resources. When we set up a new project using a framework like Rails, CORS requests are disallowed on the server-side by default (It’s actually the browser that enforces the policy, but the server sets the rules). We’re not going to go into detail about server-side settings – just know that we can allow a single domain, all domains, or whitelisted domains to make requests, and that certain requests require cookies or tokens to be sent along with them in order for the client to receive a successful response. Any server that is expected to receive CORS requests must be configured to allow them.

**CORS requests with httpOnly cookies**

Let’s say our server is configured to accept third-party requests, but data stored in a cookie is required for the request to receive a successful response. We are on catdanceparty.com, and from that page, a request is made to secretcatdances.co.uk (CORS request) to retrieve some super secret cat dance gifs. Since we are making a request to a different domain, none of catdanceparty.com’s cookies are sent by the browser, so the request is unsuccessful. An error appears in our browser. To remedy this, there are a couple of things we must do.

On the server-side, when it sets the cookie on the client, the cookie’s domain/path properties will have to be set in order to tell the browser where that cookie is allowed to be sent.

On the client-side, we must tell the browser that we want to send credentials with the request. When making a fetch request, we set ‘credentials’ to ‘include’ (in an XMLHTTPRequest, we would set ‘withCredentials’ to ‘true’). As long as the server is configured to receive CORS requests, and the cookie and request have been set properly, the request should result in a successful response.

**CORS requests with tokens in web storage**

Once again, our server is configured to accept CORS requests, but this time we’ve stored our authorization data in a JWT in web storage, instead of an httpOnly cookie. Since the token is accessible by JavaScript, we use JavaScript to access the token and add it to the Authorization header with our request to secretcatdances.co.uk. The CORS request completes with a successful response from secretcatdances.co.uk.

**Summary**

I’m not going to say which sounds easier than the other, but I think you know which one I’m thinking of.

![Oprah hands out tokens](./images/oprah.png)

*I’m sorry, I couldn’t help it.*

## Native app auth

If we’re designing native apps or an API that will communicate with them, tokens seem like the way to go. There are a lot of debates regarding whether or not cookies should or can be used for authentication and authorization in native mobile apps. The short story is this: some say it’s easy to implement auth with cookies and others say it’s difficult, and a few say it’s impossible! It is possible to implement auth either way, but there does appear to be more support – including tutorials, out-of-the-box solutions, and opinions – for using tokens in native apps. As of writing, [React Native has issues with implementing fetch and cookie-based authentication](https://facebook.github.io/react-native/docs/network#known-issues-with-fetch-and-cookie-based-authentication). However, both iOS and Android native apps support the use of cookies.

Whether the API uses cookies or tokens, the native app will have to explicitly send that data with each request and handle it, as needed, with every response.
