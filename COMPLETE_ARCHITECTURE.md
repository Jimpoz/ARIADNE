# Ariadne Complete Architecture

Indoor navigation platform: graph-based floor maps, PostGIS geometry mirror,
RAG assistant over the graph, two computer-vision services, a web mapmaker
and two mobile clients (iOS primary, Android prototype).

## At a glance

The whole platform fits in one picture: three clients, one gateway, six
internal services, four stores. Everything external goes through the
middleware; nothing else is publicly addressable.

```
   ┌─────── Clients ────────┐   ┌─ Gateway ──┐   ┌──── Internal services ─────┐   ┌──── Stores ─────┐
   │                        │   │            │   │                            │   │                 │
   │  iOS                   │   │            │   │  backend                   │   │  Neo4j Aura     │
   │   SwiftUI + MapKit     │   │            │   │   CRUD, routing,           │◄──┤   graph of      │
   │   + Combine            │   │            │   │   imports, auth + MFA      │   │   truth         │
   │                        │   │            │   │                            │   │                 │
   │  Android (prototype)   │   │            │   │  assistant                 │   │  Supabase       │
   │   Compose + on-device  ├──►│ middleware ├──►│   LLM + RAG over graph     │◄──┤  PostGIS        │
   │   YOLO/MobileNet       │   │  :8080     │   │                            │   │   geometry      │
   │                        │   │  HTTPS +   │   │  image_pipeline            │   │   mirror        │
   │  Mapmaker (web)        │   │  WSS +     │   │   YOLO11 batch room        │   │                 │
   │   nginx-served static  │   │  X-Api-Key │   │   summaries                │   │  ./models       │
   │   HTML / canvas / JS   │   │  + Bearer  │   │                            │◄──┤  PVCs           │
   │                        │   │  JWT       │   │  ml_vision                 │   │   YOLO26 ONNX   │
   └────────────────────────┘   │            │   │   YOLO26 ONNX live WS,     │   │   + per-platform│
                                │  open-     │   │   single-frame HTTP        │   │   model bundles │
                                │  api.json  │   │                            │   │                 │
                                │  merged    │   │  email-service             │   │  External SMTP  │
                                │  from 4    │   │   internal-only relay      │──►│  relay          │
                                │  upstreams │   │   (MFA + password reset)   │   │   (STARTTLS)    │
                                │            │   │                            │   │                 │
                                └────────────┘   └────────────────────────────┘   └─────────────────┘
                                       │
                                       │  Same /api/v1 surface that the mapmaker
                                       │  uses is what the mobile apps see — no
                                       │  hidden authoring endpoints.
                                       │
                                       └─► Cluster: ingress → middleware Service is the
                                           only LoadBalancer; everything else is ClusterIP.

   Protocols at the edges:
     Client → Gateway   HTTPS  +  WSS                          (X-Api-Key, Bearer JWT)
     Gateway → Service  HTTP   +  WS (over Docker net)         (re-stamped identity headers)
     Backend → Neo4j    Bolt   (neo4j+s://)                    (graph writes/reads)
     Backend → PostGIS  PostgreSQL (SQLAlchemy + GeoAlchemy2)  (geometry mirror, singleton engine)
     Backend → Email    HTTP   (Docker net only)               (X-Internal-Token, never public)
     Email   → SMTP     SMTP / STARTTLS                        (external relay, opt-in via env)
```

| Component                  | Path                                     | Role                                                     |
| -------------------------- | ---------------------------------------- | -------------------------------------------------------- |
| **Middleware (gateway)**   | `aau-sw8-spatial-backend/middleware`     | Public entry, API-key auth, JWT verify, HTTP+WS proxy    |
| **Backend (spatial)**      | `aau-sw8-spatial-backend/backend`        | Graph + geometry CRUD, imports, routing, auth + MFA      |
| **Assistant**              | `aau-sw8-spatial-backend/assistant`      | LLM chat with RAG over the Neo4j graph                   |
| **Image pipeline**         | `aau-sw8-spatial-backend/image_pipeline` | YOLO11 batch room summaries from uploaded photos         |
| **ML-Vision**              | `aau-sw8-ml-vision`                      | YOLO26 ONNX live object detection over WebSocket         |
| **Email service**          | `aau-sw8-spatial-backend/email-service`  | Internal-only SMTP relay for password reset / MFA OTP    |
| **Mapmaker (frontend)**    | `aau-sw8-spatial-backend/frontend`       | Web tool to author buildings, rooms, doors, connections  |
| **iOS client**             | `aau-sw8-ios`                            | Floor plan view, assistant, live camera, room uploads    |
| **Android client**         | `aau-sw8-android`                        | Compose prototype — on-device vision + wifi scan         |

Everything runs behind the middleware on `:8080`. No other service is exposed publicly.

---

## Software architecture (layered view)

Where [At a glance](#at-a-glance) shows *what talks to what at runtime*,
this diagram shows *how the code is organised*: which layer a module
belongs to, what it depends on, and what each layer is allowed to know
about the next. The platform is a classic five-tier layered architecture
with a thin gateway in front and cross-cutting concerns running the full
height.

```
   ┌───────────────────────────── Presentation tier ────────────────────────────────┐
   │                                                                                │
   │   iOS app (SwiftUI + Combine + AVFoundation)   Android app (Jetpack Compose)   │
   │     ─────────────────────────────────────         ────────────────────────     │
   │     Views/                                        ui/                          │
   │     ViewModels (ObservableObject)                 ViewModels                   │
   │     Services/  (URLSession, WS, CoreLocation)     Utilities/ (CameraX,         │
   │     DIContainer  (Combine wire-up)                  WifiScanner, MobileNet)    │
   │                                                                                │
   │   Web mapmaker (nginx-served static HTML + vanilla JS + Canvas 2D)             │
   │     ─────────────────────────────────────────────                              │
   │     index.html   state{}   api()-helper   floor canvas renderer                │
   │                                                                                │
   └──────────────────────────────────────┬─────────────────────────────────────────┘
                                          │  HTTPS + WSS    (X-Api-Key + Bearer JWT)
   ┌──────────────────────────────────────▼─────────────────────────────────────────┐
   │                              API gateway tier                                  │
   │                                                                                │
   │   middleware/  (FastAPI + httpx.AsyncClient + websockets)                      │
   │     verify_api_key middleware · JWT decode + identity re-stamp                 │
   │     /api/v1/* reverse-proxy · WS bridge (2-task asyncio relay)                 │
   │     merged /openapi.json · /health fan-out · /debug/upstreams                  │
   │     mobile convenience: /api/v1/mobile/campuses (SVG-stripped)                 │
   │                                                                                │
   └──────┬──────────────┬──────────────┬──────────────┬─────────────┬──────────────┘
          │              │              │              │             │              
          │   HTTP (Docker network — re-stamped x-user-id / x-org-id / x-user-role) │
          ▼              ▼              ▼              ▼             ▼              
   ┌──────────────────────────────── Service tier (5 microservices) ────────────────┐
   │                                                                                │
   │  backend/        assistant/        image_pipeline/  ml_vision/   email-service/│
   │  ───────         ──────────        ─────────────   ───────────   ──────────────│
   │  FastAPI         FastAPI           FastAPI         FastAPI       FastAPI       │
   │  Spatial CRUD    Chat + RAG        Batch CV        Live CV       SMTP relay    │
   │  + routing       + embedder        (YOLO11n)       (YOLO26 ONNX) (internal)    │
   │  + imports       (MiniLM)          + OpenCV        + WS stream                 │
   │  + auth + MFA    + LLM             + RoomSummary   + LocationResolver          │
   │                                                                                │
   └──────┬──────────────┬──────────────┬──────────────┬─────────────┬──────────────┘
          │              │              │              │             │              
   ┌──────▼──────────────▼──────────────▼──────────────▼─────────────▼──────────────┐
   │                       Domain / business-logic tier                             │
   │                                                                                │
   │   Per-service, organised the same way wherever possible:                       │
   │                                                                                │
   │     routes/     thin HTTP glue — validation, status codes, principals          │
   │       │                                                                        │
   │       ▼                                                                        │
   │     services/  business logic over BOTH stores; e.g.                           │
   │       postgis_service   geometry_service   gds_service   navigation_service    │
   │       import_service    dxf_normalize     dxf_convert    space_sync            │
   │       auth_service      audit_service     email_client   embed_client          │
   │       │                                                                        │
   │       ▼                                                                        │
   │     repositories/   pure Cypher (Neo4j) — returns dicts                        │
   │     models/         Pydantic + SQLAlchemy ORM + GeoAlchemy2 (PostGIS)          │
   │                                                                                │
   └──────┬──────────────┬──────────────┬──────────────────────────────┬────────────┘
          │ Bolt         │ Bolt+SQLA    │ Bolt + PVC                   │ SMTP/STARTTLS
          ▼              ▼              ▼                              ▼            
   ┌────────────────────────────── Data + infra tier ───────────────────────────────┐
   │                                                                                │
   │   Neo4j Aura           Supabase Postgres                  ./models/ PVCs       │
   │   ───────────          + PostGIS + pgvector               ───────────────      │
   │   graph of truth       ─────────────────────              YOLO ONNX weights    │
   │   CONNECTS_TO          geometry mirror (polygons,         + per-platform       │
   │   GDS projection         centroids, geometry_global)      bundles              │
   │     'nav_graph'        spatial index (ST_DWithin)         (.mlpackage iOS,     │
   │   Space.embedding      vector index (pgvector cosine)      .tflite Android)    │
   │   (legacy RAG          RLS multi-tenant isolation                              │
   │    fallback)           audit_log, app_users, MFA          External SMTP relay  │
   │                                                                                │
   └────────────────────────────────────────────────────────────────────────────────┘

   ─── Cross-cutting concerns (vertical slice through every tier) ──────────────────
     • AuthN & AuthZ   X-Api-Key (gateway) · JWT (gateway+service) · RLS (data)
     • Audit logging   audit_action(...) context manager → audit_log (every priv op)
     • Observability   middleware /health fan-out · /debug/upstreams
     • Network trust   ONLY middleware is publicly routable (LoadBalancer); every
                       other service is ClusterIP / `expose:` — Docker net only
     • Tenancy         org_id (active, writes) + org_ids (memberships, reads) ride
                       on the JWT, are re-stamped as headers, parsed into Principal
   ─────────────────────────────────────────────────────────────────────────────────
```

**Dependency rule.** A layer only depends on the one beneath it. Routes
never reach into another service's repository; the gateway never speaks
Bolt; the data tier doesn't know who the caller is — it trusts the GUCs
the backend stamped (`app.org_id`, `app.is_service`) and lets RLS do the
admission. This keeps the service tier independently deployable and
the data tier free of credentials beyond what its own pool needs.

**Why five tiers instead of three.** The gateway is split out from the
service tier because every service has to assume `principal` exists and
is verified — centralising that in one tier removes the temptation for
a service to "just check the JWT itself" and grow its own copy of the
auth code. The domain/business-logic tier is split out from services
because the same `import_service`, `geometry_service` and `gds_service`
are reused across both the DXF-import path and the JSON-import path,
and across both the HTTP routes and the CLI scripts in `backend/scripts/`.

---

## Data flow

The only public surface is the middleware on `:8080`. Everything else lives on
the internal Docker network. Mobile clients hit `/api/v1/*` (REST) and
`/api/v1/ml-vision/ws/stream/{id}` (WebSocket); the mapmaker hits the same
REST surface from nginx-served static pages. Writes always land in Neo4j
first, then the backend mirrors to PostGIS atomically.

```
   ┌────────────────────┐   ┌──────────────────────┐   ┌────────────────────┐
   │     iOS client     │   │   Android client     │   │  Web mapmaker      │
   │  (SwiftUI+Combine) │   │  (Compose prototype) │   │  (static HTML+JS)  │
   │                    │   │                      │   │                    │
   │ FloorPlanView      │   │ FloorPlanScreen      │   │ canvas editor      │
   │ AssistantView      │   │ AssistantScreen      │   │ CRUD organizations │
   │ CameraView (WSS)   │   │ CameraScreen (local) │   │  /campuses/etc.    │
   │ RoomPhotoUpload    │   │ WifiScanner          │   │ import/export JSON │
   │ ProfileView        │   │ MobileNet (tflite)   │   │ RAG chat           │
   └────────┬───────────┘   └──────────┬───────────┘   └─────────┬──────────┘
            │ HTTPS + WSS              │ (no backend yet)        │ HTTP
            │ X-Api-Key                                          │ X-Api-Key
            ▼                                                    ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                       Middleware / gateway (:8080)                   │
   │                                                                      │
   │  • HTTP X-Api-Key guard (skipped only for /health)                   │
   │  • /api/v1/assistant/*    ──► assistant   (reverse-proxy, 900s)      │
   │  • /api/v1/room-summary/* ──► image_pipe  (reverse-proxy, 900s)      │
   │  • /api/v1/ml-vision/*    ──► ml_vision   (reverse-proxy, 900s)      │
   │  • /api/v1/*              ──► backend     (catch-all proxy)          │
   │  • /api/v1/mobile/campuses, .../map, .../map/light (svg-stripped)    │
   │  • WS /api/v1/ml-vision/ws/stream/{facility_id}  (2-task bridge)     │
   │  • /openapi.json merged from all 4 upstreams                         │
   │  • /health fan-out + /debug/upstreams (debug build)                  │
   └───────┬──────────────┬───────────────┬────────────────┬──────────────┘
           │              │               │                │
           ▼              ▼               ▼                ▼
   ┌────────────┐  ┌────────────┐  ┌──────────────┐  ┌──────────────────┐
   │  backend   │  │ assistant  │  │image_pipeline│  │    ml_vision     │
   │  :8000     │  │  :8001     │  │    :8002     │  │      :8000       │
   │            │  │            │  │              │  │                  │
   │ FastAPI    │  │ FastAPI    │  │ FastAPI +    │  │ FastAPI +        │
   │ CRUD +     │  │ LLM +      │  │ YOLO11n      │  │ YOLO26-ONNX      │
   │ routing    │  │ RAG        │  │ room summary │  │ + location       │
   │ + imports  │  │ k-NN over  │  │ + OpenCV     │  │   resolver       │
   │            │  │ Neo4j vecs │  │              │  │ + WS stream      │
   └──┬──────┬──┘  └─────┬──────┘  └──────┬───────┘  └────────┬─────────┘
      │      │           │                │                   │
      │      └───────────┼────────────────┼─────────┐         │
      │                  │                │         │         │
      ▼                  ▼                ▼         ▼         ▼
   ┌─────────────────────────────────┐ ┌─────────────────┐ ┌───────────────┐
   │  Neo4j Aura (graph of truth)    │ │ Supabase PostGIS│ │ ./models/     │
   │                                 │ │ (geometry mirror│ │  {facility}/  │
   │  Organization/Campus/Building/  │ │  + index)       │ │   .onnx       │
   │  Floor/Space/Door nodes,        │ │                 │ │   .mlpackage  │
   │  CONNECTS_TO edges,             │ │ building_spaces │ │   .tflite     │
   │  384-d MiniLM embeddings,       │ │ floors, rooms,  │ │   metadata    │
   │  GDS projection for Dijkstra    │ │ space_connections│ │               │
   └─────────────────────────────────┘ └─────────────────┘ └───────────────┘
```

**Write-order invariant.** A mapmaker click that creates a room fires
`POST /api/v1/spaces` → gateway → backend → (1) Cypher `MERGE (Space)`,
(2) `SpaceRepository` returns a dict, (3) `PostGISService.sync_space` writes
the SQLAlchemy row + `ST_GeomFromText` polygon, (4) response returned. If
step 1 fails the request aborts; if step 3 fails Neo4j is already ahead and
the next full sync reconciles. Reads that need geometry prefer PostGIS and
fall back to Neo4j (see [floors.py:92-127](aau-sw8-spatial-backend/backend/routes/floors.py#L92-L127)).

---

## Two databases — Neo4j vs PostGIS

Neo4j is the **graph of truth**: who connects to what. PostGIS is the
**geometry + spatial + vector store**: what things look like in space, and
which spaces are semantically similar. Writes always go Neo4j first, then
sync PostGIS. The two stores have different strengths — routing wants a
native graph traversal; rendering, spatial filtering and similarity search
want indexed columns and SQL joins — so we keep both rather than forcing
one to do a job it's bad at.

| Concern                                  | Neo4j Aura | Supabase PostGIS |
| ---------------------------------------- | :--------: | :--------------: |
| Source of truth for every write          |     ✓      |                  |
| Graph traversal / Dijkstra / GDS routing |     ✓      |                  |
| Geometry storage (polygons, centroids)   |     ✓      |        ✓         |
| Spatial queries (`ST_DWithin`, `ST_Intersects`, bbox) |    | ✓         |
| Vector similarity (pgvector + cosine)    | (fallback) |   ✓ (primary)    |
| Multi-tenant RLS for org isolation       |            |        ✓         |
| Auth / audit / app_users                 |            |        ✓         |

`Space.embedding` lives on the Neo4j node *and* the PostGIS row. Neo4j
keeps the vector for backward compatibility with the legacy
`db.index.vector.queryNodes('space_embedding_idx', …)` fallback path;
PostGIS holds the same vector in a `vector(384)` column
(`building_spaces.embedding`) with a pgvector cosine index, which is
what the assistant queries by default. A single SQL statement combines
`embedding <=> CAST(:q AS vector)` (k-NN by cosine distance) with
`ST_DWithin(geometry_global::geography, …)` (a spatial radius around
the user's GPS fix), so "similar spaces near me" is one indexed scan
instead of two cross-store queries.

**Neo4j label model and cardinality**

```
                     ┌──────────────┐
                     │ Organization │  (tenant root, owns everything)
                     └──────┬───────┘
                            │ [:OWNS] 1..*
                            ▼
   ┌──────────────────► ┌────────┐ ─[:HAS_OUTDOOR]─► ┌─────────┐
   │                    │ Campus │                   │  Space  │
   │                    └───┬────┘                   │ (lawn,  │
   │                        │ [:HAS_BUILDING] 1..*   │  plaza) │
   │                        ▼                        └─────────┘
   │                   ┌──────────┐
   │                   │ Building │  name, short_name, origin_lat,
   │                   └────┬─────┘  origin_lng, origin_bearing,
   │                        │        floor_count  (auto-recounted)
   │                        │ [:HAS_FLOOR] 1..*
   │                        ▼
   │                    ┌───────┐   floor_index, display_name,
   │                    │ Floor │   floor_plan_url, scale,
   │                    └───┬───┘   origin_{x,y}, bounds,
   │                        │       [optional georef override]
   │                        │       origin_lat, origin_lng,
   │                        │       origin_bearing, scale_factor
   │                        │       (null → inherit Building)
   │                        │ [:HAS_SPACE] 1..*
   │                        ▼
   │                   ┌───────┐    space_type (ROOM_*, CORRIDOR_*,
   │                   │ Space │    DOOR_*, STAIRS_*, ELEVATOR_*...),
   │                   └───┬───┘    polygon, centroid_x/y,
   │                       │        area_m2, is_accessible,
   │                       │        is_navigable, capacity,
   │                       │        traversal_cost,
   │                       │        embedding (384 floats)
   │                       │
   │                       │ [:HAS_SUBSPACE*1..]  (nested rooms, alcoves)
   │                       ▼
   │                   ┌───────┐
   │                   │ Space │
   │                   └───┬───┘
   │                       │
   │        ┌──────────────┴──────────────┐
   │        │   [:CONNECTS_TO]            │
   │        │   (door-centred pattern,    │
   │        │    4 edges per physical     │
   │        │    doorway; see below)      │
   │        ▼                             │
   │    ┌───────┐                         │
   │    │ Space │ ────────────────────────┘
   │    └───────┘
   │
   └── (a single Organization can own many Campuses)
```

**Per-floor georeferencing override.** A `Floor` normally inherits
`origin_lat / origin_lng / origin_bearing / scale_factor` from its
parent `Building` — one rigid transform places every floor on the
world map. When an editor nudges a single floor in the iOS map
overlay (via the "Edit this floor" scope), the new georef is written
onto the `Floor` node as an override; any of those four fields being
`null` means "inherit from Building." On read, `_resolve_floor_origin`
in [routes/floors.py](aau-sw8-spatial-backend/backend/routes/floors.py)
fills the nulls from the parent, so downstream consumers (PostGIS
sync, space re-projection, iOS overlay) always see concrete values
regardless of whether the floor has its own.

A building move carries through to floor overrides: `update_building`
applies the rigid delta to every floor that has its own override (via
`apply_edit_transform` in `geometry_service.py`), so a previously-
nudged floor stays in correct relative position when the whole
building is moved.

**Door-node pattern.** A physical door between room `A` and corridor `B` is
itself a `Space` node `D` with `space_type ∈ CONN_SPACE_TYPES` (DOOR_*,
PASSAGE_*, STAIRS_*, ELEVATOR_*). It is materialised as **four** directed
`CONNECTS_TO` edges so Dijkstra can enter the door from either side and
leave again:

```
   (A) ──CONNECTS_TO──► (D) ──CONNECTS_TO──► (B)
    ▲                     │                    │
    └─── CONNECTS_TO ◄─── (D) ◄─── CONNECTS_TO ─┘

   traversal_cost on each edge = f(geometry, door_type,
                                   is_accessible, is_emergency_only)
```

`connection_group_id = D.id` on every edge (and in the PostGIS mirror) so
deleting the door atomically removes all four rows, with no orphan stubs.

**Shortest-path pipeline.**

```
   /navigate?from=A&to=B&accessible_only=&avoid_stairs=&elevators_only=
        │
        ▼
   NavigationService ─► GDS projection "nav_graph" (refreshed on import
        │                and topology changes; NOT per request)
        │                ─ used only when ALL three filters are off
        │                ─ otherwise → native Cypher shortestPath
        ▼
   gds.shortestPath.dijkstra.stream(
        sourceNode: (A), targetNode: (B),
        relationshipWeightProperty: "traversal_cost"
   )
        │
        ▼
   list[Space] → stitched into turn-by-turn steps by direction math
   on consecutive centroids; iOS overlays it on the FloorPlanView.
```

**PostGIS mirror schema** (joined-table inheritance rooted at
`building_spaces`):

```
   organizations ◄──┐
       │ id         │ (FKs without ON DELETE CASCADE — we
       ▼            │  cascade manually in PostGISService)
   campuses ───────┤
       │ id        │
       ▼           │
   buildings ─────┤
       │ id       │        ┌─────────────────────────────┐
       ▼          │        │ building_spaces (root)      │
   floors ───────┘        ─┤   id (PK), organization_id, │
       │ (id = "{b}_{f}") │   campus_id, building_id,   │
       │                  │   floor_id, space_type,     │
       │                  │   polygon (GEOMETRY),       │
       │                  │   centroid, accessibility   │
       │                  └──────┬──────────────────────┘
       │                         │ joined-table inheritance
       │              ┌──────────┴──────────┬───────────┐
       │              ▼                     ▼           ▼
       │         rooms, corridors,  doors, stairs,  outdoor_spaces
       │         offices, …         elevators,       (subclass tables,
       │                             passages         each with FK
       │                                              ON DELETE CASCADE)
       │
       │    ┌──────────────────────────────────────┐
       └───►│ space_connections                    │
            │   from_space_id, to_space_id,        │
            │   door_space_id,                     │
            │   connection_group_id  (= door id),  │
            │   traversal_cost, is_bidirectional   │
            └──────────────────────────────────────┘
```

Used for `ST_Intersects` / `ST_DWithin` geometry queries, bulk floor-plan
fetches, **pgvector cosine-similarity search for the assistant's RAG path**,
and any SQL-only consumer (BI dashboards, analytics, nearest-neighbour
rendering). The routing graph never comes from here.

`building_spaces` also carries a `geometry_global` column (WGS84 polygon,
SRID 4326) alongside the local `polygon` (floor-plan meters). The global
form is what the spatial RAG filter joins against — a single
`ST_DWithin(geometry_global::geography, ST_MakePoint(lng, lat), radius_m)`
predicate scopes a vector search to "spaces within N metres of the
user," and a GiST spatial index makes it cheap.

**Rule of thumb.** Routing → Neo4j (graph traversal, Dijkstra/GDS).
Geometry, spatial filters, and semantic similarity → PostGIS
(`ST_*` queries + pgvector). Writing → always Neo4j first (the
authoritative, synchronous commit), then the PostGIS mirror is applied
**asynchronously** by a background worker draining a durable outbox —
see [Neo4j → PostGIS sync](#neo4j--postgis-sync--the-durable-outbox)
below.

**Embedding backfill scripts.** Two idempotent scripts in
[backend/scripts](aau-sw8-spatial-backend/backend/scripts/) keep the
embeddings consistent across both stores:

- [backfill_embeddings_to_postgis.py](aau-sw8-spatial-backend/backend/scripts/backfill_embeddings_to_postgis.py) —
  one-time migration: reads every `Space.embedding` from Neo4j and writes
  the same vector into `building_spaces.embedding`. Run once after the
  `pgvector` extension + column are created.
- [backfill_missing_embeddings.py](aau-sw8-spatial-backend/backend/scripts/backfill_missing_embeddings.py) —
  ongoing repair: finds Neo4j `Space` nodes with `embedding IS NULL`
  (typically created via the mapmaker's `POST /spaces` rather than the
  `/import` pipeline that runs the embedder), batch-calls the assistant's
  `/internal/embed` endpoint, then writes both Neo4j and PostGIS so the
  vector lands in the same space as import-time embeddings.

### Neo4j → PostGIS sync — the durable outbox

The mirror write is **asynchronous and eventually consistent**, not an
in-request atomic dual-write (there is no compensating rollback anywhere
in the backend). Every `PostGISService.sync_*` / `delete_*` facade is a
thin wrapper that, *after* the authoritative Neo4j write has committed,
calls `sync_outbox.enqueue(op, payload)` — which `CREATE`s a
`(:SyncOutbox {op, payload, attempts, next_attempt_at})` node in Neo4j.
The PostGIS mutation itself lives in a private `_apply_*` method that
**only the worker ever calls**.

A single `OutboxWorker` (started in the FastAPI `lifespan`, stopped on
shutdown) polls every `SYNC_OUTBOX_POLL_INTERVAL_S` (2 s), drains up to
`SYNC_OUTBOX_BATCH_SIZE` (50) rows in `created_at ASC, id ASC` order — so
dependent ops replay as campus → building → floor → space — dispatches
each `op` through a name→`_apply_*` table, and deletes the row on
success. On failure it bumps `attempts` and pushes `next_attempt_at` out
by capped exponential backoff (`base × 2^min(attempts, 6)`, base = 5 s,
so ≤ 320 s); `SYNC_OUTBOX_MAX_ATTEMPTS` (default `0`) means *retry
forever*. The drain runs with the RLS context forced to service mode
(`app.is_service = true`, `app.org_id = null`) so a row enqueued under a
request scope that has since ended still applies.

Consequences: a slow or briefly-unavailable PostGIS never blocks or
fails a Neo4j write; the two stores converge once the queue drains; and
`outbox_status()` exposes the pending count + oldest entry for an admin
endpoint. The whole subsystem is gated by `SUPABASE_ENABLE_SYNC` — when
false, `enqueue` is a no-op and PostGIS is never touched. Code:
[`services/sync_outbox.py`](aau-sw8-spatial-backend/backend/services/sync_outbox.py).

---

## Middleware

**Description.** A FastAPI/Starlette reverse-proxy that is the only
publicly reachable container in the whole stack. It exists for three
reasons: (1) centralise authentication so upstream services can stay
credential-agnostic inside the Docker network, (2) merge four separate
FastAPI apps (backend, assistant, image_pipeline, ml_vision) into a single
`/api/v1` namespace that mobile clients see, and (3) bridge WebSockets
through to `ml_vision` — FastAPI's HTTP middleware doesn't run on the WS
handshake, so auth is done inline in the WS handler. It also hosts a few
client-shaped convenience endpoints (`/api/v1/mobile/*`) that pre-trim
payload fields the mobile apps don't need (e.g. stripping base64 SVG blobs
from `map/light`).

```
            Client                                    Internal Docker network
   ═══════════════════════                     ═══════════════════════════════
   iOS / Android / mapmaker
            │    │                             ┌───── backend (:8000) ─────┐
   HTTPS ───┘    └─── WSS                      │ Neo4j writer, PostGIS     │
   X-Api-Key     X-Api-Key (on handshake)      │ mirror, routing, imports  │
            │    │                             └─────┬─────────────────────┘
            ▼    ▼                                   │
   ┌──────────────────────────────────────────────┐  │
   │  Middleware (:8080)                          │  │
   │  FastAPI + httpx.AsyncClient + websockets    │  │
   │                                              │  │
   │  @app.middleware("http") verify_api_key      │  │
   │    • bypass only for /health                 │  │
   │    • 401 Unauthorized if X-Api-Key mismatch  │  │
   │                                              │  │
   │  routes (include_in_schema=False):           │  │
   │    /api/v1/assistant/*          ────────────►│──┼─► assistant (:8001)
   │    /api/v1/room-summary/*       ────────────►│──┼─► image_pipeline (:8002)
   │    /api/v1/ml-vision/*  (HTTP)  ────────────►│──┼─► ml_vision (:8000)
   │    /api/v1/*            (catch-all)  ───────►│──┘
   │                                              │
   │  mobile convenience (tag="mobile"):          │
   │    GET /api/v1/mobile/campuses               │
   │    GET /api/v1/mobile/campuses/{id}/map      │
   │    GET /api/v1/mobile/campuses/{id}/map/light│ (strips SVG blobs
   │                                              │  from room_summary)
   │                                              │
   │  WS /api/v1/ml-vision/ws/stream/{facility}   │
   │    ├─ 4401 close if x-api-key missing        │
   │    ├─ await websockets.connect(upstream,     │
   │    │     max_size=None, ping 20/20)          │
   │    └─ two asyncio tasks:                     │
   │        • client → upstream (bytes / text)    │
   │        • upstream → client (bytes / text)    │──► ml_vision WS
   │       asyncio.wait(FIRST_COMPLETED)          │
   │                                              │
   │  OpenAPI: /openapi.json merged from 4        │
   │    upstreams (each upstream's paths are      │
   │    filtered by public_prefixes / excludes)   │
   │                                              │
   │  Observability:                              │
   │    GET /health  → fan-out check per upstream │
   │    GET /debug/upstreams (only when           │
   │           MIDDLEWARE_DEBUG_UPSTREAMS=true)   │
   │                                              │
   │  Response shaping:                           │
   │    adds X-Gateway-Upstream header so callers │
   │    know which service answered               │
   │    httpx timeout = 900s (big imports)        │
   └──────────────────────────────────────────────┘
```

**Security posture.** `API_SECRET` is read from env, compared constant-
fashion via header equality. 502 is returned when an upstream is
unreachable rather than propagating raw stack traces. The WS bridge closes
with code `4401` on auth failure and `1011` on upstream errors, so the
mobile clients can distinguish policy from infrastructure problems.

---

## Backend (spatial)

**Description.** The heart of the platform: a FastAPI service that owns
every write to Neo4j and drives every mirrored write to PostGIS. Routes are
organised by entity (`organizations`, `campuses`, `buildings`, `floors`,
`spaces`, `connections`, `navigation`, `search`) and each route layer is a
thin glue layer over `repositories/` (Cypher) and `services/`
(business-logic over both stores). It also hosts the bulk-import pipeline
the mapmaker uses to push whole campuses in a single call, the GDS-backed
shortest-path engine the mobile clients use for turn-by-turn directions,
and the embedding generator used by the assistant for RAG. If a piece of
data matters to routing, rendering, or retrieval, it started here.

```
              HTTP (gateway → backend :8000, X-Api-Key already validated)
                                      │
                                      ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                       routes/                                        │
   │                                                                      │
   │  organizations.py   ─► Organization CRUD, hard-delete cascade        │
   │  campuses.py        ─► Campus CRUD, /campuses/{id}/export bulk read  │
   │  buildings.py       ─► Building CRUD + DELETE cascade to floors      │
   │  floors.py          ─► Floor CRUD, /display, /geometry, /map-overlay,│
   │                        /connections, /spaces                         │
   │  spaces.py          ─► Space CRUD, DELETE walks CONNECTS_TO doors    │
   │  connections.py     ─► POST ↔ DELETE single writer for doors         │
   │                        (4 Cypher edges + 4 SQL rows per call)        │
   │  navigation.py      ─► /navigate?from=&to=&accessible_only=          │
   │                        &avoid_stairs=&elevators_only= ,              │
   │                        /navigate/refresh-graph                       │
   │  search.py          ─► /search/spaces?q=  (substring CONTAINS over   │
   │                        display_name + short_name + tags_text),       │
   │                        /search/nearest-space?lat=&lon=               │
   └─────────────────────────────────┬────────────────────────────────────┘
                                     │
   ┌─────────────────────────────────▼────────────────────────────────────┐
   │                       repositories/   (pure Cypher, returns dicts)   │
   │                                                                      │
   │  campus_repo.py        Organization/Campus/Building/Floor CRUD.      │
   │                        create_floor recounts b.floor_count in-query; │
   │                        delete_building / delete_floor pre-collect    │
   │                        space_ids + door_ids before DETACH DELETE.    │
   │  space_repo.py         Space CRUD, polygon & centroid updates,       │
   │                        get_floor_spaces(), get_floor_display(),      │
   │                        delete_space — enumerates connected doors     │
   │                        (space_type IN CONN_SPACE_TYPES) and cleans   │
   │                        them too.                                     │
   │  connection_repo.py    Door-node writes (4 edges), cost computation, │
   │                        list_connections_for_floor().                 │
   └───────────────────────────────┬──────────────────────────────────────┘
                                   │                          ▲
                                   │ Cypher + Bolt             │ writes also
                                   ▼                           │ mirror via
                          ┌──────────────────┐                 │ services/
                          │   Neo4j Aura     │                 │
                          │  (graph of truth)│                 │
                          └──────────────────┘                 │
                                                               │
   ┌───────────────────────────────────────────────────────────┴──────────┐
   │                       services/                                      │
   │                                                                      │
   │  postgis_service.py    SQLAlchemy + GeoAlchemy2. sync_building,      │
   │                        sync_floor, sync_space, delete_connection_    │
   │                        group, delete_floor_cascade,                  │
   │                        delete_building_cascade, delete_organization  │
   │                        (manual top-down cascade because PG FKs have  │
   │                         no ON DELETE CASCADE).                       │
   │  geometry_service.py   Polygon ops: centroid, area, ST_GeomFromText, │
   │                        (x,y) ↔ (lat,lng) via building origin/bearing.│
   │  gds_service.py        Manages the "nav_graph" GDS projection.       │
   │                        Project / drop / dijkstra.stream wrappers.    │
   │  import_service.py     Bulk import_map() — parses mapmaker JSON,     │
   │                        writes Neo4j first, computes MiniLM 384-d     │
   │                        embeddings and traversal_cost, then mirrors   │
   │                        to PostGIS and refreshes the GDS projection.  │
   │  dxf_normalize.py      Stage 1 of CAD import. ezdxf-driven walk of   │
   │                        modelspace (recursing into INSERTs) → flat    │
   │                        JSON of LWPOLYLINE / LINE / ARC / CIRCLE /    │
   │                        TEXT / MTEXT / HATCH primitives + $INSUNITS   │
   │                        header. .dwg is first transcoded via          │
   │                        ODAFileConverter or LibreDWG (shell-out).     │
   │  dxf_convert.py        Stage 2 of CAD import. Normalized JSON →      │
   │                        MapImportSchema dict: unit-scale, polygonize  │
   │                        linework, attach labels, classify, infer      │
   │                        doors (arc-sweep heuristic), emit UUID-id     │
   │                        spaces + bidirectional connection pairs.      │
   │  dxf_import_service.py Helpers library shared by both stages — label │
   │                        classification (`_classify_label`/`_space`),  │
   │                        polygon validation, label attachment,         │
   │                        ID-type pair merge, DWG transcoding, DXF      │
   │                        recovery read. Output of dxf_convert is fed   │
   │                        to import_service.import_map so the dual-     │
   │                        write atomicity is shared with the JSON path. │
   │  navigation_service.py dijkstra → list[Space] → turn-by-turn steps   │
   │                        using centroid direction math.                │
   │  space_sync.py         Non-destructive reconciliation helpers for    │
   │                        when the two stores have drifted.             │
   └───────────────────────────────┬──────────────────────────────────────┘
                                   │ SQLAlchemy Core / ORM
                                   ▼
                          ┌──────────────────┐
                          │ Supabase PostGIS │
                          │ (geometry mirror)│
                          └──────────────────┘
```

**Key flows.**

- **Write path.** Any CRUD route → repository Cypher (Neo4j commits) →
  service mirror to PostGIS → HTTP 201/204. The backend is the *only*
  process that holds PostGIS credentials.
- **Import.** `import_service.import_map(campus_id, payload)` ingests the
  mapmaker JSON as one big transaction: Neo4j writes, per-space MiniLM
  384-dim embeddings, `traversal_cost` computed from geometry + space
  type, user-authored connections mirrored, finally `gds_service.refresh()`
  so new topology is routable immediately.
- **Connections.** `POST /connections` creates the door `Space` node, 4
  Neo4j `CONNECTS_TO` edges, and 4 rows in `space_connections` tagged with
  the same `connection_group_id`. `DELETE /connections/{from}/{to}` finds
  and removes the group in both stores atomically.
- **Navigation.** `/navigate` uses the pre-projected `nav_graph`; nothing
  rebuilds the projection per request. The projection is refreshed on
  import, on structural deletes that reach topology, and manually via
  `POST /navigate/refresh-graph`.
- **Delete cascade.** Deleting a building, floor, or organization removes
  every descendant in both stores; `DETACH DELETE` in Neo4j, explicit
  top-down deletes in PostGIS (connections → spaces → floors → buildings
  → campuses → imports → organization).

**`PostGISService` is a process-wide singleton.** Earlier the service was
re-instantiated on every request, each time building a fresh SQLAlchemy
engine (`pool_size=5, max_overflow=10` → up to 15 connections per pool)
and re-running `Base.metadata.create_all` + `ALTER TABLE`. Old engines
piled up until the Supabase pooler hit its 15-client session-mode cap
and started returning `EMAXCONNSESSION`. The fix: a module-level
`_shared_engine` plus a `_schema_initialized` guard so the engine is
constructed exactly once and the schema bootstrap runs exactly once per
process. Pool size shrunk to `pool_size=2, max_overflow=5` (max 7
clients) with `pool_pre_ping=True` and `pool_recycle=300` so half-dead
connections from the pooler's recycling don't surface as request errors.

---

## Assistant

**Description.** A standalone FastAPI microservice that layers an LLM
chat interface on top of the spatial data using a lightweight RAG
(retrieval-augmented generation) pipeline. It reads from **both** stores:
PostGIS (`db_pg.py`) drives the primary RAG retrieval — `pgvector` cosine
similarity on `building_spaces.embedding` combined in the same SQL
statement with an `ST_DWithin` spatial filter around the user's GPS fix —
and Neo4j (`db.py`) is used for graph traversal queries (anchor selection,
Dijkstra distance / "farthest office from main entrance" intents, floor
listings, "where am I" projection of local centroids to lat/lng). When
`SUPABASE_DB_URL` isn't set the PostGIS engine is `None` and the
repository falls back to the legacy Neo4j vector-index path
(`db.index.vector.queryNodes('space_embedding_idx', …)`), so the service
still boots in environments that haven't been migrated yet.

The service owns both an embedding model (sentence-transformers
`all-MiniLM-L6-v2`, 384-d) and a causal LLM (Hugging Face
`AutoModelForCausalLM`), auto-selecting `MPS > CUDA > CPU` at startup.
iOS uses this endpoint as its primary conversational interface; the
mapmaker uses it for the side-panel chat.

```
                   POST /api/v1/assistant/chat
                   { user_query, campus_id,
                     building_id?,            ← scopes RAG to one building
                     user_lat?, user_lon? }   ← used for outdoor refusal
                                                  AND for spatial filter
                                │
                                │  (gateway already checked X-Api-Key)
                                ▼
   ┌────────────────────────────────────────────────────────────────┐
   │                 assistant :8001   (FastAPI + lifespan)         │
   │                                                                │
   │  routes/assistant.py                                           │
   │     chat_endpoint ─► AssistantService(db, pg_db).chat(...)     │
   │                                                                │
   │  routes/embed.py                                               │
   │     POST /embed           — public utility for any service     │
   │                             to share the 384-d MiniLM vector   │
   │                             space                              │
   │     POST /internal/embed  — internal batch endpoint used by    │
   │                             backfill_missing_embeddings.py     │
   │                                                                │
   │  ┌────────────────────────────────────────────────────────┐    │
   │  │           AssistantService.chat()                      │    │
   │  │                                                        │    │
   │  │  1. embed user_query via SentenceTransformer           │    │
   │  │     (LRU cache, max 64 entries)                        │    │
   │  │                                                        │    │
   │  │  2. AssistantRepository.search_similar_spaces(...):    │    │
   │  │       if pg_db.enabled:                                │    │
   │  │         ─► PostGIS single SQL: cosine k-NN on          │    │
   │  │            building_spaces.embedding (pgvector <=> )   │    │
   │  │            AND ST_DWithin(geometry_global::geography,  │    │
   │  │                ST_MakePoint(lng, lat), radius_m)       │    │
   │  │       else (legacy fallback):                          │    │
   │  │         ─► Cypher db.index.vector.queryNodes(          │    │
   │  │              'space_embedding_idx', k, vec)            │    │
   │  │                                                        │    │
   │  │  3. enrich top-k via Neo4j: floor / building names,    │    │
   │  │     CONNECTS_TO neighbours, is_accessible — Cypher     │    │
   │  │     for graph traversal where SQL would be awkward     │    │
   │  │                                                        │    │
   │  │  4. compose prompt: system rules + retrieved facts +   │    │
   │  │     user_query                                         │    │
   │  │                                                        │    │
   │  │  5. LLM generate (AutoModelForCausalLM +               │    │
   │  │     GenerationConfig; temperature, top_p, max_tokens)  │    │
   │  │                                                        │    │
   │  │  6. {reply, sources:[space_ids]}                       │    │
   │  └──────────────┬─────────────────────────────────────────┘    │
   │                 │                                              │
   │       device / dtype picker (torch):                           │
   │         torch.backends.mps.is_available() → mps + float16      │
   │         torch.cuda.is_available()         → cuda + float16     │
   │         else                               → cpu + float32     │
   │       HF_TOKEN env selects gated / private HF repos            │
   └────┬───────────────────────────────────────────────┬───────────┘
        │ SQLAlchemy + pgvector                         │ Bolt / neo4j+s://
        ▼ (RAG retrieval + spatial filter)              ▼ (graph traversal,
   ┌──────────────────────────┐                          enrichment, anchors)
   │  Supabase PostGIS        │                    ┌──────────────────┐
   │  + pgvector              │                    │ Neo4j (same      │
   │  building_spaces.embedding│                   │ graph as backend)│
   │  geometry_global         │                    │ reads only —     │
   │  (GiST + ivfflat index)  │                    │ assistant never  │
   │  Optional — Neo4j vector │                    │ writes.          │
   │  index is the fallback   │                    └──────────────────┘
   │  when SUPABASE_DB_URL    │
   │  is unset.               │
   └──────────────────────────┘
```

**Why the embeddings are computed once at import.** The backend computes
the 384-d MiniLM vector for each space at `import_service.import_map`
time and writes it to **both** stores: `Space.embedding` in Neo4j and
`building_spaces.embedding` (pgvector `vector(384)`) in PostGIS. The
assistant only ever re-embeds the user's *query* — never the corpus —
so the hot path is one small query-side embed, one indexed SQL or
Cypher call, one LLM generation. The dual-write keeps the migration
reversible: if the pgvector path is disabled the legacy Neo4j vector
index still works.

**Why retrieval lives on PostGIS.** Two reasons. (1) Combining vector
similarity with a spatial filter is one query in PostGIS — `ORDER BY
embedding <=> :q` and `ST_DWithin(geometry_global, :p, :r)` in the same
WHERE clause — but two cross-store calls in the Neo4j-only path (k-NN
in Neo4j, then a `point.distance` pass-or-filter loop in Python).
(2) The pgvector index (`ivfflat` or `hnsw`) is faster than Neo4j's
native vector index at the scales we expect (~10⁴–10⁵ rooms per
deployment) and benefits from the same `org_id`-scoped RLS policies that
already protect the rest of the spatial data.

**Scope.** The assistant answers factual and exploratory questions about
the graph. Strict A→B routing still goes through the backend's
`/navigate` endpoint — the assistant is stateless over its graph reads
and doesn't project a GDS graph of its own.

**Building-scoped retrieval.** When the iOS client sends `building_id`
in the request, the retrieval Cypher narrows to that one building:
`AND building.id = $building_id` is appended to the `WHERE` clause in
`AssistantRepository.search_similar_spaces()` and
`search_spaces_on_floor()`. The reason is that an unscoped k-NN over a
multi-building campus routinely returns spaces in adjacent buildings
that are textually similar but physically irrelevant ("Office 301 in
the *other* building"); scoping collapses the vocabulary the LLM sees
to spaces the user can actually walk to from where they are now.

**Outdoor-query refusal.** The system prompt instructs the LLM to refuse
indoor-direction queries when the caller is outside any mapped
building. The iOS `FloorPlanView.resolveRoute(_:)` short-circuits *before*
the chat call when `nearestVisibleBuilding()` reports the user as more
than 80 m from every visible building's `origin_lat / origin_lng`, and
returns a deterministic "you're outside any indoor space" response with
the closest building's name and distance. The assistant prompt is the
second line of defence in case the iOS gate is bypassed.

### Concrete LLM stack

The default LLM is **`HuggingFaceTB/SmolLM2-360M-Instruct`** —
([services/assistant_service.py:14–92][as]) a 360 M-parameter instruct
model that fits in CPU RAM and runs without a GPU. Hardware detection
picks `mps > cuda > cpu` and `float16 > float32` accordingly. An
`ASSISTANT_MODE=online` env var swaps in an API-backed model (Anthropic /
OpenAI) but the default deployment is offline-first because the platform
needs to work in lab environments without outbound LLM calls.

The embedding model is **`sentence-transformers/all-MiniLM-L6-v2`**
(384-d vectors). Same model on both sides of retrieval — the assistant
re-embeds the query, the import wrote space embeddings into Neo4j +
PostGIS at the same `embedding` field name.

The response shape is a flat JSON `{answer: string, sources: [name…]}`
([models/assistant.py:12–14][am]). **No streaming, no session history.**
Each `/chat` is stateless — the iOS / mapmaker views maintain the
conversation buffer locally and re-send the relevant context as part of
the next user query when they need follow-ups. A streaming SSE upgrade
is the obvious next step but isn't on the critical path because most
answers complete in under 800 ms on `mps` and a chat session rarely runs
more than ten messages.

[as]: aau-sw8-spatial-backend/assistant/services/assistant_service.py
[am]: aau-sw8-spatial-backend/assistant/models/assistant.py

### Intent routing before RAG (the deterministic-first strategy)

A pure RAG-then-LLM pipeline hallucinates: ask SmolLM2 "what rooms are
on the first floor?" with the right rooms in context and it confidently
emits a sentence that contradicts its own context ("…GR13 is not on
First Floor" while GR13 is in the listed sources). The mitigation is a
**regex-based intent layer that runs *before* RAG** and answers
straight from the graph whenever the question fits one of the patterns
we can answer with 100 % grounding. The LLM never sees those.

The pre-filter is `AssistantService.chat()`
([services/assistant_service.py][as]) and dispatches in this exact order
(the order matters — `_floor_count_intent` runs before `_where_am_i_intent`
so "how many floors are in **this building**" isn't hijacked by the
where-am-i match):

| #  | Intent                  | Triggers on                                              | Resolver                                                                          |
| -- | ----------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------------- |
| 1  | `_smalltalk_reply`      | "hi", "thanks", "bye", greetings/farewells              | Canned reply, no graph hit                                                        |
| 2  | `_floor_count_intent`   | "how many floors / levels / storeys"                     | `list_building_floors(campus_id, building_id)`                                    |
| 3  | `_which_floor_intent`   | "which floor am I on", "what's my floor"                 | Reads `floor_index` from the client; falls back to nearest-space floor from GPS  |
| 4  | `_about_building_intent`| "tell me about this building", "what's this building"    | `list_building_floors` formatted as a 1-sentence summary                          |
| 5  | `_vertical_intent`      | "elevator / stairs / lift / escalator / ramp / upstairs" | `vertical_transport_in_building` filtered by the keyword, case-insensitive dedup |
| 6  | `_where_am_i_intent`    | "where am I" (campus / building / general variants)      | `locate_user` (point-in-polygon + nearest, **campus-scoped**)                     |
| 7  | `_needs_global_map` + `_distance_intent` (no GPS) | "closest / farthest X" without a GPS fix | `extreme_space_by_distance` anchored on the main entrance                          |
| 8  | `_navigation_intent` + `_floor_intent`             | "how do I get to floor N"                | `_floor_change_answer` — names the actual stair/elevator node from the graph      |
| 9  | `_floor_intent`         | "what's on floor N"                                      | `search_spaces_on_floor` formatted directly (deterministic listing, no LLM)       |
| 10 | `_find_place_intent`    | "where is X", "is there a Y", "which floor is Z on"      | Vector search with the **two-tier acceptance gate** described below              |
| 11 | catch-all               | nothing else matched                                     | Refuses with example queries (or hands the context to the LLM only when `ASSISTANT_MODE=online`) |

**The find-place two-tier gate (the hallucination guard).** A single
cosine threshold doesn't work: short opaque room codes ("GR5", "2.0.048")
embed weakly with MiniLM-L6-v2 and sit at cosine ~0.18–0.37, while
"where is room 999?" (a non-existent room) also sits at cosine ~0.46
against a plausible-looking match. Either threshold fails one case.

The gate is layered instead:

1. **HIGH cosine (≥ 0.55) on the top result wins outright** — covers
   semantic paraphrases like *"where can I find a restroom"* → Bathroom
   where lexical overlap is zero but embeddings cluster.
2. **Otherwise, scan the top-30 results for a *token match*** — a
   non-stopword token from the query (or its singular form, or a
   sanctioned synonym from a small map: toilet=bathroom=restroom=wc,
   lift=elevator, etc.) must appear in the candidate's `display_name`
   or `space_type`. Accept the first (cosine-ranked) hit that passes.
3. **Otherwise refuse.** *"Where is room 999?"* fails the token check
   against every candidate's name, so we never emit a wrong answer.

This is implemented in [`_find_place_token_match`][as] +
[`_depluralise`][as] and chosen in [`chat()`][as] inside the
`_find_place_intent` branch. The deeper top-K (30, not 10) exists so
weak-embedding queries like *"Where are the WCs?"* — where Bathroom
sits at cosine ~0.15, far below East Corridor at the top — still get
the right answer via the token-match scan.

**Cross-tenant safety in `locate_user`.** The Cypher used to start with
`MATCH (b:Building)` and never filter by campus, so a wrong-coast lat /
lng with `campus_id=campus-demo` would return *"You're near 2.001b
Kontor in A.C. Meyers Vænge 15 (about 80 000 m away)"* — an AAU-CPH
building, leaked across tenants. The fixed query is
`MATCH (b:Building) WHERE b.campus_id = $campus_id` and the result is
gated by `max_nearest_distance_m = 1500` so a user who isn't physically
near *any* mapped building gets a clean refusal instead of a nonsense
proximity claim.

**LLM fallthrough is opt-in.** When no deterministic intent matches,
the default (`ASSISTANT_MODE=offline`) refuses with a polite "try one
of these questions" message. The small default model (SmolLM2-360M) is
never invoked — past experience showed it contradicting its own
context (the GR13 case). Set `ASSISTANT_MODE=online` plus
`ASSISTANT_ONLINE_MODEL_ID` to opt into the LLM path with a capable
hosted model; the system prompt
([core/prompt/prompt.txt][prompt]) is then the second line of defence,
enforcing refusal over fabrication.

[prompt]: aau-sw8-spatial-backend/assistant/core/prompt/prompt.txt

### Concrete trace: "where is room 2.0.048?"

```
   user message: "where is room 2.0.048?"
        │
        ▼
   AssistantService.chat()
        │  smalltalk?           no
        │  where_am_i?          no
        │  which_floor?         no
        │  floor_count?         no
        │  find_place?          YES — matches "where is …"
        ▼
   query.embed()    →  v ∈ ℝ³⁸⁴   (cached up to 64 entries)
        │
        ▼
   AssistantRepository.search_similar_spaces(v, campus_id, building_id?, k=5)
     PostGIS path:
       SELECT id, display_name, floor_index,
              embedding <=> :v  AS dist            -- pgvector cosine
       FROM   building_spaces
       WHERE  campus_id = :cid
         AND  building_id = :bid                   -- when scoped
         AND  ST_DWithin(geometry_global::geography,
                         ST_MakePoint(:lng, :lat)::geography,
                         200)                      -- 200 m around the user
       ORDER BY dist
       LIMIT 5;
        │
        ▼
   top-1 = (space_id="space-2-0-048", name="2.0.048 Lydlab", cosine=0.18)
   score = 1 - 0.18 = 0.82, well above the 0.30 threshold
        │
        ▼  enrich via Neo4j
   MATCH (s:Space {id:"space-2-0-048"})-[:ON_FLOOR]->(f:Floor)
                                       <-[:HAS_FLOOR]-(b:Building)
   RETURN f.floor_index AS floor, b.display_name AS building
        │
        ▼
   Deterministic answer (no LLM call):
   {
     "answer":  "2.0.048 Lydlab is on floor 2 of Cassiopeia.",
     "sources": ["2.0.048 Lydlab"]
   }
```

If the same question is asked with `building_id=null` (the user hasn't
selected a building yet), the scoped `building_id = :bid` filter is
dropped from the SQL; if `cosine > 0.7` (meaning *nothing* in the top-k
is a real match for "2.0.048"), the resolver returns "I don't have
information about that room" instead of falling through to the LLM —
SmolLM2 will happily invent a floor number, the deterministic path
will not.

The LLM-path counterpart ("what's on floor 3?") looks identical until
step 3, then formats the top-k as context lines and asks SmolLM2 to
summarize them under a 1200-token budget. The reply carries a
`sources` list of every space whose name appears in `answer`, which
the iOS / mapmaker UI renders as tappable pills under the chat bubble.

---

## Image pipeline

**Description.** A FastAPI microservice for **batch** (not real-time)
computer vision, running YOLO11n from Ultralytics on uploaded room photos.
Its job is to turn 4 room photos (one per cardinal direction) into a
semantic summary — what furniture is in the room, how many seats, whether
the entrance looks accessible, which fire extinguishers and signs are
visible — and to store that summary plus an image embedding on the
`Space` node in Neo4j. The pipeline also supports **room-to-room similarity
search** so the iOS Explore view and the mapmaker can suggest "rooms that
look like this one." Image-level work happens here to keep the main backend
free of heavy ML dependencies, and to let the YOLO weights + OpenCV install
live in a single specialised container.

```
   iOS ProfileView / RoomPhotoUploadView           mapmaker room editor
         (4 photos: N, E, S, W)                       (single upload)
                    │                                         │
                    └────────────── multipart ────────────────┘
                                         │
                                         ▼
   ┌────────────────────────────────────────────────────────────────────┐
   │                image_pipeline :8002   (FastAPI)                    │
   │                                                                    │
   │  POST /api/v1/room-summary                    (raw images in)      │
   │  POST /api/v1/room-summary/by-room            (room + images)      │
   │  POST /api/v1/room-summary/rooms/{room}/photos (mobile: ≤4 imgs)   │
   │  GET  /api/v1/room-summary/rooms              (list summaries)     │
   │  GET  /api/v1/room-summary/similarity         (k-NN by room_id)    │
   │  GET  /api/v1/room-summary/similarity/nearest (k-NN by space)      │
   │  POST /api/v1/room-summary/similarity/ad-hoc  (k-NN by upload)     │
   │                                                                    │
   │  ┌──────────────────────────────────────────────────────────────┐  │
   │  │              RoomSummaryService (orchestrator)               │  │
   │  │                                                              │  │
   │  │   RoomImageInput → RoomObjectDetector (YOLO11n .pt)          │  │
   │  │                        │ Ultralytics, cv2 pre-processing     │  │
   │  │                        ▼                                     │  │
   │  │                 raw detections (label, conf, bbox)           │  │
   │  │                        │                                     │  │
   │  │                        ├──► RoomVectorizer                   │  │
   │  │                        │     (frequency/label histogram →    │  │
   │  │                        │      dense vector for similarity)   │  │
   │  │                        │                                     │  │
   │  │                        ├──► RoomImageEmbedder                │  │
   │  │                        │     (image-level embedding for      │  │
   │  │                        │      visual nearest-neighbour)      │  │
   │  │                        │                                     │  │
   │  │                        ├──► RoomTextDetector                 │  │
   │  │                        │     (signage OCR where present)     │  │
   │  │                        │                                     │  │
   │  │                        └──► ViewSummary per direction        │  │
   │  │                              (counts, accessibility flags,   │  │
   │  │                               capacity hints, fire safety)   │  │
   │  │                                                              │  │
   │  │   → NamedRoomSummaryResult (all four views + aggregates)     │  │
   │  │   → ImageSimilarity for k-NN search                          │  │
   │  └────────────────────┬─────────────────────────────────────────┘  │
   │                       │                                            │
   │  Configuration lifecycle:                                          │
   │    YOLO_MODEL_PROFILE / YOLO_MODEL_PATH env                        │
   │    YOLO_CONFIDENCE_THRESHOLD env (default 0.25)                    │
   │    modelConfig.cfg + classConf.cfg bundled per profile             │
   │    get_room_summary_service() is @lru_cache(32) so a hot profile   │
   │    stays warm across requests                                      │
   └───────────────────────────────┬────────────────────────────────────┘
                                   │ Bolt
                                   ▼
                          ┌──────────────────┐
                          │  Neo4j (same     │   writes summary, counts,
                          │  graph as backend│   image vectors onto the
                          │  and assistant)  │   Space node under
                          └──────────────────┘   metadata.room_summary
```

**Why this is a separate service.** YOLO11 + Ultralytics + OpenCV pulls in
~2 GB of Torch + CUDA wheels we don't want inside the main backend
container. Isolating it lets the backend stay small and lets this service
be scaled (or GPU-scheduled) independently. The results land back on the
same Neo4j graph the backend and assistant read from, so nothing downstream
cares that the writer is a separate process.

---

## ML-Vision

**Description.** A standalone, separately-repo'd FastAPI service for
**real-time** (WebSocket) and **single-frame** (HTTP) object detection over
a YOLO26 ONNX model fine-tuned on indoor landmarks: doors, staircases,
elevators, exit signs, room-number placards. It's what powers the iOS
CameraView AR overlay and is designed to also serve as the source-of-truth
for the indoor YOLO weights that mobile clients can download for on-device
inference (CoreML for iOS, TFLite for Android). It also hosts a lightweight
`LocationResolver` that maps a detection set to a probable graph location,
so a client can say "where am I?" and get a `Space` id back without running
any routing logic itself. It lives outside the spatial-backend repo because
the ONNX runtime + model artifacts are heavy and the release cadence is
driven by model training, not API changes.

```
           iOS CameraView / Android CameraScreen (when wired)
                          │
                          │ JPEG frames (binary WS messages)
                          │   or single-shot multipart POST
                          ▼
   ┌───────────────────────────────────────────────────────────────────┐
   │               ml_vision :8000   (FastAPI + ONNX Runtime)          │
   │                                                                   │
   │  GET  /health                                                     │
   │                                                                   │
   │  WS   /ws/stream/{facility_id}                                    │
   │    loop:                                                          │
   │      bytes = await ws.receive_bytes()                             │
   │      result = await run_in_executor(_stream_executor, process)    │
   │      await ws.send_text(json.dumps(result))                       │
   │                                                                   │
   │  POST /v1/detect                                                  │
   │    multipart: facility_id, image, navigation_to?, request_id?,    │
   │               confidence_threshold?, iou_threshold?               │
   │    returns: InferenceResponse {                                   │
   │      detections: [ {label, confidence, bbox_xyxy} ],              │
   │      location: LocationEntity,                                    │
   │      route: Optional[list[RouteStep]],                            │
   │      debug: { timings, onnx_path, resolver_debug }                │
   │    }                                                              │
   │                                                                   │
   │  GET  /v1/models/{facility_id}         (list artifacts)           │
   │  GET  /v1/models/{facility_id}/{platform}   ← server/ios/android/ │
   │                                               metadata            │
   │       (directory .mlpackage is zipped on the fly)                 │
   │                                                                   │
   │  ┌─────────────────────────────────────────────────────────────┐  │
   │  │               Per-frame processing                          │  │
   │  │                                                             │  │
   │  │   @lru_cache(16) get_detector(facility_id)                  │  │
   │  │     → loads yolo26_indoor.onnx from                         │  │
   │  │       MODEL_DIR/{facility_id}/                              │  │
   │  │                                                             │  │
   │  │   detector.infer(bytes) ─► list[Detection]                  │  │
   │  │                                                             │  │
   │  │   normalize bboxes to (0..1) in Vision-style coords         │  │
   │  │     (left / bottom origin for iOS overlay compatibility)    │  │
   │  │                                                             │  │
   │  │   LocationResolver.resolve(facility, detections,            │  │
   │  │                            navigation_to?)                  │  │
   │  │     → LocationEntity + optional RouteStep list via          │  │
   │  │       Neo4j queries against the spatial graph               │  │
   │  └──────────────────────────────┬──────────────────────────────┘  │
   │                                 │                                 │
   │  ThreadPoolExecutor(max_workers=2) keeps ONNX inference off the   │
   │  asyncio loop so the WS can stay responsive to backpressure.      │
   └─────────────────────────────────┼─────────────────────────────────┘
                                     │
                                     ▼  (optional graph lookups)
                              ┌──────────────┐
                              │  Neo4j       │
                              │ (same graph) │
                              └──────────────┘
```

**Gateway bridging.** The iOS/Android client never talks to this service
directly. The middleware's WS handler at
`/api/v1/ml-vision/ws/stream/{facility}` opens a second upstream WS via
`websockets.connect()` and runs two asyncio tasks (client→upstream,
upstream→client) until either side closes, at which point it cancels the
other. For the HTTP detection and model-download endpoints it's just the
generic reverse-proxy with a 900s timeout.

---

## Mapmaker (frontend)

**Description.** A single-page, static HTML + JS app served by nginx that
is the authoring surface for the entire platform. Everything a mobile
client eventually renders — campuses, buildings, floors, rooms, corridors,
doors, connections, room-type metadata — is authored here. It is
intentionally "no framework": one `index.html` with vanilla JS, canvas
rendering, and a tiny in-memory `state` object; it calls the middleware
over `/api/v1/*` with `X-Api-Key` just like a mobile client would. Having
the authoring tool speak the same public API as the consumers means the
backend can't have secret endpoints that only the tool uses — if the
mapmaker can do it, the mobile apps can too.

```
  ┌──────────────────────────── Sidebar ────────────────────────────┐
  │  Map panel                                                      │
  │    Organization select  ──► GET /organizations                  │
  │    Campus select        ──► GET /campuses?organization_id=…     │
  │    Building select      ──► GET /campuses/{id}/export           │
  │    Floor select         ──► GET /buildings/{id}/floors          │
  │    [+ Building] [+ Floor]                                       │
  │    [Delete Building] [Delete Floor]                             │
  │    [Delete Organization]  ← double-confirm (type the name)      │
  │    [+ New Map]  [Export JSON]                                   │
  │                                                                 │
  │  Import Map (JSON)   ──► POST /campuses/{id}/import             │
  │  Import DWG / DXF    ──► POST /campuses/import-dxf              │
  │      (multipart upload + form fields for campus / building /    │
  │       floor IDs and names, optional org + origin lat/lng)       │
  │                                                                 │
  │  Search panel ──► GET /search?q=                                │
  │  AI Assistant panel ──► POST /assistant/chat                    │
  │  Edit Tools (visible once a floor is loaded):                   │
  │    Select · + Space · Connect                                   │
  │    add-space form: name, space_type, accessibility,             │
  │                    capacity, dimensions                         │
  └─────────────────────────────────────────────────────────────────┘

  ┌────────────────────── Canvas (HTML5 2D) ────────────────────────┐
  │    zoom (wheel) + pan (drag empty space)                        │
  │    grid snapping while drawing                                  │
  │    renders:                                                     │
  │      • floor polygons + fill by space_type                      │
  │      • doors / connections (edges across rooms)                 │
  │      • centroid labels, accessibility icons                     │
  │      • pending polygon while creating                           │
  │    context menu: rename, edit type, delete,                     │
  │                  "connect from here"                            │
  └─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                 state = { organizations, campuses,
                           buildings, floors, spaces,
                           connections, selectedFloorId,
                           pan, zoom, mode, … }
                               │
                               ▼
            api(path, { method, body }) helper
              • injects X-Api-Key from window.API_KEY
              • parses JSON
              • on !ok ⇒ notify('…', 'error')
                               │
                               ▼
          Middleware /api/v1/*  (same surface as mobile)
                               │
                               ▼
               backend ──► Neo4j + PostGIS
          assistant ──► Neo4j (chat panel)
```

**Why mutations go through the middleware.** A room drop fires
`POST /api/v1/spaces`, and a door drop fires `POST /api/v1/connections` —
the latter is the single call that creates 4 Neo4j edges *and* 4 PostGIS
rows. Because there is no direct DB access from the browser, the two
stores can't drift apart regardless of which tab or editor a user is
running.

**Delete flow.** Each delete button is disabled until its corresponding
select has a value. Building / floor use one `confirm()`; organisation
requires a `confirm()` plus a `prompt()` where the user must type the
organisation's exact name. After success the relevant parent select is
re-dispatched so cascading state (child dropdowns, buttons) resets without
a page reload.

**DWG / DXF import flow.** The "Import DWG / DXF" panel posts a
`multipart/form-data` request to `POST /api/v1/campuses/import-dxf`
carrying the CAD file plus the metadata the canonical `MapImportSchema`
needs (campus / building / floor ids, optional organization, optional
origin lat / lng). The browser auto-fills the multipart boundary —
the panel deliberately does *not* set a `Content-Type` header on the
`fetch()`, because doing so would suppress the boundary and the server
would parse the body as a single opaque blob. The `api()` helper is a
thin wrapper that injects auth headers and parses the JSON response;
it is unopinionated about the body type, which is what makes the
multipart upload work cleanly with no special-casing.

---

## Indoor data pipeline (CAD import)

ARIADNE ingests floor plans through two sibling pipelines that converge
on the same atomic write. The JSON pipeline is what an editor uses for
hand-authored or programmatically-generated maps; the DWG / DXF pipeline
is what an editor uses when the architect ships a CAD file.

The DXF leg is itself **two stages** — a library-driven normalize step
that owns all ezdxf interaction, and a project-specific convert step
that owns all room/door/schema logic. Stage 1 produces an intermediate
`normalized_entities.json` (or in-memory dict in the HTTP path); Stage 2
consumes that dict and emits the canonical `MapImportSchema`. Splitting
the work this way means we never reinvent CAD geometry handling (Stage
1 is essentially `ezdxf` plus a flat-record emitter) and every
ARIADNE-specific rule lives in one auditable place (Stage 2).

```
   Hand-authored / generated                  Architect CAD export
   ─────────────────────────                  ────────────────────────
   .json (MapImportSchema)                    .dwg                 .dxf
            │                                    │                   │
            │                         (.dwg only) shell-out          │
            │                            to ODAFileConverter         │
            │                            or LibreDWG → .dxf          │
            │                                    │                   │
            │                                    └─────────┬─────────┘
            │                                              │ DXF bytes
            │                                              ▼
            │                       ┌──────── STEP 1 — DXF → JSON ─────────┐
            │                       │  services.dxf_normalize              │
            │                       │  ezdxf-driven walk of modelspace;    │
            │                       │  emits flat per-entity records       │
            │                       │  (no ARIADNE schema yet, no          │
            │                       │   geometry interpretation)           │
            │                       └──────────────────┬───────────────────┘
            │                                          │
            │                                          ▼  normalized JSON
            │                                          │  { entities[],
            │                                          │    layers[],
            │                                          │    header: $INSUNITS }
            │                                          ▼
            │                       ┌──── STEP 2 — JSON → MapImportSchema ─┐
            │                       │  services.dxf_convert                │
            │                       │  Owns every ARIADNE-specific rule:   │
            │                       │  unit scaling, polygonize, label     │
            │                       │  attach, classify, door inference,   │
            │                       │  UUIDs, paired bidirectional         │
            │                       │  connections                         │
            │                       └──────────────────┬───────────────────┘
            │                                          │  MapImportSchema dict
            │                       ◄──────────────────┘
            ▼
   POST /campuses/{id}/import         POST /campuses/import-dxf
   (Pydantic validation)              (Pydantic validation)
            │                                          │
            └──────────────────┬───────────────────────┘
                               │
                               ▼
                 ImportService.import_map(schema)
                               │
                               ▼
       Neo4j (entities + edges)   +   PostGIS (geometry mirror)
        Neo4j write (sync, authoritative) → :SyncOutbox enqueue
              → background worker mirrors to PostGIS (eventual)
                               │
                               ▼
                  GdsService.refresh_projection()
                               │
                               ▼
              Routing endpoint sees the new topology
```

**Stage 1 — `services.dxf_normalize.normalize_bytes(file_bytes, source_name)`**
turns a DXF (or transcoded DWG) buffer into a library-shaped JSON
record. This stage does no geometry interpretation — it just walks
modelspace and emits one dict per CAD primitive. The output mirrors
the well-known `dxf-json` Node parser so a different converter (or a
non-Python consumer) could in principle replace Stage 2.

- **Robust DXF read.** Uses `ezdxf.recover.readfile` rather than the
  strict `ezdxf.read(StringIO)` path. The recovery reader handles
  ASCII DXF, binary DXF, files with non-UTF-8 encodings, and slightly
  malformed files that the strict reader rejects. Every recovery hint
  the auditor produces is collected into `auditor_warnings` so the
  editor can see what was salvaged.
- **DWG transcoding + sanitization.** A `.dwg` source is shelled out
  to ODAFileConverter or LibreDWG `dwgread` to produce a DXF; if
  LibreDWG emits an out-of-spec DXF, `_sanitize_libredwg_dxf` strips
  malformed group codes and the recover-read is retried once.
- **INSERT explosion with layer inheritance.** Every `INSERT` block
  reference is recursively exploded via `entity.virtual_entities()`
  (depth-capped at 6) so rooms authored as repeated blocks come out
  as concrete geometry. A child sitting on layer `"0"` inherits its
  parent INSERT's layer so classification works on the named layer
  rather than the anonymous default.
- **Flat record shape.** Each emitted entity is `{type, layer, …}`
  with type-specific fields: vertex lists for polylines, start/end
  for lines, center+radius(+angles) for circles/arcs, position+text
  for text entities, loop point-lists for hatches. Plus a top-level
  `layers` summary (per-layer color + entity count) and a `header`
  subset that always includes `$INSUNITS`.

**Stage 2 — `services.dxf_convert.convert(normalized, …)`** turns the
normalized dict into the canonical export schema:

- **Unit scaling from `$INSUNITS`.** AAU CAD files ship as
  `$INSUNITS=4` (millimeters) so every coordinate is multiplied by
  0.001 before polygonization compares it to the meter-scale room-area
  thresholds. When the header is missing (`$INSUNITS=0`) the bbox
  span of the geometry is used as a fallback heuristic
  (>5000 → mm, >200 → cm, else m).
- **Closed-polyline + hatch rooms first.** Any LWPOLYLINE / POLYLINE
  flagged closed (or whose first vertex equals its last) is taken as
  a room directly. HATCH boundary loops are taken as rooms too. These
  are the rooms that need no inference — the draftsman drew them as
  enclosures.
- **Polygonization from linework.** The bulk of architectural DXFs
  draw rooms as wall *lines* rather than closed shells, so all LINE /
  ARC / CIRCLE / open-polyline segments are unioned, endpoint-snapped
  to a ~20 mm grid (in mm-scale files), and fed to
  `shapely.ops.polygonize`. Snap tolerance is the load-bearing knob:
  too tight and corners don't close, too loose and unrelated rooms
  merge. The outer-envelope detector drops a polygon that contains
  half-or-more of the other polygons and is at least 8× the median
  area — that's the whole-floor outline, not a room.
- **Polygon validation + dedupe.** Self-intersecting shells are
  repaired with `shapely.validation.make_valid` (largest piece kept);
  polygons under `_MIN_ROOM_AREA_M2` (1.5 m²) or over
  `_MAX_ROOM_AREA_M2` (500 m²) are dropped as noise/envelopes. Vertex
  sequences are rounded to 4 decimal places and deduped.
- **Label attachment with fallbacks.** Each TEXT / MTEXT / ATTRIB is
  point-in-polygon-tested against every polygon. Within-polygon
  matches preferred over near-boundary (`_LABEL_BOUNDARY_BUFFER_M`,
  0.6 m), smallest qualifying polygon wins (innermost room). Polygons
  still unlabeled try a nearest-external-label sweep within 3 m of
  their centroid — rescues leader-line / outside-the-wall labels.
  Polygons that strike out on every pass are *dropped* unless their
  layer is in the optional `layer_mapping` override, because the
  pre-strict version of this rule leaked sinks, hatch fragments and
  partial wall loops through as `ROOM_GENERIC`.
- **Classification.** Label-first (`*LAGER*` → `ROOM_STORAGE`,
  `*KONTOR*` → `ROOM_OFFICE`, `*WC*` → `RESTROOM`, …), layer-second
  fallback, `layer_mapping` override beats both. The full pattern
  list is in `_LABEL_TYPE_PATTERNS` in `dxf_import_service.py`.
- **ID-type pair merge.** CAD plans often place a room's ID label
  (`"2.0.041"`) and its type label (`"Audio Workshop"`) several
  meters apart inside the same physical room, which produces two
  adjacent polygons after polygonization. Pairs with centroids within
  5 m where one carries digits and the other doesn't are folded into
  the ID-bearing space, with the type-label promoted into the
  classification.
- **Door inference.** ARCs whose radius is 0.4–1.5 m and whose sweep
  is 45°–130° are tagged as door swings — that's the geometric
  signature of a drawn door leaf. The mid-arc point is offset half a
  radius into the swing arc and checked against the two nearest
  polygons within 0.4 m. Any ARC matching two distinct rooms becomes
  a `DOOR` connection; entities on a `DOOR*` / `DØR*` layer
  contribute candidates too.
- **Export shape.** Spaces get fresh UUID ids stamped with
  `organization_id`. Connections are emitted as **bidirectional
  pairs** (`a→b` plus `b→a`) so the navigation graph treats each door
  as two directed edges. The floor record carries
  `floor_plan_scale=1`, `floor_plan_origin_x=0`,
  `floor_plan_origin_y=0`; the building carries `origin_bearing` and
  `short_name`. Output validates against `MapImportSchema` without
  modification.

**Single source of truth, two callers.** The same two functions —
`normalize_bytes` / `normalize_path` and `convert` — power both the
HTTP route and the CLI tools:

- The backend route `POST /campuses/import-dxf` calls
  `await asyncio.to_thread(normalize_bytes, …)` then
  `dxf_convert.convert(…)` on a worker thread (both are CPU-bound).
- The CLI ships three thin shells in `scripts/`:
  `dxf_normalize.py` (Stage 1 only), `normalized_to_export.py`
  (Stage 2 only), and `dxf_pipeline.py` (both in one shot). All three
  add `aau-sw8-spatial-backend/backend` to `sys.path` and call the
  same backend modules — there is no duplicated implementation.

**Dry-run mode.** The route accepts a `dry_run=true` form field. When
set, the pipeline runs end-to-end (including all the validation,
classification, and door inference above) but the response is the
parsed schema preview —
`{rooms_detected, classification_summary, warnings, preview_spaces,
preview_connections}` — *without* writing to either database,
refreshing the GDS projection, or appending to `audit_log`. The
mapmaker exposes this as a "Preview (dry-run)" button next to
"Import DXF" so editors can sanity-check the layer-mapping result
before committing.

**Layer-mapping override.** A second form field, `layer_mapping`,
takes a JSON object of the form `{"WALLS_OFC_01": "ROOM_OFFICE",
"WALLS_HALL_01": "CORRIDOR"}` and overrides the default heuristic for
the named layers. Editors with non-conventional CAD layer names use
this to avoid the everything-is-`ROOM_GENERIC` outcome the heuristic
otherwise produces. The mapping is JSON-validated server-side; a
malformed mapping returns 422 with the parse error.

**DWG support is shell-out only.** DWG is a closed proprietary AutoCAD
binary format with no robust pure-Python parser. The service detects a
`.dwg` upload and shells out to whichever converter is available, in
order of preference:

1. **ODAFileConverter** (proprietary, freely downloadable from the
   Open Design Alliance, not redistributable). Highest fidelity on
   the long tail of AutoCAD versions and edge cases.
2. **`dwgread` from LibreDWG** (GPL, freely redistributable, but not
   packaged as a runnable binary in Debian — you have to build it
   from upstream source). Sufficient fidelity for the floor-plan
   vocabulary the parser actually consumes (`LWPOLYLINE`,
   `POLYLINE`, `TEXT`, `MTEXT`, `INSERT`).

If neither converter is on `PATH` the service raises `FileNotFoundError`
and the route surfaces it as **HTTP 415** with a remediation message
listing both install paths plus the recommended workaround (convert the
`.dwg` to `.dxf` locally, upload the `.dxf`).

The default backend Docker image ships **neither** converter — DWG
support is opt-in via a build arg so the default image stays small and
free of long compile steps. The two opt-in paths are:

```
# Free / redistributable: builds LibreDWG from source. ~2 minutes on
# first build, cached after that.
docker build --build-arg INSTALL_LIBREDWG=1 -t ariadne-backend backend/

# Higher fidelity: install ODAFileConverter from a mirror you control
# (the binary itself is freely downloadable but not redistributable).
docker build \
  --build-arg INSTALL_ODA=1 \
  --build-arg ODA_DEB_URL=https://your-mirror/ODAFileConverter_QT5_lnxX64_8.3dll_25.4.deb \
  -t ariadne-backend backend/
```

When ODAFileConverter is installed the Dockerfile pulls in the Qt +
Xvfb runtime dependencies and replaces `/usr/bin/ODAFileConverter`
with a small shim that wraps the real binary in `xvfb-run` — the
binary is a Qt GUI app and refuses to start without an X display, so
the shim is what makes the headless server-side transcoding actually
work.

The two flags can be combined (`--build-arg INSTALL_LIBREDWG=1
--build-arg INSTALL_ODA=1 ...`); the runtime tries ODA first and
falls back to `dwgread`, so installing both gives you the best
converter at no extra cost.

**Why both pipelines land on the same `ImportService`.** The sync
guarantee — that Neo4j commits authoritatively and PostGIS converges to
match via the durable outbox — is the most expensive thing in the
system to get right. Putting the DXF parser upstream of
`ImportService.import_map` rather than letting it write directly means
the DXF flow inherits that guarantee unchanged: the parser produces a
`MapImportSchema` dict, the schema is Pydantic-validated, and the same
Neo4j-first-then-outbox path that backs JSON imports backs DXF imports
too.

---

## Routing & navigation

End-to-end: how a tap on a room turns into the blue polyline the user
follows from their current position to the destination, what runs where,
and what each leg costs.

```
   iOS                Gateway              Backend                  Neo4j
   ───                ───────              ───────                  ─────
   user taps a room
     in FloorPlanView
            │
            │ resolveRoute(to: destination)
            │ → NavigationService
            ▼
   GET /api/v1/navigate?from=&to=&accessible_only=&avoid_stairs=&elevators_only=
            │
            ▼
   Gateway forwards (injects principal headers)
            │
            ▼
   routes/navigation.py:navigate(from, to, accessible_only,
                                  avoid_stairs, elevators_only)
            │
            ▼
   NavigationService(db).get_route(from, to,
                                   accessible_only=…,
                                   avoid_stairs=…, elevators_only=…)
            │
            ▼
   NavigationRepository.find_path
            │   excluded_types = _excluded_types(avoid_stairs, elevators_only)
            │     avoid_stairs   → {STAIRCASE, ESCALATOR}
            │     elevators_only → {STAIRCASE, ESCALATOR, RAMP}
            │
            │── if no filter is active, try GDS Dijkstra first  ─►  gds.shortestPath
            │   (weighted by 'weight' = traversal_cost,             .dijkstra.stream
            │    fastest on dense graphs)                           on "navigation-graph"
            │                                                       projection
            │
            │── otherwise (accessible_only=true OR any space type   shortestPath(
            │    excluded) fall through to native Cypher       ───►   (s)-[:CONNECTS_TO
            │    shortestPath. Per-request node-type filters         *..100]->(t))
            │    can't be expressed in the GDS projection            filtered by
            │    without rebuilding it, so the native path is        is_navigable +
            │    where accessibility AND space_type filters live.    is_accessible +
            │    Endpoints (from/to) bypass the type filter so       space_type ∉
            │    a STAIRCASE destination still resolves.             excluded_types
            ▼
   list[dict] of path nodes (id, display_name, space_type,
                             floor_index, centroid_x/y, centroid_lat/lng,
                             traversal_cost)
            │
            ▼
   NavigationService wraps each node in a RouteStep, generates the
   per-step instruction ("Walk through 2.0.001 Gang A", "Go through
   door to 2.0.048 Lydlab", "Take elevator to 1. Sal", …),
   detects floor changes + building changes from consecutive nodes,
   returns a Route { steps, total_cost, floor_changes, building_changes }
            │
            │ JSON response (Pydantic-serialized)
            ▼
   iOS NavigationService decodes into NavigationRoute / NavigationStep
            │
            ▼
   FloorPlanView.rebuildRouteCoordinates(_:)
            │
            │ routeCoordinates = [userLocation] + steps.map(centroid_lat/lng)
            ▼
   MapViewWithOverlay.updateUIView(_:) renders two MKPolyline overlays:
     · 9 pt white "route-halo" underneath
     · 5 pt systemBlue "route-main" on top
     · MKPointAnnotation "Destination" pinned at the final coordinate
```

**The graph itself.** The route is *not* a straight line from user to
destination — it's a sequence of edges through the navigation graph
that the CAD-import pipeline emitted. Every door becomes two directed
`CONNECTS_TO` edges (a→b and b→a). Floors stitch together through
STAIRCASE / ELEVATOR / ESCALATOR / RAMP nodes; buildings stitch
through BRIDGE / TUNNEL / COVERED_WALKWAY. Edge weight is
`traversal_cost` (defaults to a unit for plain rooms, higher for
vertical circulation, capped per heuristic). So a 7-segment polyline
on screen corresponds to 7 consecutive graph hops the server
computed, not 7 visual waypoints.

**Two routing strategies, one repository.** [navigation_repo.py](aau-sw8-spatial-backend/backend/repositories/navigation_repo.py)
tries the GDS projection first because it's weighted (`traversal_cost`)
and substantially faster on dense graphs:

```cypher
CALL gds.shortestPath.dijkstra.stream($projection, {
    sourceNode: start, targetNode: end,
    relationshipWeightProperty: 'weight'
})
```

The GDS path is taken **only when no per-request filter is active** —
i.e. `accessible_only=false`, `avoid_stairs=false`, `elevators_only=false`.
The projection bakes the graph at refresh time, so per-request node-
type filters would require rebuilding it for every navigation call,
which is wasteful for an interactive endpoint. As soon as any filter
is on, or the GDS plugin / projection isn't available, the repo falls
through to a native Cypher `shortestPath` that applies all three
filters at the graph level:

```cypher
MATCH path = shortestPath((start)-[:CONNECTS_TO*..100]->(end))
WHERE ALL(n IN nodes(path) WHERE n.is_navigable = true OR n.id IN [$from, $to])
  AND ($accessible_only = false OR ALL(n IN nodes(path) WHERE n.is_accessible))
  AND ALL(n IN nodes(path)
          WHERE n.id IN [$from, $to] OR NOT n.space_type IN $excluded_types)
```

Endpoints (the user's start/destination spaces) are always allowed
through the type filter — you don't get a no-path because your
destination happens to be on a `STAIRCASE` node. Either strategy
returns the same `path_nodes` shape so `NavigationService` doesn't
care which one ran.

The iOS client reads `@AppStorage("nav.avoidStairs")` and
`@AppStorage("nav.elevatorsOnly")` (the toggles in `ProfileView`)
and forwards them on every `computeRoute` call, so flipping a toggle
on the Profile tab immediately changes the next route the user asks
for.

**Why each Space carries a centroid in BOTH coordinate systems.** The
import writes `centroid_x` / `centroid_y` (local floor-plan meters,
relative to the building origin) AND `centroid_lat` / `centroid_lng`
(WGS84, computed in `geometry_service.py` from the building's origin
+ bearing). The iOS map renders against the lat/lng pair so the
polyline lives in the same coordinate system as the user's CoreLocation
fix; floor-plan-relative tools (the floor-plan canvas on the mapmaker,
the indoor-positioning math) use the x/y pair. The CAD import's
`origin_bearing` is what makes the rotation between the two systems
correct.

**Polyline rendering details.** In `FloorPlanView.swift`, the
`UIViewRepresentable`-wrapped `MKMapView` calls
`rebuildRouteCoordinates` whenever `NavigationService.currentRoute` or
the user's location changes. The user's CoreLocation fix is *prepended*
to the step centroids so the line starts at the blue dot, not at the
first step. Two `MKPolyline` overlays are added — a 9 pt white halo
(`route-halo`) and a 5 pt `systemBlue` main line (`route-main`) — so
the route reads against any MapKit basemap. An `MKPointAnnotation` is
pinned at the last coordinate so an offscreen destination still has a
visible target.

**Cancel / rebuild path.** `cancelRoute()` zeroes
`navigationService.currentRoute`, clears `routeCoordinates`, and the
next `updateUIView(_:)` removes the polylines + destination pin. A
new `resolveRoute(to:)` (typically from the assistant tab's "Show on
map" or from a list tap) issues a fresh `GET /navigate` and the
overlays repaint from the new step list.

**Graph refresh after import.** After any successful
`ImportService.import_map`, the route handler calls
`GdsService(db).refresh_projection()` so the dijkstra path picks up
the new spaces + connections without a server restart. The
`POST /navigate/refresh-graph` endpoint exposes the same hook for
manual rebuilds.

### Edge-weight math (door-aware + cross-floor 3D)

The dijkstra weight is **not** just "1 per edge" — it's wall-clock
seconds for an average walker (1.4 m/s), with two non-trivial
geometric refinements that make the polyline render where humans
actually walk. The full logic lives in `_edge_weight` /
`_room_exit_point` in [python_dijkstra.py][pd].

[pd]: aau-sw8-spatial-backend/backend/services/python_dijkstra.py

**Three branches** depending on which kind of edge is being weighted:

```
   1. SAME-FLOOR, SAME-BUILDING                  ┌──────────────────┐
   ─────────────────────────────                 │   Room A         │
                                                 │      *  ───┐     │
                                                 │   centroid │     │
   The two endpoints are real rooms on           │            ▼     │
   the same floor. If one of them is a           │       ┌────●─────┘  ← door (connector)
   connector (door / corridor / hub),            │       │
   we snap the non-connector room's              │       ●  ←  closest_point_on_polygon
   centroid to the polygon EDGE nearest          │       │     of A's outline
   to the connector's centroid, then             │   Room B (a corridor connector)
   measure euclidean distance to the             │
   door. This stops the polyline from            └──────────────────┘
   cutting through walls.

       sx, sy = _room_exit_point(room=A, door=B.centroid)
       dist_m = hypot(sx − B.centroid_x, sy − B.centroid_y)
       cost_s = dist_m / 1.4 + B.traversal_cost

   2. CROSS-FLOOR, SAME-BUILDING (stairs / lift)
   ─────────────────────────────────────────────

   Real 3D Pythagorean distance.  The two              Floor 3   ●
   endpoints have the same building but                          │\
   different floor_index — the edge crosses                      │ \
   vertical transport.  We compute                               │  \  ← actual diagonal
                                                                 │   \    travelled
   horizontal = hypot(sx − dx, sy − dy)             Floor 2  ────│────●
   vertical   = | floor_step | * 3.3 m              floor_step=1 │    (next-floor exit)
   dist_m     = hypot(horizontal, vertical)                      │
                                                                 │  3.3 m default floor
   (3.3 m is _DEFAULT_FLOOR_HEIGHT_M; would                      │  height (constant
   come from floor.elevation_m once that's                       │   _DEFAULT_FLOOR_HEIGHT_M)
   uniformly populated, but defaulting to a            Floor 1 ──┴
   global constant lets routing work on
   buildings that haven't yet been surveyed
   for slab heights.)

   3. FALLBACK
   ───────────

   Either side missing a centroid, or cross-building, or unparseable
   polygon — fall back to a flat 1 m so dijkstra still terminates with
   a sensible-ish answer rather than NaN-ing out.
```

The seconds-cost is then `dist_m / 1.4 + destination.traversal_cost`,
where the second term is the per-node penalty (e.g., elevators cost
30 s of waiting, stairs cost 10 s of climbing time on top of the
horizontal-equivalent stride). Without the penalty a 1-floor staircase
trip would tie with a 4 m corridor walk, which doesn't match how long
either actually takes.

### Polygon-edge snapping (the "doors not centroids" trick)

The 2-D refinement in branch 1 above and the polyline endpoint snap in
`navigation_service._build_polyline._endpoint_exit_point` share one
helper: [`closest_point_on_polygon`][gs] in `geometry_service.py`. Given
a polygon `[(x₁,y₁), …, (xₙ,yₙ)]` and a target point `(tx, ty)`, it
projects the target onto every edge segment and returns the projection
with minimum squared distance:

```python
for each segment (a, b):
    dx, dy = b.x - a.x,  b.y - a.y
    t = ((tx - a.x)*dx + (ty - a.y)*dy) / (dx*dx + dy*dy)
    t = clamp(t, 0.0, 1.0)               # constrain to segment
    p = (a.x + t*dx, a.y + t*dy)
    d2 = (tx - p.x)**2 + (ty - p.y)**2
keep the (p, d2) with min d2
```

This is used in two places end-to-end:

1. **During path search** ([python_dijkstra.py][pd] `_room_exit_point`)
   to give the dijkstra weight a "human-walkable" distance instead of a
   centroid-to-centroid one. Otherwise a 30 m room with a door at one
   end gets a 15 m weight when the user actually only walks 5 m.
2. **During polyline build** ([navigation_service.py][ns]
   `_endpoint_exit_point`) — the *visual* polyline's first segment is
   "user-position → polygon-edge-of-start-room nearest the next step"
   rather than "user-position → start-room centroid". Same on the last
   segment. The other interior segments stay at centroids because
   they're connector-to-connector and the centroids already lie on the
   walkable path.

```
   Before (centroid-only)               After (polygon-edge snap)
   ────────────────────────             ────────────────────────────

      ┌─────────┐                          ┌─────────┐
      │   ●─────┼──────→  to dest          │         │
      │  centroid                          │         ●─────→  to dest
      └─────────┘                          └─────────┘
        polyline cuts the              polyline exits at the door —
        polygon at an arbitrary        the same point a human would
        boundary point because         walk through
        of the straight line
```

[gs]: aau-sw8-spatial-backend/backend/services/geometry_service.py
[ns]: aau-sw8-spatial-backend/backend/services/navigation_service.py

### Distance calculations across the platform

Every "how far is X from Y" in the codebase is one of these five
distance functions. Each has a specific reason for the metric it uses.

| Distance                       | Formula                                             | Where it lives                                                       | Why this metric                                                                 |
| ------------------------------ | --------------------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| **Same-floor walk**            | `sqrt((x₂−x₁)² + (y₂−y₁)²)`  (local meters)         | `python_dijkstra._edge_weight` (branch 1) + `geometry_service.distance_m` | Local-frame floor plans are already in meters and effectively flat; no projection error. |
| **Cross-floor walk (3D)**      | `sqrt(horizontal² + (Δfloor × 3.3)²)`               | `python_dijkstra._edge_weight` (branch 2)                            | Vertical transport is a real distance cost, not a flat penalty; 3.3 m is `_DEFAULT_FLOOR_HEIGHT_M`. |
| **Polygon edge projection**    | min over edges of `t-clamped (tx−ax)·d + (ty−ay)·d` | `geometry_service.closest_point_on_polygon`                          | Snaps a centroid to the nearest *walkable* boundary point (door / corridor entry). Used by routing + polyline build. |
| **Room traversal cost**        | `sqrt(width² + length²) / 2 / 1.4`                  | `navigation_service` (per-node penalty)                              | Half-diagonal is the expected through-room walk at the average pace; better than counting "one second per room". |
| **WiFi fingerprint distance**  | `Σ (rssi_live − rssi_db)² + 25² × |miss|`           | `wifi_positioning._fingerprint_distance`                             | Squared dBm distance with a fixed missing-AP penalty; tolerates one AP flicker, distinguishes rooms with materially different visibility. |
| **iOS arrival check / dot proximity** | Haversine on lat/lng                         | `FloorPlanView.haversineMeters` + `CLLocation.distance(from:)`       | Polyline / annotations are in WGS84, so the proximity check uses the same projection (great-circle, not flat). |

```
   Geographic vs local frames
   ──────────────────────────

   Indoor math (rooms, doors, edges, fingerprints):
      LOCAL FRAME — origin at the building corner, meters,
      flat earth.  All Euclidean distances above.
                                            
                                            ▲ y (m)
                                            │
                                            │   ┌─────┐
                                            │   │ room│
                                            │   └─────┘
                                            └────────────► x (m)

   Outdoor + map rendering (polyline, user dot, snap proximity):
      WGS84 — lat / lng. Haversine.  Local-frame coords get
      lifted to WGS84 via the floor's
        (origin_lat, origin_lng, origin_bearing, scale_factor)
      at write time, so consumers don't have to project on every read.
```

The pre-write conversion is why every space + every landmark carries
**both** coordinate pairs: indoor math wants the flat meters, map
rendering wants the geographic coords, and we'd rather pay 16 bytes per
row than redo trigonometry on every API response.

---

## Landmark precise location

**Description.** A landmark is a registered visual cue — a poster, a sign,
a door placard — that the camera ML pipeline can recognise on the wire
to produce a "you are at exactly *here*" snap. Until recently every
landmark resolved to the centroid of its containing room, which on a
30 m corridor meant the dot could teleport 15 m away from where the
user was actually standing. The landmark schema and the snap pipeline
now carry **four coordinate fields end-to-end** so the dot lands on the
landmark, not the room's middle.

```
                       LANDMARK COORDINATE FIELDS
                       ──────────────────────────

   centroid_x, centroid_y          centroid_lat, centroid_lng
   ─────────────────────           ───────────────────────────
   Local floor frame, in           WGS84, derived from the same point
   meters from the building's      via the floor's origin_lat/lng/bearing
   origin.  Same coordinate        + scale_factor.  Same coordinate
   system as Space.centroid_x/y    system as the iOS / Android map dot
   (used by mapmaker canvas,       (used for the polyline rendering, the
   image-pipeline / orb_matcher,   forced-location annotation, distance
   floor-plan editing).            calculations).
```

Both pairs are persisted because each downstream consumer uses one or
the other:

- The **floor-plan editing canvas** in the mapmaker operates in local
  meters (the floor image is drawn in its own coordinate frame), so a
  landmark drawn at `(centroid_x, centroid_y)` lands at the right pixel
  regardless of the building's bearing.
- The **iOS map** is a `MKMapView` in WGS84, so the forced-location
  annotation must be set from `(centroid_lat, centroid_lng)` directly.

Storing both is cheap (16 bytes per landmark) and saves a coordinate
conversion on every read.

### Registration: client → API → Neo4j → PostGIS

```
   iOS / Android camera tap "Register landmark"
        │
        │ jpeg + name + space_id + (lat, lng) if known
        ▼
   POST /api/v1/ml-vision/landmarks/register   (multipart/form-data)
        │   centroid_x, centroid_y, centroid_lat, centroid_lng  all optional
        ▼
   routes/landmarks.py:register_landmark
        │
        │ _resolve_landmark_coords(...)
        │   if (lat,lng) present and (x,y) missing:
        │       (x,y) = global_to_local_coordinates(lat, lng,
        │                                           floor.origin_lat,
        │                                           floor.origin_lng,
        │                                           floor.origin_bearing,
        │                                           floor.scale_factor)
        │   if (x,y) present and (lat,lng) missing:
        │       (lat,lng) = local_to_global_coordinates(x, y, …same params)
        │   else: pass through (idempotent)
        │
        ▼
   LandmarkRepository.create_landmark(...)
        │  Cypher MERGE on (space, name), SET all four coords + image_b64
        ▼
   Neo4j: (:Landmark {centroid_x, centroid_y, centroid_lat, centroid_lng,
                      floor_id, building_id, image_b64, descriptors_blob})
        │
        │ Outbox sync (sync_outbox table → PostGIS writer)
        ▼
   PostGIS: landmarks(id, name, space_id, centroid_x, centroid_y,
                     centroid_lat, centroid_lng, …)
```

The cross-fill in `_resolve_landmark_coords` lets the **registration
client send whichever pair it has more reliably**. Phones with a fresh
GPS fix attach `(lat, lng)` (which is what `CameraScreen.kt` on Android
and `CameraView.swift` on iOS now do) and let the backend derive the
local-frame pair. The mapmaker, which knows local coords directly from
canvas placement, attaches `(x, y)` and lets the backend derive the
global pair. Either path produces a landmark with both pairs populated.

### Recognition: ml-vision → location resolver → frame

The ml-vision serving process keeps an in-memory cache of landmarks for
each facility ([orb_matcher.py][orb]). The cache row carries the
landmark's `floor_id`, `floor_index`, and all four coordinates:

```python
@dataclass
class CachedLandmark:
    space_id: str
    floor_id: Optional[str]
    floor_index: Optional[int]
    centroid_x: Optional[float]
    centroid_y: Optional[float]
    centroid_lat: Optional[float]
    centroid_lng: Optional[float]
    keypoints: np.ndarray           # ORB descriptors
    image_thumb: bytes              # for debug overlays
```

When a frame comes in over WSS, `OrbMatcher.match_frame(frame_bytes)`
runs ORB feature extraction, matches against the cache via FLANN/BFMatcher,
keeps matches above a Lowe-ratio threshold, and if the inlier count
exceeds the homography threshold returns the best `CachedLandmark`. The
location resolver ([location_resolver.py][lr]) then constructs a
`LocationEntity` with kind="landmark" and copies `floor_id`, `floor_index`,
`centroid_x`, `centroid_y`, `centroid_lat`, `centroid_lng` onto it. The
WS server attaches that entity to the next frame the client gets as
`frame.location`.

[orb]: aau-sw8-ml-vision/serving/orb_matcher.py
[lr]: aau-sw8-ml-vision/serving/location_resolver.py

### Client-side snap: forced-location wiring

On iOS ([CameraViewModel.swift][cvm-ios]) the Combine pipeline subscribes
to `VisionStreamingService.$resolvedLocation` and on each new
`VisionResolvedLocation`:

```swift
container.forcedUserSpaceId = location.spaceId
container.forcedUserCoordinate = (lat, lng).map(CLLocationCoordinate2D.init)
mapNav.pendingFloorIndex = location.floorIndex
mapNav.pendingFloorId    = location.floorId        // primary key
mapNav.selectedTab       = .floorPlan              // pop the camera sheet
```

The `pendingFloorId` (added recently — older code only had `floorIndex`)
is matched by `loadFloorsAndOverlay` against `FloorSummary.id` directly,
sidestepping a long-standing bug where backend / iOS `floor_index`
conventions diverged and the user landed on the ground floor regardless
of the landmark's real floor.

On Android the equivalent wiring lives in `FloorPlanViewModel.forceUserSpace`,
which carries optional `latitude` / `longitude` on the call and snaps the
map dot to that point rather than the room centroid.

[cvm-ios]: aau-sw8-ios/ViewModels/CameraViewModel.swift

---

## Live navigation simulation

**Description.** Once the user taps "Start Navigation", the route turns
into a *guided* walk: the user's blue dot becomes a red dot that the app
moves along the polyline as the user actually walks, the polyline behind
the dot trims, and on arrival a banner reads "You have reached your
destination." The simulation runs whenever the GPS dot is unreliable
(typical inside a building) so the user gets visible movement even when
CoreLocation has frozen.

Two design questions had to be answered:

1. **What detects "the user is walking"?**
2. **At what pace should the dot advance?**

### Why not GPS, pedometer, or activity classification?

```
   GPS              fails indoors (the whole reason for sim)
   CMPedometer      step events batch over 1.5–3 s windows; the dot
                    stutters in 2-3 m hops every couple of seconds
                    instead of gliding
   CMMotionActivity walk/run/stationary; ~10 s of latency before it
                    flips from "stationary" to "walking", way too
                    slow for navigation feedback
   CMMotionManager  user-acceleration magnitude at 20–50 Hz, gravity
                    already removed by sensor fusion — about 50 ms
                    of latency
```

The implementation uses `CMMotionManager.startDeviceMotionUpdates` at
20 Hz, computes `|userAcceleration|` per sample (i.e. `sqrt(ax² + ay²
+ az²)`), and treats the user as "moving" whenever the magnitude
crosses **0.05 g** (≈ 0.5 m/s²) within the last **1.2 s**. The 0.05 g
threshold is well above the ~0.01 g noise floor of a still phone but
easily crossed by walking footfalls (peaks of 0.2–0.5 g per step are
typical). The 1.2 s decay window bridges the gap between footfalls
(an adult stride lasts ~0.5–0.8 s) so the dot doesn't stutter mid-stride.

The pace, once moving, is the **average walking speed of 1.4 m/s**, a
defensible global constant (about 5 km/h, the universal commute pace
in transit planning). Pedometer-derived custom pace would be more
accurate per-user, but the false-positive rate of CMPedometer is too
high (it counts arm swings, drives that pass over potholes, etc.) and
the average-speed defaults already feel right in user testing.

### Polyline trim during walking

On every 0.5 s tick of the simulation, after advancing
`simulatedArcLengthMeters += dt × 1.4`, the new dot position is computed
by interpolating along `routeCoordinates`. Then
`rebuildRouteCoordinates(currentRoute)` is called, which splits the
**full original** polyline at the dot's projection and writes:

```
   walkedCoordinates = pathCoords[0…bestSeg] + [bestProj]    // dimmed dashed
   routeCoordinates  = [bestProj] + pathCoords[bestSeg+1…]    // bright blue
```

Both lists are bound to the `MapViewWithOverlay`; the renderer paints
the walked list with a dashed gray (`lineDashPattern = [2, 8]`) and the
remaining list with the bright-blue main + dark-blue halo stack. The
simulated arc is reset to 0 on every rebuild because the route now
starts at the new dot position — `dt × 1.4` on the next tick is
measured from there, not the original start.

### Arrival banner

After the dot advances, if the haversine distance from the new sim
position to `routeCoordinates.last` is within 5 m, the route cancels
and a `arrivalBanner` SwiftUI state goes high for 4 s with the message
"You have reached your destination." The same banner also fires from
the GPS-driven `checkArrival()` path so a real-world arrival (with the
GPS dot snapping the user past the destination radius) gets the same
visual feedback.

---

## iOS client

**Description.** The primary mobile client: a native SwiftUI + Combine +
AVFoundation app. It's the reference implementation of every end-user flow
— browsing campuses, viewing floor plans, asking the assistant for
directions, live AR-style camera overlays during navigation, and uploading
room photos for the image pipeline to analyse. Architecture is
MVVM-with-DI: views observe view-models, view-models pull from services,
and a single `DIContainer` wires all of them together with Combine
publishers. There is no local database — state is fetched on demand from
the gateway and kept in-memory per view-model; `LocationManager` is the one
stateful singleton, managing CoreLocation and wifi-state transitions.

```
   ┌────────────────────────── Views (SwiftUI) ──────────────────────────┐
   │  ExploreView               list campuses / buildings                │
   │  FloorPlanView             pan/zoom 2D plan + route overlay         │
   │  CameraView                live AR detections via WSS                │
   │  AssistantView             chat UI, sources pill per reply          │
   │  RoomPhotoUploadView       4-photo capture (N/E/S/W)                │
   │  ProfileView               avatar, uploaded rooms, settings         │
   └───────────┬─────────────────────────────┬───────────────────────────┘
               │ @Published / @ObservedObject                            
               ▼                             ▼                           
   ┌─────────────────────────────────────────────────────────────────────┐
   │                  ViewModels (ObservableObject)                      │
   │    FloorPlanViewModel   spaces, routes, selected space              │
   │    AssistantViewModel   history, streaming reply, source refs       │
   │    CameraViewModel      WS connection state, last detections        │
   └───────────┬─────────────────────────────┬───────────────────────────┘
               │ Combine pipelines injected by                           
               ▼                             ▼                           
   ┌─────────────────────────────────────────────────────────────────────┐
   │                  DIContainer (Services/DIContainer.swift)           │
   │    wires LocationManager, AssistantService, FloorPlanService,       │
   │    NavigationService, RoomSummaryService, VisionStreamingService    │
   └───────────┬─────────────────────────────────────────────────────────┘
               │
               ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │                  Services (Services/*.swift)                        │
   │                                                                     │
   │   LocationManager                                                   │
   │     CLLocationManagerDelegate + NWPathMonitor                       │
   │     @Published lastLocation, horizontalAccuracyMeters, isOnWifi     │
   │     indoor profile (tighter accuracy) vs outdoor (kNearestTen)      │
   │     filters stale readings (> 10s) and high-uncertainty fixes       │
   │                                                                     │
   │   AssistantService                                                  │
   │     POST /api/v1/assistant/chat                                     │
   │     maps {user_query, campus_id} ⇄ ChatReply (reply, sources)       │
   │                                                                     │
   │   FloorPlanService                                                  │
   │     GET /api/v1/floors/{id}/display                                 │
   │     GET /api/v1/floors/{id}/connections                             │
   │     GET /api/v1/floors/{id}/map-overlay                             │
   │     decodes SpaceDisplayItem list for canvas rendering              │
   │                                                                     │
   │   NavigationService                                                 │
   │     GET /api/v1/navigate?from=&to=&accessible_only=                 │
   │                                  &avoid_stairs=&elevators_only=    │
   │     (avoid_stairs / elevators_only come straight from               │
   │      @AppStorage("nav.avoidStairs") / nav.elevatorsOnly             │
   │      on every call — no extra round-trip)                           │
   │     turn-by-turn Route built from consecutive Space centroids       │
   │                                                                     │
   │   RoomSummaryService                                                │
   │     POST /api/v1/room-summary/rooms/{room}/photos (multipart, ≤4)   │
   │     GET  /api/v1/room-summary/rooms  (list previously uploaded)     │
   │                                                                     │
   │   VisionStreamingService                                            │
   │     WSS /api/v1/ml-vision/ws/stream/{facility_id}                   │
   │     URLSessionWebSocketTask + JPEG frame encode pipeline,           │
   │     decodes StreamFrame { detections[], location }                  │
   │     throttles to ~N fps, drops frames under backpressure            │
   │                                                                     │
   │   Protocols.swift + DetectionBox.swift                              │
   │     shared DTOs so views never touch URLSession directly            │
   └─────────────────────────────────────────────────────────────────────┘
               │                                      │
               ▼                                      ▼
         HTTPS (REST, X-Api-Key)            WSS (binary JPEG frames,
         via NSURLSession configured              X-Api-Key on handshake,
         by DIContainer                           auto-reconnect)
```

**Map / route rendering stack.** `FloorPlanView` composes an
`MKMapView` through a `UIViewRepresentable` (`MapViewWithOverlay`) and
sits at full screen. On every `updateUIView(_:)` the view:

1. Refreshes `context.coordinator.parent = self` — the original bug that
   made the overlay flash on/off was a stale `parent` reference inside
   `MKMapViewDelegate`, which made the proximity check operate against
   an empty buildings list. Refreshing per update fixes it.
2. Maintains a **single persistent custom overlay** for room rendering:
   a `FloorRoomsOverlay` (whose `boundingMapRect = .world`) is added
   once and its companion `FloorRoomsRenderer` is mutated in-place via
   `setNeedsDisplay()` — *not* removed and re-added. This replaced an
   earlier per-room `MKPolygon` model that called
   `removeOverlays / addOverlays` on ~200 polygons every edit-gesture
   frame, which caused MapKit to render a partial / torn set ("sparse
   and detached" rooms). One CG draw pass per repaint, all 200 rooms
   in one path.
3. Inside the custom renderer's `draw(_:zoomScale:in:)`, each room's
   `polygonGlobal` is converted to a `CGPath`, filled, and stroked
   with a single dark grey outline (`UIColor(white: 0.25, alpha: 0.9)`
   at `2.0 / zoomScale` line width) so the room geometry reads
   clearly regardless of fill color. The palette is intentionally
   minimal: hallways light blue, restrooms orange, everything else a
   soft grey — three colors total, the dark outline does the work of
   separating adjacent rooms.
4. When `editState` is non-nil the renderer applies the rigid
   transform (`applyEditTransform`) to each vertex live, so a building
   or floor drag previews against the same projection math the
   backend uses on save (`apply_edit_transform` in
   `geometry_service.py` is the Python mirror of the same function).
5. Adds up to three `MKPolyline` overlays for the active route, in a
   Google-Maps-style layered stack:
   - `route-walked` — dashed gray-blue (`lineDashPattern = [2, 8]`,
     width 5) for the already-walked portion behind the dot; only
     populated when a navigation simulation is running.
   - `route-halo` — dark navy (rgb 0.05/0.30/0.66, width 11) drawn
     under the main stroke as an outline that reads against any
     basemap.
   - `route-main` — bright Material-blue (rgb 0.26/0.55/0.99, width 7)
     drawn on top.
   Polyline source is `[userLocation, ...stepCentroids]` with the start
   and end snapped to the room polygon edge (see *Polygon-edge snapping*
   above), rebuilt on every GPS / simulation tick so the line stays
   anchored to the dot.
6. Reuses a single `MKPointAnnotation` for the destination pin and a
   single `ForcedLocationAnnotation` for the snap dot — both are
   `weak`-referenced from the `Coordinator` and their `coordinate`
   property is mutated in place across ticks. The previous
   "remove + re-add on every update" pattern triggered MapKit's
   `viewFor(annotation:)` every step and was the dominant per-tick CPU
   cost during navigation.

**Diff-tracking `updateUIView`.** SwiftUI re-runs `updateUIView` on every
`@State` change in the view, not just the changes that affect the map.
Without guards every step tick re-painted every room polygon, removed
and re-added every polyline, and removed and re-added every annotation —
six dozen MapKit operations per second on a 200-room floor. The
`Coordinator` now caches signatures of the relevant state
(`(roomIDs hashed) → cachedRoomsSig`,
`(routeCount, firstPoint, lastPoint) → cachedRouteSig`, similar for
walked, edit-state, forced location) and skips the work whenever the
signature matches the previous frame. A `tickSimulation` that doesn't
change the route does ~3 hash compares and exits.

**Tap-to-select-room.** A `UITapGestureRecognizer` on the map view
(`cancelsTouchesInView=false` so MapKit's own gestures keep working)
converts a tap to an `MKMapPoint`, builds a `CGPath` per room in
`MKMapPoint` space, and uses `CGPath.contains` for point-in-polygon
hit testing. The first room that contains the tap is forwarded to
the parent via `onRoomTap(Room)`, which constructs a
`SpaceSuggestion` (id, name, building id, active floor id, centroid
lat/lon) and stores it in `pendingRouteSuggestion`. The
`BottomRouteCard` pops up with that room as the destination but does
*not* compute a route yet — the user taps the blue navigation arrow
to fire `resolveRoute(to:suggestion:)`. Same destination-card flow as
tapping a suggestion in the search bar list.

**Edit mode (building vs floor scope).** Owners and editors get an
"Edit" button that opens a confirmation dialog with two scopes:
**Edit building** moves / rotates / resizes the whole building
rigidly (every floor goes with it; per-floor georef overrides ride
along through `apply_edit_transform`); **Edit this floor** only
adjusts the active floor's georef, persisting `origin_lat / origin_lng
/ origin_bearing / scale_factor` on the `Floor` node. Pan / pinch /
rotate gesture recognizers on the map fire only while `editState`
is set; live preview transforms each room's `polygonGlobal` on the
fly. Save calls `PUT /buildings/{id}` or `PATCH /floors/{id}` and
re-fetches the floor geometry; cancel discards the deltas.

The on-screen UX layer above the map is a top safe-area inset hosting
`NextStepBanner` (next instruction + distance + "N steps left" + cancel
X). The banner advances by computing `nextStep()` — the first step the
user is more than 5 m away from — every time `userLocation` updates.

**Barometric floor detection.** Every iPhone since the 6 carries a
pressure sensor; Core Motion exposes it via `CMAltimeter`. The
`BarometerService` (`Services/BarometerService.swift`) wraps it as an
`ObservableObject` that turns relative-altitude updates into a floor
index against the active building's `floor.elevation_m` values:

```
   user_elevation = baseline_floor.elevation_m
                  + (relative_altitude_now − relative_altitude_at_baseline)

   floor_index    = argmin_f  | floors[f].elevation_m − user_elevation |
```

The "baseline" is a moment we know which floor the user is on. Three
events count as a baseline:

1. **Building entry.** When `loadFloorsAndOverlay` lands the floor
   list, we call `barometer.start(floors:baselineFloorIndex: 0)` —
   the user is assumed to enter at the ground floor unless something
   else corrects us later.
2. **Explicit floor tap.** `FloorPlanView.selectLabel(_:)` calls
   `barometer.recalibrate(toFloorIndex:)` — the user just told us
   where they are, which is an authoritative anchor.
3. **ML-Vision space match (planned).** When the camera sees a
   landmark with a known floor, the same recalibrate hook fires.

A 1.5 m hysteresis (half a typical floor) prevents the published
floor from flapping when the user lingers on a stair landing midway
between two slabs. Floor changes drive an `.onChange(of:
barometer.currentFloorIndex)` in `FloorPlanView` that auto-switches
`vm.selectedFloor` so the overlay follows the user up the stairs
without any tap. `barometer.stop()` is called on `onZoomOut` so the
altimeter isn't left subscribed when no building is active.

The barometer needs `NSMotionUsageDescription` (already required by
the `CMPedometer` cards); the service guards on its presence and
skips the call if missing, since a missing description string causes
iOS to terminate the process when the API is touched.

**Floor-elevation flow through the stack.**
- The mapmaker / import JSON authors `floor.elevation_m` per floor
  (metric height of the slab above the building's reference).
- `backend/repositories/campus_repo.py:list_floors` returns the field
  in `GET /buildings/{id}/floors`.
- iOS `FloorPlanService.FloorListItem.elevation_m` decodes it; the
  field is propagated onto `FloorSummary.elevationMeters`.
- `BarometerService` consumes the per-floor elevation list to map
  altitudes onto floor indices.

If the import is missing `elevation_m` for a floor (older imports),
that floor is skipped in the argmin and the inferred index simply
won't include it — the service degrades to "no auto-switch" rather
than crashing.

**Cross-tab navigation.** A small `MapNavigationCoordinator`
`ObservableObject` lives at `ContentView` scope and is injected via
`.environmentObject(...)`. It exposes four pending-target fields
besides `selectedTab`, each cleared on consumption:

- `pendingBuildingId` / `pendingBuildingName` / `pendingBuildingCoordinate` —
  set when the user taps a building's blue navigate button anywhere
  in Explore. `FloorPlanView` resolves the id against
  `floorService.buildings`, calls `mapProxy.flyTo(building.coordinate)`,
  sets `currentBuildingId`, and kicks off `loadFloorsAndOverlay`.
- `pendingFloorIndex` — set when the user taps a floor's blue navigate
  button in Explore's floor list. `loadFloorsAndOverlay` consumes it
  when the floor summaries arrive: if any summary's `floorIndex`
  matches, that floor is selected instead of the default ground
  floor.
- `pendingDestinationSpaceId / Name / Coordinate` — set when the user
  taps a room's blue navigate button in Explore's room list. After
  geometry loads, `consumePendingDestination()` builds a
  `SpaceSuggestion` (using the room's `centroidGlobal` or polygon
  centroid as a fallback for lat/lon) and stores it in
  `pendingRouteSuggestion`; the `BottomRouteCard` then pops up with
  that room as the destination, ready for the user to tap the blue
  arrow for directions.

The same `pendingRouteSuggestion` field is set by three other
entry points: tapping a search-bar suggestion, tapping a room
directly on the map overlay, and the Explore room-row path above. In
all four cases the user ends up in front of the same destination card
on the map.

This avoids a NotificationCenter / global-singleton pattern for what
is fundamentally just a handful of cross-tab signals.

**Profile screen.**
- **Persistent preferences.** `nav.avoidStairs`, `nav.voiceGuidance`,
  `nav.elevatorsOnly` are stored via `@AppStorage` rather than `@State`,
  so the toggles survive relaunches. **`avoidStairs` and `elevatorsOnly`
  are wired:** `FloorPlanView` reads the same `@AppStorage` keys and
  forwards them on every `NavigationService.computeRoute` call as
  `avoid_stairs` / `elevators_only` query parameters; the backend
  filters STAIRCASE / ESCALATOR (and additionally RAMP for elevators-
  only) out of the path. **`voiceGuidance` is cosmetic** — no
  `AVSpeechSynthesizer` is wired up yet, so the toggle persists but
  does not affect the app. **Dark Mode** (under Appearance) is wired
  via `ThemeSettings.isDarkMode` → `.preferredColorScheme(...)` at
  the app root.
- **Live pedometer stats.** `PedometerStats` (an `ObservableObject`)
  queries `CMPedometer.queryPedometerData(...)` for today's total and
  publishes `stepsText` / `distanceText`. Crucially, the call is gated
  by an `NSMotionUsageDescription` presence check on the bundle's
  Info.plist; without that key, iOS terminates the process the moment
  CMPedometer is queried (no exception, just SIGTERM), so we skip the
  query entirely if the description is missing.
- **Adaptive dark mode.** `Color.slate*` and `Color.blue*` are no longer
  static RGB tuples — they're `UIColor { trait in ... }` adaptive
  closures defined in `Utilities/DesignSystem.swift`. The slate scale is
  rung-flipped (`slate50` ↔ `slate900`, `slate100` ↔ `slate800`, ...)
  so backgrounds stay dark and text stays light without rewriting every
  call site. `Color.cardSurface` replaces hardcoded `Color.white` for
  card backgrounds. The `ThemeSettings.isDarkMode` toggle drives a
  `.preferredColorScheme(...)` modifier at the app root.

**Capabilities & permissions.** Camera (AVFoundation for preview + frame
capture), `NSLocationWhenInUseUsageDescription` (indoor/outdoor toggles),
`NSMotionUsageDescription` (Profile pedometer cards — without it the
app SIGTERMs on first profile open), Local Network, Bonjour for WS when
on campus wifi. No third-party SDKs.

**Why there's no offline cache yet.** Everything mobile-visible is authored
in the mapmaker → gateway → Neo4j; the app currently assumes an online
connection on campus wifi. A persistent cache is feasible (CoreData would
be the natural fit) but out of current scope.

---

## Android client

**Description.** A Jetpack Compose **prototype** that is intentionally
scoped much smaller than the iOS app. It exists to prove out on-device
sensing on Android (camera, wifi scan, tflite inference with MobileNet, AR
label-overlay UX) while reusing the same UI language as the iOS client.
It currently does **not** talk to the backend gateway — all data paths are
local — but the Compose screen structure is already shaped to match iOS's
flows so the networking layer can be slotted in without rewriting the UI.
Today it's most useful as a second-platform sanity check: can the same
real-time vision pipeline run against on-device models, and does wifi
scanning give enough signal to place a user on a floor without CoreLocation?

```
   ┌────────────────── Compose Screens (com.example.aauapp/ui) ─────────────┐
   │                                                                        │
   │  MainActivity ─► MainScreen (NavHost, bottom nav)                      │
   │    ├─ ExploreScreen           stub catalogue UI                        │
   │    ├─ FloorPlanScreen ◄──────── FloorPlanViewModel (in-mem state)      │
   │    ├─ CameraScreen             CameraX preview + overlay               │
   │    │    └─ analyser ─► ImageUtils ─► MobileNetClassifier (.tflite)     │
   │    ├─ AssistantScreen ◄──────── AssistantViewModel  (stub: returns     │
   │    │                             "currently unavailable")              │
   │    └─ ProfileScreen            local settings                          │
   │                                                                        │
   └────────────────────────────────┬───────────────────────────────────────┘
                                    │
                                    ▼
   ┌────────────────── Utilities / on-device sensors ───────────────────────┐
   │                                                                        │
   │  CameraUtils           CameraX preview binding, analyser pump          │
   │  ImageUtils            YUV→RGB convert, resize, normalise for tflite   │
   │  MobileNetClassifier   Interpreter on a cached tflite asset;           │
   │                        returns top-k labels per frame                  │
   │                                                                        │
   │  LocationDetector      MLKit image-label heuristics to guess what      │
   │                        kind of space the camera is pointed at          │
   │                                                                        │
   │  WifiScanner           WifiManager.scanResults (BSSID, RSSI, SSID),    │
   │                        intended as a fingerprinting signal to replace  │
   │                        what iOS gets from CoreLocation indoors         │
   └────────────────────────────────────────────────────────────────────────┘

   AndroidManifest permissions:
     CAMERA,  ACCESS_FINE_LOCATION, ACCESS_COARSE_LOCATION,
     ACCESS_WIFI_STATE, CHANGE_WIFI_STATE, INTERNET
```

**What's missing to reach feature parity with iOS.**

1. A networking layer (Retrofit / Ktor Client) pointed at the middleware
   gateway, with an `X-Api-Key` interceptor and 401/502 handling.
2. `AssistantViewModel` wired to `POST /api/v1/assistant/chat` (response
   shape is already identical to iOS's `ChatReply`).
3. `FloorPlanViewModel` wired to `GET /api/v1/floors/{id}/display` and a
   Compose Canvas renderer equivalent to iOS `FloorPlanView`.
4. OkHttp `WebSocket` client for `wss://.../api/v1/ml-vision/ws/stream/{id}`
   so CameraScreen can also stream frames to server-side YOLO26 (it
   currently only classifies locally with MobileNet).

None of these are structurally blocked — they're deferred because the iOS
client is the one that ships.

---

## Indoor positioning

**Description.** GPS dies indoors. Concrete attenuates the signal, walls
reflect it into multipath chaos, and Apple's `kCLLocationAccuracyBest`
typically reports 30–60 m horizontal accuracy inside a building — more
than enough to snap the user to the wrong room or even the wrong floor.
Indoor positioning replaces or augments GPS using signals that propagate
inside: WiFi RSSI from access points, WiFi RTT (802.11mc), and visual
landmarks seen by the camera. Ariadne's strategy is **hybrid and
platform-aware**: GPS handles outdoor → building selection, ML-Vision
landmark recognition provides per-room "you are here" snaps that work on
both clients, and Android additionally contributes a server-resolved
WiFi position (kNN fingerprint match + optional RTT trilateration) for
dead reckoning between snaps. iOS, blocked from raw WiFi RSSI by Apple,
falls back to GPS + barometer + camera snaps only.

**Status snapshot.** WiFi positioning is **live, not planned**: the
Android client ships a `WifiScanner` + `WifiRttRanger` + `PositioningManager`,
and the backend exposes four `/api/v1/positioning/*` endpoints backed by
`services/wifi_positioning.py`. The iOS path is read-only — it consumes
position fixes when ML-Vision produces a landmark snap but does not scan
WiFi itself (Apple denies the entitlement to third-party apps; see
*Why iOS doesn't scan WiFi* below).

### How WiFi positioning actually works

Every WiFi access point a device sees has a unique BSSID (the AP's MAC)
and broadcasts beacons whose received power is reported as RSSI
(in dBm; −30 = right next to it, −90 = barely audible). Ariadne uses
two complementary techniques — fingerprinting (always) and RTT
trilateration (when supported) — and fuses the results server-side.

**Technique 1: RSSI fingerprinting (the primary path).**
Skip the physics. At survey time a surveyor walks the floor and records,
at known points (room centroids), the visible APs and their RSSIs. The
result is a per-room signature `{BSSID → RSSI}` stored in `wifi_fingerprints`.
At runtime the phone produces its own live scan, the server computes a
fingerprint distance to every stored signature, and picks the closest.
**Accuracy:** 2–5 m once the survey is dense; brittle when IT moves /
replaces APs (the survey decays silently and is felt as drift).

The distance function ([wifi_positioning.py:13–31][wp]) is a sum of
squared RSSI differences with a fixed penalty for APs that appear in
one signature but not the other:

```
d² =   Σ      (rssi_live[B] − rssi_db[B])²
     B ∈ A∩B
     +   Σ    25²                 ← missing-AP penalty (in dBm²)
       B ∈ A△B
```

The 25 dBm penalty is one path-loss decade — i.e. an AP that's visible
in one scan but not the other is treated as if it were a strong AP that
fell off a cliff. This makes the metric robust to mild noise (an AP
flickering near the floor of detection) while still discriminating
between rooms with materially different AP visibility.

**Technique 2: kNN voting (the locator).**
([wifi_positioning.py:34–75][wp]) Once we have a fingerprint distance to
every candidate, take the k=3 nearest, weight them by inverse distance,
and let them vote on a `space_id`. The winning space's confidence is

```
conf = 0.5·dominance + 0.5·closeness
```

where `dominance` is the winning space's share of total weight (a 3-way
tie scores ~0.33; a unanimous match scores 1.0) and `closeness` is a
function of the absolute distance to the winner (sub-50 → 1.0, sub-150
→ 0.5, > 300 → 0.0). A confidence threshold (0.4 by default) decides
whether the API returns a space or a null result.

**Technique 3: WiFi RTT / 802.11mc Fine Time Measurement (FTM)**
(the precision booster). Modern APs and modern Android phones (≥ Android 9,
FTM-capable chipset) measure the signal's round-trip time directly,
sidestepping RSSI noise. **Accuracy:** 1–2 m. The Android client
([WifiRttRanger.kt:25–83][rtt]) checks `FEATURE_WIFI_RTT`, filters APs by
`is80211mcResponder`, caps to `RangingRequest.getMaxPeers()`, and returns
a `{BSSID → distance_mm}` map.

When the server receives a `/positioning/locate` request with both an
RSSI scan and an RTT distance map, it first runs the fingerprint kNN to
get a coarse `space_id`, then, if it has ≥ 3 anchor APs with known
`(x, y)` in PostGIS, runs least-squares multilateration
([wifi_positioning.py:78–118][wp]) to refine to a metric `(x, y)`. The
math is the standard linearization: write the distance equations

```
(x − x_i)² + (y − y_i)² = r_i²        for i = 1…n
```

and subtract the i=1 equation from each subsequent one to eliminate
`x² + y²`. The result is an overdetermined linear system in `(x, y)`
that `numpy.linalg.lstsq` solves in O(n) (this is **dormant** until APs
are surveyed; until then the path returns the kNN result only).

[wp]: aau-sw8-spatial-backend/backend/services/wifi_positioning.py
[rtt]: aau-sw8-android/app/src/main/java/com/example/aauapp/positioning/WifiRttRanger.kt

### Android scanner: throttle handling + rolling buffer

The OS throttles `WifiManager.startScan()` to 4 calls per 2-minute window
per foreground app on Android 9+; abusing it locks the API out for the
remainder of the window. [PositioningManager.kt:32–62][pm] therefore:

1. Calls `startScan()` and registers a `BroadcastReceiver` on
   `SCAN_RESULTS_AVAILABLE_ACTION`.
2. If `startScan()` returns `false` (throttled), reads the OS-cached
   `scanResults` immediately and exits — no point waiting for a broadcast
   that won't come.
3. Sets a 3-second timeout; if no broadcast arrives, falls back to the
   cached results.
4. Merges the latest 3 scans in a rolling buffer, keeping the **strongest**
   RSSI per BSSID. This smooths the body-blocking jitter you get when the
   user turns 180° between scans.

The result map is shipped to the backend's `/positioning/locate` with the
RTT distances (when available). No client-side trilateration — the math
lives on the server because the AP coordinates and fingerprint database
do.

[pm]: aau-sw8-android/app/src/main/java/com/example/aauapp/positioning/PositioningManager.kt

### Why iOS doesn't scan WiFi

Apple deliberately blocks third-party access to raw WiFi RSSI. The only
API that returns it is `NEHotspotHelper`, which requires the
`com.apple.developer.networking.HotspotHelper` entitlement — Apple grants
this to carriers and a tiny vetted partner list and rejects most other
applications. `CNCopyCurrentNetworkInfo` returns SSID/BSSID for the
*connected* AP only (no scan list), and even that needs Location
permission + an `NSLocationWhenInUseUsageDescription`. Apps that publish
private-SPI workarounds get rejected from the App Store.

So on iOS the platform's contribution to indoor positioning is:

- **GPS** (CoreLocation, `kCLLocationAccuracyBestForNavigation`) for the
  outdoor → building snap, with stricter accuracy / age filters when the
  device is on WiFi vs cellular (`maxAcceptableHorizontalAccuracyMeters`
  is 65 m off-WiFi but 25 m on-WiFi — being on WiFi is treated as a
  proxy signal for "indoors so trust GPS less").
- **Barometer** (CMAltimeter) for floor index, with 1.5 m hysteresis.
- **Camera landmark snap** via ML-Vision — when the ORB matcher hits a
  registered landmark, `VisionResolvedLocation` arrives with `spaceId`,
  `floorId`, `centroidLat/Lng`, and the iOS client snaps the dot
  (`container.forcedUserSpaceId` + `forcedUserCoordinate`).
- *Read-only consumer* of WiFi position fixes when ML-Vision attaches
  one to the location frame — the iOS client never sends a scan up.

### The hybrid strategy (per-platform)

Each signal works at a different spatial scale; the per-platform stacks
combine them differently because of what each OS exposes.

```
   ┌──────────────────────── Android ─────────────────────────┐    ┌─────────────────── iOS ────────────────────┐
   │                                                          │    │                                            │
   │  GPS (CoreLocation eqv)                                  │    │  GPS (CoreLocation)                        │
   │   σ ≈ 30 m outdoors, > 60 m inside concrete              │    │   σ ≈ 30 m outdoors, > 60 m inside         │
   │                       │                                  │    │                       │                    │
   │  Barometer (CMAltimeter eqv)                             │    │  Barometer (CMAltimeter)                   │
   │   σ_z ≈ 0.3 m, drift ~ 1 m/h                             │    │   σ_z ≈ 0.3 m, drift ~ 1 m/h               │
   │                       │                                  │    │                       │                    │
   │  WiFi scan (RSSI)  ─► /positioning/locate  ─► space_id   │    │  Camera landmark snap ─► forcedUserSpaceId │
   │   + optional RTT (mm-accurate ranges)                    │    │   + forcedUserCoordinate                   │
   │                       │                                  │    │   (1 m when seen)                          │
   │  Camera landmark snap ─► forcedUserSpaceId               │    │                                            │
   │                       │                                  │    │  No WiFi (Apple denies entitlement)        │
   │                       ▼                                  │    │                       ▼                    │
   │  Selected: latest snap wins for ≈ 5 min,                 │    │  Selected: latest snap wins for ≈ 5 min,   │
   │  barometer for floor index, GPS for outdoor              │    │  barometer for floor index, GPS for        │
   │  ────────────────────────────────────────                │    │  outdoor                                   │
   │                                                          │    │                                            │
   └──────────────────────────────────────────────────────────┘    └────────────────────────────────────────────┘
                                  │                                                       │
                                  └───────────────────────────────┬───────────────────────┘
                                                                  ▼
                                                  Server-side fixes shared via
                                                  the WS frame `location` field
                                                  (VisionStreamingService also
                                                  attaches the resolved location
                                                  to the next frame, so a snap
                                                  on Android lands on iOS too
                                                  if both devices stream into
                                                  the same facility — useful
                                                  for guided tours).
```

The "latest snap wins" rule is implemented as a 5-minute manual-pin window
(`manualFloorPinAt` on Android, `lastAnchoredForcedSpaceId` on iOS) so the
barometer can't override a fresh landmark scan or a user-tapped floor.

### Floor-detection vertical channel

The barometer is decoupled from the horizontal channel: it doesn't help
with `(lat, lng)` and the GPS / WiFi / camera sources don't help with
floor. On both platforms the barometer computes

```
user_elevation = baseline_floor.elevation_m
               + (relative_altitude_now − relative_altitude_at_baseline)

floor_index    = argmin_f  | floors[f].elevation_m − user_elevation |
```

with a 1.5 m hysteresis (half a typical floor) so the published index
doesn't flap on stair landings. Recalibration happens on three events:

1. **Building entry** — `barometer.start(floors:baselineFloorIndex: 0)`
   when `loadFloorsAndOverlay` returns; the user is assumed to enter on
   the ground floor unless something corrects us.
2. **Explicit floor tap** — `selectLabel(_:)` calls `recalibrate(toFloorIndex:)`.
3. **ML-Vision snap** — the same hook fires when a landmark with a
   known `floor_index` is recognised, so a landmark on floor 3 instantly
   re-baselines the barometer to floor 3.

### Server-side data model (implemented)

The PostGIS schema carries three positioning tables, all wired through
the backend's `/positioning/*` routes today:

```
   wifi_aps                   (RTT anchors; dormant until coords surveyed)
     id           text PK
     bssid        text NOT NULL                indexed
     building_id  text FK → buildings.id
     floor_id     text FK → floors.id
     x, y         double precision             AP coords in local floor frame
     ref_power    double precision             P₀ (reserved for path-loss fallback)
     last_seen    timestamptz

   wifi_fingerprints          (the primary kNN database)
     id             text PK
     building_id    text FK
     floor_id       text FK
     space_id       text FK → spaces.id        the room the surveyor was in
     readings       jsonb                      { "<BSSID>": <RSSI dBm>, ... }
     rtt_distances  jsonb                      { "<BSSID>": <mm> }   when avail
     captured_at    timestamptz
     captured_by    text                        user_id of the surveyor

   audit_positioning_events  (provenance of every server-side fix)
     id, space_id, building_id, source ("rssi"|"rtt"|"fused"),
     confidence, scan_size, ts, user_id
```

### Backend endpoints (implemented)

All four are live behind the gateway today; see
[routes/positioning.py][rp].

| Endpoint                             | Method | Behaviour |
| ------------------------------------ | ------ | --------- |
| `/api/v1/positioning/fingerprints`   | POST   | Surveyor uploads `{building_id, floor_id, space_id, scan, rtt?}` → upsert into `wifi_fingerprints`. Audited. |
| `/api/v1/positioning/fingerprints`   | GET    | Per-floor survey QA: counts of fingerprints per space (for the mapmaker to render coverage heat). |
| `/api/v1/positioning/locate`         | POST   | `{building_id, scan, rtt?}` → kNN against `wifi_fingerprints`; if RTT distances + ≥ 3 AP coords are present, runs the least-squares trilateration to refine `(x, y)`. Returns `{space_id, confidence, x?, y?, source}`. |
| `/api/v1/positioning/access-points`  | POST   | Editor-only. Upsert AP coords for RTT trilateration. Until populated, the locate endpoint returns the kNN result without metric refinement. |

[rp]: aau-sw8-spatial-backend/backend/routes/positioning.py

### Why the lookup lives server-side

The fingerprint database is per-venue and changes whenever the
building's WiFi infrastructure does; pushing the whole table to every
device would inflate the app and force a re-download on every IT change.
The kNN lookup is sub-100 ms server-side and keeps the AP map private
(BSSIDs of a corporate WiFi are mildly sensitive). It also means the
math can evolve without an app release — swapping the missing-AP
penalty, adding a building-wide AP frequency-band weight, or layering
RTT on top is a server-only change.

---

## Code

The earlier sections describe *what* runs *where*. This section is a
walkthrough of the most load-bearing pieces of code — the ones that are
either non-obvious, that took multiple iterations to get right, or that
are most likely to need editing as the platform evolves. Each
walkthrough quotes the actual file, explains the design choices, and
flags the failure modes the current code is guarding against.

### 1. Dijkstra edge weights — door-aware + 3D cross-floor

**File:** [`backend/services/python_dijkstra.py`][pd]

The `_edge_weight(src, dst)` function turns a graph edge into wall-clock
seconds for an average walker. It has three branches that look small
but each fixes a real failure mode:

```python
def _edge_weight(src: dict, dst: dict) -> float:
    sx, sy = src.get("centroid_x"), src.get("centroid_y")
    dx, dy = dst.get("centroid_x"), dst.get("centroid_y")
    sf, df = src.get("floor_index"), dst.get("floor_index")
    same_building = src.get("building_id") == dst.get("building_id")

    if sx is None or sy is None or dx is None or dy is None or not same_building:
        dist_m = 1.0                                          # branch 3: fallback
    elif sf == df:                                            # branch 1: same floor
        # Snap the non-connector endpoint onto its polygon edge nearest
        # the connector — fixes "the polyline cuts through walls".
        if _is_connector(dst) and not _is_connector(src):
            sx, sy = _room_exit_point(src, dx, dy)
        elif _is_connector(src) and not _is_connector(dst):
            dx, dy = _room_exit_point(dst, sx, sy)
        dist_m = math.hypot(sx - dx, sy - dy)
    else:                                                     # branch 2: cross-floor
        horizontal = math.hypot(sx - dx, sy - dy)
        vertical   = abs(sf - df) * _DEFAULT_FLOOR_HEIGHT_M   # 3.3 m
        dist_m     = math.hypot(horizontal, vertical)

    return (dist_m / _WALKING_SPEED_MS) + (dst.get("traversal_cost") or 0.0)
```

- **Branch 1 (same floor)** uses `_room_exit_point`, which calls
  `closest_point_on_polygon` (walked through next) to snap the non-
  connector room's centroid to its boundary nearest the door. Without
  this the polyline goes centroid-to-centroid, which on a 30 m corridor
  with the door at one end gives a 15 m weight when the user actually
  walks 5 m — and the visual line cuts diagonally through the wall.
- **Branch 2 (cross-floor)** computes a 3D Pythagorean distance so a
  3 m vertical hop between floors costs the right amount of seconds.
  The 3.3 m default is `_DEFAULT_FLOOR_HEIGHT_M`; it could come from
  `floor.elevation_m` once every floor has it filled in.
- **Branch 3 (fallback)** is the floor of 1 m. It exists so dijkstra
  terminates on incomplete imports (a room with no centroid) instead of
  raising on `NaN`.

The `traversal_cost` added at the end is the per-node penalty — 30 s
for an elevator wait, ~10 s for a flight of stairs. Without it a
1-floor staircase trip would tie with a 4 m corridor walk in the
weight graph, which doesn't match how long either actually takes.

[pd]: aau-sw8-spatial-backend/backend/services/python_dijkstra.py

---

### 2. Polygon edge projection — `closest_point_on_polygon`

**File:** [`backend/services/geometry_service.py`][gs]

The geometry helper that both the routing weight (branch 1 above) and
the polyline endpoint snap (`_endpoint_exit_point` in
[`navigation_service.py`][ns]) depend on. Given a polygon and a target
point, return the point *on the polygon boundary* nearest the target:

```python
def closest_point_on_polygon(polygon, tx, ty):
    best = None
    best_d2 = math.inf
    for i in range(len(polygon)):
        ax, ay = polygon[i]
        bx, by = polygon[(i + 1) % len(polygon)]
        dx, dy = bx - ax, by - ay
        seg_len2 = dx * dx + dy * dy
        if seg_len2 == 0:
            t = 0.0                              # degenerate edge
        else:
            t = ((tx - ax) * dx + (ty - ay) * dy) / seg_len2
            t = max(0.0, min(1.0, t))            # clamp to segment
        px, py = ax + t * dx, ay + t * dy
        d2 = (tx - px) ** 2 + (ty - py) ** 2
        if d2 < best_d2:
            best_d2 = d2
            best = (px, py)
    return best
```

The body is a classic point-to-segment projection: parametrise each
edge as `A + t·(B-A)`, solve `t = ((P-A)·(B-A)) / |B-A|²`, clamp `t` to
`[0,1]` so the projection lands on the segment (not its infinite
extension), pick the edge with the smallest squared distance. The
squared-distance trick saves a `sqrt` inside the loop.

Used in two places:

```
   In routing weights (branch 1)         In polyline build (endpoint snap)
   ─────────────────────────────         ─────────────────────────────────
   Pull the room toward its door         Pull the start / end of the visual
   so the EUCLIDEAN edge weight          polyline from the room's centroid
   reflects what a human walks.          to the room's boundary, so the line
                                         exits via the actual door.
```

[gs]: aau-sw8-spatial-backend/backend/services/geometry_service.py

---

### 3. Motion-driven dot simulation — `NavigationMotionDetector` + `tickSimulation`

**File:** [`aau-sw8-ios/Views/FloorPlanView.swift`][fpv]

The simulator that advances the red dot along the polyline while the
user walks indoors and CoreLocation is unreliable. Two pieces:

**Motion detection** uses `CMMotionManager.startDeviceMotionUpdates`
at 20 Hz, looks at `userAcceleration` (gravity already removed by
sensor fusion), and treats the user as "moving" while the magnitude
crossed 0.05 g within the last 1.2 seconds:

```swift
motion.deviceMotionUpdateInterval = 1.0 / 20.0
motion.startDeviceMotionUpdates(to: queue) { [weak self] data, _ in
    guard let self = self, let d = data else { return }
    let ax = d.userAcceleration.x
    let ay = d.userAcceleration.y
    let az = d.userAcceleration.z
    let mag = (ax * ax + ay * ay + az * az).squareRoot()
    if mag > self.accelerationThreshold {            // 0.05 g
        self.lastMotionAt = Date()
    }
}

var isMoving: Bool {
    guard let last = lastMotionAt else { return false }
    return Date().timeIntervalSince(last) <= motionTimeoutSeconds   // 1.2 s
}
```

Three numbers, each chosen for a reason:

| Constant                   | Value | Why                                                                                         |
| -------------------------- | ----- | ------------------------------------------------------------------------------------------- |
| `deviceMotionUpdateInterval` | 50 ms | Fast enough to react to footfalls (1–2 Hz); slow enough to not drain battery (20 Hz vs 100). |
| `accelerationThreshold`    | 0.05 g | Well above the ~0.01 g noise floor of a still phone; well below typical 0.2–0.5 g walking peaks. |
| `motionTimeoutSeconds`     | 1.2 s | Bridges the gap between footfalls (~0.5–0.8 s) so the dot doesn't stutter mid-stride.       |

**Tick** runs every 0.5 s. When `isMoving` is true and GPS is unreliable,
the dot advances by `dt × 1.4 m/s` along the polyline and the route is
re-split at the new dot position so the walked portion behind it trims:

```swift
private func tickSimulation() {
    guard isNavigating, userOnRouteFloor, routeCoordinates.count >= 2 else { return }
    let now = Date()
    let dt = lastSimulationTick.map { now.timeIntervalSince($0) } ?? 0.5
    lastSimulationTick = now

    let blueDotReliable = ...   // fresh, accurate, no forced indoor location
    if blueDotReliable {
        simulatedPosition = nil ; simulatedArcLengthMeters = 0 ; return
    }

    guard motionDetector.isMoving else { return }       // ← key gate

    simulatedArcLengthMeters += dt * averageWalkingSpeedMS
    guard let newSim = coordinate(atArcLength: simulatedArcLengthMeters,
                                  on: routeCoordinates) else { return }
    simulatedPosition = newSim
    simulatedArcLengthMeters = 0
    rebuildRouteCoordinates(navigationService.currentRoute)   // trim behind

    if let dest = routeCoordinates.last,
       haversineMeters(newSim, dest) <= arrivalRadiusMeters {
        announceArrival()
    }
}
```

The arc-length reset after `rebuildRouteCoordinates` is the subtle
bit: the rebuild re-splits the *full* original polyline around the new
sim position, so `routeCoordinates` now starts at the new sim. From
there, the next `dt × 1.4` is measured against the trimmed line, not
the original — that's why the arc is reset to 0 on every tick that
advanced.

[fpv]: aau-sw8-ios/Views/FloorPlanView.swift

---

### 4. Diff-tracking `MapViewWithOverlay.updateUIView`

**File:** same [`FloorPlanView.swift`][fpv].

SwiftUI invokes `updateUIView` on *every* `@State` change in the
containing view, not just the changes that affect the map. Without
guards every navigation tick repainted every room polygon, removed
and re-added every polyline, and removed and re-added every
annotation — about 60 MapKit operations per second on a 200-room
floor.

The Coordinator now caches signatures of the relevant state and skips
the work when nothing material changed:

```swift
let roomsSig  = roomsSignature(visibleRooms)
let editSig   = editSignature(editState)
if roomsSig != context.coordinator.cachedRoomsSig
        || editSig != context.coordinator.cachedEditSig {
    context.coordinator.roomsRenderer?.setNeedsDisplay()
    context.coordinator.cachedRoomsSig = roomsSig
    context.coordinator.cachedEditSig  = editSig
}

let routeSig  = polylineSignature(routeCoordinates)
let walkedSig = polylineSignature(walkedCoordinates)
if routeSig != context.coordinator.cachedRouteSig
        || walkedSig != context.coordinator.cachedWalkedSig {
    // remove & re-add the (max 3) MKPolyline overlays …
    context.coordinator.cachedRouteSig  = routeSig
    context.coordinator.cachedWalkedSig = walkedSig
}

if let forced = forcedLocation {
    if let existing = context.coordinator.forcedAnnotation {
        existing.coordinate = forced                  // mutate in place,
    } else {                                          // never re-add
        let dot = ForcedLocationAnnotation()
        dot.coordinate = forced
        uiView.addAnnotation(dot)
        context.coordinator.forcedAnnotation = dot
    }
}
```

Signatures are cheap — `roomsSignature` hashes `(count, each id)`,
`polylineSignature` hashes `(count, first point, last point)`. The
last point of the route doesn't change during a walk (only the first
does as the dot advances), so the polyline rebuild *does* fire each
tick — but the rooms overlay (the expensive one) only repaints when
floors actually change, and the dot annotation is mutated in place
instead of removed-and-re-added, which used to trigger
`mapView(_:viewFor:)` every step.

End result: a tick that doesn't change the rooms is ~3 hash compares,
a polyline swap, and a `.coordinate =` assignment. The previous code
was a full overlay repaint.

---

### 5. Assistant dispatch — the deterministic-first router

**File:** [`aau-sw8-spatial-backend/assistant/services/assistant_service.py`][as]

`AssistantService.chat()` is the hub that decides whether to answer
from the graph, from a vector match, or refuse. The ordering of the
guards is load-bearing — see the table earlier in this doc. The
relevant flow:

```python
async def chat(self, user_query, campus_id, building_id=None,
               user_lat=None, user_lon=None, floor_index=None):

    if (s := self._smalltalk_reply(user_query)) is not None:
        return {"answer": s, "sources": []}

    loc = self.repo.locate_user(campus_id, user_lat, user_lon) if user_lat else None
    effective_building_id = (loc or {}).get("building_id") or building_id

    if self._floor_count_intent(user_query):
        return self._answer_floor_count(campus_id, effective_building_id)
    if self._which_floor_intent(user_query):
        return self._answer_which_floor(loc, floor_index, ...)
    if self._about_building_intent(user_query) and effective_building_id:
        return self._answer_about_building(campus_id, effective_building_id)
    if (v := self._vertical_intent(user_query)) is not None:
        return self._answer_vertical(v, campus_id, effective_building_id)
    if (where := self._where_am_i_intent(user_query)) is not None:
        return self._answer_where_am_i(where, loc, campus_id)
    # … _distance_intent (no GPS), _navigation_intent + _floor_intent,
    # … _floor_intent alone (deterministic listing) …

    if self._find_place_intent(user_query):
        chosen = self._pick_find_place_candidate(similar_spaces, user_query)
        if chosen is not None:
            return {"answer": f"{chosen['name']} is on {floor} of {bldg}{near}.", ...}
        return {"answer": "I don't have that information.", "sources": []}

    # Generic catch-all: refuse, unless ASSISTANT_MODE=online has explicitly
    # opted into the LLM path with a capable hosted model.
    if (os.getenv("ASSISTANT_MODE") or "offline").lower() != "online":
        return REFUSAL_WITH_EXAMPLES
    return self._llm_summarise(system_prompt, similar_spaces, user_query)
```

The earlier table shows the same 11-step dispatch tabularly; the code
above is the live source. Two things worth highlighting:

- **`_about_building_intent` and `_vertical_intent` run *before*
  `_where_am_i_intent`** because the where-am-i regex used to match
  "in **this building**" inside *"is there an elevator in this
  building?"* and return the wrong refusal. Ordering catches this.
- **`effective_building_id` falls back from the GPS-derived building
  to the iOS-supplied `building_id`** so a user asking about "this
  building" with the floor-plan view open answers from their open
  building even when GPS is muddy.

[as]: aau-sw8-spatial-backend/assistant/services/assistant_service.py

---

### 6. Find-place hallucination guards

**File:** same [`assistant_service.py`][as].

The find-place gate is the most complex piece of the assistant because
embedding similarity alone fails in both directions: short opaque
queries ("GR5") embed too weakly to clear a strict threshold, and
"plausible nonsense" queries ("where is room 999?") embed *just*
strongly enough to a real room (Luca's Room at cosine 0.46) to look
acceptable. Neither cosine threshold works on its own.

The current implementation pairs a high-cosine fast path with a
token-match scan:

```python
_FIND_PLACE_HIGH_SCORE = 0.55
_FIND_PLACE_LOW_SCORE  = 0.15   # only used for logging now; scan ignores it

if self._find_place_intent(user_query):
    chosen = None
    if top_in and top_in["score"] >= _FIND_PLACE_HIGH_SCORE:
        chosen = top_in
    else:
        for cand in similar_spaces:                            # top-K = 30
            if _find_place_token_match(user_query, cand["name"], cand["type"]):
                chosen = cand
                break
    if chosen:
        return {"answer": f"{chosen['name']} is on {floor} of {bldg}{near}.", ...}
    return {"answer": "I don't have that information.", "sources": []}
```

The token-match function is the safety net:

```python
def _find_place_token_match(query, candidate_name, candidate_type) -> bool:
    haystack = f"{candidate_name or ''} {candidate_type or ''}".lower()
    tokens = re.findall(r"[a-z0-9][a-z0-9'_-]*", (query or "").lower())
    content = [t for t in tokens if t not in _FIND_PLACE_STOPWORDS and len(t) >= 2]
    if not content:
        return True                                    # all stopwords → defer to cosine
    for raw in content:
        for tok in {raw, _depluralise(raw)}:           # "wcs" -> {wcs, wc}
            if tok in haystack:
                return True
            for needle in _FIND_PLACE_SYNONYMS.get(tok, ()):
                if needle in haystack:                 # "toilet" -> "bathroom"
                    return True
    return False
```

Three behaviors fall out of this:

| Query                            | Top result (cosine)            | Path                                    | Answer                          |
| -------------------------------- | ------------------------------ | --------------------------------------- | ------------------------------- |
| *"Where is the bathroom?"*       | Bathroom (~0.65)               | HIGH-cosine fast path                  | Bathroom                        |
| *"Where is GR5?"*                | GR5 (~0.37)                    | scan → token "gr5" hits "GR5"          | GR5                             |
| *"Where are the WCs?"*           | East Corridor (~0.20), Bathroom rank ~12 (~0.15) | scan → token "wcs" depluralised to "wc", synonym "bathroom" hits "Bathroom" at rank 12 | Bathroom                        |
| *"Where is room 999?"*           | Luca's Room (~0.46)            | no candidate's name contains "999"     | refuse                          |
| *"Where is GR99?"*               | GR4 (~0.30)                    | "gr99" doesn't appear in any name      | refuse                          |
| *"Where can I find a coffee machine?"* | corridors (~0.10)         | no token match in any candidate        | refuse                          |

The top-K is bumped to **30** for find-place queries (vs 10 for other
RAG paths) specifically because the right answer for weak-embedding
queries like WCs sits well below rank 10. The synonym map is small
(~20 entries) and `_depluralise` is conservative (only trims when ≥ 3
chars remain after stripping) so we don't eat real tokens.

---

### 7. WiFi positioning — kNN fingerprint distance + RTT trilateration

**File:** [`backend/services/wifi_positioning.py`][wp]

The Android client uploads `{building_id, scan: [{bssid, rssi}], rtt?: {bssid: mm}}`.
The server resolves to a `space_id` (always) and optional metric
`(x, y)` (when ≥ 3 RTT-capable APs have surveyed coordinates).

**Fingerprint distance** is a sum of squared dBm differences with a
fixed penalty for APs visible in one signature but not the other:

```python
_MISSING_AP_PENALTY = 25.0          # ≈ one path-loss decade

def _fingerprint_distance(live: dict[str, float], stored: dict[str, float]) -> float:
    bssids = set(live) | set(stored)
    total = 0.0
    for b in bssids:
        if b in live and b in stored:
            total += (live[b] - stored[b]) ** 2
        else:
            total += _MISSING_AP_PENALTY ** 2
    return math.sqrt(total)
```

The 25 dBm penalty treats a missing AP as if a strong AP fell off a
cliff — robust to one AP flickering near the floor of detection, but
still discriminating between rooms whose AP visibility is materially
different.

**kNN voting** (k=3, inverse-distance weighted) picks the winning
space:

```python
def locate_by_rssi(scan, fingerprints, k=3):
    scored = sorted(
        ((_fingerprint_distance(scan, fp["readings"]), fp) for fp in fingerprints),
        key=lambda x: x[0],
    )[:k]
    votes = {}
    for d, fp in scored:
        w = 1.0 / (d + 1e-6)
        votes[fp["space_id"]] = votes.get(fp["space_id"], 0.0) + w

    winner_id, winner_w = max(votes.items(), key=lambda kv: kv[1])
    dominance = winner_w / sum(votes.values())                       # 0.33 ↔ 1.0
    closeness = max(0.0, 1.0 - scored[0][0] / 300.0)                 # ~50 → 1.0
    confidence = 0.5 * dominance + 0.5 * closeness
    return {"space_id": winner_id, "confidence": confidence, "source": "rssi"}
```

`confidence` is the gate the API uses to decide between returning a
space and returning a `null` result; 0.4 is the threshold by default.

**RTT trilateration** (when ≥ 3 known-coord APs returned a distance)
linearises the system of distance equations around the first anchor
and solves by least squares:

```python
def trilaterate_rtt(anchors: list[tuple[float, float, float]]):
    # anchors[i] = (x_i, y_i, r_i)   where r_i is mm → meters
    if len(anchors) < 3:
        return None
    x1, y1, r1 = anchors[0]
    A, b = [], []
    for (xi, yi, ri) in anchors[1:]:
        A.append([2 * (xi - x1), 2 * (yi - y1)])
        b.append((xi**2 - x1**2) + (yi**2 - y1**2) + (r1**2 - ri**2))
    xy, *_ = numpy.linalg.lstsq(numpy.asarray(A), numpy.asarray(b), rcond=None)
    return (float(xy[0]), float(xy[1]))
```

The closed-form derivation: from `(x − xᵢ)² + (y − yᵢ)² = rᵢ²` expand
each and subtract the i=1 equation to cancel `x² + y²`. What remains
is linear in `(x, y)`. With ≥ 3 anchors that's an overdetermined
system, and `lstsq` is the natural fit. The result is the metric
refinement of the kNN's `space_id` — useful for the dot's exact
position, not just "which room".

[wp]: aau-sw8-spatial-backend/backend/services/wifi_positioning.py

---

### 8. PostGIS pgvector + spatial filter — the single-SQL RAG retrieval

**File:** [`assistant/repositories/assistant_repo.py`][ar]

The reason retrieval lives on PostGIS (rather than Neo4j's vector
index) is that pgvector and PostGIS combine in a single SQL query:
"top-k semantically similar **and** within 200 m of the user." In
Neo4j that's two queries glued in Python.

```sql
SELECT bs.display_name, bs.space_type, bs.floor_index,
       f.display_name AS floor_name, b.name AS building_name,
       1.0 - (bs.embedding <=> CAST(:q AS vector)) AS score,
       COALESCE((
         -- 1-and-2-hop neighbours (real spaces, hopping through doors),
         -- so the assistant can say "near North Corridor" without naming
         -- door nodes
         SELECT json_agg(DISTINCT label) FROM (
           SELECT nb.display_name AS label
           FROM space_connections sc
           JOIN building_spaces nb ON nb.id = sc.to_space_id
           WHERE sc.from_space_id = bs.id
             AND nb.space_type NOT IN ('DOOR_STANDARD','PASSAGE',…)
           UNION
           SELECT nb2.display_name AS label
           FROM space_connections sc1
           JOIN building_spaces mid ON mid.id = sc1.to_space_id
           JOIN space_connections sc2 ON sc2.from_space_id = mid.id
           JOIN building_spaces nb2 ON nb2.id = sc2.to_space_id
           WHERE sc1.from_space_id = bs.id
             AND mid.space_type IN ('DOOR_STANDARD','PASSAGE',…)
             AND nb2.space_type NOT IN ('DOOR_STANDARD','PASSAGE',…)
         ) neigh
       ), '[]'::json) AS connected_to
FROM building_spaces bs
LEFT JOIN floors    f ON f.floor_id = bs.floor_id AND f.building_id = bs.building_id
LEFT JOIN buildings b ON b.id = bs.building_id
WHERE bs.campus_id = :campus_id
  AND bs.is_navigable = TRUE
  AND bs.embedding IS NOT NULL
  AND (:building_id IS NULL OR bs.building_id = :building_id)
  AND (:radius_m IS NULL OR ST_DWithin(
        bs.geometry_global::geography,
        ST_SetSRID(ST_MakePoint(:user_lng, :user_lat), 4326)::geography,
        :radius_m))
ORDER BY bs.embedding <=> CAST(:q AS vector)
LIMIT :limit;
```

Three pieces of leverage in one statement:

1. **`<=>` cosine distance** on the `embedding vector(384)` column — the
   `ivfflat` (or `hnsw`) index makes this sub-millisecond at our scale.
2. **`ST_DWithin` spatial filter** on the `geometry_global` (a PostGIS
   geography column populated by the import) so the LIMIT k results
   are the k nearest-by-meaning *and* near-by-distance.
3. **The recursive neighbour CTE** hops one or two `space_connections`
   edges, *skipping connector types* (`DOOR_STANDARD`, `PASSAGE`,
   `STAIRCASE`, etc.) so the `connected_to` list is real-room names
   the assistant can say in a sentence. Without the connector hop the
   neighbours would be "DOOR_STANDARD, PASSAGE, DOOR_STANDARD" — not
   useful for "near X".

This is the only place in the platform where vector similarity and
metric distance combine — every other consumer of either path uses
just one of the two.

[ar]: aau-sw8-spatial-backend/assistant/repositories/assistant_repo.py

---

### 9. Landmark coordinate cross-fill — `_resolve_landmark_coords`

**File:** [`backend/routes/landmarks.py`][lr]

A landmark may arrive with `(lat, lng)` (when the registering phone
has a GPS fix), with local `(x, y)` (when the mapmaker dropped a pin
on the canvas), with both, or neither. The endpoint accepts every
combination and back-fills the missing pair from the floor's georef:

```python
def _resolve_landmark_coords(db, space, x, y, lat, lng):
    georef = _space_georef(db, space)         # floor.origin_{lat,lng,bearing,scale}
    if georef is None:
        return x, y, lat, lng                 # nothing to project from
    if x is not None and y is not None and (lat is None or lng is None):
        lat, lng = local_to_global_coordinates(x, y, **georef)
    elif lat is not None and lng is not None and (x is None or y is None):
        x, y    = global_to_local_coordinates(lat, lng, **georef)
    return x, y, lat, lng
```

`local_to_global_coordinates` is the same trig — rotate by
`origin_bearing`, scale, translate by `(origin_lat, origin_lng)` —
that the mapmaker uses to draw a building on the world map. Storing
*both* pairs costs 16 bytes per row but saves a re-projection on
every read from the indoor canvas and from the iOS map. The
ml-vision serving cache (`CachedLandmark`) carries both pairs along
with `floor_index`, so when the camera recognises a landmark the WS
frame the iOS client receives can set both:

```swift
container.forcedUserSpaceId    = location.spaceId
container.forcedUserCoordinate = CLLocationCoordinate2D(lat, lng)   // for the map
mapNav.pendingFloorId          = location.floorId                   // exact floor
mapNav.pendingFloorIndex       = location.floorIndex                // fallback
```

The `pendingFloorId` is the recent change that lets the iOS map land
on the right floor even when the backend's `floor_index` numbering
doesn't match the iOS client's — `floorId` is a stable UUID, immune
to convention mismatch.

[lr]: aau-sw8-spatial-backend/backend/routes/landmarks.py

---

### 10. Landmark recognition — ORB + BFMatcher + Lowe + homography

**File:** [`aau-sw8-ml-vision/serving/orb_matcher.py`][om]

Recognising a registered landmark in a live camera frame is a 4-stage
pipeline. Each stage has a knob that's tunable via environment
variable, so the pipeline can be tightened or loosened per deployment
without a code change.

**Stage 1 — feature extraction (ORB).** ORB (Oriented FAST + Rotated
BRIEF) is the open-source, patent-free alternative to SIFT/SURF. It
extracts up to N keypoints per image and a binary descriptor per
keypoint (32 bytes), which makes it fast to extract *and* fast to
match.

```python
_ORB_FEATURES = _env_int("ORB_FEATURES", 1000)
...
orb = cv2.ORB_create(nfeatures=_ORB_FEATURES)
frame_keypoints, frame_descriptors = orb.detectAndCompute(frame, None)
```

The reference image of every registered landmark is run through the
same `ORB_create(nfeatures=1000)` at cache-warm time, so the
matcher only compares descriptor sets at request time — no re-running
ORB on the references per frame.

**Stage 2 — descriptor matching (BFMatcher with Hamming distance).**
Brute-force matcher because ORB descriptors are binary (256-bit
strings), and `NORM_HAMMING` is the right distance for those.
`crossCheck=False` because we want k-NN (top-2) matches so we can run
the Lowe ratio test next.

```python
bf = cv2.BFMatcher(cv2.NORM_HAMMING, crossCheck=False)
pairs = bf.knnMatch(frame_descriptors, entry.descriptors, k=2)
```

For each frame descriptor, BFMatcher returns its 2 nearest
descriptors in the reference set (by Hamming distance).

**Stage 3 — Lowe's ratio test (cull ambiguous matches).** Among the
two nearest, keep only the match where the nearest is significantly
closer than the second-nearest. This throws out repetitive textures
(carpets, brick walls) that would otherwise produce hundreds of
"good enough" matches with no discriminative power.

```python
_RATIO_TEST = _env_float("ORB_RATIO_TEST", 0.75)
...
good = []
for pair in pairs:
    if len(pair) < 2: continue
    m, n = pair
    if m.distance < _RATIO_TEST * n.distance:    # the Lowe ratio test
        good.append(m)
```

A ratio of **0.75** is David Lowe's original SIFT recommendation and
holds up well for ORB too. Lower → stricter (fewer good matches,
higher precision); higher → looser (more good matches, more false
positives).

**Stage 4 — geometric verification (RANSAC homography).** Even after
the Lowe ratio test, you can get spurious matches when the frame
*happens* to share a few descriptors with a landmark. The fix is
geometric: a real landmark match should have a consistent
perspective transform between the reference image and the frame.
RANSAC finds the homography that explains the most matches and
returns the *inlier* count.

```python
_MIN_INLIERS = _env_int("ORB_MIN_INLIERS", 12)
_RANSAC_REPROJ_PX = _env_float("ORB_RANSAC_REPROJ_PX", 5.0)
...
_h, mask = cv2.findHomography(src, dst, cv2.RANSAC, _RANSAC_REPROJ_PX)
inliers = int(mask.sum()) if mask is not None else 0
```

The 5-pixel reprojection threshold tolerates phone camera shake and
slight perspective error; the 12-inlier floor means a match needs at
least 12 geometrically consistent point pairs before it's accepted.
Combined with the Lowe ratio test, this almost completely eliminates
the "random poster wall matches the registered landmark" failure
mode.

**Pipeline summary** with the gate at each stage:

```
   frame JPEG
        │
        ▼   ORB.detectAndCompute    (~ 1000 keypoints)
   descriptors_frame  (Nx32 binary)
        │
        ▼   bf.knnMatch(k=2)        for each landmark in the facility cache
   pair lists
        │
        ▼   Lowe ratio test (0.75)   → "good matches"
   good_matches
        │
        ▼   threshold ≥ 8           else: reject (not enough discriminative matches)
        │
        ▼   findHomography(RANSAC, 5px)
   inliers
        │
        ▼   threshold ≥ 12          else: reject (geometric inconsistency)
        │
        ▼
   pick the landmark with the highest good-match count
        │
        ▼   attach (space_id, floor_id, floor_index, centroid_lat/lng/x/y)
        │
        ▼   send as `frame.location` on the WS reply
```

The matcher is invoked once per WS frame. Match latency on a 1080p
JPEG with 50 registered landmarks is roughly **30–60 ms on CPU,
< 10 ms on GPU** when OpenCV is built with CUDA. The expensive part
is ORB extraction on the frame; the descriptor matching is sub-ms per
landmark.

[om]: aau-sw8-ml-vision/serving/orb_matcher.py

---

### 11. Polyline drawing — three-layer Google-Maps-style stack

**File:** [`aau-sw8-ios/Views/FloorPlanView.swift`][fpv]

The active route is drawn as up to three overlaid `MKPolyline`
overlays, and the rendering logic dispatches them by `title`:

```swift
func mapView(_ mapView: MKMapView,
             rendererFor overlay: MKOverlay) -> MKOverlayRenderer {
    if let polyline = overlay as? MKPolyline {
        let r = MKPolylineRenderer(polyline: polyline)
        if polyline.title == "route-walked" {
            r.strokeColor = UIColor(red: 0.55, green: 0.58, blue: 0.62, alpha: 0.7)
            r.lineWidth = 5.0
            r.lineDashPattern = [2, 8]               // tight dots
        } else if polyline.title == "route-halo" {
            r.strokeColor = UIColor(red: 0.05, green: 0.30, blue: 0.66, alpha: 1.0)
            r.lineWidth = 11.0                       // dark navy outline
        } else {                                     // "route-main"
            r.strokeColor = UIColor(red: 0.26, green: 0.55, blue: 0.99, alpha: 1.0)
            r.lineWidth = 7.0                        // bright Material-blue
        }
        r.lineJoin = .round
        r.lineCap  = .round
        return r
    }
    // ... FloorRoomsOverlay branch elsewhere ...
}
```

Three layers, layered in `updateUIView` insertion order (halo first,
main second, walked already in place from a prior tick):

```
   ┌────────────────────────────────────── route layers ──────────────────────────────────────┐
   │                                                                                          │
   │   walked  : gray-blue 70%, width 5, dash [2,8]   "where you've been" (sim ticks fill it) │
   │   halo    : dark navy   100%, width 11           outer outline – reads on any basemap   │
   │   main    : bright blue 100%, width 7            inner fill – the active route          │
   │                                                                                          │
   │   Rendered z-order on the map:                                                           │
   │                                                                                          │
   │       ════════════════════════════════════════════ ← halo (width 11, dark)              │
   │      ====================================         ← main (width 7,  bright)             │
   │   · · · · · · · ●                                  ← walked (dashed, behind dot)         │
   │                 │                                                                        │
   │                 forced-location dot (red circle with white stroke)                       │
   │                                                                                          │
   └──────────────────────────────────────────────────────────────────────────────────────────┘
```

Two design choices that make this look right on every basemap:

1. **The halo is dark, not white.** The previous code used a 95%-opaque
   white halo, which disappears against light maps (overcast / satellite
   tiles). Navy-on-blue gives the polyline a strong outline regardless
   of basemap brightness.
2. **The walked portion is dashed, not solid.** Apple Maps uses dotted
   for the "covered" portion of a route and that visual idiom is
   already well-known. The walked line uses `lineDashPattern = [2, 8]`
   — 2 pixels on, 8 pixels off — which reads as a tight trail of dots.

The construction (where the polylines come from) is in
`rebuildRouteCoordinates`:

```swift
let pathCoords = sliceStepsForFloor(r.steps, displayedFloor: target)
let routeStartFloor = r.steps.first(where: { $0.floorIndex != nil })?.floorIndex
let displayedIsRouteStart: Bool = {
    guard isMultiFloor else { return true }
    return target == routeStartFloor
}()

let anchor = liveDotLocation ?? userLocation
if let anchor, userOnThisFloor, displayedIsRouteStart {
    let (walked, remaining) = splitRoute(pathCoords, anchor: anchor)
    walkedCoordinates = walked
    routeCoordinates  = [anchor] + remaining            // start at the dot
} else {
    walkedCoordinates = []
    routeCoordinates  = pathCoords                      // start at the first step
}
```

The `displayedIsRouteStart` guard is the recent fix for cross-floor
preview: when the user manually taps a destination-floor selector,
the polyline must start at the floor slice's first coord (the stairs
exit on that floor), not at the user's actual Floor-1 GPS dot. Without
this, a manual floor tap recalibrates the barometer to the displayed
floor, `userOnThisFloor` flips true, and the polyline gets prepended
with the user's wrong-floor anchor — drawing a 50-metre diagonal
through the floor.

---

### 12. WebSocket frame streaming — binary-up, JSON-down

**File:** [`aau-sw8-ml-vision/serving/main.py`][mlv]

The realtime ML-Vision endpoint is a WebSocket per device, binary
upstream (JPEG frames), JSON downstream (detection + location result
per frame). The server runs the model + ORB matcher on a thread-pool
executor so the event loop stays responsive to many concurrent devices.

```python
@app.websocket("/ws/stream/{facility_id}")
async def stream_vision(websocket: WebSocket, facility_id: str):
    await websocket.accept()
    # Refresh ORB cache when a new device connects — picks up any
    # landmark registered since the last warm-up without a server restart.
    _resolver.orb_matcher.invalidate(facility_id)
    loop = asyncio.get_event_loop()
    try:
        while True:
            image_bytes = await websocket.receive_bytes()                       # binary up
            if not image_bytes: continue
            frame_result = await loop.run_in_executor(                          # offload
                _stream_executor, _process_stream_frame, facility_id, image_bytes
            )
            await websocket.send_text(json.dumps(frame_result))                 # JSON down
    except WebSocketDisconnect:
        pass
```

The downstream frame shape:

```json
{
  "detections": [
    { "label": "person", "confidence": 0.87,
      "x": 0.12, "y": 0.34, "width": 0.21, "height": 0.55,
      "is_landmark_match": false }
  ],
  "location": {
    "kind": "landmark",        "id": "lm-2-0-048-poster",
    "name": "Joker poster",    "confidence": 0.84,
    "space_id": "space-2-0-048", "floor_id": "floor-cassiopeia-2",
    "floor_index": 2,
    "centroid_lat": 57.0144, "centroid_lng": 9.9870,
    "centroid_x": 12.4,      "centroid_y": 8.2
  }
}
```

Two design choices behind this protocol:

1. **Per-frame request/response, not server push.** The client sends a
   frame; the server replies with the result for that exact frame.
   That makes backpressure natural — if the model is slow, the client's
   `receive` blocks, so the client throttles itself naturally instead
   of the server having to drop frames. The Android/iOS clients also
   rate-limit at ~15 fps on their side, so the server isn't asked to
   keep up with a 60 fps preview stream.
2. **The executor offload (`run_in_executor`)** is what makes a single
   uvicorn worker handle many devices. The YOLO model + ORB matcher
   are blocking CPU/GPU work; running them inline on the event loop
   would serialise *all* device streams onto a single CPU. Routing
   each frame to a thread pool lets N devices share K threads (K
   default = the executor size, which scales with CPU count).

The "binary up, text down" split is also pragmatic — binary frames are
smaller (no base64), text replies are easier to debug than CBOR / MsgPack
and the response is small (< 1 KB).

[mlv]: aau-sw8-ml-vision/serving/main.py

---

### 13. Building edit transform — the rigid 4-DOF that lives in two languages

**Files:** [`backend/services/geometry_service.py`][gs] (Python),
[`aau-sw8-ios/Views/FloorPlanView.swift`][fpv] (Swift).

Both the mapmaker (web canvas) and the iOS client (MapKit) can move /
rotate / resize a building or a floor live, with all rooms following
rigidly. The transform is 4 degrees of freedom — `(deltaLat, deltaLng,
deltaBearing, scaleMultiplier)` — and the *same math* runs in two
languages so the preview matches what the backend persists on save:

**Python (server-side, on save):**

```python
def apply_edit_transform(lat, lng,
                         pivot_lat, pivot_lng,
                         delta_lat, delta_lng,
                         delta_bearing_deg, scale_mult):
    m_per_deg_lat = 111_000.0
    m_per_deg_lng = 111_000.0 * math.cos(math.radians(pivot_lat))
    dx = (lng - pivot_lng) * m_per_deg_lng       # meters east of pivot
    dy = (lat - pivot_lat) * m_per_deg_lat       # meters north of pivot
    b  = math.radians(delta_bearing_deg)
    rx = dx * math.cos(b) - dy * math.sin(b)     # rotate around pivot
    ry = dx * math.sin(b) + dy * math.cos(b)
    sx = rx * scale_mult                         # scale
    sy = ry * scale_mult
    new_lng = pivot_lng + delta_lng + sx / m_per_deg_lng
    new_lat = pivot_lat + delta_lat + sy / m_per_deg_lat
    return new_lat, new_lng
```

**Swift (client-side, live preview):**

```swift
static func applyEditTransform(_ coord: CLLocationCoordinate2D,
                               _ e: BuildingEditState) -> CLLocationCoordinate2D {
    let mPerDegLat = 111_000.0
    let mPerDegLng = 111_000.0 * cos(e.originLat * .pi / 180)
    let dxM = (coord.longitude - e.originLng) * mPerDegLng
    let dyM = (coord.latitude  - e.originLat) * mPerDegLat
    let b   = e.deltaBearing * .pi / 180
    let rx  = dxM * cos(b) - dyM * sin(b)
    let ry  = dxM * sin(b) + dyM * cos(b)
    let sx  = rx * e.scaleMultiplier
    let sy  = ry * e.scaleMultiplier
    return CLLocationCoordinate2D(
        latitude:  e.originLat + e.deltaLat + sy / mPerDegLat,
        longitude: e.originLng + e.deltaLng + sx / mPerDegLng
    )
}
```

The two functions are deliberately mirror-image so a future
re-implementation can be diffed line-for-line against the other. The
sequence — translate-to-pivot-origin → rotate → scale → translate-back
— is the standard 2D rigid+uniform-scale transform, applied to *every
vertex* of *every polygon* of *every room*. The math runs in local
flat-earth meters around the pivot (the building's `origin_lat,
origin_lng`) because at building scale the curvature of the earth is
< 1 mm and not worth dragging in a full geodesic library.

**Edit-scope choice.** The UI prompts for "Edit building" vs "Edit this
floor". The transform is the same either way; only what it's applied
to changes:

- **Edit building** — the deltas accumulate into the *building*'s
  `(origin_lat, origin_lng, origin_bearing, scale_factor)` on save.
  Every floor of the building inherits the new origin, so they all
  follow rigidly.
- **Edit floor** — the deltas accumulate into the *floor*'s own
  georef (the per-floor overrides for `origin_lat/lng/bearing/scale`).
  Only the active floor moves; the rest of the building stays put.

This is why every `Floor` node carries optional georef fields that
override the parent `Building`'s — the per-floor edit case needs
somewhere to store the override, and we want the override resolution
("use floor's if present, else fall back to building's") to be a
trivial null-check rather than a database join.

---

### 14. Floor-elevation chain — `floor.elevation_m` → barometer → cross-floor cost

The `elevation_m` field on each `Floor` node ties together three
otherwise-independent subsystems:

```
   CAD import
      writes Floor.elevation_m (meters above the building's reference)
        │
        ├──► backend.routing
        │     python_dijkstra._edge_weight branch 2 uses elevation
        │     when both endpoints have it; falls back to
        │     _DEFAULT_FLOOR_HEIGHT_M = 3.3 m otherwise.
        │
        ├──► iOS BarometerService
        │     start(floors:) caches Floor.elevation_m for every floor;
        │     converts CMAltimeter relativeAltitude → user_elevation,
        │     picks  floor_index = argmin_f  | floors[f].elevation_m − user_z |
        │     with 1.5 m hysteresis.
        │
        └──► Android equivalent on the prototype client (same formula).
```

This is one of the few "system invariants" worth flagging because it
crosses subsystem boundaries: the *same* field is the source of truth
for routing cost *and* for indoor floor detection. If a building is
imported with `elevation_m` left null on its floors, both subsystems
silently degrade — routing falls back to the 3.3 m default (works,
just less accurate); barometer floor detection can't run at all (the
service refuses to start without floors having elevations) and the
client falls back to manual floor selection.

The simplest sanity check during import is "every floor has a non-null
`elevation_m`". The CAD importer enforces this on the way in; the
mapmaker prompts when an operator authors a floor manually without one.

[gs]: aau-sw8-spatial-backend/backend/services/geometry_service.py

---

## API reference (through gateway)

### Graph (backend)

- `GET / POST / DELETE /api/v1/organizations[/{id}]` — org tenancy CRUD
- `GET / POST / DELETE /api/v1/campuses[/{id}]` — campus CRUD
- `GET /api/v1/campuses/{id}/export` — full nested campus dump for mapmaker
- `POST /api/v1/campuses/{campus_id}/import` — bulk import (JSON `MapImportSchema`)
- `POST /api/v1/campuses/import-dxf` — bulk import (multipart upload of `.dxf` /
  `.dwg`; the form fields supply `campus_id`, `campus_name`, `building_id`,
  `building_name`, `floor_id`, `floor_index`, optional `organization_id`,
  optional `origin_lat` / `origin_lng` (range-checked), optional
  `layer_mapping` (JSON object overriding the default layer→SpaceType
  heuristic), optional `dry_run` (when true, returns the parsed schema
  preview + `classification_summary` + `warnings` without writing to either
  database). Server-side parser turns the CAD into a `MapImportSchema` and
  runs the same import pipeline (Neo4j write + outbox-mirrored PostGIS).
  `.dwg` uploads use
  ODAFileConverter when installed and fall back to LibreDWG's `dwgread`
  otherwise; both are opt-in build args (`INSTALL_ODA=1`,
  `INSTALL_LIBREDWG=1`). If neither is on `PATH` the route returns
  HTTP 415 with install instructions plus the local-conversion
  workaround.)
- `GET / POST / DELETE /api/v1/buildings[/{id}]` — building CRUD (delete cascades)
- `GET /api/v1/buildings/{id}/floors` — list floors under a building
- `GET / POST / DELETE /api/v1/floors[/{id}]` — floor CRUD (delete cascades)
- `GET /api/v1/floors/{id}/display` — spaces with polygons (PostGIS-first)
- `GET /api/v1/floors/{id}/connections` — door graph for a floor
- `GET /api/v1/floors/{id}/geometry` — rooms with metadata for rendering
- `GET /api/v1/floors/{id}/map-overlay` — bounds/scale/origin for MapKit
- `GET / POST / DELETE /api/v1/spaces[/{id}]` — space CRUD (delete cleans
  connected doors in both stores)
- `POST /api/v1/connections` — add a door (4 Neo4j edges + 4 PostGIS rows)
- `DELETE /api/v1/connections/{from}/{to}` — remove a door (both stores)
- `GET /api/v1/navigate?from=&to=&accessible_only=&avoid_stairs=&elevators_only=` —
  Dijkstra on GDS projection when no filter is active; native Cypher
  `shortestPath` with per-node filters (is_accessible + excluded
  space types) when any of `accessible_only`, `avoid_stairs`, or
  `elevators_only` is `true`. `avoid_stairs` excludes
  STAIRCASE/ESCALATOR; `elevators_only` excludes STAIRCASE/ESCALATOR/RAMP.
  Endpoints bypass the type filter so STAIRCASE/ELEVATOR destinations
  still resolve.
- `POST /api/v1/navigate/refresh-graph` — rebuild GDS projection manually
- `GET /api/v1/search/spaces?q=&limit=` — substring search over Space
  `display_name`, `short_name`, and `tags_text` (case-insensitive
  `CONTAINS`). Replaced the earlier Lucene fulltext-index path
  because token matching couldn't surface `"A201"` when the user typed
  `"2"`. Results ranked: names that *start with* the query first,
  then shorter names, then alphabetical.
- `GET /api/v1/search/campuses/{campus_id}/spaces?q=` — same matcher
  scoped to one campus.
- `GET /api/v1/search/nearest-space?lat=&lon=&limit=` — Neo4j
  `point.distance` over Space centroids; used by mobile to resolve a
  `from_space_id` when the user is already inside a building.

### Assistant

- `POST /api/v1/assistant/chat` — `{user_query, campus_id}` → `{reply, sources}`
- `POST /api/v1/embed` — embed text with the shared MiniLM L6 v2 model

### Image pipeline

- `GET  /api/v1/room-summary/rooms`
- `POST /api/v1/room-summary` — raw images in, summary out
- `POST /api/v1/room-summary/by-room` — attach summary to an existing room
- `POST /api/v1/room-summary/rooms/{room}/photos` — mobile, ≤4 images
- `GET  /api/v1/room-summary/similarity` — k-NN by room_id
- `GET  /api/v1/room-summary/similarity/nearest` — k-NN by space_id
- `POST /api/v1/room-summary/similarity/ad-hoc` — k-NN by uploaded image

### ML-Vision

- `GET /api/v1/ml-vision/health`
- `GET /api/v1/ml-vision/v1/models/{facility_id}` — list model artifacts
- `GET /api/v1/ml-vision/v1/models/{facility_id}/{platform}` —
  download server (ONNX) / ios (mlpackage, zipped) / android (tflite) /
  metadata
- `POST /api/v1/ml-vision/v1/detect` — single-frame detection + location
- `WSS /api/v1/ml-vision/ws/stream/{facility_id}` — live stream

### Mobile convenience (middleware-owned)

- `GET /api/v1/mobile/campuses` — trimmed `{id, name}` list
- `GET /api/v1/mobile/campuses/{id}/map` — full campus export
- `GET /api/v1/mobile/campuses/{id}/map/light` — full export without SVG blobs

### Auth (backend, mounted only when `AUTH_JWT_SECRET` is set)

- `POST /api/v1/auth/signup` — create user + (optional) viewer membership
- `POST /api/v1/auth/guest` — anonymous JWT (`role=guest`, public-only via RLS)
- `POST /api/v1/auth/login` — email/password → access JWT or MFA challenge
- `POST /api/v1/auth/login/mfa` — exchange challenge + code for access JWT
- `POST /api/v1/auth/mfa/setup` — TOTP enrolment (returns secret + recovery codes)
- `POST /api/v1/auth/mfa/confirm` — flip `mfa_enabled=true` after a TOTP code
- `POST /api/v1/auth/mfa/email/setup` — emails an OTP, returns setup challenge
- `POST /api/v1/auth/mfa/email/confirm` — flip `mfa_enabled=true` (email method)
- `POST /api/v1/auth/mfa/disable` — re-prompts password, wipes MFA state
- `POST /api/v1/auth/password/change` — logged-in change (re-verifies old)
- `POST /api/v1/auth/password/forgot` — emails a 6-digit OTP (always 202)
- `POST /api/v1/auth/password/reset` — redeems the OTP and rotates the hash
- `GET  /api/v1/auth/me` — principal + org membership + MFA state

---

## Communication protocols

| Link                         | Protocol                                       |
| ---------------------------- | ---------------------------------------------- |
| Backend ↔ Neo4j              | Bolt (`bolt://` / `neo4j+s://`)                |
| Backend ↔ PostGIS            | SQLAlchemy + GeoAlchemy2 over `postgresql://`  |
| Backend ↔ Email service      | HTTP over Docker net, `X-Internal-Token`       |
| Email service ↔ SMTP relay   | SMTP/STARTTLS (port 587 by default)            |
| iOS ↔ gateway (REST)         | HTTPS, `X-Api-Key` + `Authorization: Bearer`   |
| iOS ↔ gateway (streaming)    | WSS, `X-Api-Key` on handshake                  |
| Middleware ↔ upstream (REST) | HTTP over Docker network, identity headers     |
| Middleware ↔ ml-vision (WS)  | WS bridge via `websockets`                     |

---

## Email service

**Description.** A standalone FastAPI container ([`email-service/main.py`](aau-sw8-spatial-backend/email-service/main.py))
that owns every outbound mail Ariadne sends — password-reset OTPs and
email-based MFA codes. It is deliberately separated from the backend so SMTP
credentials and outbound traffic stay off the main image, and so it can be
swapped for a SaaS provider (SES, SendGrid) without touching the auth code.
It is **never exposed publicly**: only the backend talks to it, over the
private Docker network, with a shared secret in `X-Internal-Token`. Without
SMTP env vars it runs in dry-run mode and logs payloads to stdout, so a
fresh `docker compose up` works without configuring real mail.

```
   ┌──────────────────────────┐                ┌──────────────────────────┐
   │   backend (:8000)        │                │   email service (:8003)  │
   │   services/auth_service  │                │   FastAPI + smtplib      │
   │                          │                │                          │
   │  setup_mfa_email()       │                │   POST /send             │
   │     ▼                    │  HTTP, internal │     • _check_token       │
   │  _send_mfa_email_otp()   │  Docker net     │       (constant-time     │
   │     ▼                    │  X-Internal-    │        compare; refuses  │
   │  email_client.send_email │  Token          │        to run when       │
   │     ▼                    │ ─────────────► │        token is unset)   │
   │  POST {to, subject,      │                 │     • build EmailMessage │
   │        text, html?}      │ ◄───────────── │     • SMTP/STARTTLS send │
   │                          │  {success,     │       OR dry-run + log   │
   │  login()  with method=   │   dry_run}     │                          │
   │  "email"  → identical     │                 │   GET /health            │
   │  send path               │                 │     {status, dry_run}    │
   │                          │                 │                          │
   │  request_password_reset()│                 │   No public port.        │
   │     ▼                    │                 │   `expose: 8003` only —  │
   │  send_email(reset OTP)   │                 │   middleware does NOT    │
   └────────┬─────────────────┘                 │   proxy to it.           │
            │                                   └────────┬─────────────────┘
            │                                            │
            │                                            ▼
            │                                   ┌──────────────────────────┐
            │                                   │  External SMTP relay     │
            │                                   │  (SMTP_HOST:SMTP_PORT)   │
            │                                   │  STARTTLS by default     │
            │                                   │  Optional auth via       │
            │                                   │  SMTP_USERNAME/PASSWORD  │
            │                                   └──────────────────────────┘
            │
            ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │  Postgres (audit_log, password_reset_tokens, app_users.mfa_*)        │
   │   • password_reset_tokens stores ONLY the bcrypt hash of the OTP     │
   │   • email-MFA stores no per-user secret — the OTP hash is sealed     │
   │     into the short-lived JWT challenge instead (stateless)           │
   │   • every send is recorded in audit_log with subject + reason        │
   └──────────────────────────────────────────────────────────────────────┘
```

**Stateless email-MFA.** Unlike TOTP (per-user secret stored on the user
row), email-MFA generates a fresh 6-digit OTP per challenge, bcrypt-hashes
it, and seals the hash into the `mfa-challenge` JWT (`otp_hash` claim). The
JWT signature binds the hash to the user; verification at
`/auth/login/mfa` only needs to bcrypt-compare the submitted code against
the hash in the challenge. No DB row to expire, no GC job, no session
table — and the backend is free to scale horizontally.

**Configuration.** Both `EMAIL_SERVICE_URL` and `INTERNAL_EMAIL_TOKEN` must
be set on the backend; without either, `email_client.is_configured()`
returns false and the auth flows that need email fail closed (HTTP 503,
"Could not send MFA email"). The email container fails closed on its side
too: a missing `INTERNAL_EMAIL_TOKEN` returns 503 from `/send` to prevent
an accidentally-deployed open relay. SMTP is opt-in: `SMTP_HOST` unset →
dry-run for local development.

---

## Security measures

The platform is built defence-in-depth: each layer assumes the previous
one might be bypassed. Every measure below is enforced in code today, not
aspirational.

### 1. Edge / gateway

```
                                public internet
                                      │  HTTPS  +  WSS
                                      ▼
   ┌────────────────────────────────────────────────────────────────┐
   │  Middleware :8080  — only port exposed by the compose stack    │
   │                                                                │
   │  HTTP middleware (every request, /health excepted):            │
   │    constant-time X-Api-Key check  ─────►  401 on mismatch      │
   │    STRIP caller-supplied identity headers                      │
   │       (x-user-id, x-org-id, x-org-ids, x-user-role)            │
   │    decode Bearer JWT, RE-STAMP same headers from claims        │
   │                                                                │
   │  WS handshake (HTTP middleware does NOT run on it):            │
   │    inline X-Api-Key check  ──►  close(4401) on missing/wrong   │
   └──────────────────────────┬─────────────────────────────────────┘
                              │  Docker network — never the host
   ═══════════════════ trust boundary ═══════════════════════════════
                              ▼
        backend · assistant · image_pipeline · ml_vision · email
              (all ClusterIP / expose: only — no public port)
```

- **Single public port.** Only the middleware (`:8080`) is published; every
  other container uses `expose:` so it is reachable on the Docker network
  but never from the host or the internet. The email service in particular
  is unreachable from outside the compose network.
- **`X-Api-Key` on every request.** The gateway enforces it on every HTTP
  path (`/health` is the only exception) and on every WebSocket handshake
  (the gateway's HTTP middleware does not run on WS, so the auth check is
  inline in the WS handler — closes with code `4401` on failure).
- **Identity-header anti-spoofing.** Before a request is forwarded
  upstream the gateway strips any caller-supplied `X-User-Id`,
  `X-Org-Id`, `X-User-Role` headers and only re-stamps them from a
  successfully-verified JWT. A client can not forge org membership by
  setting headers — it has to forge a JWT, which requires the HS256 secret.
- **Upstream timeouts** keep slow upstreams from tying up gateway sockets
  (15s for SMTP, 5s for health probes, 900s for big imports).

### 2. Authentication

**Bearer JWT primer.** Two ideas combined: a *JSON Web Token* (RFC 7519,
the credential format) carried under the *Bearer* HTTP scheme (RFC 6750,
the transport convention). A JWT is three Base64URL-encoded segments
joined by dots:

```
   eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9 . eyJzdWIiOiJ1c2VyXzQyIiwib3JnX2lkIjoiYWNtZSIsImV4cCI6MTcxMjM0NTY3OH0 . K8FZ7q...sig
   └────── header ──────────────────────┘ └────────────────── payload (claims) ─────────────────────────────────┘ └─── HMAC ──┘

   header  = {"alg":"HS256","typ":"JWT"}
   payload = {"sub":"user_42","org_id":"acme","org_ids":["acme","aau"],
              "role":"viewer","iss":"ariadne-backend","exp":1712345678,
              "typ":"access"}
   sig     = HMAC-SHA256(header.payload, AUTH_JWT_SECRET)
```

The signature proves *integrity* (the claims came from a holder of
`AUTH_JWT_SECRET`), not *confidentiality* — anyone can Base64-decode
the middle segment and read the claims, so secrets never go in there.
"Bearer" means: whoever presents this token is treated as authenticated
— no second factor, no proof of possession, the token itself is the
credential. Clients attach it on every request after login:

```
   Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOi...
```

Why Ariadne uses this shape:

- The JWT *is* the identity. The gateway verifies the signature once per
  request and reads the claims; no per-request database lookup of
  "which user is this token?" is needed because the claims are
  cryptographically vouched for.
- Bearer is supported by every HTTP client without extra libraries
  (`URLRequest.attachBearer()` on iOS, plain header string on the
  mapmaker's `fetch`).
- The trade-off is theft sensitivity: a stolen token grants the
  thief the bearer's privileges until `exp`. Mitigated below
  (transport TLS, 12 h TTL, `is_active` per-request check).

```
   ┌────── /auth/login ─────────────────────────────────────────────┐
   │  email + password                                              │
   │     ▼                                                          │
   │  rate-limit check (per-process sliding window — section 4)     │
   │     ▼                                                          │
   │  bcrypt.checkpw   ─►  Postgres: app_users.password_hash        │
   │     ▼                                                          │
   │  issue_token():                                                │
   │    iss = backend, exp = +12h, typ = "access"                   │
   │    sub = user.id                                               │
   │    org_id  = active org    (scopes writes — section 6)         │
   │    org_ids = ALL memberships (scopes reads — section 6)        │
   └───────────────────────┬────────────────────────────────────────┘
                           │  access JWT
                           ▼
       client carries  Authorization: Bearer <jwt>  on every
       subsequent request, plus the static X-Api-Key

   ┌────── every protected request ─────────────────────────────────┐
   │  Gateway:                                                      │
   │    decode_token(typ = "access")                                │
   │      iss + exp + signature + typ all checked                   │
   │      → 401 on any mismatch                                     │
   │    strip + re-stamp x-user-id / x-org-id / x-org-ids /         │
   │                     x-user-role                                │
   │  Backend:                                                      │
   │    get_principal() reads the trusted headers into a Principal  │
   │    is_active check  ─►  401 on revoked / disabled user         │
   │  → route handler runs with the Principal                       │
   └────────────────────────────────────────────────────────────────┘

   Revocation: set app_users.is_active = false → every outstanding
   token immediately invalidates (the is_active check happens per
   request, not per token issue).
```

- **Bearer JWTs (HS256)** signed with `AUTH_JWT_SECRET`.
  - Boot-time guard: backend refuses to start if the secret is shorter
    than 32 bytes (see [`backend/main.py:31-43`](aau-sw8-spatial-backend/backend/main.py#L31-L43)).
  - Required claims `sub` + `exp`; `iss` is verified.
  - Distinct `typ` claims (`access` vs `mfa-challenge`) — the wrong-type
    token is rejected at `decode_token`, so a challenge can not be replayed
    against `/auth/me`.
  - 12h default access TTL; 5-minute MFA challenge TTL.
  - **`org_id` (active org) and `org_ids` (full membership list).**
    Both claims ride on the access token. `org_id` scopes write
    operations (RLS, `require_org_match`, audit-log subject); `org_ids`
    is the full `[m.organization_id for m in memberships]` list and is
    consumed by *read* endpoints that union "your orgs + public" content
    (`/campuses/visible`, `/buildings/visible`). Without `org_ids` a user
    in two orgs would only ever see one of them. The middleware decodes
    the claim and stamps `x-org-ids: a,b,c` (CSV) onto the upstream
    request; `core/auth_principal.py:get_principal()` parses the header
    back into a tuple on the `Principal` model.
- **Token revocation.** Setting `app_users.is_active = false` invalidates
  every outstanding token for that user — `get_principal` checks the flag
  on every call and the gateway re-resolves it for every request.
- **Bcrypt password hashing** with configurable rounds (default 12).
  Re-verifies the current password on `password/change` and `mfa/disable`
  so a stolen access token alone can not lock a user out.

### 3. MFA + email OTPs

```
   ─── Enrolment ───────────────────────────────────────────────────

   ┌── TOTP ─────────────────────────┐  ┌── Email OTP ──────────────┐
   │  /auth/mfa/setup                │  │  /auth/mfa/email/setup    │
   │    pyotp.random_base32()        │  │    secrets.randbelow(1e6) │
   │    store secret on app_users    │  │    bcrypt-hash the OTP    │
   │    return secret + QR + 10      │  │    seal hash into JWT     │
   │    recovery codes (bcrypted)    │  │    challenge (otp_hash    │
   │  /auth/mfa/confirm              │  │    claim, signed)         │
   │    verify TOTP code             │  │    send OTP via email-svc │
   │    flip mfa_enabled = true      │  │    return setup challenge │
   └─────────────────────────────────┘  │  /auth/mfa/email/confirm  │
                                        │    verify OTP, flip       │
                                        │    mfa_enabled = true     │
                                        └───────────────────────────┘

   ─── Login ────────────────────────────────────────────────────────

   POST /auth/login (email + password)
        │
        ▼  user.mfa_enabled ?
        │
        ├── no  → issue access JWT (typ=access) — section 2
        │
        └── yes → issue mfa-challenge JWT (typ=mfa-challenge, +5 min)
                   ▼
              POST /auth/login/mfa { challenge_token, code }
                   ▼  decode_token(typ = "mfa-challenge")
                   │
                   ├── TOTP:  pyotp.TOTP(secret).verify(
                   │              code, valid_window=1)
                   │
                   └── Email: bcrypt.checkpw(code, challenge.otp_hash)
                              (hash lives ONLY in the JWT — no DB row,
                               no expiry job, no orphan session table)
                   ▼
              issue access JWT  ──►  same lifecycle as section 2

   Recovery codes: 10 per user, each bcrypt-hashed; verify walks ALL
                   10 hashes (no early break) so the response time
                   doesn't leak which slot held the match.
```

- **TOTP** (RFC 6238) via `pyotp` with `valid_window=1`. `pyotp.TOTP.verify`
  is constant-time internally; we additionally sanitise the input
  (strip spaces, reject non-digit, fix length) before passing it.
- **Email OTP** generated with `secrets.randbelow(1_000_000)` (uniform,
  unpredictable). Bcrypt-hashed before storage / JWT sealing — the
  plaintext is never persisted server-side.
- **Recovery codes.** 10 per user, one-time, bcrypt-hashed at rest.
  Verification walks **all** stored hashes (no early break) so the response
  time does not leak which slot held the match.
- **Stateless email-MFA challenges.** OTP hash is sealed into the
  `mfa-challenge` JWT (signed) instead of a DB row — no replay window
  beyond the JWT `exp`, no orphan rows.
- **Internal-token auth on the email service.** `X-Internal-Token` is
  matched constant-time (`hmac.compare_digest`); fail-closed when unset to
  prevent an accidentally-deployed open relay.

### 4. Brute-force / enumeration defence

```
   POST /auth/login (email, password)
        │
        ▼
   ┌────────── rate-limit check ──────────┐
   │ key    = email                       │
   │ window = 15 min                      │
   │ max    = 10 attempts                 │
   │ store  = per-process sliding window  │
   └────────────────┬─────────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼ over limit            ▼ under limit
   ┌──────────┐         ┌────────────────────┐
   │ 429 Too  │         │ bcrypt.checkpw     │  ← expensive (~80 ms)
   │   Many   │         │ (only runs AFTER   │    so attackers can't
   │ Requests │         │  the cheap check)  │    burn CPU on bad guesses
   └──────────┘         └─────────┬──────────┘
                                  │
                  ┌───────────────┴───────────────┐
                  ▼ any failure                   ▼ ok (+ MFA if enabled)
            ┌─────────────────────┐         ┌────────────────────┐
            │  401 "Invalid       │         │  see section 2 / 3 │
            │   credentials"      │         │                    │
            │  (generic — same    │         └────────────────────┘
            │   for bad pw, bad   │
            │   email, wrong org, │
            │   bad MFA code)     │
            └─────────────────────┘

   /auth/password/forgot uses the same shape: always returns 202
   regardless of whether the email exists; the attempt is recorded
   in audit_log either way so an attacker can't enumerate accounts
   from response codes or timing.
```

- **Per-process sliding-window rate limit** on `/auth/login` (default 10
  attempts per email per 15 min, configurable). Applied **before** bcrypt
  so credential-stuffing can not burn server CPU.
- **Generic "Invalid credentials" error** on every login failure
  (bad password, missing user, wrong-org, bad MFA code) — the API does
  not leak whether an email exists.
- **`/auth/password/forgot` always returns 202** regardless of whether
  the email exists, with the audit log recording the attempt either way.
- **Constant-time walks** on recovery-code verification and password-reset
  token lookup so attackers can not time-side-channel which row matched.

### 5. Database / SQL

```
   Backend opens a Postgres session per request, then BEFORE any
   user-data query, stamps the connection's transaction-local GUCs:

     SET LOCAL app.org_id     = :org_id      (parametrised)
     SET LOCAL app.is_service = 'true'       (only for service-mode calls)

   Every org-scoped table (campuses, buildings, floors, imports) has
   one RLS policy — org_isolation USING (...) WITH CHECK (strict):

          ┌──────── incoming SELECT / INSERT / UPDATE ─────────┐
          ▼                          ▼                         ▼
   ┌────────────────┐  ┌───────────────────┐  ┌────────────────────────┐
   │  service lane  │  │   tenant lane     │  │      public lane       │
   │                │  │                   │  │                        │
   │ app.is_service │  │ row.org_id =      │  │ row.is_public = true   │
   │  = 'true'      │  │  app.org_id       │  │ (read-only — guests,   │
   │                │  │                   │  │  unscoped sessions)    │
   │ backend's own  │  │ tenant user's     │  │                        │
   │ internal call  │  │ regular query     │  │  WITH CHECK rejects    │
   │ (sync, jobs)   │  │                   │  │  writes from this lane │
   └───────┬────────┘  └────────┬──────────┘  └────────────┬───────────┘
           │                    │                          │
           └────────────────────┴────────────┬─────────────┘
                                             ▼
                                       row admitted

   SET LOCAL scopes to the transaction; values can never leak between
   requests, even when the underlying connection is reused from the
   pool. AUTH_RLS_ENABLED=false bypasses every policy (dev convenience).
```

- **SQLAlchemy ORM** for every read and write — queries are parameterised;
  there is no f-string SQL anywhere in the request path. The only `text(...)`
  calls take their interpolated values from a hard-coded constant tuple
  (`_RLS_TABLES`, `_ADDED_COLUMNS`) or use named bind parameters
  (`{"org_id": ...}`).
- **Postgres Row-Level Security.** When `AUTH_RLS_ENABLED=true`, every
  org-scoped table (`campuses`, `buildings`, `floors`, `imports`) has an
  `org_isolation` policy admitting rows where:
  - `app.is_service` GUC is `true` (service-mode lane: backend internal calls), OR
  - `organization_id` matches `app.org_id` GUC (tenant lane), OR
  - `is_public = true` for read-only tables (guest / public lane).
  `WITH CHECK` stays strict — guests and unscoped sessions can read public
  rows but can never write them.
- **Per-request GUC stamping.** The backend opens its session and runs
  `SET LOCAL app.org_id = :org_id` (parametrised) plus
  `SET LOCAL app.is_service = 'true'` for internal calls. `SET LOCAL`
  scopes to the transaction — leaking these between requests is impossible.
- **Joined-table inheritance with manual cascade.** PostGIS FKs do not use
  `ON DELETE CASCADE`; the backend does explicit top-down deletes so
  cross-store consistency with Neo4j is preserved.

### 6. Multi-tenancy

```
   Postgres                Backend                Gateway              Client
   ────────                ───────                ───────              ──────
   organization_members
   (user_id, org_id, role)
       │
       │ at login time:
       │ memberships = SELECT WHERE user_id = u
       │ org_id   = chosen / first
       │ org_ids  = [m.org_id for m in memberships]
       ▼
   issue_token()
   JWT claims  ─►  { sub, exp, typ = "access",
                     org_id:  "acme",            (active — scopes writes)
                     org_ids: ["acme", "aau"]    (all — scopes reads)
                   }
                                  │
                                  ▼  on every later request
   ┌─────── Gateway ────────┐
   │  decode JWT            │
   │  stamp upstream headers:
   │    x-org-id:    acme              ← single value
   │    x-org-ids:   acme,aau          ← CSV
   │    x-user-role: viewer
   └──────────┬─────────────┘
              ▼
   ┌─────── Backend ────────────────────────────────────────────────┐
   │  get_principal() parses headers back into Principal:           │
   │     principal.org_id   = "acme"        (active)                │
   │     principal.org_ids  = ("acme", "aau")                       │
   │                                                                │
   │  WRITE routes call require_org_match(principal, target_org):   │
   │     allowed only if principal.org_id == target_org             │
   │     → 403 otherwise (writes scope to ONE active tenant)        │
   │                                                                │
   │  READ routes pass org_ids straight into Cypher:                │
   │     /campuses/visible, /buildings/visible:                     │
   │       WHERE c.org_id IN $org_ids                               │
   │          OR coalesce(c.is_public, false) = true                │
   │     (a user in multiple orgs sees all their content + public)  │
   └────────────────────────────────────────────────────────────────┘

   Guest token: sub = "guest_<uuid>",  org_id = null,  org_ids = [],
                role = "guest"  →  only is_public rows, never writes.
```

- **Org-scoped JWT claim.** Every access token carries `org_id`;
  `get_principal` re-checks the user is still a member of that org on
  every `/auth/me` (failing closed with 403 on revoked memberships, not
  silently stripping the claim).
- **Org membership on signup.** `signup` admits a user to a single org
  with role `VIEWER`; promotion to higher roles is operator-only (no
  self-serve).
- **Multi-org visibility (`org_ids`).** A user may belong to several
  organisations simultaneously (a researcher with cross-faculty
  affiliation, a service account with lab + campus roles). `org_id`
  in the JWT is the *active* one; `org_ids` is the union. Visibility
  queries take `org_ids` and run `c.organization_id IN $org_ids OR
  coalesce(c.is_public, false) = true` so users see every org they
  belong to plus any public campus / building. The same `org_ids` is
  read off `Principal` in `routes/campuses.py:list_visible_campuses` and
  `routes/buildings.py:list_visible_buildings` and forwarded as the
  Cypher `$org_ids` parameter in `CampusRepository.list_visible_*`.
- **Guest tokens** (`sub=guest_<uuid>`, `org_id=null`, `role=guest`,
  `org_ids=[]`) bypass the DB lookup but inherit the same RLS path:
  they read only `is_public=true` rows and can never write.

### 7. Auditing

```
   ┌── any privileged route handler ──────────────────────────────┐
   │                                                              │
   │   with audit_action("create_building", principal,            │
   │                     organization_id=org) as detail:          │
   │       detail["building_id"] = …                              │
   │       repo.create_building(…)                                │
   │       detail["spaces_recomputed"] = n                        │
   │                                                              │
   │   (the context manager records success on clean exit and     │
   │    failure on any raised exception — same row shape either   │
   │    way, so the table is uniform for SIEM ingestion)          │
   └────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼   INSERT INTO audit_log VALUES (
                                  occurred_at, action,
                                  actor_user_id, actor_email,
                                  organization_id,
                                  ip_address, user_agent,
                                  success, detail (jsonb) )

   Actions covered:
     signup, login (success + failure), guest login,
     MFA setup / confirm / disable,
     password change, password/forgot, password/reset,
     import_map (JSON), import_dxf (CAD — detail carries filename
       + spaces_imported), create / update / delete on every
       graph entity (org / campus / building / floor / space /
       connection).
```

Every privileged operation writes a row to `audit_log` with subject user,
subject email, organization, IP address, user-agent, action name, success
flag, and a JSON `detail` blob. This includes signups, logins (success +
failure), MFA setup / confirm / disable, password change / reset
request / reset, guest logins, and bulk imports (`import_map` for the
JSON path, `import_dxf` for the CAD path — both record the campus id, the
filename, and the spaces-imported count in `detail`). Failures keep the
same schema as successes so the table is uniform for SIEM ingestion.

### 8. Transport

- **HTTPS** terminates at the gateway in production (the K8s ingress).
  Inside the cluster traffic is plaintext but bound to the Docker /
  cluster network only.
- **STARTTLS** for the SMTP relay (`SMTP_USE_TLS=true` by default).
- **WSS** for the live ML-Vision stream — same `X-Api-Key` check at
  handshake.

### 9. Secrets

- All secrets (`AUTH_JWT_SECRET`, `INTERNAL_EMAIL_TOKEN`, SMTP creds,
  Neo4j + Postgres credentials, `API_SECRET`, `HF_TOKEN`) are read from
  env. None are committed; the example `.env` lives outside source
  control and is referenced from `docker-compose.yml` via `${VAR:-default}`.
- A boot-time helper (`generate_default_jwt_secret`) prints a 32-byte
  URL-safe secret — used during initial setup, never called at runtime.

### 10. File uploads + import

```
   iOS / mapmaker
        │  multipart upload (e.g. 4 room photos + room_id)
        │  Authorization: Bearer <jwt>,  X-Api-Key
        ▼
   ┌──────── Gateway ────────────────────────────────────────────┐
   │  X-Api-Key check        ─►  401                             │
   │  decode JWT (typ=access) ─► 401                             │
   │  strip + re-stamp identity headers (section 1)              │
   └────────────────────┬────────────────────────────────────────┘
                        │  HTTP, with x-user-id / x-org-id /
                        │  x-user-role on the upstream request
                        ▼
   ┌──── image_pipeline /room-summary/* (POST routes) ───────────┐
   │  require user-bound JWT:                                    │
   │     role == "guest"       ─►  403                           │
   │     org_id is None        ─►  403                           │
   │                                                             │
   │  YOLO11 inference, vectorise, embed                         │
   │     ▼                                                       │
   │  write back to the Space node in Neo4j:                     │
   │     metadata.room_summary       (full summary)              │
   │     uploaded_by_user_id         (FK → Postgres app_users)   │
   │     uploaded_by_org_id          (matches principal.org_id)  │
   └─────────────────────────────────────────────────────────────┘

   Cross-DB FK note: app_users lives in Postgres, Space lives in
   Neo4j — uploaded_by_* are scalar references, not graph nodes,
   so the two stores stay decoupled but the audit trail is intact.

   Same shape protects: /room-summary, /room-summary/by-room,
   /room-summary/room-objects/setup, /room-summary/similarity/ad-hoc.
```

- Bulk-import payloads are validated against the Pydantic
  `MapImport` / `BuildingImport` / `FloorImport` schemas; unknown fields
  are rejected, types are coerced, and the resulting rows pass through
  the same RLS policies on write.
- Per-request size limit on the gateway prevents oversize JSON imports
  from exhausting backend memory.
- **Room-photo uploads are tenant-scoped.** Every `POST` route on the
  image pipeline (`/room-summary`, `/room-summary/by-room`,
  `/room-summary/room-objects/setup`, `/room-summary/similarity/ad-hoc`)
  requires a JWT that decodes to a real user with an `org_id`. Guest
  tokens (`sub=guest_<uuid>`, `role=guest`) get a `403`; tokens with no
  `org_id` also get a `403`. The middleware re-stamps `X-User-Id` /
  `X-Org-Id` / `X-User-Role` from the verified token, so the pipeline
  trusts the headers without re-decoding the JWT itself.
- The setup route additionally records who uploaded what:
  `space.uploaded_by_user_id` and `space.uploaded_by_org_id` are written
  alongside the room embedding on the `Space` node, plus mirrored into
  `metadata.room_summary` for export. Users live in Postgres, so this is
  a cross-DB foreign key — there is no `User` node in Neo4j, just the
  scalar reference. This gives an audit trail (who set up which room)
  without coupling the two databases.

---

## Cluster topology and pod-to-pod communication

This section describes how the services in [Data flow](#data-flow) actually
talk to each other once the stack is deployed on Kubernetes. The focus is
the *plumbing* — how a request reaches a pod, how one pod reaches another,
and what happens when the responding pod isn't on the same node.

### Single-node view (everything on one host)

In dev / staging the entire backend can run on a single Kubernetes node
(or its Docker Compose equivalent). The pod-to-pod calls then never
leave the host's network namespace; each service still resolves its
peers through the cluster DNS and the in-kernel `kube-proxy` rules,
not through the public gateway address.

```
   ┌────────────────────────────────────────── Internet ─────────────────────────────────────┐
   │                                                                                          │
   │   iOS / Android / mapmaker                                                               │
   │   HTTPS GET /api/v1/...                                                                  │
   │   (X-Api-Key + Authorization: Bearer <jwt>)                                              │
   │                                                                                          │
   └────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                            │
                                            ▼   :443 (TLS terminated by ingress controller)
   ┌────────────────────────────────────────────────────────────────────────────────────────┐
   │  Kubernetes node 'ariadne-0'  (one VM / bare-metal host)                                │
   │                                                                                         │
   │  ┌─────────────────────────────────── Ingress layer ────────────────────────────────┐  │
   │  │   nginx-ingress-controller pod                                                   │  │
   │  │     • holds the public TLS cert (cert-manager / Let's Encrypt)                   │  │
   │  │     • terminates TLS, re-encodes as plain HTTP into the cluster network          │  │
   │  │     • host: api.ariadne.example  →  Service "middleware" :8080                   │  │
   │  └────────────────────────────────────────┬─────────────────────────────────────────┘  │
   │                                            │  10.244.0.1:8080  (ClusterIP, virtual)    │
   │                                            ▼                                            │
   │  ┌──────────────────── ClusterIP "middleware" (kube-proxy iptables/IPVS) ──────────┐   │
   │  │   resolves to one of the gateway pods via the Endpoints object,                  │   │
   │  │   round-robined per L4 connection                                                │   │
   │  └──────────────────────────────┬───────────────────────────────────────────────────┘   │
   │                                  │                                                       │
   │  ┌───────────────────────────────▼─────────────────────────────────────────────────┐   │
   │  │  Pod  middleware-78d4-xyz   (image: ariadne/middleware)                          │   │
   │  │    container 'middleware'   listening on :8080                                   │   │
   │  │    container's veth ─►  cni0 bridge ─►  node netns                               │   │
   │  │                                                                                  │   │
   │  │    inside the container:                                                         │   │
   │  │      verify_api_key(X-Api-Key)                                                   │   │
   │  │      decode JWT (HS256, AUTH_JWT_SECRET from Secret)                             │   │
   │  │      stamp x-user-id / x-org-id / x-org-ids / x-user-role                        │   │
   │  │      route by path prefix:                                                       │   │
   │  │         /assistant/*    →  http://assistant.ariadne.svc.cluster.local:8001       │   │
   │  │         /room-summary/* →  http://image-pipeline.ariadne.svc.cluster.local:8002  │   │
   │  │         /ml-vision/*    →  http://ml-vision.ariadne.svc.cluster.local:8003       │   │
   │  │         /*              →  http://backend.ariadne.svc.cluster.local:8000         │   │
   │  └────┬────────────────┬────────────────┬─────────────────────────┬─────────────────┘   │
   │       │                │                │                         │                     │
   │       │ DNS lookup of  │                │                         │                     │
   │       │ 'backend.ariadne.svc.cluster.local'                       │                     │
   │       │   → CoreDNS    │                │                         │                     │
   │       │   → ClusterIP  │                │                         │                     │
   │       │     10.244.0.4 │                │                         │                     │
   │       │                │                │                         │                     │
   │       ▼                ▼                ▼                         ▼                     │
   │  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐  ┌─────────────────────────┐    │
   │  │ ClusterIP   │  │ ClusterIP   │  │ ClusterIP      │  │ ClusterIP               │    │
   │  │  backend    │  │  assistant  │  │ image-pipeline │  │  ml-vision              │    │
   │  └──────┬──────┘  └──────┬──────┘  └────────┬───────┘  └────────────┬────────────┘    │
   │         │                │                  │                       │                   │
   │         ▼                ▼                  ▼                       ▼                   │
   │  ┌────────────┐   ┌────────────┐    ┌────────────────┐     ┌─────────────────┐        │
   │  │ Pod        │   │ Pod        │    │ Pod            │     │ Pod             │        │
   │  │ backend-…  │   │ assistant-…│    │ image-pipeline-│     │ ml-vision-…     │        │
   │  │ :8000      │   │ :8001      │    │   …  :8002     │     │ :8003           │        │
   │  │            │   │            │    │                │     │                 │        │
   │  │ FastAPI    │   │ FastAPI    │    │ FastAPI +      │     │ FastAPI +       │        │
   │  │ + Neo4j    │   │ + LLM      │    │ YOLO11 +       │     │ YOLO26 ONNX +   │        │
   │  │ driver     │   │ + embedder │    │ OpenCV         │     │ ws bridge       │        │
   │  │ + SQLAlc.  │   │            │    │                │     │                 │        │
   │  └─────┬──────┘   └─────┬──────┘    └────────┬───────┘     └────────┬────────┘        │
   │        │                │                    │                      │                  │
   │        │ Bolt           │ Bolt (read-only)   │ HTTP                 │ HTTP             │
   │        │ neo4j+s://     │                    │ (model files via PVC)│                  │
   │        ▼                ▼                    ▼                      ▼                  │
   │   ┌─────────────────────────┐         ┌─────────────────┐  ┌──────────────────┐       │
   │   │ Neo4j Aura  (external,  │         │ ./models/       │  │ ./models/        │       │
   │   │ outside the cluster)    │         │ PersistentVol.  │  │ PersistentVol.   │       │
   │   └─────────────────────────┘         │ Claim 'pvc-img' │  │ Claim 'pvc-mlv'  │       │
   │                                       └─────────────────┘  └──────────────────┘       │
   │        │                                                                                │
   │        ▼  egress NAT through node IP                                                    │
   │   ┌─────────────────────────────────────────┐                                          │
   │   │ Supabase PostGIS  (external,            │  every backend pod opens a               │
   │   │ aws-1-eu-west-1.pooler.supabase.com)    │  shared SQLAlchemy engine (singleton),   │
   │   │ session-mode pooler, 15-client cap      │  pool_size=2  max_overflow=5 → ≤7 conns  │
   │   └─────────────────────────────────────────┘  per pod, ≤14 across 2 backend replicas. │
   │                                                                                         │
   │  Internal DNS:  CoreDNS pod resolves *.svc.cluster.local from the kube-apiserver's      │
   │                  Services API.  TTL is 5 s.                                             │
   │  Pod-to-pod L4: kube-proxy programs iptables (or IPVS) rules per Service that DNAT      │
   │                  ClusterIP → one of the Endpoints (i.e. one of the matching pods).      │
   │  Pod network:   the CNI (Flannel / Calico / Cilium) gives each pod a routable IP on     │
   │                  the node's cni0 bridge; cross-pod traffic is bridged in-kernel.        │
   └─────────────────────────────────────────────────────────────────────────────────────────┘
```

### How a single request reaches the right pod

Take a representative call: `GET /api/v1/buildings/visible` from the iOS
client. Steps 1-4 are the *external* path; 5-8 are the *internal* hop.

1. **DNS / TLS / Ingress.** The phone resolves `api.ariadne.example` to
   the LoadBalancer IP fronting the ingress-controller pod. The
   controller terminates TLS using its mounted cert and forwards the
   request as plain HTTP to the `middleware` ClusterIP.
2. **kube-proxy DNAT.** The Service's ClusterIP is a virtual IP — no
   pod actually owns it. `kube-proxy` on the node has installed
   iptables (or IPVS) rules that intercept packets to that IP and DNAT
   them to one of the Service's healthy Endpoints. The selection is
   round-robin per connection.
3. **Pod ingress.** The packet arrives at the gateway pod's veth, the
   CNI bridges it into the pod's network namespace, and FastAPI accepts
   the connection on `:8080`.
4. **Auth at the edge.** The gateway runs its `verify_api_key`
   middleware (constant-time `hmac.compare_digest`), decodes the JWT
   with the secret read from the Kubernetes `Secret` mounted as an env
   variable, strips any caller-supplied identity headers, and stamps
   `x-user-id`, `x-org-id`, `x-org-ids`, `x-user-role` from the verified
   claims.
5. **Internal DNS lookup.** The gateway's outgoing request targets
   `http://backend.ariadne.svc.cluster.local:8000/buildings/visible`.
   The Linux resolver consults the pod's `/etc/resolv.conf`, which
   points at CoreDNS. CoreDNS asks the kube-apiserver "which pods back
   this Service?" and returns the ClusterIP for `backend`.
6. **Second kube-proxy hop.** Same pattern as step 2: the ClusterIP is
   DNATed to one of the backend pods. If two backend replicas exist,
   the connection goes to whichever the iptables/IPVS rule selects.
7. **Backend handles the request.** `routes/buildings.py:list_visible_
   buildings` reads `principal.org_ids` from the trusted header, calls
   `CampusRepository.list_visible_buildings(org_ids=...)`, which runs a
   single Cypher query against Neo4j Aura (egress NAT through the
   node's external IP).
8. **Response retraces the path.** Backend → gateway → ingress
   controller → phone, each hop using the same connection it received
   the request on.

### How pods communicate when the cluster spans multiple nodes

In production the deployment scales horizontally onto multiple nodes.
Two facts change; the rest stays the same.

```
   Node A                                                Node B
   ┌──────────────────────────────────┐                 ┌──────────────────────────────────┐
   │  middleware pod (replica 1)      │                 │  middleware pod (replica 2)      │
   │       │                          │                 │       │                          │
   │       │ HTTP to                  │                 │       │ HTTP to                  │
   │       │ backend.ariadne.svc      │                 │       │ backend.ariadne.svc      │
   │       ▼                          │                 │       ▼                          │
   │  kube-proxy chooses an Endpoint  │                 │  kube-proxy chooses an Endpoint  │
   │       │                          │                 │       │                          │
   │       │  if Endpoint is on Node A: bridge in kernel│       │  if on Node B: bridge    │
   │       │  if Endpoint is on Node B: VXLAN/eBPF      │       │  if on Node A: VXLAN/eBPF│
   │       │  encapsulated to node-B's IP, decapped     │       │  encapsulated to node-A  │
   │       │  on arrival                                │       │                          │
   │       ▼                          │                 │       ▼                          │
   │  backend pod (replica 1)         │                 │  backend pod (replica 2)         │
   │  10.244.0.4                      │  ◄── overlay ──►│  10.244.1.7                      │
   │                                  │     network     │                                  │
   │  cni0 bridge (10.244.0.0/24)     │                 │  cni0 bridge (10.244.1.0/24)     │
   │       │                          │                 │       │                          │
   │       └─► node A eth0 ──────────►│  VXLAN tunnel  ◄│──── eth0 node B                  │
   │            (or eBPF redirect)    │ (UDP 4789, or  │                                  │
   │                                  │  IP-in-IP, or  │                                  │
   │                                  │  raw eBPF)     │                                  │
   └──────────────────────────────────┘                 └──────────────────────────────────┘
```

1. **Cross-node packets travel over an overlay.** The CNI plugin
   (Flannel uses VXLAN, Calico uses BGP-routed IP-in-IP or unencap,
   Cilium uses eBPF) encapsulates a pod-to-pod packet at the source
   node, sends it as a regular UDP / IP / eBPF redirect to the
   destination node, and decapsulates on arrival. To the application
   the packet looks identical to a same-node call — `10.244.0.4 →
   10.244.1.7` — but the kernel handled the cross-node hop transparently.
2. **The Service abstraction is unchanged.** A pod still resolves
   `backend.ariadne.svc.cluster.local`, still gets a ClusterIP, still
   has its packets DNATed by the local node's kube-proxy. The
   destination Endpoint may be local or remote; the application
   doesn't know and doesn't care.

### Why the gateway is the only externally-routable Service

The middleware Service is the only `LoadBalancer` (or `NodePort`) in
the manifest; every other service is `ClusterIP`-only and therefore
unreachable from outside the cluster. This collapses the auth surface
to one place: an attacker cannot reach `backend`, `assistant`,
`image-pipeline`, or `ml-vision` directly even if they're on the
cluster network's NAT gateway. The trust boundary is a single Service
manifest entry.

If a future deployment needed the assistant or ML-Vision exposed
directly (say, a partner integration that wants to embed the chat),
the path would be a *second* Ingress rule with its own auth (mTLS or
a separately-issued API key) — never a second `LoadBalancer` Service
that bypasses the gateway.

### External services (Neo4j Aura, Supabase, SMTP)

The two databases and the email relay live outside the cluster on
managed services. From the cluster's point of view they're regular
egress destinations: a backend pod opens a TCP connection to
`neo4j+s://...` or `aws-1-eu-west-1.pooler.supabase.com:5432`, which
goes out through the node's external IP, hits the public internet,
and lands at the managed service.

Two operational details:

- **Connection-pool sizing assumes the cluster's replica count.** With
  the singleton-engine fix in `PostGISService`, each backend pod opens
  ≤7 connections to the pooler. Supabase's session-mode pooler caps at
  15 clients per database role. Two backend replicas (≤14 connections)
  fits; four replicas would exceed it and would need either
  transaction-mode pooling or a per-replica role.
- **Egress IP allowlisting.** A future tightening pass would put the
  cluster behind a fixed-IP egress NAT (a Cloud NAT gateway or a
  dedicated egress node) and add the IP to the Supabase / Neo4j
  allowlist. Today the egress IP rotates with the underlying node,
  which is fine while the project is small but is a TODO for any
  real production deploy.

---

## Asynchronous behaviour & multi-device support

The platform routinely fields concurrent traffic from many clients on
many devices — multiple phones streaming camera frames into ML-Vision,
multiple editors hitting the mapmaker, multiple assistant chats running
at once. This section explains *what kind* of asynchrony the stack
delivers today, where the genuine bottlenecks live, and what changes
would be needed to add **client-perceived** async (fire-and-forget jobs
the client polls for, rather than long-held HTTP connections).

### Two senses of "async" in the architecture

The word "asynchronous" gets used for two different things; the
distinction matters because the stack delivers one and not the other.

```
                          What we have today                  What we don't have today
                          ───────────────────────             ─────────────────────────
                                                                                          
 Server-side concurrency  asyncio + async def + await         (already covered)
                          One event loop per pod handles                                  
                          ~thousands of simultaneous in-                                  
                          flight requests without blocking                                
                          threads. Every IO point (httpx,                                 
                          Neo4j Bolt, asyncpg, websockets,                                
                          ThreadPoolExecutor) cooperates.                                 
                                                                                          
 Client-perceived async   (none — every call is request/      Jobs table + worker pool.   
                          response, the client awaits the     POST returns 202 + job id;  
                          full reply over a single open       client polls GET /jobs/{id} 
                          HTTP connection, even when the      or subscribes via SSE /     
                          server takes 30+ seconds to         WS until status=done.       
                          produce it — DXF import, RAG +      Server-side workers consume 
                          LLM, room-summary batch.)           the queue independently of  
                                                              the original HTTP socket.   
```

A request to `POST /campuses/import-dxf` today is **server-side async,
client-perceived sync**: the gateway and backend never block a thread
while waiting for Neo4j / PostGIS to acknowledge, but the iPhone or
browser tab that issued the call has to keep the HTTPS connection open
until the import finishes (the gateway's `httpx` timeout is 900 s
precisely because of this). If the wifi blinks, the client gets a
connection error even though the import succeeded server-side.

This is fine for everything we ship today because no operation a normal
user triggers actually runs that long: an authoring DXF import is the
worst case and it's still under a minute on real floor plans. The
proposal at the end of the section is what we'd build *if* a flow ever
needed to exceed the wifi's tolerance for held connections.

### Streaming inference with many concurrent devices (ML-Vision)

The WS streaming path is the one place in the stack where multi-device
behaviour matters in production, because each phone in a building can
hold a long-lived camera stream open against the same ml-vision pod.
What follows is what happens when N phones stream at once.

```
   Phone 1 ─┐                            ┌─►  inference worker 1 ─┐
   Phone 2 ─┤  WS handshake              │                        │
   Phone 3 ─┼─►  middleware  ─►  ml-vision pod  ─►  ThreadPool   ─┼─► shared YOLO26
   Phone 4 ─┤  (2-task bridge,           │     (max_workers=2,    │     ONNX session
     …      │   one per client)          │      bounded queue)    │     (thread-safe,
   Phone N ─┘                            └─►  inference worker 2 ─┘     loaded once
                                                                        per facility
                                                                        via @lru_cache(16))

   What scales linearly with N:                  What does NOT scale linearly with N:
   ─────────────────────────────                 ─────────────────────────────────────
   • WS connection count                         • Inference throughput
     (asyncio coroutines are cheap —               (capped at ~ workers × per-frame ms;
      thousands fit per pod, RAM dominates)         8 fps total on the 2-worker pool)
   • Frame receive rate                          • Per-client effective fps as N grows
     (await receive_bytes is non-blocking)         (each client gets 1/N of the budget)
   • Result dispatch                             • Memory if facilities differ widely
     (await send_text per client)                  (each new facility loads a new ONNX
                                                    session into the @lru_cache(16) bucket)
```

**Concurrent device support today.** Yes — the WS path supports multiple
simultaneous devices out of the box. The architecture is:

- **One asyncio coroutine per client connection.** asyncio coroutines
  are user-space green threads — they cost a few KB of RAM each. A
  single ml-vision pod can comfortably hold *thousands* of idle WS
  connections; per-frame work is what limits practical concurrency,
  not connection count.
- **Single shared ONNX session per facility.** `@lru_cache(16)
  get_detector(facility_id)` in [ml_vision/main.py](aau-sw8-ml-vision/main.py)
  caches up to 16 loaded models. Every client on the same `facility_id`
  shares the same `onnxruntime.InferenceSession`, which is thread-safe
  for concurrent `run()` calls. The model is loaded from disk *once*
  per facility and reused for every frame from every device pointing
  at that facility — there is no per-client model duplication.
- **Bounded thread-pool keeps the asyncio loop responsive.**
  `ThreadPoolExecutor(max_workers=2)` is the actual inference
  workhorse. Per-frame work (ONNX `run` plus the post-processing pass)
  is dispatched via `loop.run_in_executor(_stream_executor, ...)` so
  the receive / send tasks never block while a frame is being
  processed. The pool size is the throughput knob — at CPU-only
  inference, two workers ≈ 8 fps of aggregate throughput.
- **Back-pressure prevents OOM under load.** Each WS bridge tracks
  whether a frame is in flight; if a new frame arrives while the
  previous is still being processed, the new one is dropped. This is
  the right behaviour for live camera streams (a stale frame is
  useless, you want the freshest one), and it means a slow inference
  worker can not pile up an unbounded backlog of pending frames per
  client. The phone's effective fps gracefully degrades instead of
  the pod OOM-ing.

**How per-client throughput degrades with N.** The 2-worker pool's
aggregate budget is divided round-robin across clients with frames in
flight. Rough envelope on the current CPU-only deployment (250 ms / frame
on YOLO26-small):

```
   N clients     aggregate fps     per-client effective fps
   ───────────   ──────────────    ──────────────────────────
       1              8                    8
       2              8                    4
       4              8                    2
      10              8                    0.8       ← noticeable lag
      50              8                    0.16      ← unusable in practice
```

Pod-level capacity is therefore roughly **2-4 phones per pod** at a
comfortable interactive frame rate, and beyond that the user perception
is "the AR overlay updates more slowly the more people are using it"
rather than "the AR overlay breaks." More devices keep working — they
just get fewer slots.

**Scaling options when N grows past 4.**

1. **Raise `max_workers`.** Free on CPU up to the physical core count;
   on GPU, more workers does not help past the GPU's saturation point
   (it just queues at the device). Lowest-effort knob; biggest gain
   per minute of work.
2. **Frame batching.** Group N frames from N clients into one ONNX
   `run` and split the results. On GPU this is *the* highest-leverage
   change because GPU inference is dominated by per-batch overhead,
   not per-frame compute — a batch of 8 frames is roughly the cost of
   a batch of 1. On CPU the speedup is smaller (~2×) but still real.
   Requires changes to the detector wrapper to accept batched inputs
   and to demux detections back to the right client.
3. **Horizontal scaling.** Bump the ml-vision Deployment's
   `replicas`. kube-proxy round-robins new WS handshakes across pods
   at the L4 layer; existing connections stay pinned to their pod.
   Linear scaling — N pods serves ~N × 4 devices comfortably. The
   ConfigMap stays the same; only the `replicas` count changes.
4. **Per-client rate limiting.** Cap each device's send rate to a
   ceiling (say 5 fps) on the iOS / Android side. This is a code
   change in the client, not the server — but it's the single
   cheapest move because most clients don't need more than 5 fps of
   AR refresh anyway, and it prevents one client from monopolising
   the inference budget.
5. **GPU pinning.** Move ml-vision pods to GPU nodes (NVIDIA device
   plugin + `resources.limits.nvidia.com/gpu: 1`) and switch ONNX to
   the CUDA execution provider. Per-frame latency drops from ~250 ms
   to ~15 ms, which on its own makes the 2-worker pool comfortable
   at 60+ fps aggregate. Combined with batching, the same pod can
   serve dozens of devices.

**What about the LLM in the assistant?** Multi-device behaviour there
is different and *worse* per-client than ml-vision, because LLM
generation holds GPU / CPU for the entire token stream (typically
2–10 s per reply). The assistant deployment today has no batching or
streaming optimisation — N concurrent chats land on the same model
sequentially. Two mitigations: (a) horizontal-scale assistant pods
the same way as ml-vision, (b) move LLM generation to a streaming
endpoint (`text/event-stream`) so the client perceives progress even
while the GPU works through the prompt queue. Neither is wired today;
the assistant is the next service that would benefit from the
job-queue pattern described below.

### Adding client-perceived async (proposal — not implemented)

If a future flow ever needs to outlast the wifi's tolerance for held
connections (a multi-floor DWG import for a whole campus, a long-running
batch room-summary across hundreds of photos, an LLM that takes longer
than the gateway's 900 s timeout), the change is to introduce a **jobs
table** and a **worker pool** so the client can fire-and-forget. The
shape:

```
                  ┌────── Client (iOS / Android / mapmaker) ───────┐
                  │                                                │
                  │  POST /api/v1/imports/dxf  ─► 202 Accepted     │
                  │       Location: /api/v1/jobs/{job_id}          │
                  │                                                │
                  │  poll: GET /api/v1/jobs/{job_id}               │
                  │   { status: 'queued' | 'running' | 'done'      │
                  │            | 'failed',                         │
                  │     progress: 0.42,                            │
                  │     result_url?: ...,  error?: ... }           │
                  │                                                │
                  │  (or)  subscribe via SSE / WS:                 │
                  │     GET /api/v1/jobs/{job_id}/stream           │
                  │     event: progress  data: {pct: 0.42}         │
                  │     event: done      data: {url: ...}          │
                  └────────────────────┬───────────────────────────┘
                                       │
                                       ▼
                  ┌────── Gateway ─────────────────────────────────┐
                  │  /imports/dxf   ─►  backend (sync 202)         │
                  │  /jobs/{id}     ─►  backend (sync lookup)      │
                  └────────────────────┬───────────────────────────┘
                                       │
                                       ▼
                  ┌────── Backend (HTTP-facing) ───────────────────┐
                  │  POST /imports/dxf:                            │
                  │    INSERT INTO jobs (id, kind, status='queued',│
                  │            payload=req, organization_id=...);  │
                  │    NOTIFY jobs_queue;                          │
                  │    return 202 { job_id, status_url }           │
                  │                                                │
                  │  GET /jobs/{id}: SELECT FROM jobs WHERE …      │
                  └────────────────────┬───────────────────────────┘
                                       │
                                       ▼  (NOTIFY / LISTEN over the same
                                       │   Postgres connection — no Redis,
                                       │   no SQS, no extra infra)
                                       ▼
                  ┌────── Worker pool (separate deployment) ───────┐
                  │  asyncio worker loop:                          │
                  │    UPDATE jobs SET status='running', worker=…  │
                  │      WHERE id = (SELECT id FROM jobs           │
                  │                  WHERE status='queued'         │
                  │                  ORDER BY created_at           │
                  │                  FOR UPDATE SKIP LOCKED        │
                  │                  LIMIT 1)                      │
                  │    run the actual import / batch / LLM         │
                  │    UPDATE jobs SET status='done' / 'failed',   │
                  │      result=…, progress=1.0                    │
                  │  (FOR UPDATE SKIP LOCKED is the load-balancer; │
                  │   N workers cooperate without a coordinator)   │
                  └────────────────────────────────────────────────┘
```

**Cost of this change.** A new table (`jobs` with id, kind, status,
payload, progress, result, error, organization_id, created/updated
timestamps), one new Deployment (`ariadne-worker`, same image as the
backend with a different entrypoint), and an SSE/WS streaming endpoint
on the backend for clients that want push updates. RLS extends to the
new table the same way it extends to imports — `organization_id`
column, `org_isolation` policy, `app.org_id` GUC stamped per session.

**Why we haven't built it.** No current flow needs it. Every long
operation today either (a) finishes inside the 900 s gateway timeout
in practice (DXF import), or (b) is already streamable (ml-vision
WS). Adding the table prematurely would create a second source of
truth for "is this import done" that complicates the Neo4j→PostGIS
sync model — better to add it the day a real workflow asks for it
than to speculate now.

---

## File layout

```
ariadne/
├── COMPLETE_ARCHITECTURE.md    # this file
├── README.md                   # top-level module map + quick start
├── users.md                    # user model & multi-tenant plan
├── aau-sw8-spatial-backend/
│   ├── middleware/             # gateway
│   ├── backend/                # Neo4j + PostGIS writer, routing, auth, MFA, outbox worker
│   ├── assistant/              # LLM + RAG
│   ├── image_pipeline/         # YOLO11 room summaries
│   ├── email-service/          # internal-only SMTP relay (MFA + reset)
│   └── frontend/               # static mapmaker
├── aau-sw8-ml-vision/          # YOLO26 live-stream WS service
├── aau-sw8-ios/                # SwiftUI client
├── aau-sw8-android/            # Compose prototype
├── aau-sw8-indoor-data-pipeline/  # standalone paper-topology → graph importer (research artefact)
├── db_setup/                   # one-shot DB bootstrap (Neo4j + PostGIS)
├── k8s/                        # Kubernetes manifests (see KUBERNETES.txt)
└── scripts/                    # repo-level helper scripts
```

---

## Code Snippets

This section has two parts. **Backend deep-dives** (1–6 below) take the
deepest, most load-bearing logic in the backend service
(`aau-sw8-spatial-backend/backend`) — each gives the function, a
one-paragraph description, and the real implementation with inline
comments on the non-obvious behaviour. (The earlier [Code](#code)
section samples the whole platform — iOS, Android, assistant, CV; these
go a level deeper on the hardest backend pieces.) A final subsection,
[Listings from the project report](#listings-from-the-project-report-ariadne_finalpdf),
reproduces the ten code listings shown in `ariadne_final.pdf`
verbatim, so the deck-facing report and this document stay in sync.

### 1. `python_dijkstra.find_path` — weighted A* routing core

**What it does.** Computes the cheapest walking route between two
Spaces. Despite the module name it is **A\***, not plain Dijkstra: it
loads the navigable sub-graph from Neo4j in two queries, builds a
weighted adjacency list (door-aware, 3D cross-floor — see `_edge_weight`),
then runs a priority-queue search using an *admissible* straight-line
geographic heuristic so that geometric ties break toward the
destination instead of by arbitrary UUID order. It is the portable
middle tier of the GDS → Python → native-Cypher routing fallback chain.

```python
def _edge_weight(src: dict, dst: dict) -> float:
    """Edge cost in seconds (walking-time approximation)."""
    sx, sy = src.get("centroid_x"), src.get("centroid_y")
    dx, dy = dst.get("centroid_x"), dst.get("centroid_y")
    have_centroids = None not in (sx, sy, dx, dy)

    src_fi, dst_fi = src.get("floor_index"), dst.get("floor_index")
    src_bi, dst_bi = src.get("building_id"), dst.get("building_id")
    floor_step = (abs(int(src_fi) - int(dst_fi))
                  if src_fi is not None and dst_fi is not None else 0)
    different_building = (src_bi is not None and dst_bi is not None and src_bi != dst_bi)

    dist_m: float
    if have_centroids and floor_step == 0 and not different_building:
        # Same floor: when exactly one endpoint is a connector (door/passage),
        # measure from the room's polygon edge nearest the door, not its
        # centroid — a 30 m corridor with a door at one end is ~5 m of walk.
        src_is_conn = src.get("space_type") in _CONNECTION_TYPES
        dst_is_conn = dst.get("space_type") in _CONNECTION_TYPES
        if src_is_conn and not dst_is_conn:
            ex, ey = _room_exit_point(dst, float(sx), float(sy))
            dist_m = math.hypot(float(sx) - ex, float(sy) - ey)
        elif dst_is_conn and not src_is_conn:
            ex, ey = _room_exit_point(src, float(dx), float(dy))
            dist_m = math.hypot(ex - float(dx), ey - float(dy))
        else:
            dist_m = math.hypot(float(sx) - float(dx), float(sy) - float(dy))
    elif have_centroids and floor_step > 0 and not different_building:
        # Cross-floor: 3D Pythagoras with a per-floor vertical step.
        horizontal = math.hypot(float(sx) - float(dx), float(sy) - float(dy))
        vertical = floor_step * _DEFAULT_FLOOR_HEIGHT_M
        dist_m = math.hypot(horizontal, vertical)
    else:
        dist_m = 1.0  # missing geometry → nominal unit cost

    # A connector target also costs the time to actually pass through it
    # (elevator wait, door open) carried on the node as traversal_cost.
    penalty = float(dst.get("traversal_cost") or 0.0) if dst.get("space_type") in _CONNECTION_TYPES else 0.0
    return (dist_m / _WALKING_SPEED_MS) + penalty


def find_path(db, from_id, to_id, *, accessible_only=False, excluded_types=None):
    """Run weighted A* from from_id to to_id."""
    excluded = set(excluded_types or [])

    # 1. Pull every navigable node (plus the two endpoints even if not navigable).
    nodes = db.execute("""MATCH (s:Space)
        WHERE s.is_navigable = true OR s.id IN [$from_id, $to_id]
        RETURN s.id AS id, s.space_type AS space_type, s.floor_index AS floor_index,
               s.building_id AS building_id, s.centroid_x AS centroid_x,
               s.centroid_y AS centroid_y, s.polygon AS polygon,
               s.is_accessible AS is_accessible, s.traversal_cost AS traversal_cost
               /* …other fields… */""",
        {"from_id": from_id, "to_id": to_id})
    if not nodes:
        return None
    by_id = {n["id"]: dict(n) for n in nodes}
    if from_id not in by_id or to_id not in by_id:
        return None

    # 2. Pull the CONNECTS_TO edges between those nodes.
    edges = db.execute("""MATCH (s:Space)-[:CONNECTS_TO]->(t:Space)
        WHERE (s.is_navigable = true OR s.id IN [$from_id, $to_id])
          AND (t.is_navigable = true OR t.id IN [$from_id, $to_id])
        RETURN s.id AS from_id, t.id AS to_id""",
        {"from_id": from_id, "to_id": to_id})

    # 3. Build the weighted adjacency list, applying per-request filters
    #    (accessibility, excluded types) to intermediate nodes only — the
    #    endpoints are always allowed even if they fail a filter.
    endpoints = {from_id, to_id}
    adj = {nid: [] for nid in by_id}
    for e in edges:
        src, dst = by_id.get(e["from_id"]), by_id.get(e["to_id"])
        if src is None or dst is None:
            continue
        if e["to_id"] not in endpoints:
            if dst.get("space_type") in excluded:
                continue
            if accessible_only and dst.get("is_accessible") is False:
                continue
        adj[e["from_id"]].append((e["to_id"], _edge_weight(src, dst)))

    # 4. Admissible heuristic: straight-line metres to goal ÷ walking speed.
    #    Never overestimates, so A* stays optimal; it only reorders the
    #    frontier so geometric ties resolve toward the destination.
    goal = by_id.get(to_id, {})
    gx, gy = goal.get("centroid_x"), goal.get("centroid_y")
    def _h(node_id):
        if gx is None or gy is None:
            return 0.0
        n = by_id.get(node_id, {})
        nx, ny = n.get("centroid_x"), n.get("centroid_y")
        if nx is None or ny is None:
            return 0.0
        return math.hypot(float(nx) - float(gx), float(ny) - float(gy)) / _WALKING_SPEED_MS

    dist = {from_id: 0.0}
    prev = {from_id: None}
    heap = [(_h(from_id), from_id)]            # priority = f = g + h
    while heap:
        f, u = heapq.heappop(heap)
        if u == to_id:
            break
        g = dist.get(u, math.inf)
        if f - _h(u) > g + 1e-9:               # stale heap entry (we found u cheaper later)
            continue
        for v, w in adj.get(u, []):
            ng = g + w
            if ng < dist.get(v, math.inf):     # relax
                dist[v] = ng
                prev[v] = u
                heapq.heappush(heap, (ng + _h(v), v))

    if to_id not in dist:
        return None

    # 5. Walk prev[] back from goal to start, reverse, project to plain dicts.
    path_ids, cur = [], to_id
    while cur is not None:
        path_ids.append(cur)
        cur = prev.get(cur)
    path_ids.reverse()
    path_nodes = [ { /* …projected node fields… */ } for nid in path_ids ]
    return {"path_nodes": path_nodes, "total_cost": dist[to_id]}
```

### 2. `NavigationService._build_polyline` — render-ready route line

**What it does.** Converts the raw node path into the lat/lng polyline
the clients actually draw. Three tricks make the line look like a human
route instead of a centroid zig-zag: (a) it inserts the **shared-wall
midpoint** between adjacent rooms so the line crosses at the door, not
through the wall; (b) it **snaps the very first/last point** to the
room's polygon edge nearest the next step (a "door exit" look); and
(c) for intermediate rooms it **pulls the waypoint toward the
straight line** between the surrounding anchors by a fixed factor so the
path doesn't detour into every room's centre. Local floor-plan metres
are projected to WGS84 via each node's (floor-override → building-inherit)
georef.

```python
_ROOM_PULL_FACTOR = 0.35   # 0 = straight through anchors, 1 = full centroid detour

def _build_polyline(self, path_nodes, georefs):
    n = len(path_nodes)
    # Door waypoint between each consecutive pair (None if no shared wall).
    transitions = [self._shared_edge_waypoint(path_nodes[i], path_nodes[i + 1], georefs)
                   for i in range(n - 1)]

    def _global_centroid(node, i):
        """Node centroid in world coords; if a door Space has only local
        coords, borrow a neighbour's georef to project it."""
        lat, lng = node.get("centroid_lat"), node.get("centroid_lng")
        if (lat is None or lng is None) and node.get("centroid_x") is not None:
            for nbr in (i - 1, i + 1):                     # try either neighbour
                if 0 <= nbr < n:
                    g = georefs.get(path_nodes[nbr]["id"])
                    if g and g["origin_lat"] is not None:
                        lat, lng = local_to_global_coordinates(
                            node["centroid_x"], node["centroid_y"],
                            g["origin_lat"], g["origin_lng"], g["bearing"], g["scale"])
                        node["centroid_lat"], node["centroid_lng"] = lat, lng
                        break
        return [float(lat), float(lng)] if lat is not None and lng is not None else None

    def _endpoint_exit_point(node, target):
        """Snap a terminal room's point to its polygon edge nearest `target`."""
        polygon = parse_polygon(node.get("polygon"))
        tx, ty = target.get("centroid_x"), target.get("centroid_y")
        g = georefs.get(node["id"])
        if polygon is None or tx is None or g is None or g.get("origin_lat") is None:
            return None                                    # caller falls back to centroid
        sx, sy = closest_point_on_polygon(polygon, float(tx), float(ty))
        return list(local_to_global_coordinates(sx, sy, g["origin_lat"], g["origin_lng"],
                                                g["bearing"], g["scale"]))

    centroids = [_global_centroid(node, i) for i, node in enumerate(path_nodes)]

    def _is_anchor(j):     # endpoints + connectors keep their exact point
        return j == 0 or j == n - 1 or self._is_connector(path_nodes[j])

    def _pulled_in_waypoint(i):
        """Pull a room's waypoint toward the midpoint of the nearest anchors."""
        room_c = centroids[i]
        prev_pt = next((centroids[j] for j in range(i - 1, -1, -1)
                        if _is_anchor(j) and centroids[j] is not None), None)
        next_pt = next((centroids[j] for j in range(i + 1, n)
                        if _is_anchor(j) and centroids[j] is not None), None)
        if prev_pt is None or next_pt is None:
            return room_c
        mid = [(prev_pt[0] + next_pt[0]) / 2, (prev_pt[1] + next_pt[1]) / 2]
        a = self._ROOM_PULL_FACTOR
        return [mid[0] + a * (room_c[0] - mid[0]), mid[1] + a * (room_c[1] - mid[1])]

    # Only snap terminal rooms (not if the endpoint is itself a connector).
    snapped_start = _endpoint_exit_point(path_nodes[0], path_nodes[1]) \
        if n >= 2 and not self._is_connector(path_nodes[0]) else None
    snapped_end = _endpoint_exit_point(path_nodes[n - 1], path_nodes[n - 2]) \
        if n >= 2 and not self._is_connector(path_nodes[n - 1]) else None

    # Assemble: anchor/pulled point per node, then the door waypoint after it.
    polyline, polyline_floors = [], []
    for i in range(n):
        fi = path_nodes[i].get("floor_index")
        if _is_anchor(i):
            point = (snapped_start if i == 0 and snapped_start else
                     snapped_end if i == n - 1 and snapped_end else centroids[i])
        else:
            point = _pulled_in_waypoint(i)
        if point is not None:
            polyline.append(point); polyline_floors.append(fi)
        if i < n - 1 and transitions[i] is not None:       # door between i and i+1
            polyline.append(transitions[i]); polyline_floors.append(fi)

    return polyline, _group_by_floor(polyline, polyline_floors)   # also split per floor
```

### 3. `ImportService._create_door_space_connection` — door-node materialization

**What it does.** A "door" in Ariadne is not an edge property — it is a
first-class `Space` node sitting between the two rooms it joins, so the
router can put a real waypoint there. This method synthesises that node:
it picks the door's coordinates (explicit → shared-wall midpoint →
centroid average), derives a **deterministic** id from the sorted
endpoint pair (so re-imports update in place instead of forking
duplicates), garbage-collects any stale door between the same pair,
creates the door Space in both stores, and wires the **four**
directed `CONNECTS_TO` edges (room↔door, door↔room) that make it
bidirectionally traversable.

```python
def _create_door_space_connection(self, conn, ct_upper):
    space_a = self.space_repo.get_space(conn.from_space_id)
    space_b = self.space_repo.get_space(conn.to_space_id)

    # Door coords: explicit from import → shared-edge midpoint → centroid mean.
    cx, cy = conn.door_cx, conn.door_cy
    if cx is None or cy is None:
        poly_a, poly_b = space_a.get("polygon"), space_b.get("polygon")
        if poly_a and poly_b:
            mid = find_shared_edge_midpoint(poly_a, poly_b)
            if mid: cx, cy = mid
    if cx is None or cy is None:
        a, b = space_a, space_b
        if all(v is not None for v in (a.get("centroid_x"), a.get("centroid_y"),
                                       b.get("centroid_x"), b.get("centroid_y"))):
            cx = (a["centroid_x"] + b["centroid_x"]) / 2
            cy = (a["centroid_y"] + b["centroid_y"]) / 2

    # Map connection_type + door_type → a concrete SpaceType.
    if ct_upper == "PASSAGE":
        door_space_type = SpaceType.PASSAGE
    else:
        door_space_type = {"STANDARD": SpaceType.DOOR_STANDARD,
                           "AUTOMATIC": SpaceType.DOOR_AUTOMATIC,
                           "LOCKED": SpaceType.DOOR_LOCKED,
                           "EMERGENCY": SpaceType.DOOR_EMERGENCY}.get(
                               (door_type_str or "").upper(), SpaceType.DOOR_STANDARD)

    # A door inherits floor/building only when BOTH endpoints agree (else null).
    floor_id = space_a.get("floor_id") if space_a.get("floor_id") == space_b.get("floor_id") else None
    building_id = space_a.get("building_id")
    if space_b.get("building_id") != building_id:
        building_id = None

    # Deterministic id: same unordered pair + type → same uuid5 → idempotent re-import.
    sorted_endpoints = sorted([conn.from_space_id, conn.to_space_id])
    door_id = f"door_{uuid.uuid5(uuid.NAMESPACE_URL, f'{ct_upper}:{sorted_endpoints[0]}:{sorted_endpoints[1]}').hex[:12]}"

    # GC any *other* door previously materialized between this pair.
    for stale_id in self.conn_repo.find_door_spaces_between(conn.from_space_id, conn.to_space_id):
        if stale_id != door_id:
            try: self.space_repo.delete_space(stale_id)
            except SpaceNotFound: pass
            self.postgis.delete_space(stale_id)

    # Create the door node (Neo4j) + enqueue its PostGIS mirror.
    door_space = self.space_repo.create_space(SpaceCreate(
        id=door_id, space_type=door_space_type, centroid_x=cx, centroid_y=cy,
        floor_id=floor_id, building_id=building_id, is_navigable=True,
        traversal_cost=compute_traversal_cost(door_space_type.value, None, None, None),
        # …other inherited fields (campus_id, org_id, global coords, metadata)…
    ))
    self.postgis.sync_space(build_space_sync_payload(door_space, self.campus_repo))

    # Four directed edges = bidirectional A↔door↔B traversal.
    self.conn_repo.create_connection(conn.from_space_id, door_id)
    self.conn_repo.create_connection(door_id, conn.from_space_id)
    self.conn_repo.create_connection(conn.to_space_id, door_id)
    self.conn_repo.create_connection(door_id, conn.to_space_id)
    self.postgis.sync_connection(from_space_id=conn.from_space_id, to_space_id=conn.to_space_id,
                                 door_space_id=door_id, connection_type=door_space_type.value,
                                 door_cx=cx, door_cy=cy, is_accessible=bool(getattr(conn, "is_accessible", True)))
```

### 4. `sync_outbox` — durable async Neo4j → PostGIS mirror

**What it does.** Implements the [outbox](#neo4j--postgis-sync--the-durable-outbox)
that decouples the two stores. `enqueue` persists one sync op as a
`:SyncOutbox` node right after the Neo4j write; the `OutboxWorker`
background task drains the queue in enqueue order, dispatches each op to
its `_apply_*` PostGIS writer, deletes on success, and reschedules with
capped exponential backoff on failure. Drain runs in RLS "service mode"
so rows survive the request scope that created them.

```python
def enqueue(op: str, payload: dict) -> bool:
    """Persist a sync op to the Neo4j outbox (called by every sync_* facade)."""
    if not settings.supabase_enable_sync:
        return False                                    # sync disabled → true no-op
    try:
        get_db().execute_write("""CREATE (o:SyncOutbox {
            id: $id, op: $op, payload: $payload, created_at: $created_at,
            attempts: 0, next_attempt_at: $created_at, last_error: null })""",
            {"id": str(uuid.uuid4()), "op": op,
             "payload": json.dumps(payload, default=str), "created_at": _utcnow_iso()})
        return True
    except Exception as exc:
        # The data write already committed — surface a dropped mirror loudly.
        logger.error("sync_outbox enqueue(%s) failed: %s", op, exc)
        return False


class OutboxWorker:
    def _drain_once(self, service) -> int:
        # Only rows whose backoff has elapsed; oldest first so campus→…→space replays in order.
        rows = self.db.execute("""MATCH (o:SyncOutbox) WHERE o.next_attempt_at <= $now
            RETURN o.id AS id, o.op AS op, o.payload AS payload, o.attempts AS attempts
            ORDER BY o.created_at ASC, o.id ASC LIMIT $limit""",
            {"now": _utcnow_iso(), "limit": settings.sync_outbox_batch_size})
        if not rows:
            return 0
        dispatch = _get_dispatch()                       # {op_name: lambda svc, payload: svc._apply_*(**payload)}
        # Force service mode so RLS accepts writes for ops whose request scope is gone.
        token = current_is_service.set(True); org_token = current_org_id.set(None)
        try:
            for row in rows:
                self._process(service, dispatch, row)
        finally:
            current_is_service.reset(token); current_org_id.reset(org_token)
        return len(rows)

    def _process(self, service, dispatch, row):
        handler = dispatch.get(row["op"])
        if handler is None:
            return self._mark_failure(row["id"], row.get("attempts") or 0, f"unknown op: {row['op']}")
        try:
            handler(service, json.loads(row["payload"]) if row.get("payload") else {})
        except Exception as exc:
            return self._mark_failure(row["id"], row.get("attempts") or 0, str(exc))
        self._delete(row["id"])                           # success → remove from queue

    def _mark_failure(self, outbox_id, attempts, error):
        new_attempts = attempts + 1
        max_attempts = settings.sync_outbox_max_attempts
        if max_attempts and new_attempts >= max_attempts:  # default 0 → retry forever
            self._delete(outbox_id); return
        shift = min(new_attempts, _BACKOFF_CAP_SHIFTS)     # cap at 2**6 = 64×
        delay = settings.sync_outbox_base_backoff_s * (2 ** shift)
        next_iso = datetime.fromtimestamp(
            datetime.now(timezone.utc).timestamp() + delay, tz=timezone.utc).isoformat()
        self.db.execute_write("""MATCH (o:SyncOutbox {id:$id})
            SET o.attempts=$attempts, o.next_attempt_at=$next_attempt_at, o.last_error=$last_error""",
            {"id": outbox_id, "attempts": new_attempts,
             "next_attempt_at": next_iso, "last_error": (error or "")[:_LAST_ERROR_MAX_LEN]})
```

### 5. `PostGISService.apply_rls_policies` — programmatic multi-tenant RLS

**What it does.** Installs Postgres Row-Level Security on every
org-scoped table so tenant isolation is enforced by the **database**, not
just by application `WHERE` clauses. For each table it enables RLS and
(re)creates a single `org_isolation` policy whose `USING`/`WITH CHECK`
predicate compares the row's `organization_id` to the per-session
`app.org_id` GUC the backend stamps from the trusted identity header —
with two escape hatches: a service-mode bypass (`app.is_service`, used by
the outbox worker) and an `is_public` read allowance for the
campus/building/floor tables. Called once at startup when
`AUTH_RLS_ENABLED=true`.

```python
_RLS_PUBLIC_READ_TABLES = ("campuses", "buildings", "floors")
_RLS_TABLES = ("campuses", "buildings", "floors", "imports", "landmarks",
               "wifi_fingerprints", "wifi_access_points")

def apply_rls_policies(self) -> bool:
    if not self.engine:
        return False
    try:
        with self.engine.begin() as conn:
            for table in self._RLS_TABLES:
                public_read = table in self._RLS_PUBLIC_READ_TABLES
                # Read predicate: service bypass OR same-tenant OR (public rows, read-only).
                using_clause = ("current_setting('app.is_service', true) = 'true' "
                                "OR organization_id = current_setting('app.org_id', true)")
                if public_read:
                    using_clause += " OR is_public = true"
                conn.execute(text(f'ALTER TABLE "{table}" ENABLE ROW LEVEL SECURITY'))
                conn.execute(text(f'DROP POLICY IF EXISTS org_isolation ON "{table}"'))
                conn.execute(text(f'''
                    CREATE POLICY org_isolation ON "{table}"
                        USING ({using_clause})
                        WITH CHECK (
                            current_setting('app.is_service', true) = 'true'
                            OR organization_id = current_setting('app.org_id', true)
                        )'''))   # WITH CHECK has NO is_public clause → writes are tenant-only
        return True
    except Exception as exc:
        print(f"[PostGISService] apply_rls_policies failed: {exc}")
        return False
```

### 6. `AuthService.login` — credential + MFA login state machine

**What it does.** The password-login entry point. It rate-limits *before*
touching the DB (so credential-stuffing can't burn bcrypt cycles),
verifies the password in constant-ish time, returns a deliberately
**generic 401** for every failure mode (wrong password, unknown email,
not-a-member) so nothing about account/membership existence leaks,
resolves the caller's org membership, and then **branches**: MFA-enabled
users get a short-lived *challenge token* (emailing an OTP first when the
method is email, and aborting if that send fails), while everyone else
gets a signed access JWT. Every outcome is written to the audit log.

```python
def login(self, email, password, organization_id, ip_address=None, user_agent=None):
    email = email.strip().lower()
    self._check_login_rate_limit(email)        # raises 429 before any DB / bcrypt work
    session = self._SessionLocal()
    try:
        user = session.query(AppUser).filter_by(email=email).first()
        if not user or not user.is_active or not verify_password(password, user.password_hash):
            self._record_login_attempt(email)  # only failures count toward the limit
            self._audit(session, action="login", success=False, subject_email=email,
                        subject_user_id=user.id if user else None,
                        ip_address=ip_address, user_agent=user_agent,
                        detail={"reason": "bad_credentials"})
            session.commit()
            raise AuthError("Invalid credentials", status_code=401)   # generic — no user enumeration

        # Resolve org scope: explicit id must match a membership; a lone
        # membership auto-selects; multiple memberships force the caller to choose.
        memberships = session.query(OrganizationMember).filter_by(user_id=user.id).all()
        chosen = None
        if organization_id:
            chosen = next((m for m in memberships if m.organization_id == organization_id), None)
            if not chosen:
                self._audit(session, action="login", success=False, subject_user_id=user.id,
                            subject_email=email, organization_id=organization_id,
                            ip_address=ip_address, user_agent=user_agent,
                            detail={"reason": "not_a_member"})
                session.commit()
                raise AuthError("Invalid credentials", status_code=401)  # same generic 401
        elif len(memberships) == 1:
            chosen = memberships[0]
        elif len(memberships) > 1:
            raise AuthError("Multiple organizations available; please specify organization_id", status_code=400)

        org_id = chosen.organization_id if chosen else None
        role_value = chosen.role.value if chosen else None
        all_org_ids = [m.organization_id for m in memberships]
        self._clear_login_attempts(email)      # good password → reset the limiter slate

        if user.mfa_enabled:
            method = user.mfa_method or "totp"
            otp_hash = None
            if method == "email":
                otp = _generate_email_otp()
                otp_hash = bcrypt.hashpw(otp.encode(), bcrypt.gensalt(rounds=settings.auth_bcrypt_rounds)).decode()
                if not _send_mfa_email_otp(user.email, otp):
                    raise AuthError("Could not send MFA email; try again later", status_code=503)
            self._audit(session, action="login_mfa_challenge", success=True, subject_user_id=user.id,
                        subject_email=email, organization_id=org_id, detail={"method": method})
            session.commit()
            # The email-OTP hash is sealed *inside* the challenge JWT — no server-side OTP store.
            challenge, exp = issue_mfa_challenge(user.id, org_id, otp_hash=otp_hash, method=method)
            return {"mfa_required": True, "mfa_method": method,
                    "challenge_token": challenge, "challenge_expires_at": exp.isoformat()}

        # No MFA → issue the access token directly.
        self._audit(session, action="login", success=True, subject_user_id=user.id,
                    subject_email=email, organization_id=org_id)
        session.commit()
        token, exp = issue_token(user.id, org_id, role_value, organization_ids=all_org_ids)
        return {"mfa_required": False,
                "user": {"id": user.id, "email": user.email, "full_name": user.full_name},
                "organization_id": org_id, "role": role_value,
                "token": token, "token_expires_at": exp.isoformat()}
    finally:
        session.close()
```

### Listings from the project report (`ariadne_final.pdf`)

The ten code listings printed in the project report, transcribed
verbatim (chapter 6). They are deliberately abridged "teaching" forms —
where one has since diverged from the live source, a note points to the
current implementation above.

#### Listing 6.1 — `ContentView.body` (Swift) — authentication gate

The root view's three-way branch: show a spinner until the auth probe
finishes, then either the login wall (when enforcement is on and the
user isn't authenticated) or the main tab bar.

```swift
var body: some View {
    Group {
        if !authService.didProbe {
            ZStack { Color.slate50.ignoresSafeArea(); ProgressView() }
        } else if authService.enforcementOn && !authService.isAuthenticated {
            LoginView()
        } else {
            mainTabs
        }
    }
}
```

#### Listing 6.2 — `LoginView` (Swift) — handling the authentication outcome

Calls the async login and branches on the result type: a plain success
falls through, while an MFA-enabled account stashes the returned
challenge token to drive the OTP screen.

```swift
let outcome = try await authService.login(
    email: trimmedEmail,
    password: password,
    organizationId: trimmedOrg
)
switch outcome {
case .authenticated:
    break
case .mfaRequired(let challengeToken, _):
    pendingChallenge = challengeToken
}
```

#### Listing 6.3 — `navigateToBuilding` (Swift) — navigation to a selected element

When a building is picked from the list, stash its world coordinate (if
known), id and name as a *pending* navigation, then switch to the floor-plan
tab which consumes them.

```swift
private func navigateToBuilding(_ b: BuildingDTO) {
    if let lat = b.origin_lat, let lng = b.origin_lng {
        mapNav.pendingBuildingCoordinate =
            CLLocationCoordinate2D(latitude: lat, longitude: lng)
    } else {
        mapNav.pendingBuildingCoordinate = nil
    }
    mapNav.pendingBuildingId = b.id
    mapNav.pendingBuildingName = b.name
    mapNav.selectedTab = .floorPlan
}
```

#### Listing 6.4 — `recomputeFloorIndex` (Swift) — barometric floor detection

Converts the barometer's relative altitude into an absolute user
elevation, picks the floor whose `elevationMeters` is closest, and only
switches floors once the new candidate beats the current one by more
than the hysteresis margin (1.5 m) — so a pause on a stair landing
doesn't flap the displayed floor.

```swift
private func recomputeFloorIndex(currentRelativeAltitude r: Double) {
    guard let r0 = baselineRelativeAltitude,
          let e0 = baselineFloorElevation else { return }
    let userElevation = e0 + (r - r0)
    let scored = floors.compactMap { f -> (FloorSummary, Double)? in
        guard let e = f.elevationMeters else { return nil }
        return (f, abs(e - userElevation))
    }
    guard let best = scored.min(by: { $0.1 < $1.1 }) else { return }
    if let cur = currentFloorIndex,
       let curFloor = floors.first(where: { $0.floorIndex == cur }),
       let curElevation = curFloor.elevationMeters {
        let curDelta = abs(curElevation - userElevation)
        if curDelta - best.1 < hysteresisMeters { return }   // 1.5 m
    }
    if best.0.floorIndex != currentFloorIndex {
        currentFloorIndex = best.0.floorIndex
    }
}
```

#### Listing 6.5 — RAG SQL query (vector + spatial)

The assistant's single-statement retrieval: a pgvector cosine score
(`<=>`) over `building_spaces`, optionally fenced to spaces within
`radius_m` of the user via an `ST_DWithin` predicate that is appended
only when a location is supplied.

```python
spatial_filter = ""
if user_lat and user_lng and radius_m:
    spatial_filter = """
        AND bs.geometry_global IS NOT NULL
        AND ST_DWithin(
            bs.geometry_global::geography,
            ST_SetSRID(ST_MakePoint(:user_lng, :user_lat), 4326)::geography,
            :radius_m)
    """

sql = text(f"""
    SELECT bs.display_name AS name, bs.space_type AS type,
           1.0 - (bs.embedding <=> CAST(:q AS vector)) AS score
    FROM building_spaces bs
    WHERE bs.campus_id = :campus_id
      AND bs.is_navigable = TRUE
      AND bs.embedding IS NOT NULL
      AND (:building_id IS NULL OR bs.building_id = :building_id)
      {spatial_filter}
    ORDER BY bs.embedding <=> CAST(:q AS vector)   -- cosine distance
    LIMIT :limit
""")
```

#### Listing 6.6 — Dijkstra implementation

The portable fallback router: an edge cost in seconds (centroid distance
÷ walking speed + a per-node traversal penalty) feeding a binary-heap
Dijkstra over the Cypher-loaded graph.

```python
_WALKING_SPEED_MS = 1.4

def _edge_weight(src: dict, dst: dict) -> float:
    """Edge cost in seconds: walking time between centroids + per-node penalty."""
    # local centroids only comparable on the same floor/building
    different_frame = (src["floor_index"] != dst["floor_index"]) or \
                      (src["building_id"] != dst["building_id"])
    if have_centroids and not different_frame:
        dist_m = math.hypot(src["centroid_x"] - dst["centroid_x"],
                            src["centroid_y"] - dst["centroid_y"])
    else:
        dist_m = 1.0                          # vertical hop: nominal distance
    penalty = float(dst.get("traversal_cost") or 0.0) \
              if dst["space_type"] in _CONNECTION_TYPES else 0.0
    return (dist_m / _WALKING_SPEED_MS) + penalty

# ... build adjacency from :CONNECTS_TO edges, then:
heap = [(0.0, from_id)]
while heap:
    d, u = heapq.heappop(heap)
    if u == to_id: break
    for v, w in adj.get(u, []):
        nd = d + w
        if nd < dist.get(v, math.inf):
            dist[v] = nd; prev[v] = u
            heapq.heappush(heap, (nd, v))
```

> **Note.** This is the report's original plain-Dijkstra form. The live
> code has since evolved into weighted **A\*** with an admissible
> geographic heuristic and door-aware (polygon-edge-snapped) edge
> weights — see [snippet 1](#1-python_dijkstrafind_path--weighted-a-routing-core).

#### Listing 6.7 — Routing endpoint logic

The FastAPI route: `from`/`to` plus the three preference flags as query
params, delegating to `NavigationService.get_route` and mapping the
domain errors onto a 404.

```python
@router.get("", response_model=Route)
def navigate(from_space_id: str = Query(..., alias="from"),
             to_space_id: str = Query(..., alias="to"),
             accessible_only: bool = Query(False),
             avoid_stairs: bool = Query(False),
             elevators_only: bool = Query(False),
             db: Database = Depends(get_db)):
    try:
        return NavigationService(db).get_route(
            from_space_id, to_space_id,
            accessible_only=accessible_only,
            avoid_stairs=avoid_stairs, elevators_only=elevators_only)
    except (SpaceNotFound, NavigationError) as e:
        raise HTTPException(status_code=404, detail=str(e))
```

#### Listing 6.8 — Three-tier routing fallback

The strategy that lets the same code run on Neo4j Community, Aura free,
and Enterprise: try GDS Dijkstra, fall back to the pure-Python weighted
Dijkstra (Listing 6.6), and finally to native unweighted `shortestPath`.

```python
# Tier 1: GDS Dijkstra
try:
    result = self.db.execute(_GDS_QUERY, {...})
    if result and result[0]["path_nodes"]:
        # reject if path violates accessibility / avoid-stairs filters
        if not violates:
            return {"path_nodes": ..., "total_cost": result[0]["totalCost"]}
except Exception as exc:
    print(f"[nav] GDS threw: {exc}; trying Python Dijkstra")

# Tier 2: pure-Python weighted Dijkstra (Aura free tier has no GDS)
try:
    py_result = python_dijkstra.find_path(self.db, from_id, to_id,
                                          accessible_only=accessible_only,
                                          excluded_types=excluded)
    if py_result is not None:
        return py_result
except Exception as exc:
    print(f"[nav] Python Dijkstra threw: {exc}; falling back to native")

# Tier 3: Neo4j built-in unweighted shortestPath (last-resort safety net)
result = self.db.execute(_NATIVE_QUERY, {...})
```

#### Listing 6.9 — Two pooled ngrok endpoints in front of the gateway

The report's development tunnel: two ngrok agents pointed at the same
public URL with pooling enabled, so traffic fails over between the two
local ports.

```bash
# Create two endpoints with the same URL forwarding
ngrok http 8080 --url https://your-domain.ngrok.app --pooling-enabled true
ngrok http 9090 --url https://your-domain.ngrok.app --pooling-enabled true
```

> **Note.** This is the report's tunnel choice; the current dev setup
> uses **Tailscale Funnel** (see [Deployment](#deployment) / the defence
> notes) — same role, no bandwidth cap.

#### Listing 6.10 — Trimmed `k8s-config.yaml`

The shape of the Kubernetes manifest: one `Secret` of shared
credentials and one `ConfigMap` of cross-service URLs mounted into every
pod, then one `Deployment` (+ matching `Service`) per microservice.

```yaml
# Shared credentials, mounted into every pod
apiVersion: v1
kind: Secret
metadata: { name: ariadne-secrets, namespace: ariadne }
type: Opaque
stringData:
  API_SECRET: "<gateway api key>"
  AUTH_JWT_SECRET: "<≥ 32-byte signing key>"
  NEO4J_PASSWORD: "<password>"
  SUPABASE_DB_URL: "postgresql://...:5432/postgres"
---
# Non-secret cross-service URLs
apiVersion: v1
kind: ConfigMap
metadata: { name: ariadne-config, namespace: ariadne }
data:
  BACKEND_URL: "http://spatial-backend:8000"
  ASSISTANT_URL: "http://assistant:8001"
  ML_VISION_URL: "http://ml-vision:8000"
---
# One service = a Deployment (replicas, image, env) + a Service (endpoint)
apiVersion: apps/v1
kind: Deployment
metadata: { name: assistant, namespace: ariadne }
spec:
  replicas: 2
  selector: { matchLabels: { app: assistant } }
  template:
    # ... (pod template trimmed in the report)
```
