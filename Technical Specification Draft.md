# **ChanterSync — Technical Specification (v0 .1)**

_(Internal engineering document – exhaustive detail)_

---

## Table of Contents

[[#1 . Project Rationale & Scope]]
[[#2 . Functional Requirements]]
[[#3 . Non-Functional Requirements]]
[[#4 . Domain Model (Conceptual)]]
[[#5 . High-Level Architecture]]
[[#6 . Subsystem Specifications]]
[[#7 . Data Model (SQL DDL subset)]]
[[#8 . Key Application Flows]]
[[#9 . Concurrency & Conflict-Resolution]]
[[#10 . Offline Strategy]]
[[#11 . Security & Compliance]]
[[#12 . Operational Concerns]]
[[#13 . Extensibility Roadmap]]   

_(Throughout this spec, object / table names are **PascalCase**; REST routes are **kebab-case**.)_

---

<a name="1"></a>
## 1 . Project Rationale & Scope

### 1.1 Problem Statement

Orthodox choirs routinely juggle PDFs stored across disparate sources (monastery sites, Drive links, e-mails). In liturgy, multiple tablets (iPadOS & Android) must surface the correct chant instantly in English, Arabic, and occasionally Greek. Current pain-points:

- **File fragmentation** → last-minute searching
    
- **Inconsistent PDF viewers** → wrong app, layout breaks
    
- **Page coordination** → choirmaster must micro-manage flips
    
- **Device fragility** → battery / crash = singer offline
    
- **Network jitters** → slow downloads mid-service
    

### 1.2 Proposed Solution

A cross-platform Progressive Web App (installable PWA) plus thin native shell (future) that:

- Holds a **central music library** (object store + metadata DB)
    
- Allows a **service set list** to be prepared remotely
    
- Lets a **leader session** broadcast one or more PDF “tabs” (English, Arabic, Greek, etc.)
    
- Pushes the correct documents to every active tablet; each tablet scrolls freely
    
- Enables **two-tap role reassignment** when devices fail
    
- Works **offline** once files are cached
    

### 1.3 Out-of-Scope (v0 .1)

- Synchronized page/measure highlighting
    
- Audio playback / pitch reference
    
- Rich per-user social graph (friends, chat)
    
- Native iOS / Android binaries (wrapped later via Capacitor/EAS or similar)
    

---

<a name="2"></a>
## 2 . Functional Requirements

| #     | Requirement                                                                  | Priority |
| ----- | ---------------------------------------------------------------------------- | -------- |
| FR-01 | Upload PDF/ZIP via UI or drag-and-drop                                       | P0       |
| FR-02 | Tag files with tone, feast, language                                         | P0       |
| FR-03 | Build **SetList** ordered collection; save draft; publish                    | P0       |
| FR-04 | Start **ServiceInstance** that references a SetList                          | P0       |
| FR-05 | Display up to _N_ active **Tabs** (default 1; tested with 3)                 | P0       |
| FR-06 | Broadcast PDF(s) to all sessions for given Tab                               | P0       |
| FR-07 | Device selects **ProfileType** (choirmaster/assistant/chanter) with 2 taps   | P0       |
| FR-08 | Single active **LeaderToken** enforced via heartbeat                         | P0       |
| FR-09 | Tablets auto-download PDFs for the service; no blocking on Wi-Fi once cached | P0       |
| FR-10 | Hot-swap device: new tablet claims choirmaster role in ≤ 15 s                | P0       |
| FR-11 | Local private annotations (per device, not synced)                           | P2       |
| FR-12 | Night mode (UI + PDF invert)                                                 | P2       |

---

<a name="3"></a>
## 3 . Non-Functional Requirements

- **Cross-Platform:** Safari (iPadOS ≥ 15), Chrome/Edge/Firefox (desktop & Android).
    
- **Offline Mode:** All PDFs + core UI reachable without network once service started.
    
- **Low Latency Broadcast:** < 200 ms median for `broadcast` → client update within LAN.
    
- **Horizontal Scalability:** 1,000 concurrent WebSocket connections per service cluster.
    
- **Cold Start ≤ 3 s** on 2016 iPad (A9) for PWA launch.
    
- **Storage Cost Ceiling:** ≤ US$0.02 per GB-month (object store + CDN egress).
    
- **Data Residency:** All sensitive user data stored in single region (Toronto) for PIPEDA.
    
- **99.9 % availability** during 07:00-14:00 local Sunday window.
    

---

<a name="4"></a>
## 4 . Domain Model (Conceptual)

```
Team (1) ─── (∞) LibraryFile
Team (1) ─── (∞) Tag
Team (1) ─── (∞) SetList
SetList (1) ─── (∞) SetListItem → LibraryFile

ServiceInstance (1) ─── (∞) DeviceSession
ServiceInstance (1) ── (1) BroadcastState

DeviceSession (∞) ─── (1) Team
DeviceSession (∞) ─── (1) ProfileType (enum)
```

_`BroadcastState` stores composite JSON: `{ tabs: [ {id, language, fileId} ], leaderDeviceId, updatedAt }`_

---

<a name="5"></a>
## 5 . High-Level Architecture

```
 ┌───────────────────────────────────────────────────────────────────────┐
 │                          Client PWA (Chromium/Safari)                 │
 │   • React/Vue SSG bundle                                              │
 │   • Service Worker (offline, cache)                                   │
 │   • PDF.js embedded viewer                                            │
 │   • IndexedDB persistent store                                        │
 └───────────────────────────────────────────────────────────────────────┘
                     ▲                ▲                   ▲
   HTTPS / REST      │   WebSocket    │ Signed URLs (S3)  │
                     │                │                   │
 ┌───────────────────┴────────────────┴───────────────────┴──────────────┐
 │                **API Gateway / Real-Time Hub**                        │
 │   • HTTP routing (REST/GraphQL)                                       │
 │   • WS broker (fan-out per ServiceInstance)                           │
 │   • Auth middleware (JWT/Cookie)                                      │
 └───────────────────────────────────────────────────────────────────────┘
                     │                │
         gRPC/REST   │                │   Event bus (NATS/Kafka)
                     ▼                ▼
 ┌─────────────────────────┐  ┌─────────────────────────┐  ┌────────────┐
 │  Core App Service       │  │  Auth & Session Service │  │  Workers   │
 │  • Business logic       │  │  • Magic-link issuance  │  │  • File    │
 │  • LeaderToken mgr      │  │  • Token verification   │  │    virus   │
 │  • SetList CRUD         │  │  • Role policy          │  │    scan    │
 └─────────────────────────┘  └─────────────────────────┘  └────────────┘
                     │                │
             SQL (RDS/PG)             │
                     ▼                │
 ┌─────────────────────────┐  ┌─────────────────────────┐
 │  Relational Database    │  │   Object Storage + CDN  │
 │  • Teams, Files, Tags   │  │   (S3-compatible)       │
 │  • ServiceInstance      │  │   • Origin bucket       │
 │  • BroadcastState       │  │   • Signed GET for PDFs │
 └─────────────────────────┘  └─────────────────────────┘
```

---

<a name="6"></a>
## 6 . Subsystem Specifications

### 6.1 Front-End (PWA Client)

| Module                              | Responsibilities                                                                                                        |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **AppShell**                        | Routing, error boundaries, responsive layout (tablet-first 1024×1366).                                                  |
| **AuthGate**                        | Consumes JWT from cookie / localStorage; if absent, shows **RolePicker**.                                               |
| **RolePicker**                      | Three buttons: Choirmaster, Assistant, Chanter → POST `/sessions` to obtain `deviceSessionId` + JWT.                    |
| **LibraryView**                     | CRUD for LibraryFile & Tag (if profile ≥ assistant). Drag-upload; progress indicators; virus scan pending state.        |
| **SetListBuilder**                  | Drag-sortable list; auto-save draft; “Publish” posts `/set-lists/:id/publish`.                                          |
| **ServiceController (Leader-side)** | Start/Stop service, create tabs, broadcast LibraryFile to tab.                                                          |
| **TabBar & PDFPane**                | For each active tab render PDF in `<canvas>` via PDF.js; independent scroll context; local page state in IndexedDB.     |
| **BannerStatus**                    | Shows current hymn title + page range; highlights if user scrolls outside range.                                        |
| **WSService**                       | Wraps WebSocket; subscribes to `broadcast:{serviceId}` channel; debounced heartbeat emit.                               |
| **OfflineCache**                    | Service Worker `install` caches AppShell; `fetch` intercepts PDFs—if `Cache.match()` exists, serve; else fetch & cache. |

### 6.2 API Gateway / Real-Time Hub

- **TLS termination** (ALB/Ingress).
    
- **REST paths** (example):
    
    - `POST /sessions` → returns JWT w/ `deviceSessionId`, `profileType`
        
    - `POST /service-instances/:id/leader-heartbeat`
        
    - `POST /broadcast` body `{tabId, fileId}`
        
    - `GET /library-files?tag=Pascha&language=AR`
        
- **WebSocket multiplexing**: subprotocol `csync-v1`; first message includes `deviceSessionId`.
    
- On `broadcast` publish: JSON diff pushed to all devices in service room.
    

### 6.3 Core Application Service

- **LeaderToken Manager**
    
    - Acquire: atomic `UPDATE ... WHERE leaderDeviceId IS NULL OR updatedAt < now() - 45s`.
        
    - Heartbeat: leader posts every 15 s; failure to update releases token.
        
- **SetList Engine**
    
    - `status: draft | published | archived`.
        
    - Publishing freezes order and emits `setlistPublished` event → offline prefetch tasks.
        
- **BroadcastState**
    
    - JSON: `{tabs:[{tabId, fileId, label, sort}], version, leaderId}`; bump `version` every broadcast to bust SW cache if needed.
        

### 6.4 Storage Layer

- **Object Storage**
    
    - Bucket naming: `music-prod-{teamId}`.
        
    - PDF max size 25 MiB (configurable).
        
    - Lifecycle rule: move >365-day unused to Glacier/DeepArchive.
        
- **CDN**: signed URL (HMAC) valid 7 days; delivered via edge PoP for low latency.
    

### 6.5 Authentication / Session Service

- **Magic-Link** (later). For v0.1, **anonymous team secret** embedded in RolePicker until hardened.
    
- JWT Claims: `iss, exp, teamId, deviceSessionId, profileType, leaderEligible (bool)`.
    

### 6.6 Background Workers

|Worker|Trigger|Task|
|---|---|---|
|**VirusScan**|`libraryFile.created`|Scan via ClamAV; tag `quarantine` if fail.|
|**ThumbnailGen**|`libraryFile.created`|Render first page → PNG 256× etc.|
|**PrefetchWarmup**|`setlistPublished`|Queue CDN pre-warm for each LibraryFile.|

---

<a name="7"></a>
## 7 . Data Model (SQL DDL subset)

```sql
CREATE TABLE Teams (
  id UUID PRIMARY KEY,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL
);

CREATE TYPE ProfileType AS ENUM ('choirmaster','assistant','chanter');

CREATE TABLE DeviceSessions (
  id UUID PRIMARY KEY,
  team_id UUID REFERENCES Teams(id),
  profile_type ProfileType NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  last_heartbeat TIMESTAMPTZ
);

CREATE TABLE LibraryFiles (
  id UUID PRIMARY KEY,
  team_id UUID REFERENCES Teams(id),
  file_key TEXT UNIQUE NOT NULL,
  original_filename TEXT,
  language CHAR(2),       -- EN, AR, EL
  page_count INT,
  size_bytes INT,
  uploaded_by UUID REFERENCES DeviceSessions(id),
  uploaded_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE Tags (
  id SERIAL PRIMARY KEY,
  team_id UUID REFERENCES Teams(id),
  name TEXT
);

CREATE TABLE FileTags (
  file_id UUID REFERENCES LibraryFiles(id),
  tag_id INT REFERENCES Tags(id),
  PRIMARY KEY (file_id, tag_id)
);

CREATE TABLE SetLists (
  id UUID PRIMARY KEY,
  team_id UUID REFERENCES Teams(id),
  title TEXT,
  status TEXT CHECK (status IN ('draft','published','archived')),
  created_by UUID REFERENCES DeviceSessions(id),
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE SetListItems (
  id SERIAL PRIMARY KEY,
  setlist_id UUID REFERENCES SetLists(id),
  file_id UUID REFERENCES LibraryFiles(id),
  ordinal INT
);

CREATE TABLE ServiceInstances (
  id UUID PRIMARY KEY,
  team_id UUID REFERENCES Teams(id),
  setlist_id UUID REFERENCES SetLists(id),
  started_by UUID REFERENCES DeviceSessions(id),
  started_at TIMESTAMPTZ DEFAULT now(),
  leader_device_id UUID REFERENCES DeviceSessions(id),
  leader_updated_at TIMESTAMPTZ
);

CREATE TABLE BroadcastStates (
  service_id UUID PRIMARY KEY REFERENCES ServiceInstances(id),
  state_json JSONB,        -- tabs + version + meta
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

---

<a name="8"></a>
## 8 . Key Application Flows

### 8.1 Remote Preparation

1. Choirmaster opens PWA at home (`https://chantersync.app/st-george`).
    
2. Uploads `GreatLitany_EN_AR.pdf` (POST `/library-files`).
    
3. Adds tags `[Tone 1, Feast:Pascha]`.
    
4. Creates new SetList; drags files; `POST /set-lists/{id}/publish`.
    
5. Worker fires `setlistPublished`; prefetch tasks queued.
    

### 8.2 Sunday Startup

1. Parish tablet loads `/role-picker`. Choirmaster taps _Choirmaster_:
    
    - `POST /sessions` returns `{jwt, deviceSessionId}`.
        
2. Client subscribes WS channel `broadcast:{serviceId}`.
    
3. Choirmaster `POST /service-instances` ⇒ returns `serviceId`; acquires LeaderToken.
    

### 8.3 Broadcast Document

1. Choirmaster clicks SetListItem #3 (Arabic Communion).
    
2. Client emits WS message `broadcast` `{tabId:'AR', fileId:'uuid-B'}`.
    
3. API verifies leader; updates `BroadcastStates.state_json`; pushes diff.
    
4. Chanters’ PWA receives diff, fetches signed PDF URL (if not cached), opens.
    

### 8.4 Device Hot-Swap

1. Primary iPad crashes (no heartbeat).
    
2. After 45 s, LeaderToken expires (`UPDATE … WHERE leader_device_id = old AND age > 45s`).
    
3. Spare Android tablet taps _Choirmaster_ → obtains session.
    
4. Presses “Take Control” → leader acquisition succeeds; BroadcastState unchanged (all PDF viewers keep position).
    

---

<a name="9"></a>
## 9 . Concurrency & Conflict-Resolution

- **LeaderToken**: RDBMS row-level atomic compare-and-swap (see SQL earlier).
    
- **SetList Draft vs Published**: `status` column + version; published setlist immutable (duplicate for edits).
    
- **Broadcast collisions**: Only requests with `deviceSessionId == leader_device_id` accepted; others receive `409 LEADER_CONFLICT`.
    

---

<a name="10"></a>
## 10 . Offline Strategy

|Asset|Cache Layer|Invalidation|
|---|---|---|
|AppShell JS/CSS|SW static cache (`cacheVersion`key)|On new service worker activate|
|Library PDFs (current service)|IndexedDB `pdf-store`|`BroadcastStates.version` mismatch triggers re-download|
|Thumbnails|Cache-First|30-day max-age|

_If completely offline:_

- RolePicker still loads (guest Choirmaster allowed).
    
- Library UI disabled; only cached SetList + PDFs visible.
    
- LeaderToken defaults to “local-only” (assumes no concurrency).
    

---

<a name="11"></a>
## 11 . Security & Compliance

- **Transport:** HSTS, TLS 1.3 only.
    
- **Auth:** JWT HS256 (edge verified) with 30-min TTL; silent refresh every 20 min.
    
- **File Access:** Signed GET (V4) with IP-range lock (optional).
    
- **Virus Scan:** ClamAV + PDF structural validation.
    
- **PIPEDA:** Data residency flag; single region bucket.
    
- **Audit Logs:** Insert triggers on `ServiceInstances`, `LeaderToken`, `BroadcastStates`; shipped to log pipeline (ELK).
    

---

<a name="12"></a>
## 12 . Operational Concerns

|Concern|Mitigation|
|---|---|
|**WS fan-out spikes** (Christmas, Pascha)|Auto-scale Real-Time Hub (stateless) via Kubernetes HPA on connection count.|
|**Large PDF upload**|100 MiB “edge reject” limit; encourage compression.|
|**PDF.js CPU on low-end Android 5.1**|Render at 1× viewport scale + progressive page render (lazy).|
|**Clock drift** for heartbeat TTL|Server authoritative; TTL generous (45 s).|

---

<a name="13"></a>
## 13 . Extensibility Roadmap

| Phase    | Additions                                                                        |
| -------- | -------------------------------------------------------------------------------- |
| **v0.2** | Magic-link e-mail auth; Team admin UI; per-device note sync.                     |
| **v0.3** | Fully synced page turning (optional flag); Bluetooth foot pedal events.          |
| **v1.0** | Native shell build (App Store / Play), in-app donations, multi-choir federation. |
| **v1.1** | OCR + searchable text layers; instant transposition for Western notation.        |

---

**End of Technical Specification**