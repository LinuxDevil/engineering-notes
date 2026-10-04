---
title: "Case Study - News Feed"
tags: [frontend, frontend/system-design, interview, case-study]
aliases: [News Feed, Interview Social Media News Feed, Design a news feed]
updated: 2026-09-27
---
# Case Study - News Feed

<Note>
**TL;DR**

Design a Facebook/Twitter-style feed: an infinitely scrolling list of posts with text and media, likes and comments, and new posts arriving in near real time. The interesting parts are **cursor pagination + infinite scroll**, **virtualization**, a **normalized store**, **optimistic likes**, **real-time "new posts" without jumping the scroll**, safe **rich text**, and **performance/a11y** for a very long list.
</Note>

## R: Requirements
**Functional (agree on these)**
- Show a feed of posts from people/pages the user follows, newest or ranked first.
- Post = author, timestamp, text (with links, mentions, hashtags), optional images/video, like count, comment count.
- Infinite scroll to load more.
- Like/unlike a post; open and add comments.
- Create a post (text + images).
- New posts arrive while the user is reading: show a "N new posts" pill instead of inserting them under the reader.

**Out of scope**: ads auction, stories, DMs, moderation tools, notifications center.

**Non-functional**
- Fast first load on mobile (LCP ≤ 2.5 s), smooth scroll (60 fps), INP ≤ 200 ms on the like button.
- Works on slow networks; basic offline reading of already loaded posts.
- Accessible with keyboard and screen readers.
- i18n + RTL.
- SEO: not needed for a logged-in feed (public post permalinks do need it).

## A: Architecture

{/* diagram source: /diagrams/diagram-343.mmd */}
![diagram](/diagrams/diagram-343.svg)

**Rendering strategy**
- Logged-in feed: **SSR (or streaming SSR) the app shell + first page of posts** so the first screen paints fast, then hydrate and continue client-side. A pure CSR feed is acceptable if you explain the LCP cost.
- Public post permalink pages (`/post/:id`): SSR with meta tags for SEO and link previews.

**State placement**
- Server state (posts, users, comments): a server-state cache (TanStack Query `useInfiniteQuery` or RTK Query) backed by normalized entities.
- Client UI state: composer draft, expanded comment sections, "new posts" count.
- Persisted: the composer draft (localStorage) and the last feed page (IndexedDB) for offline reading.

## D: Data model

```ts
type User = { id: string; name: string; avatarUrl: string; verified: boolean };

type Media =
  | { type: 'image'; url: string; width: number; height: number; alt: string; blurhash?: string }
  | { type: 'video'; url: string; posterUrl: string; width: number; height: number };

type Post = {
  id: string;
  authorId: string;            // reference, not nested
  createdAt: string;           // ISO
  text: string;                // plain text + entity ranges, NOT raw HTML
  entities: Array<{ type: 'mention' | 'hashtag' | 'link'; start: number; end: number; value: string }>;
  media: Media[];
  likeCount: number;
  commentCount: number;
  viewerHasLiked: boolean;
};

type FeedState = {
  users: Record<string, User>;
  posts: Record<string, Post>;
  feed: { ids: string[]; nextCursor: string | null; status: 'idle' | 'loading' | 'error' };
  pendingNewIds: string[];     // arrived via SSE, not yet shown
};
```

Why **normalized**: the same author appears in many posts; the same post may appear in the feed, a permalink modal and a notification. One `posts[id]` means a like updates everywhere.

Why **width/height on media**: reserve space before the image loads, so no layout shift (CLS) and virtualization can estimate row heights.

Why **text + entity ranges instead of HTML**: the client renders mentions/links as React elements, so there is no `dangerouslySetInnerHTML` and no XSS surface.

## I: Interface (API)

```http
GET  /api/feed?cursor=<opaque>&limit=10
200  { "posts": [Post], "users": [User], "nextCursor": "c_123" | null }

POST /api/posts                         body: { text, mediaIds[] }  Idempotency-Key: <uuid>
201  { "post": Post }

POST   /api/posts/:id/like              200 { likeCount, viewerHasLiked: true }
DELETE /api/posts/:id/like              200 { likeCount, viewerHasLiked: false }

GET  /api/posts/:id/comments?cursor=&limit=
POST /api/media   (multipart or presigned upload URL)  -> { mediaId }

GET  /api/feed/stream  (SSE)  event: new-post  data: { "id": "p_999" }
```

**Why cursor pagination**: new posts are constantly inserted at the top. With `?page=2`/offset, an insert shifts everything and you get duplicates or skipped posts. A cursor ("items older than X") is stable.

**Why SSE for new posts**: updates only flow server → client, SSE is plain HTTP, auto-reconnects, and works through most proxies. WebSocket is overkill unless you also need chat/typing. Polling every 30–60 s is an acceptable simpler fallback.

**Side-loading users**: the response includes `users` separately so the client can normalize without extra requests.

## O: Optimizations & deep dives

### 1. Infinite scroll with IntersectionObserver
Do not listen to `scroll` and compute positions (runs on the main thread constantly). Observe a sentinel near the bottom.

<Info>
**Infinite scroll hook**

```tsx
function useLoadMore(onLoadMore: () => void, enabled: boolean) {
  const ref = useRef<HTMLDivElement>(null);
  useEffect(() => {
    if (!enabled || !ref.current) return;
    const io = new IntersectionObserver(
      ([entry]) => { if (entry.isIntersecting) onLoadMore(); },
      { rootMargin: '800px 0px' }        // start loading ~1 screen early
    );
    io.observe(ref.current);
    return () => io.disconnect();
  }, [onLoadMore, enabled]);
  return ref;
}

// with TanStack Query
const q = useInfiniteQuery({
  queryKey: ['feed'],
  queryFn: ({ pageParam, signal }) => fetchFeed(pageParam, { signal }),
  initialPageParam: null as string | null,
  getNextPageParam: (last) => last.nextCursor,
});
const sentinel = useLoadMore(() => q.fetchNextPage(), q.hasNextPage && !q.isFetchingNextPage);
```
</Info>

Also: guard against double-fetch (`isFetchingNextPage`), show a skeleton row while loading, show an inline "Couldn't load more, Retry" on error, and a clear end-of-feed message.

### 2. Virtualization
After a few hundred posts the DOM has thousands of nodes: memory grows and scrolling/INP degrade. Render only visible posts plus an overscan. See [Virtualization & Large Lists](/Software%20Engineering/Frontend/10%20Performance/Virtualization%20%26%20Large%20Lists).
- Posts have **variable heights** → use a virtualizer that measures rows (`@tanstack/react-virtual` with `measureElement`) and estimate heights from text length + media aspect ratios.
- Keep keyboard/screen reader usability: `role="feed"` with `aria-setsize`/`aria-posinset` on each `article`, and keep the focused item mounted.
- Cheaper alternative for moderate lists: `content-visibility: auto; contain-intrinsic-size: auto 500px;` on each post lets the browser skip rendering off-screen posts without JS virtualization.
- Memory offloading: drop far-away pages from memory and refetch if the user scrolls back.

### 3. Optimistic like

{/* diagram source: /diagrams/diagram-344.mmd */}
![diagram](/diagrams/diagram-344.svg)

<Info>
**React 19 `useOptimistic`**

```tsx
function LikeButton({ post }: { post: Post }) {
  const [optimistic, setOptimistic] = useOptimistic(
    { liked: post.viewerHasLiked, count: post.likeCount },
    (state, liked: boolean) => ({ liked, count: state.count + (liked ? 1 : -1) })
  );
  const [, startTransition] = useTransition();
  const toggle = () => startTransition(async () => {
    const next = !optimistic.liked;
    setOptimistic(next);
    await (next ? likePost(post.id) : unlikePost(post.id));
    // When the transition ends, the optimistic value is dropped and `post` props win again.
    // So on success likePost must also update the store/query cache; on failure nothing
    // is updated and the UI naturally falls back to the old values (show a toast).
  });
  return (
    <button aria-pressed={optimistic.liked} onClick={toggle}>
      {optimistic.liked ? 'Unlike' : 'Like'} · {optimistic.count}
    </button>
  );
}
```
With TanStack Query the same idea uses `onMutate` (snapshot + update cache), `onError` (rollback), `onSettled` (invalidate).
</Info>

Rapid double clicks: debounce the network call or send the final state only (`PUT /like {liked: true}` is idempotent and safer than toggle).

### 4. Real-time new posts without scroll jumps
- SSE pushes new post ids → store them in `pendingNewIds` and show "3 new posts" pill.
- On click: fetch those posts, prepend, scroll to top, and move focus to the first new post.
- Never insert above the viewport while the user is reading. If you must (e.g., an edit changes height), use scroll anchoring (`overflow-anchor`, which the browser does by default) or adjust `scrollTop` by the height delta.
- Pause/close the SSE connection when the tab is hidden (`visibilitychange`) and catch up on return.

### 5. Rich text and media
- Render text by splitting on entity ranges into `<a>` / `<Link>` elements. Never inject server HTML. If HTML is unavoidable, sanitize (DOMPurify / Sanitizer API) and enforce Trusted Types. See [Frontend Security - XSS, CSRF, CSP & Trusted Types](/Software%20Engineering/Frontend/11%20Security/Frontend%20Security%20-%20XSS%2C%20CSRF%2C%20CSP%20%26%20Trusted%20Types).
- Links: `rel="noopener noreferrer"` for external ones; show link previews from server-side unfurling (never fetch arbitrary URLs from the client).
- Images: `loading="lazy"` below the fold, `fetchpriority="high"` for the first post's image, `srcset`/`sizes` with AVIF/WebP from an image CDN, explicit `width`/`height`, a blurhash/LQIP placeholder.
- Video: `preload="none"` + poster; autoplay muted only when ≥50% visible (IntersectionObserver), pause when out of view.
- Post truncation with "See more" (CSS `line-clamp` + a button, not a JS measurement loop).

### 6. Composer
- Controlled textarea with `field-sizing: content` (Baseline 2026) or auto-grow; character count with `aria-live="polite"`.
- Image upload: get a presigned URL, upload directly to storage with progress, show previews via `URL.createObjectURL`, allow cancel (AbortController).
- Optimistically insert the new post at the top with a "Posting..." state; send an `Idempotency-Key` so a retry doesn't double-post.
- Save the draft to localStorage on change (debounced).

### 7. Accessibility
- Each post is an `<article>` with `aria-labelledby` pointing to the author name; the list uses `role="feed"` (the WAI-ARIA feed pattern) and `aria-busy="true"` while loading more.
- Like button: a real `<button>` with `aria-pressed`. Counts announced via the button name, not a separate live region spam.
- "New posts" pill is a button; after loading, move focus to the first new post.
- Keyboard: Tab through interactive elements; optional `j`/`k` shortcuts to move between posts (with a way to disable them).
- Images need `alt` (the composer should ask for it); respect `prefers-reduced-motion` for autoplay/animations.

### 8. Performance summary
| Concern | Technique |
|---|---|
| First paint | SSR/streaming shell + first page, inline critical CSS, preload LCP image |
| JS size | Route-level code splitting; lazy-load composer, comments, video player, emoji picker |
| Scroll | Virtualization or `content-visibility`, passive listeners, no layout reads in scroll handlers |
| Interactions (INP) | Optimistic updates, `startTransition` for non-urgent renders, React Compiler/memo on `PostItem` |
| Network | Cursor pagination, dedupe via query cache, prefetch next page early, HTTP caching for avatars/media |
| Images | Image CDN, responsive sizes, lazy loading, reserved space |

### 9. Offline and poor network
- Service worker caches the app shell (cache-first) and API responses (stale-while-revalidate); IndexedDB keeps the last N posts.
- Show an offline banner (`navigator.onLine` + `online`/`offline` events); queue likes/comments and replay when back online (Background Sync where available, else on reconnect).
- Timeouts + retry with exponential backoff and jitter for feed requests.

### 10. Observability
Track LCP/INP/CLS with `web-vitals`, feed load errors, time-to-first-post, scroll depth, like success rate. Roll out changes (e.g., new ranking UI) behind feature flags with A/B experiments.

<Warning>
**Common pitfalls**

- Offset pagination on a live feed (duplicates/missing posts).
- Using the array index as `key` when prepending new posts (React reuses the wrong DOM and state).
- Inserting new posts above the reader.
- Rendering server HTML without sanitizing.
- Forgetting image dimensions (CLS) and forgetting the error state for "load more".
</Warning>

## Interview questions
<Accordion title="Why cursor-based pagination for a feed?">
Items are inserted at the head constantly. Offset pagination shifts results, causing duplicates or skipped posts, and deep offsets are slow on the database. A cursor encodes "after this item", so each page is stable and cheap.
</Accordion>

<Accordion title="How do you keep a 5,000-post feed smooth?">
Virtualize (render only visible rows + overscan, measure variable heights), or at least use `content-visibility: auto`. Memoize post rows, avoid layout thrash, lazy-load media, and evict far-off pages from memory.
</Accordion>

<Accordion title="How do you handle a like that fails?">
Optimistic update first, then call the API; on failure roll back to the snapshot and show a non-blocking error. Make the endpoint idempotent (set-state rather than toggle) so retries are safe.
</Accordion>

<Accordion title="Polling, SSE or WebSocket for new posts?">
SSE: one-way server → client, plain HTTP, automatic reconnect. Polling is a simpler fallback; WebSocket is justified if the same page needs bidirectional features like chat or typing indicators.
</Accordion>

<Accordion title="How do you render user-generated rich text safely?">
Store plain text plus entity ranges and render them as elements. If you must accept HTML, sanitize with an allowlist (DOMPurify or `setHTML`), enforce CSP and Trusted Types, and never trust client-side sanitizing alone.
</Accordion>

<Accordion title="What makes the feed accessible?">
`role="feed"` with `article` items, `aria-busy` while loading, real buttons with `aria-pressed`, alt text on media, focus management when new posts load, keyboard navigation, and reduced-motion support.
</Accordion>

## Related
- [RADIO Framework](/Software%20Engineering/Frontend/15%20System%20Design/RADIO%20Framework) · [Frontend System Design Index](/Software%20Engineering/Frontend/15%20System%20Design/Frontend%20System%20Design%20Index)
- [Virtualization & Large Lists](/Software%20Engineering/Frontend/10%20Performance/Virtualization%20%26%20Large%20Lists) · [Observer APIs](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Observer%20APIs) · [Server State - TanStack Query & RTK Query](/Software%20Engineering/Frontend/06%20State%20Management/Server%20State%20-%20TanStack%20Query%20%26%20RTK%20Query)
- [Networking - HTTP, Caching & Real-time](/Software%20Engineering/Frontend/03%20Browser%20%26%20Web%20Platform/Networking%20-%20HTTP%2C%20Caching%20%26%20Real-time) · [Forms & Actions in React](/Software%20Engineering/Frontend/05%20React/Forms%20%26%20Actions%20in%20React) · [Accessibility Essentials](/Software%20Engineering/Frontend/12%20Accessibility/Accessibility%20Essentials)

## Sources
- WAI-ARIA feed pattern: https://www.w3.org/WAI/ARIA/apg/patterns/feed/
- MDN IntersectionObserver: https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API
- MDN Server-sent events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
- react.dev `useOptimistic`: https://react.dev/reference/react/useOptimistic
- TanStack Query infinite queries: https://tanstack.com/query/latest/docs/framework/react/guides/infinite-queries
- web.dev `content-visibility`: https://web.dev/articles/content-visibility
