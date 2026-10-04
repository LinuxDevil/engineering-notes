---
title: "Intermediate React V6"
---
- SSG: To export the pages as fully static site. “Not interactive, just content”
- CSR: The usual rendering on the client, where this diagram shines:
![image 12](/Attachments/image%2012.png)
- SSR: Pre-rendering pages on the server and serving the full-rendered html to the client
![image 13](/Attachments/image%2013.png)

<Tip>
**💡**

Client-side React apps (no server), static site generation apps (server runs only at compile time), and server-side rendered app (server renders initial page but then client does the rest).
</Tip>

RSC → React server components

- SSR you do the initial page load and send that to the user then the client does the rest of the rendering, once it does the initial render it’s DONE, and no longer involved.
- RSC are component that only renders on the server.
SSR and RSC are not related.

