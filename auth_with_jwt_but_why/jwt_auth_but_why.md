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

