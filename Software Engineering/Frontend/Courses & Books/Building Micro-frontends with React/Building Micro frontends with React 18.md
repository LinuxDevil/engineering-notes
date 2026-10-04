---
title: "Building Micro frontends with React 18"
---
> This book will teach you the team building for Micro-Frontends and the code part using Nx workspace & React.
> [Book Link](https://subscription.packtpub.com/book/web-development/9781804610961/2/ch02lvl1sec05/understanding-microfrontend-patterns)
> 
> → The book uses *next@12*
> 
> → The book uses *Nx@14*

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


> Microfrontends are double-edged sword, when done right they will bring a great amount of productivity; however, if not, they can make things way worse.

> Its important that the Software teams to lay down a set of rules and guiding principles that all team members and code they write adhere to. This ensures that the team will come to a decision a lot faster.

Principles of the Microfrontends
    Domain Driven Teams
> Is Microfrontends solving a technology problem or an organizational problem ?

        DDD (Domain driven design) Is a well-established concept in the microservices world. Backend teams are commonly organized around these domain models. In the Frontend we extend this thinking and by re-organizing the team with these domain models, we are now able to create vertically sliced teams, where a domain driven team can own the responsibility of a business functionality from end to end and can work independently.
![Untitled 52](/Attachments/Untitled%2052.png)

    Isolating Failure
        Microfrontends are inherently designed to be “decentralized”, this is to reduce the error blast radius (That is one common problem in the monolith SPAs). When designing a Microfrontend architecture, you ensure a graceful service degradation if one more microfrontend fails.
> Microfrontends like microservices <u>**MUST NOT**</u> depend on each other.

    Deploying Independently
        Another key principle of the Microfrontend architecture is the ability to deploy each app independently without having to redeploy the other apps. When a new app is deployed, it should immediately be available to a user and should not require a restart of the host app or servers for the change to take effect. 
        A very interesting observation with different teams working on microfrontends is that we need the DevOps pipeline to be configured to deploy that app independently without impacting the other apps, the best way to tackle this, is with having a “<u>**Full life cycle teams**</u>” who are responsible for building the app and also responsible fro deploying it to production. These teams work closely with the DevOps team to build the CI/CD pipelines and then take over the control of managing and running them.

    Preferring Runtime Integrations
![Screenshot_2023-11-26_at_11.13.58_AM](/Attachments/Screenshot_2023-11-26_at_11.13.58_AM.png)
        Runtime Integration: 
> Always preferred, Your microfront-ends should be immediately available for use the moment they are deployed.

            The team responsible for the script will build and push it, and other teams will embed that script inside, like for example bootstrap 
![Screenshot_2023-11-26_at_11.15.44_AM](/Attachments/Screenshot_2023-11-26_at_11.15.44_AM.png)
        Build-time Integration:
            The different teams build and publish their micro apps either to a version control system or an artifact repository such as NPM or Nexus.
            During this build-time, all these micro apps are brought together to build a single app bundle, which is then deployed to production. 
> This pattern is discouraged as it breaks the aforementioned principle of independent deployment.

> **In most microfrontend patterns, we can make use of a host application or a shell app that keeps a tab of the different micro apps that load within it, but care must be taken to ensure that this host/app shell is built with scalability in mind. If the process of checking for new versions of a micro app takes up a lot of CPU or memory resources, then there is a high risk that **<u>**it will become a single point of failure**</u>** when your application scales, in terms of the micro apps and the traffic it receives.**

    Avoid the “Distributed Monolith” trap
        DRY (Don’t repeat yourself) in the microservice/microfrontend world refers to automating tasks so that you don’t have to manually repeat the steps for reach microservice or micro app. These could be thins as quality gates, or performance and security checks as part of the developer’s pipelines.
        It’s means slightly different that the one we are used to (Code reusability), When working with microfrontends don’t go overboard creating libraries and utilities which eventually get imported and used in each of the micro apps. (Even with Tree-Shaking, some of the imports will remain) which will create additional unused code that is being imported into the micro apps. Also, it have a very high risk of breaking changes, with one change made to one microfrontend that is not adapted by the other. This practice will lead to a distributed monolith which is the very worst thing of the two worlds combined.

    Technology agnostic
        The reason is to allow incremental upgrades.
        Microfrontends should be technology agnostic, meaning in theory, each micro app could be built using different framework/languages. However, just because it’s possible doesn’t mean teams should go all out and use either Vue, Angular or React to build different micro apps.
        Multiple libraries/frameworks mean an additional payload being sent down the wire to users devices. and it makes it difficult to rotate team members, and moving from one team to another means having to get comfortable with new framework/library.

    Granular Scaling
        Granular scaling means when a certain pages get a lot of traffic, only the servers (pods) serving those pages should scale, while the rest of the pods serving other parts of your microfrontend can remain the same.

    Culture of Automation and DevOps
        It is very critical to build a strong culture of automation to ensure the success of the microfrontend architecture.


Components of Microfrontend Architecture (4 components that make the architecture)
    Routing Engine
        Depending on the approach you want to take, (Multi SPA, Micro apps). You can use Ngnix and have a list of all the primary routes that maps to respective apps in the multi-SPA pattern. If they are deployed to Kubernetes cluster, we can make use of the ingress routes to map the primary routes to the respective apps.
    A global state and a Communication Channel
        The communication channel between different apps (Event driven) and the global state the can be shared between different apps.
        With a monolith SPA, we use Redux or MobX where everything is written into that store and read from it. With microfrontends, the recommendation is to avoid such global client-side stores and instead let each micro app get its data from the BE and have it’s own store.
        With microfrontend pattern, it also become important to establish a common communication channel that the different apps use to communicate with each other, like for example add to cart button, the mini cart icon should be automatically incremented, in these cases <u>event-driven</u> communication works best.
    Source code Version Control
        Ployrepos
            Are where you have each of your multi-SPAs or micro apps managed in its own independent Git repository, These are easiest to start with and give complete team independence. From a devops standpoint too, it’s a lot easier to manage. But the risk of teams being isolated and reduced inter-team collaboration. Another drawback is duplication and higher maintenance costs for tooling.
        Monorepos
            Are where all your multi-SPAs or mirco apps are co-located in a single Git repo, where each app located within its own individual folder.
            The main advantage of monorepo is increased team collaboration, as everybody sees everybody’s code. Tooling and automation scripts can be centralized, one optimization done by one team are available to all. Some risks include the DevOps setup being a bit complicated, also teams to setup a fine folder-level permissions to prevent teams from overwriting each other’s code.
    Component Library 
        Make sure all apps use of a common design system and component library. A common pattern is to publish a component library as a NPM module and set up all other apps to import and use it.
        Another pattern (”Monorepo”) is to build directly from source (It is stored within the libs section)


There are many strategy/patterns to manage the repos, like Git submodules, Git subtrees, Polyrepos, Monorepos and many more.

Monorepos
    As the name suggests, mono is single and the code is managed in a single Git repository. This means all the team members work on a common single repository that contains multiple applications.
![Untitled 53](/Attachments/Untitled%2053.png)

Polyrepos
    This is where each app has its own repository, Team generally work on multiple repos switching repos as they work on different apps (But ideally, each time work on 1 repo).
![Untitled 54](/Attachments/Untitled%2054.png)


Differences between Monorepo and Polyrepos
    Team Collaboration
        With Polyrepos, teams create their own individual repositories and decide and define their own rules and guidelines on how the code is maintained. This is easy and quick, but has a few drawbacks. Here teams tend to become more isolated and requires to set up and maintain the build pipelines and recommit hooks as duplication on each repo.
        With monorepos, teams are forced to collaborate as the need to find a common way to how the code will be maintained. 
    Build tools and Quality Gates
        With Polyrepos, teams need to implement its own build system and quality gates, this will result in duplication of effort and leads to higher maintenance costs, also it will make the Teams with strong leads way better than Junior teams which will need intervention between teams.
        With monorepos, all build tools and quality gates can be centrally managed, reducing the duplication and we can leverage the strength and skillsets across all teams immediately.
    Code Ownership
        In Polyrepo, the permission are set at the repo level in terms of who has permission to view the code in a repo or make changes to it.
        In Monorepo, all teams have access to view and edit all the files in the code. The permissions and control in monorepo are maintained via a **CODEOWNERS** file, which allows us to set a granular permissions at a folder level.
    Flexibility
        Polyrepos provide the highest level of flexibility, in terms how the code within each team is managed.
        In Monorepo, this flexibility is “Intentionally” restricted to ensure all team members benefit from the best coding practices and tooling setup that the team can offer.
    Refactoring Code
        In Polyrepo, In case you are refactoring code across multiple repos, it can be very time-consuming as you will need to check out all the different repos and individually raise a merge request or pull request for each repository.
        In Monorepo, doing such large-scale refactoring is quite easy by making atomic commits where a single merge request can contain the necessary changes for all apps.
    Ownership of Upgrades
        The time to upgrade libraries or tooling, In Polyrepo teams have the full decision on to take the upgrade or to defer it if they have other priorities. This can be good and bad, as there is always a risk that some teams may get far behind in upgrading their libraries.
        With monorepos, if a shared library or tool is being upgraded, it is easy to make atomic commit across all the apps within monorepo, which means all teams directly get the benefit of the latest versions. 
    Code base Size
        With Polyrepo, your code base gradually increases over time; however with monorepo, you are dealing with a large code base right from day one. and in monorepo it grows exponentially as the application grows.


Popular Monorepo tools:
	1. Lerna (Adopted by NRWL team who built Nx)
		1. This is using packages-based monorepo style.
	2. Nx
		2. This is using integrated monorepo
		3. Supports package based monorepos 
	3. Turborepo
		4. This is using packages-based monorepo style


## Module Federation

Module Federation is a new feature in Webpack 5 that allows us to load external JS bundles in real time. (To dynamically load app bundle in realtime) 

Previously, we’ve built our component libraries and publish them on npm, and import them into our application during build time, With module federation these modules can be hosted at independent URL and import them into the application at <u>runtime</u>. We will take this feature in our advantage when we build our microfrontends, where we will have our micro apps independently hosted and loaded into the app shell in realtime.

All of the Module Federation features are made available in Webpack 5 via ModuleFederationPlugin, this is where we define the settings of how the Module Federation should work, this plugin allows us to provide or consume modules during **runtime** with other independent builds.

[https://webpack.js.org/plugins/module-federation-plugin/](https://webpack.js.org/plugins/module-federation-plugin/)

So in our host app (App shell) we should configure it like this:

`**module-federation.config.js**`

```javascript
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'hostAppName',
      remotes: {
        app1: '<app1's URL path to remoteEntry.js>',
	      app2: '<app2's URL path to remoteEntry.js>',
      },
    }),
  ],
};
```

Here, we let the Module federation know the name of the host app and provide a list of remote apps and the path to their corresponding `*remoteEntry.js*` file.

For our Remote apps (The ones that will be dynamically loaded within our host app) These ones are referred to as containers in Module federation, The JS bundle of these remote apps is usually exposed via a single `**.js**` file usually called `*remoteEntry.js*` which host the app.

Every one of these remote apps (containers) should expose the module configurations like this:

`**module-federation.config.js**`

```javascript
new ModuleFederationPlugin({
	name: 'remoteAppName',
	exposes: ['./public-path/remoteEntry.js'],
});
```

For the `**remoteEntry.js**`, this is just a small file that is <u>generated by module federation at runtime</u>. It contains the metadata for each remote app. This file is what the host app relies on to know which modules to load into.

We can also use the Module federation to dynamically load common libraries or modules such as <u>*design system*</u> for example which will negate the need to publish these common libraries to npm or to have to rebuild and re-deploy every time a common library has changed. 

> **Any kind of JS module can be dynamically imported into Module Federation**

![Untitled 55](/Attachments/Untitled%2055.png)

To setup the microfrontend with a Host (App shell) and a remote app to make the users get that “true” single-page experience while still ensuring that the app is independently built and deployed you’ll need to:

- Create a host app.
- Define the remote apps to be loaded into the host app.
- Define the remote entry for the your micro apps.

> In Module-Federation the host app will take care of the routing, so we really don’t need any proxy configurations.

![Untitled 56](/Attachments/Untitled%2056.png)


For the remote micro-apps 

- Create remote entries in the module-federation.config.js


```javascript
const moduleFederationConfig = {
  name: 'catalog',
  exposes: {
    './Module': './src/app/app.tsx'
  },
};

module.exports = moduleFederationConfig;
```

- Swap the builder in project.json to use the module-federation plugin


```javascript
"serve": {
      "executor": "@nrwl/react:module-federation-dev-server",
			...
}
```

- Add serve-static executor


```javascript
"serve-static": {
      "executor": "@nrwl/web:file-server",
      "options": {
        "buildTarget": "catalog:build",
        "port": 4201
      }
    },


targets: {
	"webpackConfig": "apps/catalog/webpack.config.js"
}
```

- Use custom webpack config that defines the remote entry modules

```javascript
const { withModuleFederation } = require('@nrwl/react/module-federation');
const baseConfig = require('./module-federation.config');

const defaultConfig = {
  ...baseConfig,
}

module.exports = withModuleFederation(defaultConfig);
```


## State Management With Module Federation

> This Book will use *Zustand* as a state management library.

If we want to use something like context api in the regular manner, we will think to put the context in the app-shell and export the context api as a remote app. This will make a cyclic dependency issue where we also need to reimport the context for the components that acts as a shell too.

![Untitled 57](/Attachments/Untitled%2057.png)

When you work with module federation, you ideally need a unidirectional flow of how the remote and host apps are loaded. So ideally you will want the store to be defined as a micro app to make all other apps consume it.

> This is like the SDK in our Verticals 

![Untitled 58](/Attachments/Untitled%2058.png)

This will make sure that our host app will never be bloated and our dependencies is will defined.
