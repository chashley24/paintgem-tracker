# Paint Gem Tracker — Technical Guide

A comprehensive reference for understanding, maintaining, and extending the Paint Gem Tracker application.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture: Why Everything Lives in One File](#2-architecture-why-everything-lives-in-one-file)
3. [Firebase Fundamentals](#3-firebase-fundamentals)
4. [Data Model](#4-data-model)
5. [How the UI Renders](#5-how-the-ui-renders)
6. [Application State](#6-application-state)
7. [Function Reference](#7-function-reference)
8. [Logic Specification](#8-logic-specification)
9. [How to Add a New Feature](#9-how-to-add-a-new-feature)
10. [Environment Setup](#10-environment-setup)
11. [Common Troubleshooting](#11-common-troubleshooting)

---

## 1. Project Overview

The Paint Gem Tracker is a Progressive Web App (PWA) that helps manage a collection of diamond painting kits. It tracks which kits exist, which designs within those kits are completed, and provides tools for choosing which kit to work on next.

**What it does:**

- Manage 150+ diamond painting kits (add, edit, delete)
- Track individual designs within each kit (not started → gemming → completed)
- Pick a random kit to work on next (spinner feature)
- Upload and view photos of completed designs
- Record start and completion dates at both the design and kit level
- Keep notes on individual kits

**How it's hosted:**

- A single HTML file hosted on GitHub Pages (free static hosting)
- Data stored in Google's Firebase cloud services (Firestore for data, Storage for photos)
- Works on any device with a web browser — primarily used on iPhone

---

## 2. Architecture: Why Everything Lives in One File

The entire application — HTML structure, CSS styles, and JavaScript logic — lives in a single `index.html` file. This might seem unusual, but it's a deliberate choice for a project like this:

**Why a single file works here:**

- **Simple deployment**: Push one file to GitHub, and it's live. No build tools, no package managers, no compilation step.
- **No server needed**: GitHub Pages serves static files for free. The file loads in the browser, and Firebase handles all the data storage in the cloud.
- **Easy to share**: You can send someone the file and they can open it directly in a browser.

**The three parts of the file:**

```
┌─────────────────────────────────────────┐
│ <head>                                  │
│   - Meta tags (PWA settings, viewport)  │
│   - Firebase SDK script tags            │
│   - <style> block (all CSS)             │
│ </head>                                 │
│                                         │
│ <body>                                  │
│   - <div id="app"></div>  ← the app     │
│     renders everything inside here      │
│   - Hidden file input for photo uploads │
│   - <script> block (all JavaScript)     │
│ </body>                                 │
└─────────────────────────────────────────┘
```

**How the three parts interact:**

1. **CSS** (lines ~14–81) defines how things look — colors, spacing, animations. It uses CSS custom properties (variables) defined in `:root` so you can change the entire color scheme by modifying a few values at the top.

2. **HTML** is minimal — just `<div id="app"></div>`. The JavaScript dynamically generates ALL of the visible content and places it inside this div.

3. **JavaScript** (lines ~86–352) does everything: connects to Firebase, manages application state, generates HTML strings for each screen, and handles user interactions.

**The key concept**: There are no separate pages. The entire app is one HTML page, and JavaScript swaps out the content inside `<div id="app">` based on what "screen" you're viewing. This is called a **Single Page Application (SPA)**.

---

## 3. Firebase Fundamentals

Firebase is Google's cloud platform for building apps. This project uses two Firebase services:

### 3.1 Firestore (Database)

**What it is:** A cloud database that stores data as documents organized into collections. Think of it like folders (collections) containing files (documents), where each file is a bunch of named fields.

**How we use it:**

```
Firebase Project
  └── Firestore Database
        ├── kits (collection)              ← live app
        │     ├── document_abc123          ← one kit
        │     │     ├── number: 1
        │     │     ├── name: "Ocean Sunset"
        │     │     ├── designs: [...]
        │     │     ├── notes: "..."
        │     │     ├── kitStartDate: 1708646400000
        │     │     └── kitCompletedDate: null
        │     ├── document_def456          ← another kit
        │     └── ...
        ├── pickHistory (collection)       ← live spinner history
        ├── kits_test (collection)         ← test environment
        └── pickHistory_test (collection)  ← test spinner history
```

**Key Firestore concepts:**

- **Collection**: A group of documents. Like a database table. We have `kits` and `pickHistory`.
- **Document**: A single record in a collection. Each kit is one document. Firebase auto-generates a unique ID for each document (like `abc123xyz`).
- **Fields**: Named values in a document. A kit document has fields like `number`, `name`, `designs`, etc.
- **Realtime listeners**: Instead of asking "give me the data" every time, we tell Firestore "watch this collection and tell me whenever anything changes." This is what `onSnapshot` does — it's like a live subscription.

**How data flows between the app and Firestore:**

```
┌──────────────┐         ┌──────────────────┐
│   Browser    │ ──save──→│   Firestore      │
│  (your app)  │         │  (Google cloud)   │
│              │ ←─live──│                   │
│  S.kits = [] │ updates │  kits collection  │
└──────────────┘         └──────────────────┘
```

1. When the app loads, it calls `loadData()` which sets up **realtime listeners** on the `kits` and `pickHistory` collections.
2. Whenever any data changes in Firestore (whether from this device or another), the listener fires and updates `S.kits` in memory, then calls `render()` to refresh the screen.
3. When the user does something (adds a kit, checks a design, etc.), the app calls one of the `fb` functions (`fbAdd`, `fbUp`, `fbDel`) which writes to Firestore.
4. That write triggers the listener, which updates the local state, which triggers a re-render. The app never manually updates `S.kits` after a save — it waits for Firestore to confirm via the listener.

**Offline support**: The line `db.enablePersistence()` tells Firestore to cache data locally. This means the app can load even without internet (using cached data), and any changes made offline will sync when connectivity returns.

### 3.2 Firebase Storage (Photo Storage)

**What it is:** A cloud file storage service (like Google Drive, but for apps). We use it to store uploaded photos.

**Why we need it (instead of just storing photos in Firestore):**

Firestore documents have a **1MB size limit**. Photos, even compressed, are typically 100–200KB each. After about 6 photos stored as base64 strings (text representations of images) directly in a Firestore document, the document exceeds 1MB and all further saves fail silently. Firebase Storage has a 5GB free tier — enough for ~25,000 photos.

**How photo upload works:**

```
User picks photo
       │
       ▼
Browser resizes image (max 1200px, 80% JPEG quality)
       │
       ▼
Upload compressed image to Firebase Storage
  path: photos/{kitId}/{designId}.jpg
       │
       ▼
Get download URL from Storage (a https:// link)
       │
       ▼
Save that URL string to the design's `photo` field in Firestore
```

The Firestore document only stores a small URL string (like `https://firebasestorage.googleapis.com/...`), not the actual image data. When the app displays the photo, it uses this URL as the `src` of an `<img>` tag, and the browser fetches it directly from Storage.

### 3.3 Firebase Configuration

The `firebaseConfig` object at the top of the JavaScript tells the Firebase SDK which project to connect to. These values are:

| Field | Purpose |
|-------|---------|
| `apiKey` | Identifies your project to Google's servers. Despite the name, this is NOT secret — it's designed to be in client-side code. |
| `projectId` | The unique name of your Firebase project |
| `storageBucket` | The URL of your Storage bucket |
| `authDomain`, `databaseURL`, `messagingSenderId`, `appId`, `measurementId` | Other identifiers used by various Firebase services |

**Security comes from rules, not the API key.** Firestore Rules and Storage Rules (configured in the Firebase Console) control who can read and write data. Currently, our rules allow anyone to read/write — which is fine for a personal app but would need to be locked down if this were public-facing.

### 3.4 The Firebase Functions in Our Code

These are the "bridge" functions between our app and Firebase:

| Function | What it does | Firestore operation |
|----------|-------------|-------------------|
| `fbAdd(data)` | Creates a new kit document | `kitsCol.add(data)` |
| `fbUp(id, data)` | Updates fields in an existing kit | `kitsCol.doc(id).update(data)` |
| `fbDel(id)` | Deletes a kit document entirely | `kitsCol.doc(id).delete()` |
| `fbHist(kit)` | Adds a new entry to pick history | `histCol.add({...})` |
| `fbDelHist(id)` | Deletes a pick history entry | `histCol.doc(id).delete()` |

**Important**: `fbUp` uses `.update()` not `.set()`. This means it only changes the fields you specify — it doesn't overwrite the entire document. If you call `fbUp(id, {name: "New Name"})`, only the `name` field changes; `designs`, `notes`, etc. remain untouched.

---

## 4. Data Model

### 4.1 Kit Document

Each kit is a Firestore document with these fields:

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `number` | number | Kit identifier number (user-assigned) | `42` |
| `name` | string | Display name of the kit | `"Ocean Sunset"` |
| `designs` | array | Array of design objects (see below) | `[{...}, {...}]` |
| `notes` | string | Free-text notes about the kit | `"Missing bag #3"` |
| `createdAt` | number | Timestamp when the kit was added (ms since epoch) | `1708646400000` |
| `kitStartDate` | number or null | Timestamp when work began on this kit | `1708646400000` |
| `kitCompletedDate` | number or null | Timestamp when all designs were finished | `null` |

**Note on timestamps:** All dates are stored as milliseconds since January 1, 1970 (Unix epoch). JavaScript's `Date.now()` returns this format. For example, `1708646400000` = February 23, 2024.

### 4.2 Design Object

Each design is an object inside the kit's `designs` array:

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `id` | string | Unique identifier (auto-generated) | `"m1abc2def3"` |
| `name` | string | Display name (empty string = "Design #N") | `"Blue Butterfly"` |
| `status` | string | Current workflow state | `"not_started"`, `"in_progress"`, or `"completed"` |
| `completed` | boolean | Whether the design is done | `true` or `false` |
| `photo` | string | Firebase Storage URL, or empty string | `"https://..."` or `""` |
| `startDate` | number or null | When gemming started on this design | `1708646400000` |
| `completedDate` | number or null | When this design was finished | `1708732800000` |

**Why both `status` and `completed`?** The `completed` boolean is a legacy field from before the three-state workflow was added. It's kept in sync with `status` for backward compatibility — older code that checks `d.completed` still works. The rule is: `completed` is `true` if and only if `status === 'completed'`.

**Backward compatibility for old designs**: Some designs created before the status feature was added won't have a `status` field. The code handles this with a fallback:
```javascript
const st = d.status || (d.completed ? 'completed' : 'not_started');
```
This means: use `status` if it exists, otherwise infer it from `completed`.

### 4.3 Pick History Document

Each spinner pick is a document in the `pickHistory` (or `pickHistory_test`) collection:

| Field | Type | Description |
|-------|------|-------------|
| `kitId` | string | The Firestore document ID of the kit that was picked |
| `kitNumber` | number | Kit number at time of pick (stored separately in case kit is later deleted) |
| `kitName` | string | Kit name at time of pick |
| `timestamp` | number | When the pick happened (ms since epoch) |

### 4.4 Visual: How Data Connects

```
Firestore                           Firebase Storage
────────                           ────────────────
kits_test/                          photos/
  ├── doc_A                           ├── doc_A/
  │   ├── number: 1                   │   ├── design_x.jpg
  │   ├── name: "Sunset"              │   └── design_y.jpg
  │   ├── kitStartDate: 170864...     └── doc_B/
  │   ├── kitCompletedDate: null          └── design_z.jpg
  │   └── designs: [
  │       {
  │         id: "design_x",
  │         status: "completed",
  │         completed: true,
  │         photo: "https://storage.../doc_A/design_x.jpg",
  │                    │
  │                    └──── This URL points to ──────────┐
  │         startDate: 170864...,                         │
  │         completedDate: 170873...                      │
  │       },                                              │
  │       {                                               │
  │         id: "design_y",             Firebase Storage ──┘
  │         status: "in_progress",     has the actual image file
  │         photo: "https://storage.../doc_A/design_y.jpg"
  │       }
  │     ]
  └── doc_B
        └── ...

pickHistory_test/
  ├── hist_1 { kitId: "doc_A", kitNumber: 1, kitName: "Sunset", timestamp: ... }
  └── hist_2 { kitId: "doc_B", ... }
```

---

## 5. How the UI Renders

### 5.1 The Render Loop

The app uses a pattern called **state-driven rendering**. Here's the core idea:

1. There is ONE global state object called `S` that holds everything the app needs to know.
2. There is ONE `render()` function that reads `S` and produces the entire screen.
3. Whenever anything changes, you update `S` and call `render()`.

```
User taps something
       │
       ▼
Event handler updates S (state)
       │
       ▼
render() is called
       │
       ▼
render() checks S.view to decide which screen to show
       │
       ▼
The appropriate screen function (rHome, rDetail, etc.)
  builds an HTML string from the current state
       │
       ▼
app.innerHTML = h  ← the entire page content is replaced
       │
       ▼
User sees the updated screen
```

**This is a complete replacement**, not a partial update. Every time `render()` runs, it throws away ALL existing HTML inside `<div id="app">` and replaces it with freshly generated HTML. This is simpler to reason about (the screen always exactly reflects the state) but means you lose things like scroll position or text cursor position if you're not careful.

### 5.2 The render() Function

```javascript
function render(){
  const app = document.getElementById('app');
  if(!app) return;

  // Show loading screen until data arrives from Firebase
  if(!S.loaded){
    app.innerHTML = '...loading...';
    return;
  }

  // Pick which screen to render based on S.view
  let h = '';
  switch(S.view){
    case 'home':    h = rHome();    break;
    case 'add':     h = rAdd();     break;
    case 'detail':  h = rDetail();  break;
    case 'edit':    h = rEdit();    break;
    case 'picker':  h = rPicker();  break;
    case 'history': h = rHistory(); break;
    case 'gallery': h = rGallery(); break;
  }

  // Append overlays (toast, modals, lightbox) on top
  if(S.toast) h += '...toast HTML...';
  if(S.showDeleteModal) h += '...modal HTML...';
  if(S.showUncompleteModal) h += '...modal HTML...';
  if(S.showDateEditModal) h += '...modal HTML...';
  if(S.showSwitchGemModal) h += '...modal HTML...';
  if(S.lightboxPhoto) h += '...lightbox HTML...';

  // Replace all content at once
  app.innerHTML = h;

  // Post-render setup (e.g., attach swipe handlers)
  if(S.view === 'history') setTimeout(initSwipeHandlers, 50);
}
```

### 5.3 Screen Functions

Each screen is a function that returns an HTML string. They all follow the same pattern:

```javascript
function rHome(){
  let h = '';  // Start with an empty string

  // Build up the HTML piece by piece
  h += '<div class="screen">...';
  h += '<h1>Paint Gem Tracker</h1>';

  // Use state to decide what to show
  if(S.kits.length > 0){
    h += '...kit list...';
  } else {
    h += '...empty state...';
  }

  h += '</div>';
  return h;  // Return the completed HTML string
}
```

**Screen functions and their purposes:**

| Function | Screen | What it shows |
|----------|--------|--------------|
| `rHome()` | Home / Kit List | Currently gemming card, stats, filters, searchable kit list |
| `rAdd()` | Add Kit | Form for adding a new kit (single or bulk mode) |
| `rDetail()` | Kit Detail | Kit header with progress ring, tabs for Designs/Photos/Notes |
| `rEdit()` | Edit Kit | Form to change kit number, name, or total designs |
| `rPicker()` | Random Picker | Spinner animation to randomly select a kit |
| `rHistory()` | Pick History | Timeline of past spinner picks with swipe-to-delete |
| `rGallery()` | Photo Gallery | Grid of all photos across all kits |

### 5.4 Navigation

Navigation between screens is handled by the `nav()` function:

```javascript
function nav(view, kitId){
  S.view = view;                    // Set which screen to show
  if(kitId !== undefined)
    S.selectedKitId = kitId;        // Track which kit is selected
  S.editingDesignId = null;         // Reset any editing state
  S.editingNotes = false;
  if(view === 'detail')
    S.detailTab = 'designs';        // Default to designs tab
  render();                         // Re-render with new state
  window.scrollTo(0, 0);           // Scroll to top
}
```

For example, tapping a kit in the list calls `nav('detail', 'abc123')` which sets the view to the detail screen and tells it which kit to display.

### 5.5 Bottom Navigation Bar

The four tabs at the bottom (Kits, Pick, History, Gallery) are generated by the `bnav()` function. Each tab calls `nav()` with the appropriate view name. The currently active tab is highlighted by comparing against `S.view`.

---

## 6. Application State

The entire app state lives in a single object called `S`:

```javascript
let S = {
  // === DATA (populated by Firebase listeners) ===
  kits: [],              // All kit documents from Firestore
  pickHistory: [],       // All pick history documents
  loaded: false,         // Whether initial Firebase data has arrived

  // === NAVIGATION ===
  view: 'home',          // Current screen: 'home','add','detail','edit','picker','history','gallery'
  selectedKitId: null,   // Which kit is selected for detail/edit views
  detailTab: 'designs',  // Which tab is active in detail view: 'designs','photos','notes'

  // === FILTERS & SEARCH ===
  filter: 'all',         // Kit list filter: 'all','ip' (started),'ns' (not started),'done'
  search: '',            // Current search text

  // === SPINNER ===
  spinning: false,       // Whether the spinner animation is running
  pickerResult: null,    // The kit object that was picked (shown after spin)

  // === ADD KIT FORM ===
  addNumber: '',         // Form field values
  addName: '',
  addTotal: '',
  addCompleted: '0',
  bulkMode: false,       // Whether bulk add mode is on

  // === EDIT KIT FORM ===
  editNumber: '',
  editName: '',
  editTotal: '',

  // === INLINE EDITING ===
  editingDesignId: null, // Which design is being renamed (null = none)
  editingNotes: false,   // Whether the notes textarea is open

  // === UI FEEDBACK ===
  toast: null,           // Toast message to display (null = hidden)
  toastTimer: null,      // Timer ID for auto-hiding toast

  // === PHOTO LIGHTBOX ===
  lightboxPhoto: null,   // Whether the fullscreen photo viewer is open
  lightboxDesignName: '',
  lightboxDesignId: '',

  // === PHOTO UPLOAD ===
  photoTargetDesignId: null,  // Which design the photo picker is uploading to

  // === MODALS ===
  showDeleteModal: false,     // "Delete Kit?" confirmation
  deleteTargetId: null,
  showUncompleteModal: false, // "Mark Incomplete?" confirmation
  uncompleteKitId: null,
  uncompleteDesignId: null,
  showDateEditModal: false,   // Date picker modal
  dateEditTarget: null,       // {type:'kit'|'design', kitId, designId?}
  dateEditValue: '',          // Current value in the date input
  dateEditField: '',          // Which field to update: 'startDate','completedDate','kitStartDate','kitCompletedDate'
  showSwitchGemModal: false,  // "Design In Progress" switch confirmation
  switchGemKitId: null,
  switchGemDesignId: null
};
```

**Key principle**: If it's visible on screen or affects what's visible, it's in `S`. This makes the app predictable — you can always look at `S` to understand what the user should be seeing.

---

## 7. Function Reference

### 7.1 Helper Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `gid()` | Generate a unique ID for a new design | `"m1abc2def3"` |
| `showToast(message)` | Show a temporary notification at the bottom | `showToast('Photo added!')` |
| `mkDesigns(count, completed)` | Create an array of fresh design objects | `mkDesigns(12, 3)` → 12 designs, first 3 completed |
| `ks(kit)` | Calculate kit stats | Returns `{c:3, t:12, p:25, done:false, hasGem:true}` |
| `overall()` | Calculate stats across all kits | Returns `{tk:100, ck:5, ip:10, ns:85, td:1200, cd:150}` |
| `filtered()` | Get kits matching current filter + search | Returns array of kit objects |
| `E(string)` | HTML-escape a string (prevent injection) | `E('<script>')` → `'&lt;script&gt;'` |
| `fmtDate(timestamp)` | Format as "Jan 15, 2025" | Human-readable date |
| `fmtDateTime(timestamp)` | Format as "Jan 15, 2025 · 3:30 PM" | Date with time |
| `daysBetween(a, b)` | Count days between two timestamps | `daysBetween(start, end)` → `14` |
| `daysAgo(timestamp)` | Relative time string | `"today"`, `"1 day ago"`, `"5 days ago"` |
| `toInputDate(timestamp)` | Convert timestamp to `YYYY-MM-DD` for date inputs | `"2025-01-15"` |
| `fromInputDate(string)` | Convert `YYYY-MM-DD` back to timestamp | Returns milliseconds |
| `ring(completed, total, size)` | Generate SVG progress ring | Returns SVG HTML string |
| `sparkles()` | Trigger celebratory animation | 30 colored dots fall from top |
| `findGemming()` | Find the currently in-progress design | Returns `{kit, design, idx}` or `null` |
| `selKit()` | Get the currently selected kit | Uses `S.selectedKitId` |

### 7.2 Stats Functions Explained

**`ks(kit)`** — Kit Stats:

```javascript
// Input: a kit object
// Output: { c, t, p, done, hasGem }
//   c = completed design count
//   t = total design count
//   p = percentage (0-100)
//   done = true if ALL designs are completed
//   hasGem = true if any design has status 'in_progress'
```

**`overall()`** — All-Kits Stats:

```javascript
// Output: { tk, ck, ip, ns, td, cd }
//   tk = total kits
//   ck = complete kits (all designs done)
//   ip = in-progress kits (some progress OR has gemming, but not 100%)
//   ns = not-started kits (zero completed, no gemming)
//   td = total designs across all kits
//   cd = completed designs across all kits
```

---

## 8. Logic Specification

This section documents every business rule in the app. Use this to verify behavior matches your expectations.

### 8.1 Design Status Flow

A design has three possible states, forming a one-way cycle:

```
                    ┌───────────────────────────────┐
                    │                               │
                    ▼                               │
             ┌─────────────┐     ┌─────────────┐   │
             │ NOT STARTED │────→│  GEMMING IN  │   │
             │             │ tap │  PROGRESS    │   │
             └─────────────┘     └──────┬───────┘   │
                    ▲                   │ tap        │
                    │            ┌──────▼───────┐   │
                    │            │  COMPLETED   │   │
                    └────────────│              │───┘
                   confirm modal └──────────────┘
```

**Rules:**

| Action | What happens | Automatic side effects |
|--------|-------------|----------------------|
| Tap a NOT STARTED design (no other design gemming) | Status → `in_progress`, `startDate` set to now | `kitStartDate` set to now (if not already set) |
| Tap a NOT STARTED design (another design IS gemming) | **Modal appears** asking to switch or keep current | If user taps "Switch": old design → `not_started`, new design → `in_progress` |
| Tap a GEMMING IN PROGRESS design | Status → `completed`, `completed` → true, `completedDate` set to now | If this was the last design: `kitCompletedDate` set to now, sparkle animation plays |
| Tap a COMPLETED design | **Modal appears** asking to confirm marking incomplete | If confirmed: status → `not_started`, `completed` → false, `completedDate` → null. If kit was complete: `kitCompletedDate` → null |

### 8.2 Gemming In Progress — Global Constraint

**RULE: Only ONE design across ALL kits can be "Gemming In Progress" at any time.**

- The `findGemming()` function scans every kit to find the active design.
- When starting a new gemming design, the code checks if one already exists.
- If in the **same kit**: The old design is reverted to `not_started` in the same database write.
- If in a **different kit**: A confirmation modal appears. If the user confirms the switch, the old design is reverted via a separate database write to its kit, then the new one is started.

### 8.3 Kit Status Categories

A kit falls into exactly one category:

| Category | Condition | Filter pill |
|----------|-----------|-------------|
| **Complete** | All designs have `completed: true` (and at least one design exists) | "Complete" |
| **Started** | At least one design is `completed` OR any design has `status: 'in_progress'`, but NOT all designs are completed | "Started" |
| **Not Started** | Zero designs completed AND no design is in progress | "Not Started" |

These categories are **mutually exclusive** — a complete kit does NOT appear under "Started."

### 8.4 Kit Dates

| Date | When it's auto-set | Can be manually edited? |
|------|-------------------|----------------------|
| `kitStartDate` | When the first design in the kit is set to "Gemming In Progress" | Yes — tap the date in the kit detail header |
| `kitCompletedDate` | When the last design in the kit is marked as completed | Yes — tap the date in the kit detail header |

**Additional rules:**
- If a user marks a completed design as incomplete and the kit had a `kitCompletedDate`, the kit completion date is cleared (set to null).
- Kit dates are **not** automatically cleared if all designs are reverted to not-started (only `kitCompletedDate` is cleared when un-completing a design).
- The "+ Set start date" link appears in the kit header if no `kitStartDate` exists, allowing manual entry for kits that were started before the tracking feature existed.

### 8.5 Design Dates

| Date | When it's auto-set | Can be manually edited? |
|------|-------------------|----------------------|
| `startDate` | When the design is set to "Gemming In Progress" | Yes — tap "date" button on in-progress designs |
| `completedDate` | When the design is tapped to mark complete (from in-progress state) | Yes — tap "dates" button on completed designs |

**Additional rules:**
- `startDate` is preserved if a design is switched away from (another design takes over gemming). This means if you start gemming Design A, switch to Design B, then come back to Design A, the original start date is kept.
- When a completed design is marked incomplete, `completedDate` is cleared but `startDate` is **not** — this is intentional because the start date represents when work actually began.
  - **Note**: Currently the uncomplete action clears `completedDate` and sets `status` back to `not_started`. The `startDate` is preserved so if she re-starts it, the original start date remains.

### 8.6 Stat Card Display

The home screen shows three stat cards:

| Card | Number shown | Label |
|------|-------------|-------|
| Left (green) | Count of complete kits | "X of Y Complete" |
| Middle (purple) | Count of started-but-not-complete kits | "X of Y Started" |
| Right (gray) | Count of not-started kits | "Not Started" |

### 8.7 Photo Upload Rules

- Photos are resized to a maximum of 1200px on the longest side, maintaining aspect ratio.
- Compressed to JPEG at 80% quality.
- Uploaded to Firebase Storage at path `photos/{kitId}/{designId}.jpg`.
- The download URL is saved to the design's `photo` field in Firestore.
- When a photo is removed, it's deleted from both Storage and Firestore.
- There is no limit on the number of photos per kit (Storage-based approach).

### 8.8 Spinner / Random Picker

- Only kits that are NOT complete are included in the spinner pool.
- The spinner runs through kit names visually for 25–40 ticks, slowing down at the end.
- The winning kit is pre-determined randomly before the animation starts.
- Each spin is logged to the `pickHistory` collection with the kit's name, number, and current timestamp.
- Re-rolling logs a new entry (the previous spin is also kept).
- History entries can be removed via swipe-to-delete.

### 8.9 Kit Add / Edit Rules

**Adding:**
- Kit number and total designs are required.
- Kit name defaults to "Kit #N" if left empty.
- "Already Completed" pre-marks the first N designs as completed (status: `completed`).
- Bulk mode keeps the form open and auto-increments the kit number.

**Editing:**
- Changing total designs adds new blank designs to the end or removes from the end.
- When reducing designs, completed designs are sorted to the front (preserved), and uncompleted ones are removed from the end.

### 8.10 Notes

- Notes are free-text stored on the kit document.
- Notes are edited inline on the Notes tab of the kit detail screen.
- The notes indicator (●) appears on the tab when notes exist.

---

## 9. How to Add a New Feature

Here's a step-by-step walkthrough of adding a hypothetical feature: **"Favorite Kits"** — the ability to mark kits as favorites and filter by them.

### Step 1: Plan the Data

What new data do you need? A `favorite` boolean on each kit.

```
Kit document:
  └── favorite: true/false (new field)
```

### Step 2: Update State (if needed)

If the feature needs UI state (like a "show favorites" toggle), add it to `S`:

```javascript
let S = {
  // ... existing fields ...
  showFavoritesOnly: false,   // NEW
};
```

### Step 3: Add the Action Function

Write a function that handles the user interaction:

```javascript
function toggleFavorite(kitId) {
  const k = S.kits.find(x => x.id === kitId);
  if (!k) return;
  fbUp(kitId, { favorite: !k.favorite });
  showToast(k.favorite ? 'Removed from favorites' : 'Added to favorites!');
}
```

Key pattern: **Don't update `S.kits` directly.** Call `fbUp()` to write to Firestore, and the realtime listener will update `S.kits` and trigger a re-render automatically.

### Step 4: Update the Render

Modify the relevant screen function to show the new UI. For example, in `renderKitItems()`, add a star icon:

```javascript
h += `<div class="kit-item" onclick="nav('detail','${k.id}')">
  ${ring(s.c, s.t, 44)}
  <div style="flex:1">
    <div>${k.favorite ? '⭐ ' : ''}#${k.number} — ${E(k.name)}</div>
  </div>
  <button onclick="event.stopPropagation(); toggleFavorite('${k.id}')">
    ${k.favorite ? '★' : '☆'}
  </button>
</div>`;
```

**Important**: `event.stopPropagation()` prevents the button tap from also triggering the parent div's `onclick` (which navigates to the kit detail).

### Step 5: Update Filters (if applicable)

If you want a "Favorites" filter pill, update the `filtered()` function:

```javascript
function filtered() {
  return S.kits
    .filter(k => {
      if (S.filter === 'fav') return k.favorite;
      // ... existing filters ...
    })
    .filter(k => { /* search filter */ });
}
```

And add the pill to `rHome()`:

```javascript
['all','All'], ['fav','Favorites'], ['ip','Started'], ...
```

### Step 6: Test

1. Push to `test.html` on your repo.
2. Open in browser, test all scenarios.
3. Check the Firebase Console to verify data looks correct.
4. When satisfied, merge into `index.html`.

### General Tips for Adding Features

- **Always modify data through `fbUp()`** — never modify `S.kits` directly.
- **Call `render()` only for local UI changes** (like opening a modal). Data changes flow automatically through the Firestore listener.
- **Use `event.stopPropagation()`** on buttons inside clickable containers to prevent double-actions.
- **New CSS classes** go in the `<style>` block at the top of the file.
- **New state variables** go in the `S` object.
- **New modals** get rendered at the bottom of the `render()` function (after all screens but before `app.innerHTML = h`).

---

## 10. Environment Setup

### 10.1 Live vs Test

| | Live (her app) | Test (your testing) |
|---|---|---|
| **File** | `index.html` | `test.html` |
| **URL** | `username.github.io/repo/` | `username.github.io/repo/test.html` |
| **Kits collection** | `kits` | `kits_test` |
| **History collection** | `pickHistory` | `pickHistory_test` |
| **Visual indicator** | None | Red "TEST" badge |

Both use the same Firebase project, same API keys, and same Storage bucket. The only difference is which Firestore collections they read/write.

### 10.2 Switching Test to Live

When a feature is ready to go live:

1. In `test.html`, change `kits_test` → `kits` and `pickHistory_test` → `pickHistory`
2. Remove the red TEST badge from the title
3. Copy the contents into `index.html`
4. Commit and push to `main`

### 10.3 Firebase Console

Access at: https://console.firebase.google.com

- **Firestore Database**: View/edit documents, manage rules
- **Storage**: View uploaded photos, manage rules
- **Rules**: Firestore rules and Storage rules are configured separately

### 10.4 Firestore Rules (Current)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /kits/{kitId} { allow read, write: if true; }
    match /pickHistory/{docId} { allow read, write: if true; }
    match /kits_test/{kitId} { allow read, write: if true; }
    match /pickHistory_test/{docId} { allow read, write: if true; }
  }
}
```

### 10.5 Storage Rules (Current)

```
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /{allPaths=**} {
      allow read, write: if request.time < timestamp.date(2026, 3, 23);
    }
  }
}
```

**⚠️ Expires March 23, 2026.** Replace before then with permanent rules:
```
match /photos/{kitId}/{fileName} { allow read, write: if true; }
```

---

## 11. Common Troubleshooting

### "Photos stopped uploading"

- **Check Storage rules expiration** — if past March 23, 2026, rules have expired.
- **Check browser console** (F12 → Console) for error messages.
- **Verify Storage is enabled** in Firebase Console → Storage.

### "App shows loading forever"

- **Internet connection** — the app needs to fetch data from Firebase on first load.
- **Check Firestore rules** — if rules were changed, access might be blocked.
- **Clear browser cache** — cached offline data might be corrupted. Try incognito mode.

### "Changes aren't saving"

- **Check Firestore rules** — write access might be blocked.
- **Check browser console** for "Save failed" toast or errors.
- **Document size** — if somehow a document exceeds 1MB, all saves to it will fail.

### "The page looks broken on her phone"

- **Force refresh** — have her swipe down to refresh, or close and reopen the browser.
- **Clear Safari cache** — Settings → Safari → Clear History and Website Data.
- **Check GitHub Pages** — make sure the latest commit has deployed (Settings → Pages shows last deploy time).

### "I accidentally broke the live version"

- **Git revert** — use `git revert` or `git checkout` to go back to a previous commit.
- **GitHub web UI** — you can view file history and restore previous versions directly on github.com.

### How to debug

1. Open the app in Chrome on your computer.
2. Press F12 to open Developer Tools.
3. Go to the **Console** tab to see errors.
4. Go to the **Application** tab → **Firestore** to inspect cached data.
5. Go to the **Network** tab to see Firebase requests.

---

## Glossary

| Term | Meaning |
|------|---------|
| **Firestore** | Google's cloud database service. Stores kit and history data as documents in collections. |
| **Firebase Storage** | Google's cloud file storage. Stores uploaded photos as actual image files. |
| **Collection** | A group of documents in Firestore (like a database table). |
| **Document** | A single record in a Firestore collection (like a row in a table). |
| **SPA** | Single Page Application — the entire app runs in one HTML page without full page reloads. |
| **State** | The `S` object — all the data the app needs to determine what to show on screen. |
| **Render** | The process of generating HTML from the current state and placing it on the page. |
| **Listener** | A Firestore `onSnapshot` callback that fires whenever data in a collection changes. |
| **PWA** | Progressive Web App — a website that can be "installed" on a phone's home screen and work somewhat like a native app. |
| **Base64** | A way to encode binary data (like images) as text. The old photo storage method that hit the 1MB limit. |
| **Timestamp** | A number representing a point in time as milliseconds since January 1, 1970. |
| **DID** | Design ID — the unique identifier string for a design within a kit. |
| **KID** | Kit ID — the Firestore document ID for a kit. |
