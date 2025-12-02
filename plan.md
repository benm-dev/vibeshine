# Complete Architecture Plan: Dual-Client Streaming

## Full System Analysis & Change Plan

---

# PART 1: CURRENT STATE ANALYSIS

## 1.1 System Overview - How It Works Today

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          CURRENT SINGLE-CLIENT ARCHITECTURE                          │
│                                                                                      │
│                                                                                      │
│    MOONLIGHT CLIENT                           VIBESHINE SERVER                       │
│    ────────────────                           ────────────────                       │
│                                                                                      │
│    ┌──────────────┐     GET /serverinfo       ┌──────────────┐                      │
│    │              │ ─────────────────────────►│              │                      │
│    │   NvHTTP     │     currentgame: 0        │   nvhttp     │                      │
│    │              │◄───────────────────────── │              │                      │
│    └──────────────┘     state: FREE           └──────────────┘                      │
│           │                                          │                              │
│           │ GET /launch?appid=1                      │                              │
│           │─────────────────────────────────────────►│                              │
│           │                                          │                              │
│           │     sessionUrl: rtsp://...               │                              │
│           │◄─────────────────────────────────────────│                              │
│           │                                          │                              │
│    ┌──────────────┐                           ┌──────────────┐                      │
│    │   Session    │     RTSP Handshake        │    rtsp      │                      │
│    │              │ ────────────────────────► │              │                      │
│    └──────────────┘                           └──────────────┘                      │
│           │                                          │                              │
│           │                                          ▼                              │
│    ┌──────────────┐                           ┌──────────────┐                      │
│    │   Input      │     Control Stream        │   stream     │                      │
│    │   Handler    │ ────────────────────────► │   session_t  │                      │
│    └──────────────┘     (keyboard/mouse)      └──────────────┘                      │
│           │                                          │                              │
│           │                                          ▼                              │
│    ┌──────────────┐                           ┌──────────────┐                      │
│    │   Video      │◄────────────────────────  │   video      │                      │
│    │   Decoder    │     Video Stream          │   capture    │                      │
│    └──────────────┘                           └──────────────┘                      │
│                                                      │                              │
│                                               ┌──────────────┐                      │
│                                               │   Display    │                      │
│                                               │   (single)   │                      │
│                                               └──────────────┘                      │
│                                                                                      │
│    ═══════════════════════════════════════════════════════════════════════          │
│    PROBLEM: Second client connection triggers quit dialog / terminates first        │
│    ═══════════════════════════════════════════════════════════════════════          │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## 1.2 Key Data Structures

### SERVER (Vibeshine)

| Component | Location | Purpose | Multi-Session Ready? |
|-----------|----------|---------|---------------------|
| `proc::proc` | `process.cpp` | Tracks running app | ❌ Single `_app_id` |
| `session_t` | `stream.cpp` | Streaming session state | ✅ Vector of sessions |
| `running_sessions` | `stream.cpp` | Session count | ✅ Atomic counter |
| `chosen_encoder` | `video.cpp` | Global encoder | ⚠️ Singleton |
| `capture_thread_sync` | `video.cpp` | Capture context | ⚠️ Shared |
| `platf_input` | `input.cpp` | Input injection | ❌ Global singleton |

### CLIENT (Moonlight-Qt)

| Component | Location | Purpose | Multi-Session Ready? |
|-----------|----------|---------|---------------------|
| `currentGameId` | `nvcomputer.h` | Running app ID | ❌ Single value |
| `Session` | `session.h` | Active session | ❌ Singleton pattern |
| `SdlInputHandler` | `input.h` | Input capture | ✅ Per-session |
| `AppModel` | `appmodel.cpp` | App list model | ✅ Data model |

## 1.3 Current Blocking Points Identified

### Server Blockers

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         SERVER BLOCKING POINTS                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  1. SERVERINFO RESPONSE (nvhttp.cpp:1027-1030)                             │
│     ─────────────────────────────────────────                              │
│     Reports single currentgame and binary BUSY/FREE state                  │
│     → Client sees "another app running" → triggers quit dialog             │
│                                                                            │
│  2. CANCEL ENDPOINT (nvhttp.cpp:1645-1670)                                 │
│     ────────────────────────────────────                                   │
│     Terminates ALL sessions unconditionally                                │
│     → No way to quit just one session                                      │
│                                                                            │
│  3. PROCESS TRACKING (process.cpp)                                         │
│     ────────────────────────────                                           │
│     Single _app_id tracks one app                                          │
│     → Can't track multiple concurrent apps                                 │
│                                                                            │
│  4. DISPLAY CAPTURE (video.cpp)                                            │
│     ───────────────────────────                                            │
│     Single display_p index for all sessions                                │
│     → All sessions stream same display                                     │
│                                                                            │
│  5. INPUT HANDLING (input.cpp)                                             │
│     ─────────────────────────                                              │
│     No session role concept                                                │
│     → All sessions can inject input                                        │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Client Blockers

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         CLIENT BLOCKING POINTS                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  1. LAUNCH DECISION (AppView.qml:200-213)                                  │
│     ────────────────────────────────────                                   │
│     if (runningId != 0 && runningId != appId) → show quit dialog           │
│     → Can't launch second app without quitting first                       │
│                                                                            │
│  2. RESUME VS LAUNCH (session.cpp:1592)                                    │
│     ──────────────────────────────────                                     │
│     currentGameId != 0 → use /resume instead of /launch                    │
│     → Wrong endpoint called for second display app                         │
│                                                                            │
│  3. SINGLE CURRENTGAMEID (nvcomputer.h:95)                                 │
│     ─────────────────────────────────────                                  │
│     One integer tracks "the running app"                                   │
│     → No concept of multiple concurrent apps                               │
│                                                                            │
│  4. BUSY STATE DISPLAY (computermodel.cpp:38)                              │
│     ─────────────────────────────────────────                              │
│     Shows computer as "busy" if currentGameId != 0                         │
│     → UX implies can't connect                                             │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

# PART 2: TARGET ARCHITECTURE

## 2.1 Desired End State

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          TARGET DUAL-CLIENT ARCHITECTURE                             │
│                                                                                      │
│    LAPTOP (Moonlight #1)                      VIBESHINE SERVER                       │
│    ─────────────────────                      ────────────────                       │
│    App: "Display 1"                                                                  │
│    Role: PRIMARY                              ┌──────────────────────────────┐       │
│    Input: ENABLED                             │                              │       │
│         │                                     │    SESSION MANAGER           │       │
│         │                                     │    ──────────────────        │       │
│         │   Control Stream                    │                              │       │
│         │─────────────────────────────────────│──► Session 1 (PRIMARY)       │       │
│         │   (input processed)                 │    - Display: 0              │       │
│         │                                     │    - Input: PROCESS          │       │
│         │◄────────────────────────────────────│    - Encoder: NVENC #1       │       │
│         │   Video Stream (Display 1)          │                              │       │
│                                               │                              │       │
│    SURFACE PRO (Moonlight #2)                 │                              │       │
│    ──────────────────────────                 │                              │       │
│    App: "Display 2"                           │                              │       │
│    Role: DISPLAY_ONLY                         │                              │       │
│    Input: DISABLED (or ignored)               │                              │       │
│         │                                     │                              │       │
│         │   Control Stream                    │                              │       │
│         │─────────────────────────────────────│──► Session 2 (DISPLAY_ONLY)  │       │
│         │   (input IGNORED)                   │    - Display: 1              │       │
│         │                                     │    - Input: IGNORE           │       │
│         │◄────────────────────────────────────│    - Encoder: NVENC #2       │       │
│         │   Video Stream (Display 2)          │                              │       │
│                                               └──────────────────────────────┘       │
│                                                             │                        │
│                                                             ▼                        │
│                                               ┌──────────────────────────────┐       │
│                                               │     WINDOWS DESKTOP          │       │
│                                               │     ─────────────────        │       │
│                                               │  ┌────────┐    ┌────────┐    │       │
│                                               │  │Display │◄──►│Display │    │       │
│                                               │  │   1    │    │   2    │    │       │
│                                               │  └────────┘    └────────┘    │       │
│                                               │       Cursor moves freely    │       │
│                                               │       (handled by Windows)   │       │
│                                               └──────────────────────────────┘       │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## 2.2 New Concepts Required

### Session Role Enum

```
┌─────────────────────────────────────────────────────────────────┐
│                      SESSION ROLES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  PRIMARY                                                        │
│  ───────                                                        │
│  • First client to connect (or explicitly requested)            │
│  • Full input privileges (keyboard, mouse, gamepad)             │
│  • Only ONE primary session allowed at a time                   │
│  • Can transfer primary role to another session                 │
│                                                                 │
│  DISPLAY_ONLY                                                   │
│  ────────────                                                   │
│  • Subsequent clients (or explicitly requested)                 │
│  • Video + Audio streaming only                                 │
│  • Input packets received but IGNORED                           │
│  • Multiple display-only sessions allowed                       │
│                                                                 │
│  OBSERVER (Future Enhancement)                                  │
│  ────────                                                       │
│  • View-only of primary session's display                       │
│  • No unique display assignment                                 │
│  • Useful for screen sharing / spectating                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Display Binding Model

```
┌─────────────────────────────────────────────────────────────────┐
│                    DISPLAY BINDING MODEL                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  APP CONFIGURATION                                              │
│  ─────────────────                                              │
│  Each "app" in Vibeshine can specify:                           │
│  • display_index: Which monitor to capture (0, 1, 2, ...)       │
│  • session_role: Default role for sessions launching this app   │
│  • allow_concurrent: Whether multiple sessions can stream this  │
│                                                                 │
│  EXAMPLE APPS.JSON:                                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ {                                                         │  │
│  │   "name": "Display 1 (Primary)",                          │  │
│  │   "display_index": 0,                                     │  │
│  │   "session_role": "primary",                              │  │
│  │   "allow_concurrent": true                                │  │
│  │ },                                                        │  │
│  │ {                                                         │  │
│  │   "name": "Display 2",                                    │  │
│  │   "display_index": 1,                                     │  │
│  │   "session_role": "display_only",                         │  │
│  │   "allow_concurrent": true                                │  │
│  │ }                                                         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# PART 3: SERVER (VIBESHINE) CHANGES

## 3.1 Change Summary Matrix

| Area | File(s) | Change Type | Risk | Effort |
|------|---------|-------------|------|--------|
| Session Roles | `stream.h`, `stream.cpp` | New field + filtering | Low | Small |
| Input Filtering | `stream.cpp` | Add role check | Low | Small |
| ServerInfo Response | `nvhttp.cpp` | Modify XML response | Medium | Small |
| Launch Logic | `nvhttp.cpp` | Allow concurrent apps | Medium | Medium |
| Cancel Logic | `nvhttp.cpp` | Per-session termination | Medium | Medium |
| Display Binding | `video.cpp` | Per-session display | High | Large |
| App Configuration | `config.cpp`, `process.cpp` | New app fields | Medium | Medium |
| Encoder Allocation | `video.cpp` | Multiple encoders | High | Large |

## 3.2 Detailed Change Specifications

### 3.2.1 Session Role System

**Location:** `src/stream.h`, `src/stream.cpp`

**Current State:**
- `session_t` has no role concept
- All sessions treated equally for input

**Required Changes:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CHANGE: Add session_role_e to session_t                                 │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  NEW ENUM in stream.h:                                                   │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  enum class session_role_e {                                       │  │
│  │    PRIMARY,       // Input enabled, first/main session             │  │
│  │    DISPLAY_ONLY   // Video only, input ignored                     │  │
│  │  };                                                                │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  NEW FIELD in session_t:                                                 │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  struct session_t {                                                │  │
│  │    // ... existing fields ...                                      │  │
│  │    session_role_e role;           // NEW                           │  │
│  │    int target_display_index;      // NEW                           │  │
│  │  };                                                                │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  INITIALIZATION in session::alloc():                                     │
│  - Read role from launch_session_t                                       │
│  - Read display_index from launch_session_t                              │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2.2 Input Filtering by Role

**Location:** `src/stream.cpp` (controlBroadcastThread)

**Current State:**
```cpp
server->map(packetTypes[IDX_INPUT_DATA], [&](session_t *session, ...) {
    // All input processed for all sessions
    input::passthrough(session->input, std::move(plaintext));
});
```

**Required Changes:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CHANGE: Filter input by session role                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  MODIFY IDX_INPUT_DATA handler:                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  server->map(packetTypes[IDX_INPUT_DATA], [...](session_t *session,│  │
│  │      const std::string_view &payload) {                            │  │
│  │                                                                    │  │
│  │    // NEW: Role check                                              │  │
│  │    if (session->role != session_role_e::PRIMARY) {                 │  │
│  │      BOOST_LOG(debug) << "Ignoring input from non-primary session";│  │
│  │      return;                                                       │  │
│  │    }                                                               │  │
│  │                                                                    │  │
│  │    // Existing input processing...                                 │  │
│  │    input::passthrough(session->input, std::move(plaintext));       │  │
│  │  });                                                               │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ALSO MODIFY IDX_ENCRYPTED handler:                                      │
│  - Same role check for decrypted input packets                           │
│                                                                          │
│  LOCATIONS TO CHECK:                                                     │
│  - Line ~1014-1038: IDX_INPUT_DATA (legacy unencrypted)                  │
│  - Line ~1040-1104: IDX_ENCRYPTED (encrypted control)                    │
│  - Specifically line ~1097-1100 where IDX_INPUT_DATA is reprocessed      │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2.3 ServerInfo Response Modification

**Location:** `src/nvhttp.cpp` (serverinfo function)

**Current State:**
```cpp
auto current_appid = proc::proc.running();
tree.put("root.currentgame", current_appid);
tree.put("root.state", current_appid > 0 ? "SUNSHINE_SERVER_BUSY" : "SUNSHINE_SERVER_FREE");
```

**Required Changes:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CHANGE: Report multiple active sessions                                 │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STRATEGY: Maintain backwards compatibility while adding extension       │
│                                                                          │
│  MODIFICATION:                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  auto active_sessions = rtsp_stream::get_active_sessions_info();   │  │
│  │  auto primary_appid = get_primary_session_appid(active_sessions);  │  │
│  │                                                                    │  │
│  │  // BACKWARDS COMPATIBLE: Legacy clients see primary session       │  │
│  │  tree.put("root.currentgame", primary_appid);                      │  │
│  │  tree.put("root.state", active_sessions.empty() ?                  │  │
│  │           "SUNSHINE_SERVER_FREE" : "SUNSHINE_SERVER_BUSY");        │  │
│  │                                                                    │  │
│  │  // NEW: Multi-session aware clients can see all sessions          │  │
│  │  tree.put("root.sessionCount", active_sessions.size());            │  │
│  │                                                                    │  │
│  │  // Optional: Detailed session list for advanced clients           │  │
│  │  pt::ptree sessions_node;                                          │  │
│  │  for (auto& s : active_sessions) {                                 │  │
│  │    pt::ptree session_info;                                         │  │
│  │    session_info.put("appid", s.appid);                             │  │
│  │    session_info.put("display", s.display_index);                   │  │
│  │    session_info.put("role", s.role_string);                        │  │
│  │    sessions_node.push_back({"session", session_info});             │  │
│  │  }                                                                 │  │
│  │  tree.add_child("root.activeSessions", sessions_node);             │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  NEW HELPER NEEDED:                                                      │
│  - rtsp_stream::get_active_sessions_info() to return session details     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2.4 Launch Logic Modification

**Location:** `src/nvhttp.cpp` (launch function)

**Current State:**
- Launches app via `proc::proc.execute(appid, launch_session)`
- No check for concurrent display apps

**Required Changes:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CHANGE: Allow concurrent launch for different display apps              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  DECISION LOGIC:                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                                                                    │  │
│  │  GET /launch?appid=X                                               │  │
│  │         │                                                          │  │
│  │         ▼                                                          │  │
│  │  ┌─────────────────────────┐                                       │  │
│  │  │ Is another app running? │                                       │  │
│  │  └───────────┬─────────────┘                                       │  │
│  │              │                                                     │  │
│  │    ┌────────┴────────┐                                             │  │
│  │    │ NO              │ YES                                         │  │
│  │    ▼                 ▼                                             │  │
│  │  Normal           ┌───────────────────────────┐                    │  │
│  │  Launch           │ Is requested app a        │                    │  │
│  │                   │ different-display app?    │                    │  │
│  │                   └─────────────┬─────────────┘                    │  │
│  │                        ┌───────┴───────┐                           │  │
│  │                        │ NO            │ YES                       │  │
│  │                        ▼               ▼                           │  │
│  │                   Return error      ┌───────────────────┐          │  │
│  │                   (app already      │ Allow concurrent  │          │  │
│  │                   running)          │ session creation  │          │  │
│  │                                     │ - Skip proc exec  │          │  │
│  │                                     │ - Create session  │          │  │
│  │                                     │ - Assign display  │          │  │
│  │                                     │ - Assign role     │          │  │
│  │                                     └───────────────────┘          │  │
│  │                                                                    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  KEY INSIGHT:                                                            │
│  - proc::proc.execute() starts a HOST PROCESS (like a game)              │
│  - For display-only streaming, we don't need a new process               │
│  - We just need a new STREAMING SESSION capturing a different display    │
│                                                                          │
│  IMPLEMENTATION APPROACH:                                                │
│  - Check if requested appid is a "display streaming" app                 │
│  - If concurrent display streaming allowed:                              │
│    - DON'T call proc::proc.execute() (no new process needed)             │
│    - DO create new streaming session with different display_index        │
│    - Assign appropriate role (PRIMARY or DISPLAY_ONLY)                   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2.5 Cancel Logic Modification

**Location:** `src/nvhttp.cpp` (cancel function)

**Current State:**
```cpp
void cancel(...) {
    rtsp_stream::terminate_sessions();  // Kills ALL
    if (proc::proc.running() > 0) {
        proc::proc.terminate();
    }
}
```

**Required Changes:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CHANGE: Support per-session termination                                 │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  NEW PARAMETER: sessionId (optional)                                     │
│                                                                          │
│  BEHAVIOR:                                                               │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                                                                    │  │
│  │  GET /cancel                      → Terminate ALL (legacy)         │  │
│  │  GET /cancel?sessionId=<id>       → Terminate specific session     │  │
│  │  GET /cancel?sessionId=primary    → Terminate primary session      │  │
│  │                                                                    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  LOGIC:                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  auto session_id = get_arg(args, "sessionId", "");                 │  │
│  │                                                                    │  │
│  │  if (session_id.empty()) {                                         │  │
│  │    // LEGACY: Terminate all                                        │  │
│  │    rtsp_stream::terminate_sessions();                              │  │
│  │    proc::proc.terminate();                                         │  │
│  │  } else if (session_id == "primary") {                             │  │
│  │    // Terminate primary session, promote next to primary           │  │
│  │    rtsp_stream::terminate_primary_session();                       │  │
│  │  } else {                                                          │  │
│  │    // Terminate specific session by ID                             │  │
│  │    rtsp_stream::terminate_session(session_id);                     │  │
│  │  }                                                                 │  │
│  │                                                                    │  │
│  │  // Only terminate process if NO sessions remain                   │  │
│  │  if (rtsp_stream::session_count() == 0) {                          │  │
│  │    proc::proc.terminate();                                         │  │
│  │  }                                                                 │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  NEW HELPERS NEEDED:                                                     │
│  - rtsp_stream::terminate_session(session_id)                            │
│  - rtsp_stream::terminate_primary_session()                              │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2.6 Per-Session Display Capture

**Location:** `src/video.cpp`

**Current State:**
- Single `display_p` index for all sessions
- One capture thread shared by all sessions
- `capture_thread_sync` or `capture_thread_async` are singletons

**Required Changes:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CHANGE: Per-display capture contexts                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  OPTION A: Multiple Capture Threads (Recommended for Simplicity)         │
│  ─────────────────────────────────────────────────────────────           │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                                                                    │  │
│  │  CURRENT:                                                          │  │
│  │  ┌────────────────────┐                                            │  │
│  │  │ capture_thread     │ ──► All sessions get same display          │  │
│  │  │ (display_p = 0)    │                                            │  │
│  │  └────────────────────┘                                            │  │
│  │                                                                    │  │
│  │  PROPOSED:                                                         │  │
│  │  ┌────────────────────┐                                            │  │
│  │  │ display_capture[0] │ ──► Sessions targeting display 0           │  │
│  │  │ (display_p = 0)    │                                            │  │
│  │  └────────────────────┘                                            │  │
│  │  ┌────────────────────┐                                            │  │
│  │  │ display_capture[1] │ ──► Sessions targeting display 1           │  │
│  │  │ (display_p = 1)    │                                            │  │
│  │  └────────────────────┘                                            │  │
│  │                                                                    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  NEW DATA STRUCTURE:                                                     │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  struct display_capture_ctx_t {                                    │  │
│  │    int display_index;                                              │  │
│  │    capture_thread_async_ctx_t capture_ctx;  // or sync             │  │
│  │    std::vector<session_t*> sessions;                               │  │
│  │    bool active;                                                    │  │
│  │  };                                                                │  │
│  │                                                                    │  │
│  │  // Map of display index to capture context                        │  │
│  │  std::map<int, display_capture_ctx_t> active_display_captures;     │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  LIFECYCLE:                                                              │
│  - Session starts → Find/create capture for target_display_index         │
│  - Session ends → Detach from capture, stop capture if no sessions       │
│                                                                          │
│  OPTION B: Single Capture Thread with Display Switching                  │
│  ─────────────────────────────────────────────────────────               │
│  - More complex, requires capture thread to manage multiple displays     │
│  - Frame routing logic needed                                            │
│  - Not recommended for initial implementation                            │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2.7 App Configuration Extension

**Location:** `src/config.cpp`, `src/process.cpp`, `apps.json`

**Current State:**
- Apps have: name, cmd, working_dir, image_path, etc.
- No display binding or role information

**Required Changes:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CHANGE: Add display and role fields to app configuration                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  NEW FIELDS in proc::ctx_t:                                              │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  struct ctx_t {                                                    │  │
│  │    // ... existing fields ...                                      │  │
│  │                                                                    │  │
│  │    // NEW: Display streaming configuration                         │  │
│  │    bool is_display_app = false;         // True for display-only   │  │
│  │    int display_index = -1;              // Which display (-1=auto) │  │
│  │    std::string default_role = "auto";   // primary/display_only    │  │
│  │    bool allow_concurrent = false;       // Allow multi-session     │  │
│  │  };                                                                │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  APPS.JSON EXAMPLE:                                                      │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  {                                                                 │  │
│  │    "name": "Desktop (All Displays)",                               │  │
│  │    "is_display_app": true,                                         │  │
│  │    "display_index": -1,                                            │  │
│  │    "default_role": "auto",                                         │  │
│  │    "allow_concurrent": false                                       │  │
│  │  },                                                                │  │
│  │  {                                                                 │  │
│  │    "name": "Display 1 (Primary)",                                  │  │
│  │    "is_display_app": true,                                         │  │
│  │    "display_index": 0,                                             │  │
│  │    "default_role": "primary",                                      │  │
│  │    "allow_concurrent": true                                        │  │
│  │  },                                                                │  │
│  │  {                                                                 │  │
│  │    "name": "Display 2",                                            │  │
│  │    "is_display_app": true,                                         │  │
│  │    "display_index": 1,                                             │  │
│  │    "default_role": "display_only",                                 │  │
│  │    "allow_concurrent": true                                        │  │
│  │  }                                                                 │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  PARSING in process.cpp:                                                 │
│  - Read new fields from JSON                                             │
│  - Pass to launch_session_t during launch                                │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2.8 Encoder Allocation

**Location:** `src/video.cpp`

**Current State:**
- Single `chosen_encoder` pointer
- One encoder instance shared

**Required Changes:**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  CHANGE: Allow multiple encoder instances                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  CONSIDERATION: NVENC Session Limits                                     │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Consumer GPUs: 2-3 concurrent NVENC sessions                      │  │
│  │  Professional GPUs: Unlimited                                      │  │
│  │                                                                    │  │
│  │  For Laptop + Surface Pro: Need 2 sessions → OK on consumer GPU    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  APPROACH:                                                               │
│  - Keep encoder SELECTION as singleton (chosen_encoder)                  │
│  - Allow encoder INSTANCES per capture context                           │
│  - Each display_capture_ctx_t has its own encoder instance               │
│                                                                          │
│  CURRENT FLOW:                                                           │
│  1. probe_encoders() selects best encoder → chosen_encoder               │
│  2. capture() creates encode_device from chosen_encoder                  │
│  3. All sessions share this                                              │
│                                                                          │
│  NEW FLOW:                                                               │
│  1. probe_encoders() selects best encoder → chosen_encoder (unchanged)   │
│  2. Each display_capture_ctx creates its own encode_device               │
│  3. Sessions on same display share encoder, different displays don't     │
│                                                                          │
│  RESOURCE MANAGEMENT:                                                    │
│  - Track encoder instance count                                          │
│  - Fail gracefully if NVENC session limit reached                        │
│  - Log warning when approaching limit                                    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

# PART 4: CLIENT (MOONLIGHT-QT) CHANGES

## 4.1 Change Summary Matrix

| Area | File(s) | Change Type | Required? | Risk | Effort |
|------|---------|-------------|-----------|------|--------|
| Multi-App Awareness | `nvcomputer.h/cpp` | Extend data model | Optional | Low | Small |
| Launch Decision | `AppView.qml` | Modify condition | Optional | Low | Small |
| Resume vs Launch | `session.cpp` | Modify decision | Optional | Low | Small |
| Busy State Display | `computermodel.cpp` | Modify role | Optional | Low | Small |
| Display-Only Mode | `streamingpreferences.h` | New preference | Optional | Low | Small |
| Input Suppression | `input/*.cpp` | Skip sending | Optional | Low | Medium |

**KEY INSIGHT:** Client changes are **OPTIONAL** for basic functionality!

With server-side changes alone:
- User runs TWO Moonlight instances (separate processes)
- Each instance connects to a different app ("Display 1", "Display 2")
- Server handles role assignment and input filtering
- No quit dialog because each instance has fresh state

## 4.2 Optional Client Enhancements

### 4.2.1 Multi-Session Awareness (Enhanced UX)

**Purpose:** Show user that multiple sessions are possible

```
┌──────────────────────────────────────────────────────────────────────────┐
│  ENHANCEMENT: Track multiple running apps                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  CURRENT in NvComputer:                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  int currentGameId;  // Single value                               │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ENHANCED:                                                               │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  int currentGameId;           // Keep for backwards compat         │  │
│  │  QList<int> activeAppIds;     // NEW: All running apps             │  │
│  │  int sessionCount;            // NEW: Number of sessions           │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  PARSE from serverinfo:                                                  │
│  - Read new sessionCount field if present                                │
│  - Read activeSessions list if present                                   │
│  - Fall back to currentGameId for legacy servers                         │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.2.2 Smart Launch Decision (Skip Quit Dialog)

**Purpose:** Allow launching different display apps without quit prompt

```
┌──────────────────────────────────────────────────────────────────────────┐
│  ENHANCEMENT: Smart app launch logic                                     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  CURRENT LOGIC (AppView.qml):                                            │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  if (runningId != 0 && runningId != appId) {                       │  │
│  │    // Show quit dialog - BLOCKS concurrent apps                    │  │
│  │    quitAppDialog.open()                                            │  │
│  │  }                                                                 │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ENHANCED LOGIC:                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  if (runningId != 0 && runningId != appId) {                       │  │
│  │    // NEW: Check if concurrent is allowed                          │  │
│  │    if (appModel.isConcurrentAllowed(appId, runningId)) {           │  │
│  │      // Proceed with launch - no quit needed                       │  │
│  │      launchApp(appId)                                              │  │
│  │    } else {                                                        │  │
│  │      // Show quit dialog for incompatible apps                     │  │
│  │      quitAppDialog.open()                                          │  │
│  │    }                                                               │  │
│  │  }                                                                 │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  SERVER MUST PROVIDE:                                                    │
│  - Per-app "allow_concurrent" flag in applist response                   │
│  - Or client infers from app name pattern ("Display X")                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.2.3 Client-Side Display-Only Mode (Efficiency)

**Purpose:** Skip input sending entirely for efficiency

```
┌──────────────────────────────────────────────────────────────────────────┐
│  ENHANCEMENT: Client-side input suppression                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  NEW PREFERENCE in StreamingPreferences:                                 │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  Q_PROPERTY(bool displayOnlyMode MEMBER displayOnlyMode ...)       │  │
│  │  bool displayOnlyMode;  // Don't send any input                    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  MODIFY Input Handlers:                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  void SdlInputHandler::handleMouseMotionEvent(...) {               │  │
│  │    if (m_DisplayOnlyMode) return;  // Skip entirely                │  │
│  │    // ... existing code                                            │  │
│  │  }                                                                 │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  BENEFITS:                                                               │
│  - Reduces network traffic                                               │
│  - Reduces server processing                                             │
│  - Clear user intent (I know this is display-only)                       │
│                                                                          │
│  NOT STRICTLY NECESSARY:                                                 │
│  - Server filters input anyway                                           │
│  - But nice optimization for Surface Pro use case                        │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

# PART 5: INTERACTION PROTOCOL

## 5.1 New Protocol Flow

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          DUAL-CLIENT LAUNCH SEQUENCE                                 │
│                                                                                      │
│  TIME    LAPTOP                 SERVER                  SURFACE PRO                 │
│  ────    ──────                 ──────                  ───────────                 │
│                                                                                      │
│  T0      GET /serverinfo        ─────►                                              │
│          ◄─────────────────────  currentgame: 0                                     │
│                                  state: FREE                                        │
│                                                                                      │
│  T1      GET /launch?appid=1    ─────►                                              │
│          (Display 1 - Primary)         │                                            │
│                                        ▼                                            │
│                                  Create session #1                                  │
│                                  - role: PRIMARY                                    │
│                                  - display: 0                                       │
│          ◄─────────────────────  sessionUrl: rtsp://...                             │
│                                                                                      │
│  T2      RTSP + Streaming       ═════►                                              │
│                                  Session #1 active                                  │
│                                                                                      │
│  T3                                                     GET /serverinfo ────►       │
│                                        │                ◄────────────────────       │
│                                        │                currentgame: 1              │
│                                        │                state: BUSY                 │
│                                        │                sessionCount: 1             │
│                                        │                                            │
│  T4                                                     GET /launch?appid=2 ────►   │
│                                                         (Display 2)                 │
│                                        │                                            │
│                                        ▼                                            │
│                                  Check: Different display app?                      │
│                                  YES → Allow concurrent                             │
│                                  Create session #2                                  │
│                                  - role: DISPLAY_ONLY                               │
│                                  - display: 1                                       │
│                                        │                                            │
│                                        │                ◄────────────────────       │
│                                        │                sessionUrl: rtsp://...      │
│                                                                                      │
│  T5                                                     RTSP + Streaming ════►      │
│                                  Session #2 active                                  │
│                                                                                      │
│  ════════════════════════════════════════════════════════════════════════════════   │
│                                                                                      │
│  BOTH STREAMING CONCURRENTLY:                                                       │
│  - Session #1: Video from Display 0, input PROCESSED                                │
│  - Session #2: Video from Display 1, input IGNORED                                  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## 5.2 Error Handling Scenarios

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          ERROR SCENARIOS                                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SCENARIO 1: Second client requests same display                         │
│  ───────────────────────────────────────────────                         │
│  Client B: GET /launch?appid=1 (Display 1 - already in use)              │
│  Server Response:                                                        │
│  - status_code: 409 (Conflict)                                           │
│  - message: "Display 1 already has an active session"                    │
│  Client B: Show error, suggest different display                         │
│                                                                          │
│  SCENARIO 2: NVENC session limit reached                                 │
│  ──────────────────────────────────────────                              │
│  Client C: GET /launch?appid=3 (Display 3)                               │
│  Server: Encoder allocation fails                                        │
│  Server Response:                                                        │
│  - status_code: 503 (Service Unavailable)                                │
│  - message: "Maximum encoder sessions reached (2/2)"                     │
│  Client C: Show error, explain GPU limitation                            │
│                                                                          │
│  SCENARIO 3: Primary session disconnects                                 │
│  ────────────────────────────────────────                                │
│  Session #1 (PRIMARY) disconnects unexpectedly                           │
│  Server: Promote Session #2 to PRIMARY? Or keep DISPLAY_ONLY?            │
│  Options:                                                                │
│  A) Auto-promote: Session #2 becomes PRIMARY                             │
│  B) No promotion: Session #2 stays DISPLAY_ONLY until explicit request   │
│  RECOMMENDATION: Option B (explicit control, no surprises)               │
│                                                                          │
│  SCENARIO 4: Legacy client connects to multi-session server              │
│  ───────────────────────────────────────────────────────────             │
│  Legacy Moonlight sees currentGameId != 0                                │
│  Legacy Moonlight shows quit dialog                                      │
│  User clicks "Yes" → sends /cancel                                       │
│  Server: Terminates ALL sessions (legacy behavior)                       │
│  MITIGATION: Warn users to use separate Moonlight instances              │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

# PART 6: IMPLEMENTATION PHASES

## Phase 1: Foundation (Server-Only, Minimum Viable)

**Goal:** Enable dual-client streaming with existing Moonlight clients

**Duration:** ~1-2 weeks

```
┌──────────────────────────────────────────────────────────────────────────┐
│  PHASE 1 DELIVERABLES                                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Session Role System                                                  │
│     ☐ Add session_role_e enum                                            │
│     ☐ Add role field to session_t                                        │
│     ☐ Add role field to launch_session_t                                 │
│                                                                          │
│  2. Input Filtering                                                      │
│     ☐ Modify IDX_INPUT_DATA handler                                      │
│     ☐ Modify IDX_ENCRYPTED handler                                       │
│     ☐ Log ignored input for debugging                                    │
│                                                                          │
│  3. App Configuration                                                    │
│     ☐ Add display_index to ctx_t                                         │
│     ☐ Add is_display_app flag                                            │
│     ☐ Add allow_concurrent flag                                          │
│     ☐ Parse new fields from apps.json                                    │
│                                                                          │
│  4. Launch Logic                                                         │
│     ☐ Check for concurrent display app scenario                          │
│     ☐ Skip proc::execute for display-only sessions                       │
│     ☐ Assign role based on app config + existing sessions                │
│                                                                          │
│  5. Testing                                                              │
│     ☐ Configure two display apps in apps.json                            │
│     ☐ Launch Moonlight #1 → Display 1                                    │
│     ☐ Launch Moonlight #2 → Display 2                                    │
│     ☐ Verify both stream simultaneously                                  │
│     ☐ Verify input only works on Display 1                               │
│                                                                          │
│  USER INSTRUCTIONS FOR PHASE 1:                                          │
│  - Add "Display 1 (Primary)" and "Display 2" apps to apps.json           │
│  - Run TWO separate Moonlight instances                                  │
│  - Connect Instance 1 to "Display 1"                                     │
│  - Connect Instance 2 to "Display 2"                                     │
│  - Cursor moves between displays via Display 1 input                     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## Phase 2: Multi-Display Capture (Critical Enhancement)

**Goal:** Each session captures its assigned display

**Duration:** ~2-3 weeks

```
┌──────────────────────────────────────────────────────────────────────────┐
│  PHASE 2 DELIVERABLES                                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Display Capture Architecture                                         │
│     ☐ Design display_capture_ctx_t structure                             │
│     ☐ Create map of display_index → capture_ctx                          │
│     ☐ Implement get_or_create_display_capture()                          │
│                                                                          │
│  2. Per-Display Capture Threads                                          │
│     ☐ Modify start_capture_async/sync for display parameter              │
│     ☐ Implement capture thread per display                               │
│     ☐ Handle display enumeration and selection                           │
│                                                                          │
│  3. Session-Display Binding                                              │
│     ☐ Pass target_display_index through video::capture()                 │
│     ☐ Route session to correct capture context                           │
│     ☐ Handle session cleanup when display capture stops                  │
│                                                                          │
│  4. Encoder Instance Management                                          │
│     ☐ Track encoder instances per display                                │
│     ☐ Implement encoder limit checking                                   │
│     ☐ Graceful failure when limit reached                                │
│                                                                          │
│  5. Testing                                                              │
│     ☐ Verify Display 1 captures monitor 0                                │
│     ☐ Verify Display 2 captures monitor 1                                │
│     ☐ Verify cursor appears on correct display stream                    │
│     ☐ Test encoder limit behavior                                        │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## Phase 3: Protocol Extensions (Improved UX)

**Goal:** Better client experience with multi-session awareness

**Duration:** ~1-2 weeks

```
┌──────────────────────────────────────────────────────────────────────────┐
│  PHASE 3 DELIVERABLES                                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. ServerInfo Extensions                                                │
│     ☐ Add sessionCount field                                             │
│     ☐ Add activeSessions list                                            │
│     ☐ Maintain backwards compatibility                                   │
│                                                                          │
│  2. AppList Extensions                                                   │
│     ☐ Add is_display_app flag to app response                            │
│     ☐ Add allow_concurrent flag                                          │
│     ☐ Add display_index hint                                             │
│                                                                          │
│  3. Cancel Endpoint Extensions                                           │
│     ☐ Add sessionId parameter                                            │
│     ☐ Implement per-session termination                                  │
│     ☐ Maintain legacy behavior for no parameter                          │
│                                                                          │
│  4. Documentation                                                        │
│     ☐ Document new API fields                                            │
│     ☐ Update protocol documentation                                      │
│     ☐ Create multi-display setup guide                                   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## Phase 4: Client Enhancements (Optional)

**Goal:** Native multi-session support in Moonlight

**Duration:** ~2-3 weeks

```
┌──────────────────────────────────────────────────────────────────────────┐
│  PHASE 4 DELIVERABLES (OPTIONAL)                                         │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Multi-Session Awareness                                              │
│     ☐ Parse sessionCount from serverinfo                                 │
│     ☐ Parse activeSessions list                                          │
│     ☐ Update NvComputer model                                            │
│                                                                          │
│  2. Smart Launch Logic                                                   │
│     ☐ Detect display apps from applist                                   │
│     ☐ Skip quit dialog for concurrent-allowed apps                       │
│     ☐ Show concurrent status in UI                                       │
│                                                                          │
│  3. Display-Only Mode                                                    │
│     ☐ Add displayOnlyMode preference                                     │
│     ☐ Skip input sending when enabled                                    │
│     ☐ Show indicator in overlay                                          │
│                                                                          │
│  4. Session Management UI                                                │
│     ☐ Show active sessions count                                         │
│     ☐ Allow quitting specific sessions                                   │
│     ☐ Show session role indicators                                       │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

# PART 7: RISK ANALYSIS

## 7.1 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| NVENC session limit on consumer GPU | Medium | High | Check limit, fail gracefully, document |
| Capture thread resource contention | Medium | Medium | Separate threads per display, careful sync |
| Memory usage with multiple encoders | Low | Medium | Monitor, limit concurrent sessions |
| Input timing issues with filtering | Low | Low | Process input synchronously |
| Display enumeration race conditions | Medium | Medium | Lock during enumeration, retry logic |
| Legacy client breaks multi-session | Medium | High | Detect legacy clients, warn users |

## 7.2 Compatibility Risks

| Scenario | Risk | Mitigation |
|----------|------|------------|
| Old Moonlight + New Vibeshine | Medium | Backwards-compatible serverinfo |
| New Moonlight + Old Vibeshine | Low | Client graceful degradation |
| Mixed client versions | Medium | Document version requirements |
| Existing apps.json configs | Low | New fields are optional |

## 7.3 User Experience Risks

| Risk | Mitigation |
|------|------------|
| Confusion about input control | Clear documentation, role indicators |
| Accidental session termination | Confirm dialogs, session-specific cancel |
| Performance degradation | Resource monitoring, quality scaling |
| Setup complexity | Setup wizard, default configs |

---

# PART 8: SUMMARY

## What Must Change (Server)

1. **Session Roles** - Add role field, filter input by role
2. **Launch Logic** - Allow concurrent display apps
3. **App Config** - Add display binding fields
4. **Display Capture** - Per-display capture threads
5. **ServerInfo** - Report session count (backwards compatible)
6. **Cancel** - Support per-session termination

## What Can Change (Client - Optional)

1. **Session Awareness** - Parse multi-session info
2. **Launch Logic** - Skip quit dialog for display apps
3. **Display-Only Mode** - Client-side input suppression
4. **UI Enhancements** - Show session status

## Simplest Path to Working Dual-Client

**Server changes only, Phase 1:**
1. Add session role field
2. Filter input by role
3. Allow concurrent display app launches
4. Configure "Display 1" and "Display 2" apps

**User workflow:**
1. Run Moonlight Instance 1 → Connect to "Display 1"
2. Run Moonlight Instance 2 → Connect to "Display 2"
3. Both stream, input works only on Display 1
4. Windows cursor moves between displays naturally

---

This architecture provides a clear path from the current single-client model to full dual-client streaming while maintaining backwards compatibility and allowing incremental implementation.