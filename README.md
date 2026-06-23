# Ariadne

**Adaptive Routing & Indoor Autonomous Directional Navigation Engine** — an
indoor-navigation platform that takes you the last mile GPS can't: from the
building door to the exact room.

It pairs a **Neo4j** graph of buildings/floors/rooms/doors (the source of
truth) with a **PostGIS + pgvector** geometry-and-vector mirror, fronts them
with a **service-oriented backend** behind one secured gateway, and adds an
**LLM assistant** (retrieval over the graph, with turn-by-turn directions),
**two computer-vision services** (real-time landmark recognition + batch room
labelling), a web **mapmaker**, a **CAD→graph data pipeline**, and **native
iOS and Android clients**.

For the in-depth architecture read [COMPLETE_ARCHITECTURE.md](COMPLETE_ARCHITECTURE.md);
for the design rationale + maths read [defence.md](defence.md); per-module
detail lives in each module's own `README.md`.

## Modules

| Path | Module | What it is |
| ---- | ------ | ---------- |
| [`aau-sw8-spatial-backend/`][backend]    | Spatial backend  | Compose stack: gateway, graph+geometry API, routing, auth+MFA, assistant (LLM+RAG), image pipeline, internal email relay, mapmaker |
| [`aau-sw8-ml-vision/`][mlvision]         | Real-time vision | ORB landmark recognition → YOLO26 ONNX detection over a WebSocket; serves per-facility model artifacts (ONNX / CoreML / TFLite) |
| [`aau-sw8-ios/`][ios]                    | iOS client       | SwiftUI/MVVM: floor plan, navigation, assistant, live camera + landmark snap, barometric floor, room-photo upload |
| [`aau-sw8-android/`][android]            | Android client   | Jetpack Compose: same five tabs; **Android-exclusive** WiFi RSSI/RTT positioning, barometer, camera streaming, osmdroid maps |
| [`aau-sw8-indoor-data-pipeline/`][datap] | Data pipeline    | Stateless **DXF/DWG → canonical `MapImportSchema` JSON** converter (two-stage: normalise, then apply ARIADNE rules) |

[backend]: aau-sw8-spatial-backend/README.md
[mlvision]: aau-sw8-ml-vision/README.md
[ios]: aau-sw8-ios/README.md
[android]: aau-sw8-android/
[datap]: aau-sw8-indoor-data-pipeline/README.md

## Capabilities

- **Hybrid positioning** — no single signal is trusted. GPS → which *building*;
  barometer → which *floor* (relative altitude + 1.5 m hysteresis); camera ORB
  landmark → exact *room*; WiFi RSSI/RTT fingerprinting → *area* (Android only —
  iOS blocks raw WiFi). A forced/landmark snap overrides GPS for "where am I".
- **Routing** — weighted Dijkstra over the graph (edge weight = walking time +
  per-mode transition penalty). Three tiers for portability: Neo4j **GDS**
  Dijkstra → pure-Python heap Dijkstra → native `shortestPath`. Door-aware
  weights; the rendered polyline is built server-side and projected to lat/lng.
- **Assistant (LLM + RAG)** — **deterministic-first**: 11 regex intent
  classifiers answer from the graph in 50–300 ms (incl. turn-by-turn
  *directions* to a room); only open-ended queries fall back to RAG (MiniLM
  384-d embedding → pgvector cosine + `ST_DWithin`, building-scoped) and the
  offline `SmolLM2-360M`. Refuses below a 0.30 score — no invented rooms.
- **Computer vision** — real-time service runs **ORB landmark matching first**
  (a hit snaps the user), **YOLO/ONNX detection as fallback**; the image
  pipeline batch-summarises rooms (YOLO11 object counts + OCR + OpenCLIP 512-d
  embeddings) for similarity search.
- **Security** — `X-Api-Key` on every request + JWT (HS256) on user actions;
  gateway-stamped trusted identity headers; bcrypt; login rate-limit + generic
  401; MFA (TOTP primary, email-OTP fallback); PostGIS row-level security;
  append-only audit log; TLS at the edge.

## Quick start

```bash
cd aau-sw8-spatial-backend
docker compose up -d --build
```

This brings up the gateway on `:8080` plus its upstreams — `backend`,
`assistant`, `image_pipeline`, the internal `email` relay — and the static
`frontend` mapmaker on `:3000`. The middleware enforces `X-Api-Key` on every
public route. (A reachable Neo4j and, for the full feature set, a PostGIS /
Supabase instance are expected — see the backend README for env vars.)

Bring up the real-time vision service alongside it (separate repo):

```bash
cd ../aau-sw8-ml-vision
docker build -t aau-sw8-ml-vision .
docker run --rm -p 8010:8000 \
  -e FACILITY_ID=default_facility \
  -e MODEL_ONNX_URL=https://huggingface.co/zwh20081/yolo26-onnx/resolve/main/yolo26n.onnx \
  -e MODEL_DIR=/app/models \
  aau-sw8-ml-vision
```

Mobile clients:

- **iOS** — open [aau-sw8-ios/](aau-sw8-ios/) in Xcode, create `AppSecrets.swift` with `backendURL` + `apiSecret`, run on a real device (camera + WebSocket need hardware).
- **Android** — open [aau-sw8-android/](aau-sw8-android/) in Android Studio, set the gateway URL + key in `local.properties`, run.

## Two-store invariant

Neo4j is the **graph of truth** — which spaces exist, what connects to what
(doors/lifts/stairs are nodes too), space types, and the 384-d embeddings.
PostGIS is the **geometry + vector mirror** — WGS84 polygons, centroids,
spatial indexes, the same embeddings, plus all relational data a graph is bad
at (users, audit log, WiFi fingerprints, landmarks). Writes go **Neo4j first**,
then sync PostGIS in the same HTTP request; reads that need geometry prefer
PostGIS and fall back to Neo4j. See [COMPLETE_ARCHITECTURE.md][arch] for the
label model, door-node pattern, and PostGIS schema.

[arch]: COMPLETE_ARCHITECTURE.md

## Public surface

The middleware on `:8080` is the only public port; everything else is on the
internal Docker network. Every request needs `X-Api-Key`; the WS handshake
checks the same key, and mutating routes also require a JWT bearer.

| Prefix | Reaches |
| ------ | ------- |
| `/api/v1/auth/*`              | backend (signup/login/MFA) |
| `/api/v1/assistant/*`         | assistant (chat + directions) |
| `/api/v1/room-summary/*`      | image_pipeline |
| `/api/v1/ml-vision/*`         | ml_vision |
| `WS /api/v1/ml-vision/ws/...` | ml_vision (bridged) |
| `/api/v1/mobile/*`            | middleware-owned (lightweight map download) |
| `/api/v1/*`                   | backend (catch-all: spaces, navigation, landmarks, import/export) |

## Deployment

Docker Compose is the dev source of truth; the same Dockerfiles deploy to
**Kubernetes** in prod (manifests in [`k8s/`](k8s/), notes in
[KUBERNETES.txt](KUBERNETES.txt)). HPAs on the traffic-varying pods (gateway,
assistant, ml-vision); GPU is pinned and time-sliced. During development the
local cluster is exposed via a **Tailscale + ngrok** tunnel so mobile clients
reach a stable public URL.

## Results (executed, § 8 of the report)

| What | Result |
| ---- | ------ |
| LLM generation (median) | 978 ms GPU · 2770 ms CPU (**2.8×**) |
| Deterministic assistant query (end-to-end) | ~151 ms |
| LLM/RAG query (large context) | ~7.9 s |
| Throughput @ 100 concurrent | backend `/health` 108 req/s · gateway 56 · deterministic assistant 26 · LLM/RAG 0.1 (GIL-bound) · **0 failures** on health + deterministic paths |
| Routing | Entrance Hallway → Door → Bathroom: 3 hops, **7.62 s** (weighted Dijkstra) |
| Dual-store mirror integrity | spaces **66/66**, floors **2/2** |
| Embedder separation | paraphrases 0.65–0.85, unrelated 0.14–0.29 (0.30 hallucination guard) |
| Positioning (field) | landmark recognition most reliable; WiFi Android-only (iOS blocks it); barometer a good floor-transition signal |

## Documentation

| File | What |
| ---- | ---- |
| [COMPLETE_ARCHITECTURE.md](COMPLETE_ARCHITECTURE.md) | Full architecture deep-dive |
| [defence.md](defence.md)                             | Design-decision defence + SOTA comparison + rendered maths appendix + examiner Q&A |
| [ariadne_final_speech.md](ariadne_final_speech.md)   | Presentation speaker script (per-member, ~45 min) |
| [KUBERNETES.txt](KUBERNETES.txt) · [SETUP.txt](SETUP.txt) | Deployment & setup notes |
| [diagrams/](diagrams/)                               | Architecture / Dijkstra / testing diagrams (PNG) |

## Repository layout

```
ariadne/
├── README.md                       this file
├── COMPLETE_ARCHITECTURE.md        architecture deep dive
├── defence.md                      design rationale, SOTA, maths appendix
├── ariadne_final_speech.md         presentation script
├── diagrams/                       generated architecture & results diagrams
├── aau-sw8-spatial-backend/        compose stack (gateway + services + mapmaker)
├── aau-sw8-ml-vision/              standalone real-time vision (ORB + YOLO26 ONNX)
├── aau-sw8-ios/                    SwiftUI client
├── aau-sw8-android/                Jetpack Compose client
├── aau-sw8-indoor-data-pipeline/   CAD (DXF/DWG) → MapImportSchema converter
├── db_setup/                       one-shot DB bootstrap
└── k8s/                            Kubernetes manifests (see KUBERNETES.txt)
```

## Team

Group **SW82E25** · Software, Aalborg University · Spring 2026 · Supervisor: **Hua Lu**

| Member | Focus |
| ------ | ----- |
| **Jinpeng Zhang** | Team / technical lead · system architecture · iOS client · Kubernetes & tunneling |
| **Zsombor Mátyás Kispéter** | Spatial backend · databases · mapmaker · system integration |
| **Rubens Rissi Onzi** | Computer vision · ML-Vision · indoor data pipeline |
| **Rupesh Parajuli** | Android client · mobile ↔ backend · assistant (RAG) integration |


## Official GitHub Repositories

- https://github.com/Jimpoz/aau-sw8-spatial-backend
- https://github.com/Jimpoz/aau-sw8-android
- https://github.com/Jimpoz/aau-sw8-ios
- https://github.com/Jimpoz/aau-sw8-ml-vision
- https://github.com/Jimpoz/aau-sw8-indoor-data-pipeline