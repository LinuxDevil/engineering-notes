> Microfrontends are double-edged sword, when done right they will bring a great amount of productivity; however, if not, they can make things way worse.

> Its important that the Software teams to lay down a set of rules and guiding principles that all team members and code they write adhere to. This ensures that the team will come to a decision a lot faster.

Principles of the Microfrontends
    Domain Driven Teams
> Is Microfrontends solving a technology problem or an organizational problem ?

        DDD (Domain driven design) Is a well-established concept in the microservices world. Backend teams are commonly organized around these domain models. In the Frontend we extend this thinking and by re-organizing the team with these domain models, we are now able to create vertically sliced teams, where a domain driven team can own the responsibility of a business functionality from end to end and can work independently.
![[Untitled 52.png]]

    Isolating Failure
        Microfrontends are inherently designed to be “decentralized”, this is to reduce the error blast radius (That is one common problem in the monolith SPAs). When designing a Microfrontend architecture, you ensure a graceful service degradation if one more microfrontend fails.
> Microfrontends like microservices <u>**MUST NOT**</u> depend on each other.

    Deploying Independently
        Another key principle of the Microfrontend architecture is the ability to deploy each app independently without having to redeploy the other apps. When a new app is deployed, it should immediately be available to a user and should not require a restart of the host app or servers for the change to take effect. 
        A very interesting observation with different teams working on microfrontends is that we need the DevOps pipeline to be configured to deploy that app independently without impacting the other apps, the best way to tackle this, is with having a “<u>**Full life cycle teams**</u>” who are responsible for building the app and also responsible fro deploying it to production. These teams work closely with the DevOps team to build the CI/CD pipelines and then take over the control of managing and running them.

    Preferring Runtime Integrations
![[Screenshot_2023-11-26_at_11.13.58_AM.png]]
        Runtime Integration: 
> Always preferred, Your microfront-ends should be immediately available for use the moment they are deployed.

            The team responsible for the script will build and push it, and other teams will embed that script inside, like for example bootstrap 
![[Screenshot_2023-11-26_at_11.15.44_AM.png]]
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

