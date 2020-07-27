# Tips for Integrating Third-Party Code
Adding functionality to a web app via third-party APIs (or projects) can be challenging, especially when the documentation is lacking. Here are some tips for getting third-party code working while reducing frustration.

## 1. Use the third-party playground (if provided)
Some API developers provide online playgrounds where you can test the API. This is a great way to check your understanding of the codebase and start testing out the functions or HTTP requests your project may require. You can then take note of any code that worked in the playground and save it for when you need it later.

## 2. Find an example, quickstart, or tutorial
If you can find an up-to-date example, quickstart or tutorial, your life will be much easier. The goal is to get the example working, even if that example doesn't do exactly what you want it to do. If you can't get the example working, you may need to do more research or check for a more up-to-date resource; it's possible the example code is already out of date.

Google, for example, often provides quickstarts for their APIs. Smaller outfits, on the other hand, might only provide a project. In either case, take what you're given first, get that to work, and then think about modifying the code to do what you want. If you can't get the code that is supposed to work to actually work, you'll probably have a really hard time integrating the code you need into your project.

## 3. Isolate the code
It's tempting to go straight to integration with an existing project, but avoid that temptation, especially if you've never worked with the third-party codebase before. Instead, create a new project where you can test and play with the third-party library or project all by itself. This way, you don't have to worry about the specifics of your project clashing with the new functionality.

During this phase, see if you can modify the code to perform the core tasks that you need. If you need to use the Google Calendar API to add an event, for example, attempt to add an event to your calendar with the click of a button. If you need to trigger an event every time the score changes in a game, get that working first.

Once you've achieved the core behavior, start paring down the code. Remove any functions, classes, variables, files, or other elements you don't need. After this cleanup, check that everything is still working. You now have a project that's worthy of integration!

## 4. Git it!
Always use git or some other form of version control. There's no telling how much code is going to be added or modified during the integration process, so it's important to have a working version to revert back to. Always create a new branch for the integration in the target project - never work directly on master! It's really easy to turn your project into one giant stream of errors.

## 5. Integrate slowly
Instead of dumping a ton of code into the target project, add the third-party code piece by piece. Think about this logically: how can you divide the third-party code into logical and testable chunks? This will make debugging much easier, and it'll make it easier to spot environment issues, such as missing dependencies or dependency clashes. 

As you integrate, you may need to check if assets are loading in the correct order and if they're accessible in the correct parts of the target project when they're needed. You'll also need to see if this new code impacts your project's load time and user interactions noticeably. If it does, you may need to find ways to make the project run more efficiently and/or consider adding load screens to let users know that your project is working as intended.

## 6. Watch out for the little gotchas!
Sometimes it's the little things that cause the most frustration. If it seems like you've done everything exactly as you were supposed to, you may need to check on some of the little things that documentation often glosses over.

For projects that require credentials (API keys or developer IDs, for example), make sure your credentials were generated correctly and with the correct settings. You may need to check that the credentials are associated with the correct project, have the appropriate permissions, and are whitelisting the correct domains. In some cases, you may also need to verify who you are before your credentials become active.

Also be aware of the API's rate-limiting. You don't want to exceed the allotted number of requests you're allowed to make - doing so will prevent further access to the API either for a given period of time or forever, depending on how the rate-limiting is set up. 

When testing your project, make sure the domain you're testing it on exactly matches the whitelisted domain. 

For projects that include authorization, such as OAuth, you may need to clear the browser's cookies and cache. In fact, clear this data as a habit before launching a new project. You'll be glad you did.

