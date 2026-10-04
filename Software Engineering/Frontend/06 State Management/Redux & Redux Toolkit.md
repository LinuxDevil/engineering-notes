---
title: "Redux & Redux Toolkit"
tags: [frontend, frontend/state-management, frontend/redux]
aliases: [Redux, Redux Toolkit, RTK, redux-thunk, createSlice]
updated: 2026-09-27
---
# Redux & Redux Toolkit

<Note>
**TL;DR**

**Redux** is a tiny library for a single global store: state changes only by **dispatching actions** to **pure reducers** `(state, action) => newState`. **Redux Toolkit (RTK)** is the official, recommended way to write Redux today: `configureStore`, `createSlice` (with Immer so you can "mutate" drafts), `createAsyncThunk` and **RTK Query**. **React-Redux** is the binding layer (`Provider`, `useSelector`, `useDispatch`). Hand-written `createStore` + switch reducers is legacy; learn it to understand the internals and for interviews.
</Note>

## Mental model

{/* diagram source: /diagrams/diagram-272.mmd */}
![diagram](/diagrams/diagram-272.svg)

Three principles:
1. **Single source of truth**: one store per app (having multiple stores is an anti-pattern).
2. **State is read-only**: the only way to change it is to dispatch an action (a plain object describing *what happened*).
3. **Changes are made with pure functions**: reducers compute the next state without side effects and without mutating the previous state.

Key facts from the owner's notes (corrected where needed):
- You can use Redux **not only with React**; it's plain JS and works anywhere JS runs.
- **React-Redux** is the connector between Redux and React. It uses the **Context API only to pass the store instance** down the tree. Components then subscribe to the store directly (via `useSyncExternalStore`), so context does *not* carry the state and doesn't cause the "every consumer re-renders" problem.
- **Redux Toolkit is an abstraction on top of the Redux core** (not on top of React-Redux). You still use React-Redux alongside it.
- Only put **business state** inside Redux; don't put **UI** state (is this modal open?) there. And in 2026, don't put **server data** there by hand either: use RTK Query ([Server State - TanStack Query & RTK Query](/Software%20Engineering/Frontend/06%20State%20Management/Server%20State%20-%20TanStack%20Query%20%26%20RTK%20Query)).

## Redux core: 5 functions (9 really)

Redux exports **5** top-level functions: `compose`, `combineReducers`, `createStore`, `bindActionCreators`, `applyMiddleware`. The store created by `createStore` has **4** more methods: `dispatch`, `subscribe`, `getState`, `replaceReducer`. 5 + 4 = "9 really".

### `compose`

Combines functions right-to-left into a new function: `compose(f, g, h)(x) === f(g(h(x)))`. Internally it builds an array of the functions and uses `reduce`, calling each function with the result of the previous one.

<Info>
**compose**

```js
import { compose } from "redux";

const makeLouder = (string) => string.toUpperCase();
const repeatThreeTimes = (string) => string.repeat(3);
const embolden = (string) => `<b>${string}</b>`; // String.prototype.bold() is deprecated

const combinedFunction = compose(repeatThreeTimes, makeLouder, embolden);
combinedFunction("hi"); // "<B>HI</B><B>HI</B><B>HI</B>"

/* compose(repeatThreeTimes, makeLouder, embolden)(x) is, under the hood:
 * repeatThreeTimes(
 *   makeLouder(
 *     embolden(x)
 *   )
 * )
 */

// A minimal implementation
const myCompose = (...fns) => fns.reduce((f, g) => (...args) => f(g(...args)));
```
</Info>

### Reducers and `combineReducers`

A **reducer is just a function that takes a state and an action and outputs a new state**.

```ts
type ApplicationState = {};
type Action = { type: string; [key: string]: any };

type Reducer = (state: ApplicationState, action: Action) => ApplicationState;
```

![Screenshot_2023-08-02_at_1.47.20_PM](/Attachments/Screenshot_2023-08-02_at_1.47.20_PM.png)

`combineReducers` combines several slice reducers into one root reducer. Each key in the object becomes a key in the state. **Every one of the combined reducers receives every action**; each reducer ignores actions it doesn't care about by returning its current state. Advice: the **flatter** your state object, the better (normalize nested data).

<Info>
**combineReducers (owner's example)**

```js
import { createStore, combineReducers } from "redux";

const ADD_TASK = "ADD_TASK";
const ADD_USER = "ADD_USER";

const initState = {
  users: [
    { id: 1, name: "Steve" },
    { id: 2, name: "Ali" },
  ],
  tasks: [{ title: "File Something up" }, { title: "Order more cupcakes" }],
};

const addTask = (title) => ({ type: ADD_TASK, payload: title });
const addUser = (name) => ({ type: ADD_USER, payload: name });

const userReducer = (state = initState.users, action) => {
  if (action.type === ADD_USER) return [...state, action.payload];
  return state;
};

const taskReducer = (state = initState.tasks, action) => {
  if (action.type === ADD_TASK) return [...state, action.payload];
  return state;
};

const reducer = combineReducers({ users: userReducer, tasks: taskReducer });
const store = createStore(reducer);
store.dispatch(addTask({ title: "Book flight" }));
```
</Info>

### `createStore` (and its 4 methods)

A store is **a global container for the state of your application**. `createStore(reducer, preloadedState?, enhancer?)` takes 3 arguments; the last two are optional.

<Warning>
**`createStore` is deprecated-marked**

Since Redux 4.2, `createStore` is marked `@deprecated` (it still works, and won't be removed) to nudge you toward RTK's `configureStore`. If you really want the plain one without the strike-through, import `legacy_createStore`. Use `configureStore` in real code.
</Warning>

<Info>
**A counter store (owner's example, bug fixed)**

```js
import { createStore } from "redux";

const initState = { value: 0 };

const CounterActions = { INCREMENT: "INCREMENT" };

// Action creator returning a Flux Standard Action
const increment = (amount) => ({
  type: CounterActions.INCREMENT,
  payload: amount,
  meta: {},
});

const reducer = (state = initState, action) => {
  switch (action.type) {
    case CounterActions.INCREMENT:
      return { value: state.value + action.payload }; // original had { state: ... }, which broke the shape
    default:
      return state;
  }
};

const store = createStore(reducer);
```
</Info>

**`dispatch(action)`** emits an action to the current reducer. The only requirement for an action is that it has a `type`. We use the **Flux Standard Action** (FSA) shape: `type`, `payload`, `meta`, `error`.

```js
const incrementAction = {
  type: "INCREMENT",
  payload: 5,
  meta: { /* additional information about the action */ },
  // error: true  -> in FSA, `error` is a boolean flag; when true, payload holds an Error object
};

store.dispatch(incrementAction);
```

**`subscribe(listener)`** takes a function that runs whenever the state is updated (after every dispatch). It returns a function you call to unsubscribe.

```js
const subscription = () => console.log("Subscribed", store.getState().value);
const unsubscribe = store.subscribe(subscription);
// later
unsubscribe();
```

**`getState()`** returns the current state.

**`replaceReducer(nextReducer)`** replaces the current reducer with a new one. Useful for **code splitting** (injecting reducers for lazily loaded features). RTK 2 has `combineSlices` + `slice.injectInto` for this.

### `bindActionCreators`

Wraps action creators so calling them dispatches automatically. Conceptually it's `compose(store.dispatch, increment)`: each bound creator is `(...args) => dispatch(creator(...args))`.

<Info>
**bindActionCreators (owner's example, with the missing MULTIPLY case)**

```js
import { createStore, bindActionCreators } from "redux";

const initState = { value: 0 };
const CounterActions = { INCREMENT: "INCREMENT", MULTIPLY: "MULTIPLY" };

const increment = (amount) => ({ type: CounterActions.INCREMENT, payload: amount });
const multiply = (amount) => ({ type: CounterActions.MULTIPLY, payload: amount });

const reducer = (state = initState, action) => {
  switch (action.type) {
    case CounterActions.INCREMENT:
      return { value: state.value + action.payload };
    case CounterActions.MULTIPLY:
      return { value: state.value * action.payload };
    default:
      return state;
  }
};

const store = createStore(reducer);

const actions = bindActionCreators({ increment, multiply }, store.dispatch);

actions.increment(400); // value: 400
actions.multiply(4);    // value: 1600
```
</Info>

### Store enhancers

![Screenshot_2023-08-02_at_2.53.24_PM](/Attachments/Screenshot_2023-08-02_at_2.53.24_PM.png)

An **enhancer** is a higher-order function that lets us **modify and extend `createStore`** itself: `(createStore) => (reducer, preloadedState, enhancer) => store`. It gets to wrap the reducer, `dispatch`, `getState`, etc. `applyMiddleware` and the Redux DevTools are both enhancers.

<Info>
**Monitor enhancer (measures reducer time)**

```js
const basicReducer = (state) => state;

const monitorEnhancer = (createStore) => (reducer, initState, enhancer) => {
  const monitorReducer = (state, action) => {
    const start = performance.now();
    const newState = reducer(state, action);
    const end = performance.now();
    console.log("Performance: " + Math.round(end - start) + " ms");
    return newState;
  };
  return createStore(monitorReducer, initState, enhancer);
};

const store = createStore(basicReducer, {}, monitorEnhancer);
store.dispatch({ type: "Hello" });
```
</Info>

<Info>
**Composing two enhancers with combineReducers**

```js
import { createStore, compose, combineReducers } from "redux";

const initState = { users: [{ id: 1, name: "Ali" }], tasks: [{ title: "First Task" }] };

const ACTION_ADD_TASK = "ACTION_ADD_TASK";
const ACTION_ADD_USER = "ACTION_ADD_USER";

const tasksReducer = (state = initState.tasks, action) =>
  action.type === ACTION_ADD_TASK ? [...state, action.payload] : state;

const usersReducer = (state = initState.users, action) =>
  action.type === ACTION_ADD_USER ? [...state, action.payload] : state;

const combinedReducer = combineReducers({ tasks: tasksReducer, users: usersReducer });

const stateEnhancer = (createStore) => (reducer, initState, enhancer) => {
  const stateLogger = (state, action) => {
    console.log("Previous State: ", state, "Action: ", action);
    const newState = reducer(state, action);
    console.log("New State: ", newState, "Action: ", action);
    return newState;
  };
  return createStore(stateLogger, initState, enhancer);
};

const monitorEnhancer = (createStore) => (reducer, initState, enhancer) => {
  const monitorReducer = (state, action) => {
    const start = performance.now();
    const newState = reducer(state, action);
    console.log("Performance: " + Math.round(performance.now() - start) + " ms");
    return newState;
  };
  return createStore(monitorReducer, initState, enhancer);
};

const store = createStore(combinedReducer, initState, compose(monitorEnhancer, stateEnhancer));
store.dispatch({ type: ACTION_ADD_TASK, payload: { title: "New Task" } });
```
</Info>

### `applyMiddleware`

Middleware is a way to **intercept actions as they pass through `dispatch`** (logging, analytics reports, async, crash reporting). `applyMiddleware(...mws)` is itself an enhancer that composes the whole chain. A middleware has the curried signature `store => next => action => result`, where `next` calls the next middleware (or the real `dispatch` at the end of the chain).

{/* diagram source: /diagrams/diagram-273.mmd */}
![diagram](/diagrams/diagram-273.svg)

<Info>
**Log + performance middleware (owner's example)**

```js
import { createStore, combineReducers, applyMiddleware } from "redux";
// ...same tasksReducer / usersReducer / initState as above

const logMiddleware = (store) => (next) => (action) => {
  console.log("Old state", store.getState(), action);
  const result = next(action);           // pass the action on (and return its result)
  console.log("New state", store.getState(), action);
  return result;
};

const monitorMiddleware = (store) => (next) => (action) => {
  const before = performance.now();
  const result = next(action);
  console.log("Performance after state change: " + Math.round(performance.now() - before) + " ms");
  return result;
};

const store = createStore(
  combinedReducer,
  initState,
  applyMiddleware(logMiddleware, monitorMiddleware),
);
store.dispatch({ type: ACTION_ADD_TASK, payload: { title: "New Task" } });
```
</Info>

<Info>
**Redux Thunk is ~10 lines of middleware**

```js
const thunk = (store) => (next) => (action) =>
  typeof action === "function"
    ? action(store.dispatch, store.getState) // let functions run, give them dispatch
    : next(action);
```
</Info>

### React-Redux: `Provider`, `useSelector`, `useDispatch`

`useSelector` lets you **select a piece of state from the store**. The component re-renders only when the selected value changes (compared with `===` by default).

```tsx
import { Provider, useSelector, useDispatch } from "react-redux";

<Provider store={store}><App /></Provider>;

const count = useSelector((state) => state.count);
const dispatch = useDispatch();
```

<Warning>
**Selector pitfalls**

- Returning a **new object/array** from a selector (`state => ({ a: state.a, b: state.b })` or `.filter(...)`) makes it re-render on every dispatch. Select primitives separately, pass `shallowEqual`, or memoize with `createSelector` (Reselect, re-exported by RTK).
- Derive data in selectors instead of storing it.
</Warning>

## Redux Toolkit (RTK)

RTK is the official, opinionated toolset that removes Redux boilerplate. Unlike hooks, it is **not tied to the React component life cycle**: the store lives outside React. It uses **Immer** under the hood in reducers so you can "mutate" a `draft` (a Proxy) and Immer produces the new immutable state for you.

### `configureStore`

Creates the store with good defaults. It automatically:
1. Combines slice reducers if you pass an object to `reducer`.
2. Adds default middleware: **redux-thunk**, plus in development the **immutability check** (throws if you mutate state) and **serializability check** (warns on non-serializable values like class instances or Promises in state/actions).
3. Sets up the **Redux DevTools Extension** (as an enhancer, not a middleware).

<Info>
**configureStore (owner's example fixed: `name`/`initialState` belong to slices, not the store)**

```ts
import { configureStore } from "@reduxjs/toolkit";
import { taskSlice } from "./features/tasks-slice";
import { humanSlice } from "./features/humans-slice";

export const store = configureStore({
  reducer: {
    tasks: taskSlice.reducer,
    humans: humanSlice.reducer,
  },
  // preloadedState: {...}   // optional initial state for the whole store
  // devTools: process.env.NODE_ENV !== "production",
});
```
</Info>

### `createSlice`

Think of a slice as **one piece of a feature**. It combines the **reducers, action creators and action types** in one place. The `name` differentiates slices and prefixes the generated action types (`Tasks/add`). It creates the actions on our behalf.

<Info>
**createSlice (owner's example fixed: `add` must be inside `reducers`)**

```ts
import { createSlice, nanoid, type PayloadAction } from "@reduxjs/toolkit";

type Task = { id: string; title: string; completed: boolean; assignedTo: string };

const createTask = (title: string): Task => ({
  id: nanoid(),
  title,
  completed: false,
  assignedTo: "",
});

const initState: Task[] = [createTask("Order Coffee"), createTask("Study Redux")];

export const taskSlice = createSlice({
  name: "Tasks",
  initialState: initState,
  reducers: {
    add: (state, action: PayloadAction<string>) => {
      state.push(createTask(action.payload)); // "mutation" is fine: Immer draft
    },
    toggle: (state, action: PayloadAction<string>) => {
      const task = state.find((t) => t.id === action.payload);
      if (task) task.completed = !task.completed;
    },
    assignToUser: (state, action: PayloadAction<{ taskId: string; humanId: string }>) => {
      const task = state.find((t) => t.id === action.payload.taskId);
      if (task) task.assignedTo = action.payload.humanId;
    },
  },
});

export const { add, toggle, assignToUser } = taskSlice.actions; // add("x") -> { type: "Tasks/add", payload: "x" }
export const tasksReducer = taskSlice.reducer;
```
</Info>

<Warning>
**Immer rules**

Either mutate the draft **or** return a new value, never both. `state = newValue` does nothing (it reassigns the local variable); `return newValue` instead.
</Warning>

### `createAction`

What if we want an action that is **not automatically generated** by a slice (e.g. shared by several slices)? Use `createAction`. Its optional **prepare callback formats the action** you pass to dispatch. The prepare callback must return an object with `payload` (and optionally `meta` and `error`) only.

```ts
import { createAction, nanoid } from "@reduxjs/toolkit";

// Owner's version returned { payload: taskId, completed } - `completed` at the top level is not allowed.
export const toggleTask = createAction("tasks/toggle", (taskId: string, completed: boolean) => ({
  payload: { taskId, completed },
  meta: { at: Date.now(), requestId: nanoid() },
}));

toggleTask("t1", true); // { type: "tasks/toggle", payload: { taskId: "t1", completed: true }, meta: {...} }
toggleTask.type;        // "tasks/toggle"
toggleTask.match(action); // type guard
```

### `extraReducers` (listen to other slices' actions)

What if another slice needs to react to an action from our slice? `extraReducers` creates reducers that **listen to actions defined elsewhere** (other slices, `createAction`, thunks). They don't generate action creators. Always use the **builder callback** form.

<Info>
**humanSlice reacting to `assignToUser` (owner's example, bug fixed)**

```ts
import { createSlice, nanoid } from "@reduxjs/toolkit";
import { taskSlice } from "./tasks-slice";

type Human = { id: string; name: string; taskIds: string[] };
const createHuman = (name: string): Human => ({ id: nanoid(), name, taskIds: [] });

export const humanSlice = createSlice({
  name: "Humans",
  initialState: [createHuman("Ali")] as Human[],
  reducers: {
    add: (state, action) => {
      state.push(createHuman(action.payload));
    },
  },
  extraReducers: (builder) => {
    builder.addCase(taskSlice.actions.assignToUser, (state, action) => {
      for (const human of state) {
        if (human.id === action.payload.humanId) {
          human.taskIds.push(action.payload.taskId);
        } else {
          // original called .filter() without assigning, which does nothing (filter returns a new array)
          human.taskIds = human.taskIds.filter((id) => id !== action.payload.taskId);
        }
      }
    });
  },
});
```
</Info>

### `createReducer`

Creates a standalone reducer from an **initial state** and a **builder** (same builder as `extraReducers`), with Immer built in:
- **`addCase(actionCreatorOrType, reducer)`**: like a `case` in a switch on `action.type`.
- **`addMatcher(predicate, reducer)`**: match incoming actions with your own filter function instead of only `action.type` (e.g. `isAnyOf(a, b)`, `isPending`, or "any action ending in `/rejected`").
- **`addDefaultCase(reducer)`**: runs if no case/matcher matched (the builder already returns the state unchanged by default).

Order matters: all `addCase` calls first, then `addMatcher`, then `addDefaultCase`.

```ts
import { createReducer, createAction, isAnyOf } from "@reduxjs/toolkit";

const increment = createAction<number>("counter/increment");
const reset = createAction("counter/reset");

const counterReducer = createReducer({ value: 0, lastAction: "" }, (builder) => {
  builder
    .addCase(increment, (state, action) => { state.value += action.payload; })
    .addCase(reset, (state) => { state.value = 0; })
    .addMatcher(isAnyOf(increment, reset), (state, action) => { state.lastAction = action.type; })
    .addDefaultCase(() => {});
});
```

### Async logic: Redux Thunk and `createAsyncThunk`

![Screenshot_2023-08-02_at_9.04.06_PM](/Attachments/Screenshot_2023-08-02_at_9.04.06_PM.png)

**Redux Thunk** lets you **dispatch functions that will eventually dispatch actions**. It is the simple way to hit an API and is included by default in `configureStore`.

**`createAsyncThunk(type, payloadCreator)`** creates a thunk that dispatches a lifecycle of actions:
1. `pending` when the request starts,
2. then `fulfilled` (payload = the returned value) **or** `rejected` (error / `rejectWithValue`).

{/* diagram source: /diagrams/diagram-274.mmd */}
![diagram](/diagrams/diagram-274.svg)

<Info>
**createAsyncThunk (owner's example, updated for RTK 2)**

```ts
import { createAsyncThunk, createSlice } from "@reduxjs/toolkit";

type Character = { name: string };
type State = { data: Character[]; loading: boolean; error: string | null };

const initialState: State = { data: [], loading: false, error: null };

const ENDPOINT = "https://star-wars-characters.glitch.me/api/search/";

export const fetchCharactersFromAPI = createAsyncThunk(
  "characters/fetchCharacters",
  async (searchTerm: string, { signal, rejectWithValue }) => {
    const res = await fetch(ENDPOINT + searchTerm, { signal }); // signal aborts if thunk is cancelled
    if (!res.ok) return rejectWithValue(`HTTP ${res.status}`);
    const json = await res.json();
    return json.results as Character[];
  },
);

export const charactersSlice = createSlice({
  name: "characters",
  initialState,
  reducers: {
    add: (state, action) => {
      state.data = action.payload; // original wrote to state.characters, a key not in initialState
    },
  },
  // The object syntax { [thunk.fulfilled]: fn } was REMOVED in RTK 2.0 - use the builder callback
  extraReducers: (builder) => {
    builder
      .addCase(fetchCharactersFromAPI.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchCharactersFromAPI.fulfilled, (state, action) => {
        state.data = action.payload;
        state.loading = false;
      })
      .addCase(fetchCharactersFromAPI.rejected, (state, action) => {
        state.loading = false;
        state.error = (action.payload as string) ?? action.error.message ?? "Unknown error";
      });
  },
});

// In a component
const promise = dispatch(fetchCharactersFromAPI("luke"));
promise.abort();            // cancel (e.g. in an effect cleanup)
await promise.unwrap();     // throws on rejected, returns payload on fulfilled
```
</Info>

<Tip>
**For fetching and caching data, prefer RTK Query**

`createAsyncThunk` + hand-written `loading/error/data` is exactly what RTK Query automates (caching, dedupe, refetch, invalidation). See [Server State - TanStack Query & RTK Query](/Software%20Engineering/Frontend/06%20State%20Management/Server%20State%20-%20TanStack%20Query%20%26%20RTK%20Query).
</Tip>

### Thunk vs Saga vs Observable (and listener middleware)

The owner's note: thunk is the simple way to hit an API, but if you need **debouncing** and **throttling** (and cancellation/complex orchestration), use **Redux Saga** or **Redux Observable**. Today RTK's built-in **listener middleware** covers most of those cases too.

| | Redux Thunk | Redux Saga | Redux Observable | RTK Listener middleware |
|---|---|---|---|---|
| Model | Functions that get `dispatch`/`getState` | Generator functions + effects (`take`, `call`, `put`, `debounce`, `takeLatest`) | RxJS streams ("epics": action$ in, action$ out) | Effects that run in response to matching actions |
| Learning curve | Very low | Medium-high (generators) | High (RxJS) | Low-medium |
| Debounce / throttle / cancel | Manual | Built-in effects | RxJS operators | `cancelActiveListeners`, `delay`, `take`, `condition` |
| Testability | Mock dispatch | Very testable (effects are plain objects) | Marble tests | Straightforward |
| Status | Default in RTK | Mature, less common in new code | Niche | Recommended for "side effects on actions" in new RTK code |

<Info>
**Debounced search with the listener middleware**

```ts
import { createListenerMiddleware } from "@reduxjs/toolkit";

export const listener = createListenerMiddleware();

listener.startListening({
  actionCreator: searchChanged,
  effect: async (action, api) => {
    api.cancelActiveListeners();   // cancel previous run -> "takeLatest"
    await api.delay(300);          // debounce
    await api.dispatch(fetchCharactersFromAPI(action.payload));
  },
});

// configureStore({ reducer, middleware: (gDM) => gDM().prepend(listener.middleware) })
```
</Info>

## TypeScript: typed hooks

Infer the root types from the store itself, then create pre-typed hooks once and use them everywhere.

`store.ts`:

```ts
import { configureStore } from "@reduxjs/toolkit";
import { tasksReducer } from "../features/tasks-slice";

const store = configureStore({
  reducer: {
    tasks: tasksReducer,
  },
});

export type ApplicationState = ReturnType<typeof store.getState>; // a.k.a. RootState
export type ApplicationDispatch = typeof store.dispatch;          // a.k.a. AppDispatch (knows about thunks)

export default store;
```

**Modern form (React-Redux 9.1+)**, `hooks.ts`:

```ts
import { useDispatch, useSelector, useStore } from "react-redux";
import type { ApplicationDispatch, ApplicationState } from "./store";
import type store from "./store";

export const useAppDispatch = useDispatch.withTypes<ApplicationDispatch>();
export const useAppSelector = useSelector.withTypes<ApplicationState>();
export const useAppStore = useStore.withTypes<typeof store>();
```

**Older form** (owner's version, still valid): `TypedUseSelectorHook` is an interface that makes a hook properly typed for your store's root state.

```ts
import { useSelector, useDispatch, type TypedUseSelectorHook } from "react-redux";
import type { ApplicationDispatch, ApplicationState } from "./store";

export const useAppSelector: TypedUseSelectorHook<ApplicationState> = useSelector;
export const useAppDispatch: () => ApplicationDispatch = useDispatch;
```

Usage:

```ts
const tasks = useAppSelector((state) => state.tasks.entities);
const dispatch = useAppDispatch();
```

Also: `createAsyncThunk.withTypes<{ state: ApplicationState; dispatch: ApplicationDispatch }>()` gives typed thunks.

<Tip>
**A great utility type (from the owner's notes)**

```ts
// Make only some keys required, the rest optional
type RequireOnly<T, P extends keyof T> = Pick<T, P> & Partial<Omit<T, P>>;

// Then you can use it
type DraftUser = RequireOnly<User, "realName" | "alterEgo">;
```
More in [Advanced TypeScript - Generics & Utility Types](/Software%20Engineering/Frontend/02%20TypeScript/Advanced%20TypeScript%20-%20Generics%20%26%20Utility%20Types).
</Tip>

## Other RTK tools worth knowing

- **`createEntityAdapter`**: normalized `{ ids, entities }` state with CRUD reducers (`addOne`, `upsertMany`, `removeOne`) and memoized selectors (`selectAll`, `selectById`). The `state.tasks.entities` above comes from this pattern.
- **`createSelector`** (Reselect): memoized derived data.
- **`combineSlices`** + lazy injection: code-split reducers (replaces manual `replaceReducer`).
- **`createSlice` `selectors` field** and `create.asyncThunk` inside `reducers` (RTK 2).
- **RTK Query** (`createApi`): see [Server State - TanStack Query & RTK Query](/Software%20Engineering/Frontend/06%20State%20Management/Server%20State%20-%20TanStack%20Query%20%26%20RTK%20Query).

## Legacy Redux vs RTK side by side

| Task | Legacy Redux | Redux Toolkit |
|---|---|---|
| Create store | `createStore(rootReducer, applyMiddleware(thunk), devTools...)` | `configureStore({ reducer })` |
| Action types + creators | Constants + functions by hand | Generated by `createSlice` / `createAction` |
| Immutable updates | Spreads everywhere | Immer drafts |
| Async | redux-thunk by hand, manual loading flags | `createAsyncThunk` or RTK Query |
| Handle other slices' actions | Switch cases in each reducer | `extraReducers` builder |
| TS | Lots of manual types | Types inferred |

<Warning>
**Common pitfalls**

- Mutating state in a hand-written (non-Immer) reducer, or both mutating and returning in an Immer reducer.
- Putting non-serializable values (Dates, class instances, Promises, functions) in state or actions.
- Selectors that return new references → re-render on every dispatch.
- Storing server data in slices and re-implementing caching; use RTK Query.
- Using the removed object syntax for `extraReducers` (RTK 2.0 throws).
- Importing the standalone `getDefaultMiddleware` (removed in RTK 2); use the callback argument `middleware: (getDefaultMiddleware) => ...`.
- Creating the store at module scope in Next.js SSR: create it per request (`makeStore()` inside a client `StoreProvider` with `useRef`).
</Warning>

## Interview questions

<Accordion title="Explain Redux data flow.">
UI dispatches an action → it passes through the middleware chain → the root reducer computes a new state (pure, no mutation) → the store notifies subscribers → `useSelector` re-runs selectors and re-renders components whose selected value changed.
</Accordion>

<Accordion title="What is middleware and how is it different from an enhancer?">
Middleware wraps `dispatch` (`store => next => action`) to intercept actions (logging, thunks, analytics). An enhancer wraps `createStore` itself and can change anything about the store. `applyMiddleware` is an enhancer that installs a middleware chain.
</Accordion>

<Accordion title="Why must reducers be pure and not mutate state?">
Redux and React-Redux detect changes by reference equality. Mutating keeps the same reference, so subscribers won't update, and time-travel debugging and predictability break. Purity also makes reducers trivially testable. In RTK, Immer lets you write mutating syntax while producing a new immutable state.
</Accordion>

<Accordion title="What does `configureStore` give you over `createStore`?">
Automatic reducer combination, thunk middleware, dev-only immutability and serializability checks, and DevTools setup, all with typed inference. `createStore` is marked deprecated in favour of it.
</Accordion>

<Accordion title="How does `createAsyncThunk` work?">
It returns a thunk that dispatches `pending`, runs your async payload creator, then dispatches `fulfilled` with the result or `rejected` with the error (or `rejectWithValue`). You handle those in `extraReducers` with the builder. The returned promise has `.abort()` and `.unwrap()`.
</Accordion>

<Accordion title="Thunk vs Saga?">
Thunks are simple functions with `dispatch`/`getState`, great for straightforward async. Sagas use generators and declarative effects, making complex flows (debounce, takeLatest, races, cancellation) easier and very testable, at the cost of complexity. In modern RTK, the listener middleware covers most saga use cases.
</Accordion>

<Accordion title="Does Redux use Context? Why doesn't it have the Context re-render problem?">
React-Redux puts the **store instance** (a stable reference) in context, not the state. Components subscribe to the store via `useSyncExternalStore` and re-render only when their selector result changes.
</Accordion>

<Accordion title="When would you NOT use Redux?">
Small/medium apps where state is mostly server data (use TanStack Query/RTK Query or RSC), URL state, or a bit of global client state (Context or Zustand).
</Accordion>

## Related
- [State Management Overview](/Software%20Engineering/Frontend/06%20State%20Management/State%20Management%20Overview)
- [Server State - TanStack Query & RTK Query](/Software%20Engineering/Frontend/06%20State%20Management/Server%20State%20-%20TanStack%20Query%20%26%20RTK%20Query)
- [Functions - HOF, Callbacks & Currying](/Software%20Engineering/Frontend/01%20JavaScript/Functions%20-%20HOF%2C%20Callbacks%20%26%20Currying) (compose, currying in middleware)
- [Objects, Arrays & Immutability](/Software%20Engineering/Frontend/01%20JavaScript/Objects%2C%20Arrays%20%26%20Immutability)
- [Advanced TypeScript - Generics & Utility Types](/Software%20Engineering/Frontend/02%20TypeScript/Advanced%20TypeScript%20-%20Generics%20%26%20Utility%20Types)
- [Hooks Reference](/Software%20Engineering/Frontend/05%20React/Hooks%20Reference)
- [React Interview Questions](/Software%20Engineering/Frontend/17%20Interview%20Prep/React%20Interview%20Questions)

## Sources
- Owner's original notes (Front End.md, Redux and Redux Toolkit sections)
- https://redux.js.org/
- https://redux.js.org/tutorials/essentials/part-1-overview-concepts
- https://redux.js.org/introduction/why-rtk-is-redux-today
- https://redux-toolkit.js.org/api/configureStore
- https://redux-toolkit.js.org/api/createSlice
- https://redux-toolkit.js.org/api/createAsyncThunk
- https://redux-toolkit.js.org/api/createListenerMiddleware
- https://redux-toolkit.js.org/usage/migrating-rtk-2
- https://react-redux.js.org/using-react-redux/usage-with-typescript
- https://github.com/redux-utilities/flux-standard-action
- https://redux-saga.js.org/ and https://redux-observable.js.org/
