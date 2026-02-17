# Architecture

This project is a **static, client-side web app** (vanilla ES modules) that computes the “burn rate” of a meeting over time and persists a small history of past meetings in **`window.localStorage`**. It can be served by any static web server; the repo also includes an **Nginx + Docker** setup for local hosting.

## High-level structure

### Runtime components

- **Browser (application runtime)**: all meeting logic runs in the browser. The entrypoint is `index.js` loaded as an ES module from `index.html`.
- **Storage (browser localStorage)**: persisted meeting history is stored in `window.localStorage` via the `Warehouse` wrapper.
- **Web server (optional)**:
  - **Docker**: `Dockerfile` packages the repo into `nginx:alpine` with a custom `nginx.conf`.
  - **docker-compose**: `docker-compose.yml` runs the container on `http://localhost:8080`.

### Code layout (by responsibility)

- **Entrypoint / controller**
  - `index.js`: wires DOM events to model updates and triggers view re-renders.
- **Domain models**
  - `lib/models/meeting.js`: meeting state machine and cost/time computations.
  - `lib/models/increment.js`: immutable-ish record for a time slice (“increment”) of a meeting.
  - `lib/models/Warehouse.js`: storage adapter for loading/saving meeting history.
- **Views (render functions / templates)**
  - `lib/views/meeting.js`: renders the “current meeting cost” panel.
  - `lib/views/pastMeetings.js`: renders the “past meetings” table from persisted data.
- **Constants**
  - `lib/constants.js`: time conversion constants used by the models.
- **Static shell**
  - `index.html`: layout and form elements; loads `index.js` as a module.
  - `app.css`: styling (Bootstrap is loaded via CDN in `index.html`).

## Execution flow

### On page load

`index.js` runs on `DOMContentLoaded` and:

- Grabs relevant DOM nodes (inputs, buttons, containers).
- Instantiates a **single** `Warehouse`.
- Immediately renders **past meetings** from storage (`PastMeetingsTemplate(warehouse)`).
- Instantiates a **single** `Meeting`, passing the `warehouse` in.
- Resets meeting state and initializes form values from `Meeting.defaultObject`.

### During a meeting (“clock running”)

- Clicking **Start**:
  - Calls `meeting.startMeeting()` (clears increments, sets `clockRunning`, starts current increment timer).
  - Starts a `setInterval(..., 100)` that re-renders the “current cost” view.
- While running:
  - Changes to attendee count or wage call:
    - `meeting.changeAttendeeCount(...)` or `meeting.changeAverageWage(...)`
    - which both **close the current increment** (via `pushIncrement()`) and start a new one.
  - Typing purpose updates `meeting.purpose` (no increment boundary is created on purpose changes).

### Stopping a meeting

- Clicking **Stop**:
  - Stops the render interval.
  - Calls `meeting.stopMeeting()`, which:
    - closes the current increment (`pushIncrement()`),
    - persists the meeting (`warehouse.save(this.increments)`).
  - Re-renders current meeting and past meetings.

### Clearing history

- Clicking **Clear** calls `warehouse.remove()` (default key), then re-renders the past meetings view.

## Design patterns in use

- **Lightweight MVC / MV* split**
  - `index.js` behaves like a controller: it owns DOM event wiring and orchestrates model/view updates.
  - `lib/models/*` are pure-ish domain models (no direct DOM manipulation).
  - `lib/views/*` are template/render functions (DOM rendering, no business logic beyond formatting).

- **Storage adapter**
  - `Warehouse` abstracts persistence behind a small API (`save/load/remove/clear/key`) so the rest of the app doesn’t call `localStorage` directly.

- **Incremental aggregation**
  - Meeting cost/time are computed as a sum of **increments** (time slices), allowing attendee count / wage changes mid-meeting while keeping totals accurate.

- **Event-driven UI**
  - The UI responds to DOM events (click/change/input) and re-renders.
  - Rendering is scheduled via `window.requestAnimationFrame(...)` in view functions.

## Storage engine contract (Warehouse + localStorage)

### Public API (current contract)

`lib/models/Warehouse.js` defines the “storage engine” used by the app.

- **Constructor**: `new Warehouse()`
  - Detects whether browser `Storage` is available.
  - Sets `warehouse.local` to `true`/`false`.

- **`save(payload, key = 'meetings')`**
  - **Inputs**
    - `payload`: expected to be a “meeting record” (in practice: an array of increments for one meeting).
    - `key`: localStorage key (string), defaults to `'meetings'`.
  - **Behavior**
    - Loads existing history with `load()`, parses JSON to an array (or uses `[]`).
    - Appends the new `payload`.
    - Trims history to **max 10** entries (shifts oldest).
    - Persists via `window.localStorage.setItem(key, JSON.stringify(history))`.
  - **Returns**
    - `null` if storage isn’t available.
    - Otherwise returns the return value of `setItem` (typically `undefined`).

- **`load(key = 'meetings')`**
  - **Inputs**: `key` must be a string.
  - **Returns**
    - `null` if storage isn’t available or key is not a string.
    - Otherwise returns the raw stored string: `window.localStorage.getItem(key)` (string or `null`).

- **`remove(key = 'meetings')`**
  - **Inputs**: `key` must be a string.
  - **Returns**
    - `null` if storage isn’t available or key is not a string.
    - Otherwise returns the return value of `removeItem` (typically `undefined`).

- **`clear()`**
  - Clears **all** localStorage keys for the origin.
  - Returns `null` if storage isn’t available.

- **`key(index)`**
  - Pass-through to `localStorage.key(index)`.
  - Returns `null` if storage isn’t available.

### Persistence key(s)

- **Default key**: `meetings` (see `Warehouse.defaultKey`)

### Stored data shape

At `localStorage['meetings']`, the stored JSON is:

- **Type**: `MeetingHistory`
- **Shape**:

```ts
type MeetingHistory = MeetingRecord[];

// A single “past meeting” is stored as an array of increments.
type MeetingRecord = IncrementRecord[];

type IncrementRecord = {
  // Captured at increment creation; serialized through JSON.stringify.
  // In practice these will deserialize as strings (Date ISO) unless
  // explicitly revived into Date objects.
  startTime: string | Date;
  stopTime: string | Date;

  // Computed numeric fields.
  elapsedTime: number;      // seconds
  attendeeCount: number;
  averageWage: number;      // currency per hour
  cost: number;             // cost for this increment
  totalCost: number;        // running total at end of this increment

  // Copied from Meeting.purpose at increment close.
  purpose: string;
};
```

#### Important notes about serialization

- `Meeting.stopMeeting()` saves `this.increments`, which are instances of `Increment`.
- `Increment.startTime` / `stopTime` are created from `new Date`.
- When persisted via `JSON.stringify(...)`, `Date` values serialize to strings. Code that reads history treats these values as serializable data, not `Date` objects.

### Reading contract (past meetings view expectations)

`lib/views/pastMeetings.js` expects:

- `warehouse.load()` returns a JSON string representing `MeetingHistory` or `null`.
- After `JSON.parse`, the result is an array of meetings.
- Each meeting is an array of increment-like objects with (at minimum):
  - `attendeeCount`, `averageWage`, `totalCost`, `purpose`, `startTime`
- The “date” shown in the table is derived from the **first increment’s** `startTime`.
- The “purpose” shown is read from the **last increment’s** `purpose`.

### Contract stability / extension points

If you want to evolve the storage engine cleanly, the current seams are:

- Swap `Warehouse` implementation (e.g., IndexedDB, remote API) while keeping the same methods and return shapes.
- Introduce a versioned envelope (e.g., `{version: 1, meetings: [...]}`) under the `meetings` key to support migrations.
- Add revive logic for `Date` fields on load if the UI ever needs date arithmetic rather than display-only formatting.

## Serving architecture (Docker + Nginx)

- `Dockerfile` copies the repo into Nginx’s static directory (`/usr/share/nginx/html`) and installs `nginx.conf`.
- `nginx.conf`:
  - forces `.js` MIME type to `application/javascript` (important for ES modules),
  - uses `try_files ... /index.html` to support SPA-style navigation,
  - sets long-lived cache headers for static assets.

