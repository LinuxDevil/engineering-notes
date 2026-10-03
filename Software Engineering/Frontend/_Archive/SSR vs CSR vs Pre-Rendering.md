**Client-rendered Application**
    Since frameworks like Angular, Ember.js, and Backbone exists, front-end developers have tended to render everything client-side. Thanks to Google and its ability to “read” JavaScript, it works pretty well, and it’s even SEO friendly.
    With a client-side rendering solution, you redirect the request to a single HTML file and the server will deliver it without any content (or with a loading screen) until you fetch all the JavaScript and let the browser compile everything before rendering the content.
    Under a good and reliable internet connection, it’s pretty fast and works well. But it can be a lot better, and it doesn’t have to be difficult to make it that way. 

**Server-side Rendering (SSR)**
    An SSR solution is something we used to do a lot, many years ago, but tend to forget in favor of a client-side rendering solution.
    With **old** server-side rendering solutions, you built a web page—with PHP for example—the server compiled everything, included the data, and delivered a fully populated HTML page to the client. It was fast and effective.
    **But…** every time you navigated to another route, the server had to do the work all over again: Get the PHP file, compile it, and deliver the HTML, with all the CSS and JS delaying the page load to a few hundred ms or even whole seconds.
    What if you could do the first page load with the SSR solution, and then use a framework to do dynamic routing with AJAX, fetching only the necessary data?
    This is why SSR is getting more and more traction within the community because React popularized this problem with an easy-to-use solution: The [`RenderToString`](https://reactjs.org/docs/react-dom-server.html#rendertostring) method.
    This new kind of web application is called a *universal app* or an *isomorphic app*. There’s still some controversy over the exact meanings of these terms and the relationship between them, but many people use them interchangeably.
    Anyway, the advantage of this solution is being able to develop an app server-side and client-side with the same code and deliver a really fast experience to the user with custom data. The disadvantage is that you need to run a server.
    SSR is used to fetch data and pre-populate a page with custom content, leveraging the server’s reliable internet connection. That is, the server’s own internet connection is better than that of a user with [lie-fi](https://developers.google.com/web/fundamentals/performance/poor-connectivity/#what_is_lie-fi)), so it’s able to prefetch and amalgamate data before delivering it to the user.
    With the pre-populated data, using an SSR app can also fix an issue that client-rendered apps have with social sharing and the OpenGraph system. For example, if you have only one `index.html` file to deliver to the client, they will only have one type of metadata—most likely your homepage metadata. This won’t be contextualized when you want to share a different route, so none of your routes will be shown on other sites with their proper user content (description and preview picture) that users would want to share with the world.

**Pre-rendering**
    The mandatory server for a universal app can be a deterrent for some and may be overkill for a small application. This is why pre-rendering can be a really nice alternative.
    I discovered this solution with [Preact](https://blog.logrocket.com/introduction-to-preact-a-smaller-faster-react-alternative-ad5532eb6d79) and its own CLI that allows you to compile all pre-selected routes so you can store a fully populated HTML file to a *static* server. This lets you deliver a super-fast experience to the user, thanks to the Preact/React hydration function, without the need for Node.js.
    The catch is, because this isn’t SSR, you don’t have user-specific data to show at this point—it’s just a static (and somewhat generic) file sent directly on the first request, as-is. So if you have user-specific data, here is where you can integrate a beautifully designed skeleton to show the user their data is coming, to avoid some frustration on their part:
    There is another catch: In order for this technique to work, you still need to have a proxy or something to redirect the user to the right file.


### What is client-side rendering?

Client-side rendering allows developers to make their websites entirely rendered in the browser with JavaScript. Instead of having a different HTML page per route, a client-side rendered website creates each route dynamically directly in the browser. This approach spread once JS frameworks made it easy to take.

### What is the server-side rendering?

Server-side rendering allows developers to pre-populate a web page with custom user data directly on the server. It is generally faster to make all the requests within a server than making extra browser-to-server round-trips for them. This is what developers used to do before client-side rendering.

### What is the difference between client-side and server-side rendering?

Client-side rendering manages the routing dynamically without refreshing the page every time a user requests a different route. But server-side rendering is able to display a fully populated page on the first load for any route of the website, whereas client-side rendering displays a blank page first.

### What is pre-rendering?

Pre-rendering is a tradeoff between client-side and server-side rendering. Every pre-rendered page displays a skeleton template while the data waits to be rehydrated with AJAX/XHR requests. Once the page is fetched, internal routing is done dynamically to take advantage of a client-side rendered website.

### What is a universal app?

A universal app sends to the browser a page populated with data. Then the app loads its JavaScript and rehydrates the page to get a fully client-side rendered app. This approach combines the advantages of the latest techniques available today.