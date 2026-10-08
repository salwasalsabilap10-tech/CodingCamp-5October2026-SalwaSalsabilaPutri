# Design Document — To Do List Life Dashboard

## Overview

The **To Do List Life Dashboard** is a single-page personal productivity web application built with plain HTML, CSS, and Vanilla JavaScript. It runs entirely in the browser with no server, no build step, and no external dependencies. All persistent state is stored in the browser's `localStorage` API.

The page is composed of four independent feature sections:

| Section | Purpose |
|---|---|
| Greeting | Live clock (HH:MM:SS), current date, time-of-day greeting |
| Focus Timer | 25-minute Pomodoro countdown with Start / Stop / Reset |
| To-Do List | Create, edit, complete, and delete tasks |
| Quick Links | Add and remove shortcut buttons to external URLs |

Each section owns its own DOM subtree, its own controller object in `js/script.js`, and its own `localStorage` key. The controllers are wired together only through the shared `StorageManager` module; they do not call each other directly.

---

## Architecture

### File Structure

```
index.html          ← Single HTML entry point; all markup
css/style.css       ← Single stylesheet
js/script.js        ← Single JavaScript file; all logic
```

### JavaScript Module Structure

`js/script.js` is organised as a collection of plain-object module literals (the "Revealing Module" pattern) that are all initialised on `DOMContentLoaded`. There are no ES modules, no classes, and no bundler — just `const` object literals with factory functions.

```
js/script.js
│
├── StorageManager          ← Read / write localStorage
├── GreetingController      ← Clock tick, date, greeting message
├── TimerController         ← Pomodoro countdown state machine
├── TodoController          ← Task CRUD + render
└── QuickLinksController    ← Link CRUD + render
```

Each controller exposes an `init()` function that is called once during startup. Controllers that need to update the DOM over time (Greeting, Timer) maintain a private interval ID.

### Initialisation Sequence

```
DOMContentLoaded
  └─ StorageManager.init()
  └─ GreetingController.init()   → starts setInterval (1 s tick)
  └─ TimerController.init()      → renders initial 25:00, binds buttons
  └─ TodoController.init()       → loads tasks from storage, renders list
  └─ QuickLinksController.init() → loads links from storage, renders buttons
```

### Dependency Flow

```
GreetingController   ──────────────────────────────► DOM
TimerController      ──────────────────────────────► DOM
TodoController       ──► StorageManager ──► localStorage
QuickLinksController ──► StorageManager ──► localStorage
```

---

## Components and Interfaces

### StorageManager

Centralises all `localStorage` interaction. All other modules call `StorageManager` rather than `localStorage` directly.

```js
const StorageManager = {
  KEYS: {
    TASKS: 'tld_tasks',
    LINKS: 'tld_links',
  },

  // Returns parsed array or [] on missing / corrupt data
  getTasks()      → Task[]
  saveTasks(tasks: Task[]) → void

  // Returns parsed array or [] on missing / corrupt data
  getLinks()      → Link[]
  saveLinks(links: Link[]) → void
}
```

`getTasks` and `getLinks` always return a valid array. If `JSON.parse` throws, they silently return `[]` and do not propagate the error.

---

### GreetingController

Manages the Greeting section. Owns a `setInterval` that fires every 1 000 ms. On each tick it queries `new Date()` and updates three DOM elements.

**DOM targets**

| Element ID | Content |
|---|---|
| `#clock` | `HH:MM:SS` string |
| `#date-display` | e.g. `Monday, 5 October 2026` |
| `#greeting-message` | `Good Morning / Afternoon / Evening / Night` |

**Greeting logic**

| Hour range | Message |
|---|---|
| 05 – 11 | Good Morning |
| 12 – 17 | Good Afternoon |
| 18 – 21 | Good Evening |
| 22 – 04 | Good Night |

The controller calls `tick()` once immediately at `init()` so the display is populated before the first interval fires.

---

### TimerController

Implements a finite-state machine with three states:

```
         Start           Stop
IDLE ──────────► RUNNING ────────► PAUSED
  ▲                 │                 │
  │     Reset       │     Reset       │
  └─────────────────┴─────────────────┘
                    │
              (reaches 00:00)
                    │
                    ▼
                 IDLE  (+ notify user)
```

**State → button availability**

| State | Start | Stop | Reset |
|---|---|---|---|
| IDLE | enabled | disabled | disabled |
| RUNNING | disabled | enabled | enabled |
| PAUSED | enabled | disabled | enabled |

**DOM targets**

| Element ID | Role |
|---|---|
| `#timer-display` | `MM:SS` countdown text |
| `#timer-start` | Start button |
| `#timer-stop` | Stop button |
| `#timer-reset` | Reset button |
| `#timer-complete-msg` | Hidden message shown on completion |

The controller stores `remainingSeconds` (integer) as its single source of truth. The display is derived from it on every render call.

---

### TodoController

Manages the full Task lifecycle: creation, inline editing, toggle completion, deletion, and re-rendering.

**Rendering model**: The entire task list `<ul>` is re-rendered from the in-memory `tasks` array after every mutation. This keeps render logic simple and stateless.

**Edit mode**: Only one task can be in edit mode at a time. The controller tracks `editingId` (a task `id` string or `null`). When rendering, the task whose `id === editingId` is rendered as an input field; all others are rendered as text.

**DOM targets**

| Element ID | Role |
|---|---|
| `#todo-input` | New task text input |
| `#todo-add-btn` | Add task button |
| `#todo-error` | Inline validation error |
| `#todo-list` | `<ul>` that is fully re-rendered |

**Generated element structure per task**

```html
<li data-id="{id}" class="task-item [task-complete]">
  <!-- Read mode -->
  <input type="checkbox" class="task-checkbox" [checked]>
  <span class="task-text">{text}</span>
  <button class="task-edit-btn">Edit</button>
  <button class="task-delete-btn">Delete</button>

  <!-- Edit mode (replaces above when editingId === id) -->
  <input type="text" class="task-edit-input" value="{text}">
  <button class="task-save-btn">Save</button>
  <button class="task-cancel-btn">Cancel</button>
</li>
```

Event listeners are attached to the `#todo-list` element via **event delegation** — a single `click` and `keydown` listener on the parent `<ul>` handles all task interactions by reading `event.target.closest('[data-id]')`.

---

### QuickLinksController

Manages link creation, deletion, and rendering. No edit-in-place — links are delete-and-recreate only.

**DOM targets**

| Element ID | Role |
|---|---|
| `#link-label-input` | Label field |
| `#link-url-input` | URL field |
| `#link-add-btn` | Add Link button |
| `#link-error` | Inline validation error |
| `#links-container` | Container re-rendered on every mutation |

**Generated element structure per link**

```html
<div class="link-item" data-id="{id}">
  <a href="{url}" target="_blank" rel="noopener noreferrer">{label}</a>
  <button class="link-delete-btn">×</button>
</div>
```

URL validation uses `new URL(value)` inside a `try/catch`. A value that throws is rejected as invalid. Both `http://` and `https://` schemes are accepted; bare hostnames without a scheme are rejected.

---

## Data Models

### Task

```js
/**
 * @typedef {Object} Task
 * @property {string}  id        - Unique identifier (crypto.randomUUID() or Date.now().toString())
 * @property {string}  text      - The task description (non-empty, trimmed)
 * @property {boolean} completed - Whether the task has been marked done
 */
```

**Example**

```json
{
  "id": "1748920000000",
  "text": "Review pull request #42",
  "completed": false
}
```

---

### Link

```js
/**
 * @typedef {Object} Link
 * @property {string} id    - Unique identifier (crypto.randomUUID() or Date.now().toString())
 * @property {string} label - Display label for the button (non-empty, trimmed)
 * @property {string} url   - Fully-qualified URL (passes new URL() validation)
 */
```

**Example**

```json
{
  "id": "1748920001234",
  "label": "GitHub",
  "url": "https://github.com"
}
```

---

### localStorage Schema

| Key | Type | Description |
|---|---|---|
| `tld_tasks` | `JSON string → Task[]` | Serialised array of all Task objects |
| `tld_links` | `JSON string → Link[]` | Serialised array of all Link objects |

**Example stored values**

```
localStorage["tld_tasks"] =
  '[{"id":"1","text":"Buy groceries","completed":false}]'

localStorage["tld_links"] =
  '[{"id":"2","label":"GitHub","url":"https://github.com"}]'
```

If a key is absent (fresh browser or cleared storage), `StorageManager` returns `[]`; the UI renders an empty list, which is valid behaviour (Requirements 7.2 and 9.2).

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Greeting message covers all hours

*For any* integer hour value in [0, 23], the greeting function SHALL return exactly one of the four defined messages ("Good Morning", "Good Afternoon", "Good Evening", "Good Night"), with no hour left unmapped.

**Validates: Requirements 1.3, 1.4, 1.5, 1.6**

---

### Property 2: Greeting hour-to-message mapping is deterministic

*For any* hour value, calling the greeting function twice SHALL return the same message both times (pure function, no side effects).

**Validates: Requirements 1.3, 1.4, 1.5, 1.6**

---

### Property 3: Task whitespace rejection

*For any* string composed entirely of whitespace characters (spaces, tabs, newlines), attempting to add it as a task SHALL be rejected and the task list SHALL remain unchanged.

**Validates: Requirements 3.3**

---

### Property 4: Task addition grows the list by exactly one

*For any* task list state and any valid (non-empty, non-whitespace) task description, adding that task SHALL result in the task list length increasing by exactly one.

**Validates: Requirements 3.2**

---

### Property 5: Task addition round-trip persistence

*For any* valid task added to the list, querying `StorageManager.getTasks()` immediately after SHALL return an array that contains a task with the same `text` and `completed: false`.

**Validates: Requirements 3.4, 7.1**

---

### Property 6: Task completion toggle is its own inverse

*For any* task, toggling its completion state twice SHALL leave the task in its original completion state (i.e., toggle is idempotent over two applications).

**Validates: Requirements 5.2, 5.3**

---

### Property 7: Task edit whitespace rejection

*For any* task and any whitespace-only replacement string, attempting to save the edit SHALL be rejected and the task's text SHALL remain unchanged.

**Validates: Requirements 4.4**

---

### Property 8: Task serialization round-trip

*For any* valid `Task[]` array, serializing it with `JSON.stringify` then deserializing with `JSON.parse` SHALL produce an array that is deeply equal to the original (same `id`, `text`, `completed` for every element).

**Validates: Requirements 7.3**

---

### Property 9: Link serialization round-trip

*For any* valid `Link[]` array, serializing it with `JSON.stringify` then deserializing with `JSON.parse` SHALL produce an array that is deeply equal to the original (same `id`, `label`, `url` for every element).

**Validates: Requirements 9.3**

---

### Property 10: Invalid URL rejection

*For any* string that does not pass `new URL()` validation, attempting to add it as a Quick Link SHALL be rejected and the link list SHALL remain unchanged.

**Validates: Requirements 8.3**

---

### Property 11: Timer display derivation is pure

*For any* integer `remainingSeconds` in [0, 1500] (0 to 25 minutes), formatting it to `MM:SS` SHALL always produce a string of the form `DD:DD` where both parts are zero-padded to two digits.

**Validates: Requirements 2.1, 2.3**

---

### Property 12: Timer decrement stays non-negative

*For any* sequence of decrement operations, `remainingSeconds` SHALL never go below zero. When it reaches zero the timer stops automatically.

**Validates: Requirements 2.6**

---

## Error Handling

### StorageManager

- `JSON.parse` errors on read → log to console, return `[]` (fail-safe, do not crash the page)
- `localStorage.setItem` quota errors → catch `DOMException`, log to console (data loss is acceptable here; no silent data corruption)

### TodoController / QuickLinksController

- Empty / whitespace-only input → show inline `#todo-error` / `#link-error` message; do not modify state or storage
- Input validation happens before any state mutation; state is only updated on valid input

### TimerController

- `clearInterval` is always called before setting a new interval to prevent duplicate timers
- `remainingSeconds` is clamped to `0` before formatting to avoid negative display values

### GreetingController

- `new Date()` is assumed to never throw in supported browsers; no additional error handling required

### QuickLinksController

- URL validation uses `new URL(value)` in a `try/catch`; any thrown `TypeError` is treated as invalid input

---

## Testing Strategy

### Overview

Because this is a Vanilla JavaScript browser application with no build tooling, testing is structured around two complementary layers:

1. **Unit tests** — pure functions extracted from controllers, tested with a lightweight test runner (e.g., [fast-check](https://fast-check.io/) for property-based testing, with plain assertions for examples)
2. **Manual / browser integration tests** — checklist-driven testing in each target browser for DOM interactions, timer behaviour, and localStorage persistence

### Functions Suitable for Pure Unit Testing

The following functions should be extracted as pure, side-effect-free helpers and covered with both example-based and property-based tests:

| Function | Module | Testable Properties |
|---|---|---|
| `getGreetingMessage(hour)` | GreetingController | Properties 1, 2 |
| `formatTime(seconds)` | TimerController | Property 11 |
| `isValidTask(text)` | TodoController | Properties 3, 4 |
| `isValidLink(label, url)` | QuickLinksController | Property 10 |
| `serializeTasks(tasks)` / `deserializeTasks(json)` | StorageManager | Property 8 |
| `serializeLinks(links)` / `deserializeLinks(json)` | StorageManager | Property 9 |

### Property-Based Testing

Use **fast-check** (loaded via CDN for a test HTML file, separate from the production page) to verify the correctness properties listed above.

Each property test is tagged with a comment referencing the design property:

```js
// Feature: todo-life-dashboard, Property 1: Greeting message covers all hours
fc.assert(fc.property(fc.integer({ min: 0, max: 23 }), (hour) => {
  const msg = getGreetingMessage(hour);
  return ['Good Morning', 'Good Afternoon', 'Good Evening', 'Good Night'].includes(msg);
}), { numRuns: 100 });
```

Minimum **100 iterations** per property test.

### Example-Based Unit Tests

Cover specific acceptance criteria that are not amenable to property testing:

- Timer initialises to 25:00 on page load (Requirement 2.1)
- Start button is disabled while timer is RUNNING (Requirement 2.7)
- Stop button is disabled while timer is IDLE or PAUSED (Requirement 2.8)
- Link button opens URL in new tab (Requirement 8.4)
- Empty task list renders without errors (Requirements 7.2, 9.2)

### Cross-Browser Integration Testing

Manual checklist covering:

- Chrome (current stable)
- Firefox (current stable)
- Edge (current stable)
- Safari (current stable)

Checklist items per browser:

1. Clock updates every second and shows correct HH:MM:SS
2. Greeting message changes correctly at each boundary hour
3. Pomodoro timer starts, stops, resets, and fires completion notification at 00:00
4. Tasks persist across a page reload
5. Editing a task and saving reflects the change immediately
6. Quick links open in a new tab
7. No horizontal scrollbar appears at 320px, 768px, 1280px, and 1920px viewport widths

### Accessibility and Visual Checks

- Colour contrast ratio ≥ 4.5:1 verified using browser DevTools accessibility panel
- All interactive elements (buttons, inputs, checkboxes) are keyboard-navigable
- WCAG 2.1 Level AA compliance (note: full validation requires manual testing with assistive technologies)

### Responsive Layout Testing

Use browser DevTools device emulation to verify layout at:

- 320px (minimum per Requirement 10.1)
- 768px (tablet)
- 1280px (laptop)
- 2560px (large desktop)
