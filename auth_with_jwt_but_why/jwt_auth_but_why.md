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
