---
title: "Chapter 1  Introducing Microfrontends"
---
In the BE world previously in order to solve their performance and scaling challenges they adopted the microservices pattern. Now the FE teams are looking to apply the same principles which being referred as micro-frontends.

**Microfrontends** are a composition of micro apps that can developed and deployed independently by independent teams. 

> If you break your apps into the smallest possible level you will add up to the unnecessary complexity, which will negate all the benefits that micro-front ends offer.

When you want to break your app into micro-apps, you should look for the largest app that can be deployed and managed by a scrum team without effecting other micro-apps.

Microfrontends Premium

The benefits of the Micro-Frontend pattern doesn’t show until the size and the complexity boosters kicks in.

> The whole process of decoupling the various parts of components, routing, and templates and delegating them to different systems can become an unnecessary overhead for small or medium-scale apps. **The benefits of microfrontends kick in only when your project starts reaching the size and complexity thresholds. and ideally by teams over 15 people. **

![Untitled 51](/Attachments/Untitled%2051.png)

Microfrontends benefits: 

1. Faster development and deployment
2. Easier to scale as app grows
3. Improved DX
4. Progressive upgrades 

Microfrontends patterns

> These are still early days for microfrontends, and there is no right or wrong way of building them**.**

There are two primary patterns for microfrontends, both patterns can be applied irrespective of whether you are building a SSR app or CSR app. (<u>*The Multi-SPA Pattern*</u>, <u>*The Micro-Apps Pattern*</u>)

Let’s look In the case of an e-commerce application.

The Multi-SPA Pattern
    As the name suggest, it is built up of multiple SPAs. The apps will be broken into 2-3 distinct SPAs and each app will be rendered at its own URL. when the user navigates to another they are redirected via a browser reload.
    In the case of an E-Commerce app, we can distinct the building blocks of it as follows: ([Search product listing, product details] into another app, [cart, checkout] into an app, [login, registration, profile] into another app.) <u>***3***</u>*** in total***.
![Building_Microfrontends.drawio_Mutli_SPAs](/Attachments/Building_Microfrontends.drawio_Mutli_SPAs.png)
    As you can see from the figure, our app consists of 3 main parts (3 SPAs) [Catalog, Checkout, Accounts]. Each app behaves as an independent SPA that sits within its own unique global URL. each app will be deployed at is own unique global URL.
    When the users moves from one “Macro” app to another, there will be a reload of the page in the browser, this is because we usually use the HTML href tag to navigate between apps. Don’t go and complicate your architecture to achieve a Single page experience, the truth is the user don’t really care if your app is SPA or Multi-Page App (**MPA**) as long as the experience is fast and non-janky.
    At sometimes this reload will work in our favor to reduce the risk of memory bloat due to either memory leaks or too much data being put into the “data store”.
    However, if you really want to achieve this SPA feeling, you can always create a thin app that shell that hosts the global routes and data store (You can combine each macro app store into this big one, or create a SDK layer that hosts the business logic and stores), in this pattern the routing is generally split into two parts: The global part which is in the app shell that routes the macro apps and the macro app routes. 
![Building_Microfrontends_Mutli-SPA_SPA_feelingpattern.drawio](/Attachments/Building_Microfrontends_Mutli-SPA_SPA_feelingpattern.drawio.png)
    Here you’ll notice that we introduced the notion of an app shell which will include the app header and different SPAs load within the content slot. This pattern gives a true SPA experience as the header component doesn’t refresh when transitioning from one SPA to another.

The Micro-Apps Pattern
    In this pattern the web page itself is composed of different components where each component is an independent micro app that can exists in isolation and work in tandem with other micro apps as part of the same app.
![Building_Microfrontends_Micro_Apps_Pattern.drawio](/Attachments/Building_Microfrontends_Micro_Apps_Pattern.drawio.png)
    Here you can notice we are further breaking down the central content slot into smaller micro-apps. Notice how the central content area now consists of two macro apps, namely product details and recommended products micro apps.
    This pattern is more complex that the multi-SPA pattern and its recommended mainly for very very large web apps where there will be teams that will own different elements on a single page.
    This pattern will assume that there is a dedicated team that manages the product description component and another manage the recommended component on the same page (Where each team will own and manage X components ideally 1). And we would also assume that these component updates with feature enhancements, as they will be constantly undergo with A/B tests and need to be deployed more frequently.
    In This pattern, all the route both the Primary and the secondary are **managed** by the app shell. Here in addition to managing the routing and global state the <u>app shell also needs too store/retrieve info about the page layout for each of the routes and different micro apps that need to be loaded within each page</u>. 

When you need to choose a suitable pattern, there are two primary factors that will help you choose.

Team Composition
    It’s common for teams that build with Micro-services and Mirco-frontends to be vertically sliced based on business functionality. In the *E-Commerce example*, we can see a team that focuses on the browsing journey and another team that focuses on the checkout journey. If one scrum teams own the entire browser journey and one scrum team owns the entire checkout journey, its recommended to go with **Multi-SPA pattern**. However, if you have numerous small teams that own different entities of the business domain. Such as the search, product recommendation and promotion, then it would be wise to go for the **Micro apps pattern**. <u>The rule of thumb is for each Scrum team to ideally own a single micro app</u>.

Frequency of Deployment
    If there’s a specific sections of the app that change more than others, these sections can be separated into its own microfrontend, which can be separately deployed without affecting the other sections of the app. This reduces the amount of testing needed to be done because now you isolate the test to that particular micro app and not the entire application.


> To setup different apps as they are 1, we can specify this in the nextjs config rewrites section.
And you need to add the basePath into the second app next config.

```javascript
async rewrites() {
    return [
      {
        source: '/:path*',
        destination: '/:path*'
      },
      {
        source: '/catalog',
        destination: 'http://localhost:3001/catalog'
      },
      {
        source: '/catalog/:path*',
        destination: 'http://localhost:3001/catalog/:path*'
      }
    ]
  }
```

```javascript
basePath: '/catalog'
```
