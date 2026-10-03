> Perfect your why, and your how will come naturally

- A framework serve as a tool equipped with predefined rules and conventions that offer a structured approach to construct an application.
It provides an environment that outlines
	- Overall architecture
	- Design patterns
	- Workflows
    So it’s “Pre-built solution for common functionality”

> [!note]+ Data fetching techniques
> - Client Side fetching: With useEffect
> - Server Side fetching: With marking the component async and directly hitting the API

> [!note]+ Rendering Strategies
> - Static Site generation “SSG”: Renders pages at build time for fast delivery, not good for frequently updated content. 
> - Incremental Static Re-generation “ISR”: Allows updates to static content without a full rebuild, combins benefits of SSG with dynamic updates. - This is possible via revalidation.
> export const revalidate = 3600; “route segment config” for time based revalidation. or with in-demand validation
> - Server Side Rendering “SSR” Generates content dynamically for each request, ensuring up-to-date content, ideal for real-time data.
> - Partial Pre-rendering “PPR”: Mixes static and dynamic content on a page

> For Dynamic meta-data you can export an async function called “generateMetadata” that takes the same params as your route and return the dynamic object of the meta data.

Authentication / Authorization

> [!tip] 💡
> Authentication → Verifying identity
> Authorization → Verifying role and determining the actions and permissions 

Types of authentication

> [!note]+ Session based
> A session is created on the server for each user after they login, the server sends a unique session id (Stored as a cookie) to the client
> 
> Workflow
>     - When a user logs in, the server generates a unique session ID and stores session data (like user ID, expiration time, etc.) on the server side.
>     - The server sets a cookie in the response containing the session ID.
>     - For subsequent requests, the client sends the session ID along with the request.
>     - The server verifies the session ID against the stored session data to authenticate the user.
> 
> Pros
>     - Secure: Session data is stored on the server, reducing the risk of token theft.
>     - Easy to implement: Many web frameworks provide built-in support for session management.
>     - Automatic expiration: Sessions can be set to expire after a certain period, enhancing security.
> 
> Cons
>     - Scalability issues: Storing session data on the server can lead to scalability challenges, especially in distributed systems.
>     - Server-side storage: Requires server resources to manage session data, increasing server load.
>     - Vulnerable to CSRF attacks: Session identifiers stored in cookies can be susceptible to CSRF attacks if not properly secured.

> [!note]+ Token based (JWT)
> With token based, a token containing user information is generated upon login and sent to the in the body of the request.
> 
> Workflow
>     - Upon successful authentication, the server generates a JWT containing user information and signs it with a secret key.
>     - The server sends the JWT to the client, usually in the response body or as a header.
>     - The client includes the JWT in the header of subsequent requests.
>     - The server verifies the JWT's signature and decodes its payload to authenticate the user.
> 
> Pros
>     - Stateless: Tokens contain all necessary information for authentication, eliminating the need for server-side storage.
>     - Scalable: Tokens can be easily distributed across multiple servers, improving scalability.
>     - Enhanced security: Tokens can be encrypted and signed to prevent tampering and unauthorized access.
> 
> Cons
>     - Token management: Requires additional effort to manage token lifecycle, including expiration and revocation.
>     - Token size: Tokens can be larger than session identifiers, increasing network overhead.
>     - Vulnerable to token theft: If not properly secured (e.g., through HTTPS), tokens can be intercepted and used by malicious actors.

> [!note]+ OAuth
> OAuth is a protocol for delegated authorization, allowing third-party services to access user’s resources without exposing their credentials. “Google auth for example”
> 
> Workflow
>     - Register your application with the OAuth provider and obtain client credentials (client ID and client secret).
>     - Redirect users to the OAuth provider's authentication endpoint.
>     - After authentication, the OAuth provider redirects the user back to your application with an authorization code.
>     - Exchange the authorization code for an access token and optionally a refresh token.
>     - Use the access token to access the user's resources on behalf of the user.
> 
> Pros
>     - Single sign-on (SSO): Users can access multiple services with a single set of credentials, improving user experience.
>     - Granular permissions: Users can grant limited access to their resources, enhancing privacy and security.
>     - Widely supported: OAuth is a widely adopted standard used by many popular services and platforms.
> 
> Cons
>     - Complexity: Implementing OAuth can be complex, requiring understanding of the protocol and its various flows.
>     - Trust dependency: Users must trust third-party services with access to their data, raising privacy concerns.
>     - Token management: Requires handling of access tokens and refresh tokens, adding complexity to authentication flow.

HTTP State Management Mechanism

As the HTTP is a stateless protocol, various state management mechanisms have been developed

> [!note]+ Cookies
> Cookies are small pieces of data stored in the user’s browser by the server. They are used for session management, preference management.
> 
> Client sends cookies with every request to the same server, allowing the server to identify the user and maintain state across requests.
> 
> This happen automatically.

> [!note]+ Local Storage
> Local Storage is a storage mechanism that allows data to be stored in the user’s web browser, it provide key-value storage and used for perferences, caching, and storing application state.
> 
> It provides a  (5-10MB) per domain.

> [!note]+ Session Storage
> It’s the same as local storage but it is cleared when the user’s session end (When the browser is closed). It is used for temp data.

> [!note]+ Indexed DB
> Indexed DB is a low-level API for client storage that helps store structured data, files, blobs ..etc

Storing session token (A unique identifier used to authenticate a user’s session)

> [!note]+ Cookies
> Pros
>     - Automatically sent with each request
>     - Cookies have built-in security features like HttpOnly and Secure flags
> 
> Cons
>     - Vulnerable to CSRF attacks 
>     - Limited to same-origin policy
>     - Limited in size

> [!note]+ Authorization Headers(Bearer tokens) 
> Pros
>     - Token contain all necessary information for authentication
>     - Can contain additional data like roles
> 
> Cons
>     - Requires additional effort to manage the token life cycle (Expiration and revocation)
>     - Vulnerable to token theft if not properly secured through HTTPS for example

> [!note]+ Local Storage
> Pros
>     - Provide larger storage capacity
>     - Accessible via JS API’s
> 
> Cons
>     - Not automatically sent with each request
>     - Vulnerable to XSS attacks if not properly secured

> [!note]+ Session Storage
> Pros
>     - Automatically cleared when the browser session ends.
>     - Accessible via JS API’s for temp data storage
> 
> Cons
>     - Limited to duration of browser session, requiring re-authentication after each session expiry.
>     - Not automatically sent with each request like cookies.

 
