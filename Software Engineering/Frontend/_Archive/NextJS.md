---
title: "NextJS"
---
RSC life cycle

![image 7](/Attachments/image%207.png)

RCS Update 

![image 8](/Attachments/image%208.png)


<Note>
**Static Rendering**

Static rendering is a server rendering strategy where we generate HTML pages when building our application along side the HTML that is being generated the RSC payloads for components and JS chunks for client-side hydration is created.

The Client-side navigation uses the RSC payloads and JS chunks without additional server requests.

> It is the default strategy in NextJS app router

Prefetch is a technique that preloads routes in the background as their Links become visible
</Note>

<Note>
**Dynamic Rendering (server-rendered on demand)**

Dynamic rendering is a server rendering strategy where routes are rendered uniquely for each user in the server when they make a request.  (Data available at request time and not ahead of time during pre-rendering like cookies, URL search params) like shopping pages.

NextJS dynamically switch from static to dynamic when it detect a dynamic function or dynamic API (cookies(), headers(), connection(), draftMode(), searchParams prop, after())

HTML are generated at request time.

If you want to force dynamic routes export dynamic = “force-dynamic” at the top of your page.
</Note>

<Note>
**generateStaticParams()**

![image 9](/Attachments/image%209.png)

A function that works with dynamic pages to generate static routes during build time instead of on demand at request time.

There is a config param that we can export called dynamicParams, if set to true NextJS will statically build the dynamic url that is not included in the generateStaticParams and save it to the .next folder, if set to true it will redirect the user to a 404 page.

 
</Note>

<Note>
**Streaming**

Streaming is a rendering strategy that allows progressive UI rendering from the server

Code is broken down into smaller chunks and streamed to the client as soon as they are ready, so the user can see parts of the page right away “statically” without waiting for everything to load.

This is used using `<Suspense> </Suspense>` 
</Note>

<Note>
**Parallel Routes**

Its a routing mechanism that lets us render multiple pages simultaneously within the same layout

Parallel routes uses a feature known as slots, to create one use the `@folder`  naming convention (children in the layout is a slot), these slots will be available as props in your layout file. 
</Note>

<Note>
**Intercepting routes**

A routing mechanism that lets you load a route from another part of your application within the current layout, used to display new content while keeping your user in the same context.

Like a login form, you can go to /login or you can open a modal that shows the same exact login form

![image 10](/Attachments/image%2010.png)

You first create your parent page and the page you want to intercept in

```bash
 app
	 -> parent
			 -> child
```

then, you create at the same level as the child the intercepted use the (.)child

```bash
app
	-> parent
		-> child
		-> (.)child
```

If you want to intercept a component that is in the same level as parent

```bash
app
	-> another
	-> parent
		-> child
		-> (.)child
		-> (..)another
```

![image 11](/Attachments/image%2011.png)
</Note>
