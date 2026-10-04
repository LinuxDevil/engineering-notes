---
title: "Using Node API’s"
tags: [backend, backend/node, course]
aliases: [Using Node APIs, Node APIs, req and res objects]
updated: 2026-09-27
---
# Using Node API’s

<Note>
**TL;DR**

When a request arrives, Node **auto-runs** the function you gave `http.createServer` and **auto-fills its arguments**: a `req` object built by parsing the raw HTTP message, and a `res` object whose methods (`writeHead`, `setHeader`, `end`) write straight into the response Node will send back. You never call the handler yourself.
</Note>

*Part of [The Hard Parts of Servers & Node.js](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js) (course notes).*

## Node auto-runs and auto-fills

```js
const server = http.createServer((req, res) => {
  // ...
});
```

- Node not only auto-runs our function at the right moment, it also **automatically inserts the relevant data** as arguments (inputs).
- Sometimes it inserts an **object full of functions** (`res`) that give us direct access to the message being sent back to the user and let us add data to it.

## What happens when a request comes in

1. Bytes arrive at the **network card**; the OS hands them to the socket libuv is watching.
2. Node's HTTP parser (llhttp) reads the text of the request: the **request line** (`GET /path HTTP/1.1`) and the **headers** (metadata: data about data), then the body.
3. Node creates two objects:
   - `req` (an `http.IncomingMessage`): `method`, `url`, `headers`, and a readable stream for the body.
   - `res` (an `http.ServerResponse`): functions like `setHeader`, `writeHead`, `write`, `end` that build the outgoing message.
4. Node runs our handler with those two objects in a new execution context. We look at `req.url`/`req.method` to "route" and use `res.end(...)` to send data to the client.

![Screenshot_2023-04-29_at_1.45.04_AM](/Attachments/Screenshot_2023-04-29_at_1.45.04_AM.png)

## Worked example: a tiny router

```js
const http = require('node:http');

const server = http.createServer((req, res) => {
  if (req.method === 'GET' && req.url === '/tweets') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify([{ id: 1, text: 'hi' }]));
    return;
  }
  if (req.method === 'POST' && req.url === '/tweets') {
    let body = '';
    req.on('data', chunk => (body += chunk));      // body arrives as a stream
    req.on('end', () => {
      const tweet = JSON.parse(body);
      res.writeHead(201, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify(tweet));
    });
    return;
  }
  res.writeHead(404).end('Not found');
});

server.listen(3000);
```

The raw request the parser saw looks like:

```http
POST /tweets HTTP/1.1
Host: localhost:3000
Content-Type: application/json
Content-Length: 14

{"text":"hi"}
```

<Note>
**JS is an evaluated language**

Your handler code isn't run when the file loads; it's saved in memory as a function definition and evaluated later, when Node calls it.
</Note>

## Pitfalls
- Forgetting `res.end()` leaves the client hanging until it times out.
- The body is **not** on `req` automatically: you must read the stream (frameworks do this for you with body parsers).
- `JSON.parse` on untrusted input can throw; wrap it and return `400`.

## Interview questions

<Accordion title="What are `req` and `res` in Node's http module?">
`req` is an `IncomingMessage` (a readable stream plus parsed method, URL and headers). `res` is a `ServerResponse` (a writable stream with helpers to set status and headers). Node creates both for every request and passes them to your handler.
</Accordion>

<Accordion title="Why do you need to listen to `data`/`end` to read a POST body?">
Because the body arrives in chunks over the socket and `req` is a stream. Node doesn't buffer it for you, which keeps memory bounded for large uploads.
</Accordion>

## Related
- [The Hard Parts of Servers & Node.js](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js) · [Node.js Index](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node.js%20Index)
- [Introduction to Node.js](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Introduction%20to%20Node.js) · [Node with HTTP](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Node%20with%20HTTP) · [Building HTTP Servers - Express, Fastify & Hono](/Software%20Engineering/Backend/The%20Hard%20Parts%20of%20Servers%20%26%20Node.js/Building%20HTTP%20Servers%20-%20Express%2C%20Fastify%20%26%20Hono)
- [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time)

## Sources
- [Node.js docs: http](https://nodejs.org/api/http.html)
- [Node.js: Anatomy of an HTTP transaction](https://nodejs.org/en/learn/modules/anatomy-of-an-http-transaction)
- Frontend Masters, *The Hard Parts of Servers & Node.js* (Will Sentance)
