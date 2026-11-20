# Implementation Plan: CineRole Core

## Overview

17 tasks covering all 12 packages. Tasks are ordered by dependency: infrastructure and shared packages first (T1–T3), then backend services (T4–T12), then frontend (T13–T17). The ingestion CLI (T15) is independent and can run in parallel with any wave.

## Task Dependency Graph

```
T1 (i18n)
T2 (shared-ui types)
T3 (api-client interfaces)
  └─► T4 (gateway auth)
        └─► T5 (room-service: create/join/lobby)
              └─► T6 (game-engine: role assignment)
                    └─► T7 (rag-service: pgvector + retrieval)
                          └─► T8 (npc-service: NPC response)
                                └─► T9 (game-engine: timer + turns)
                                      └─► T10 (game-engine: vote + end-game)
                                            └─► T11 (npc-service: scoring)
T12 (transport-service)
  └─► T13 (web-app: 3D view)
        └─► T14 (mobile-app: WebView 3D)
T15 (ingestion-service CLI)
T1, T2, T3, T5, T6, T8, T9, T10 ──► T16 (web-app: lobby + game UI)
T16 ──► T17 (mobile-app: lobby + game UI)
```

```json
{
  "waves": [
    { "wave": 1, "tasks": ["1", "2", "3", "15"], "description": "Shared packages and ingestion CLI — no dependencies" },
    { "wave": 2, "tasks": ["4"], "description": "Gateway — depends on api-client types from T3" },
    { "wave": 3, "tasks": ["5", "7", "12"], "description": "room-service, rag-service, transport-service — depend on gateway" },
    { "wave": 4, "tasks": ["6"], "description": "game-engine role assignment — depends on room-service and rag-service" },
    { "wave": 5, "tasks": ["8"], "description": "npc-service NPC response — depends on game-engine and rag-service" },
    { "wave": 6, "tasks": ["9"], "description": "game-engine turns and timer — depends on npc-service" },
    { "wave": 7, "tasks": ["10"], "description": "game-engine vote and end-game — depends on game timer" },
    { "wave": 8, "tasks": ["11"], "description": "npc-service scoring — depends on end-game trigger" },
    { "wave": 9, "tasks": ["13"], "description": "web-app 3D view — depends on transport-service" },
    { "wave": 10, "tasks": ["14"], "description": "mobile-app WebView 3D — depends on web-app 3D" },
    { "wave": 11, "tasks": ["16"], "description": "web-app UI screens — depends on all backend services" },
    { "wave": 12, "tasks": ["17"], "description": "mobile-app UI screens — depends on web-app screens" }
  ]
}
```

---

## Tasks

- [ ] 1. Package `i18n`: bundle locale IT e typed key catalogue
  - Packages: `i18n`
  - Create `packages/i18n/src/locales/it.json` with all UI string keys (lobby, game, vote, score, errors)
  - Create `packages/i18n/src/index.ts` that initialises i18next with Italian as default and exports a typed `t()` wrapper
  - Export a `LocaleKeys` TypeScript type derived from the Italian bundle so missing keys cause compile-time errors
  - Fallback: if a requested locale bundle is absent, `t()` falls back to Italian silently
  - _Requirements: 14.1, 14.2, 14.3, 14.4_

- [ ] 2. Package `shared-ui`: shared TypeScript types and base presentational components
  - Packages: `shared-ui`, `i18n`
  - Define shared types used by both frontends: `Player`, `Room`, `ChatMessage`, `PlayerScore`, `GameOutcome`, `RoomState`
  - Implement presentational components: `NicknameInput`, `RoomCodeInput`, `PlayerList`, `ReadyBadge`, `ChatBubble`, `ScoreCard`, `ErrorBanner`
  - Components consume the `t()` wrapper from `i18n`; no business logic inside components
  - _Requirements: 14.1_

- [ ] 3. Package `api-client`: typed HTTP, WebSocket, and WebTransport clients
  - Packages: `api-client`
  - Implement `HttpClient` with `join()` and `createRoom()` methods; request/response types derived from the gateway API contract in design.md
  - Implement `RoomClient` over WebSocket: `connect()`, `on<E>()`, `send()`, `disconnect()`; handle reconnect with exponential backoff (max 3 attempts)
  - Implement `TransportClient` over WebTransport with fallback to WebSocket when `typeof WebTransport === 'undefined'`
  - All three clients export typed interfaces; concrete implementations are internal to the package
  - _Requirements: 8.3, 9.1, 12.4_

- [ ] 4. Package `gateway`: auth guest, rate limiting, routing
  - Packages: `gateway`
  - Implement `POST /auth/join`: validate nickname (1–20 chars), look up room existence via `room-service`, issue RS256 JWT (`sub=playerId`, `roomId`, `nickname`, `exp=now+15min`)
  - Implement `POST /rooms`: proxy to `room-service` create endpoint (authenticated)
  - Implement `GET /rooms/:code`: proxy to `room-service` room info endpoint
  - Enforce sliding-window rate limit of 10 req/min per IP on `/auth/join`; return HTTP 429 on excess
  - Proxy WebSocket upgrade to `room-service`; document that WebTransport connections go directly to `transport-service`
  - Return structured errors `{ code, message }` for all 4xx/5xx responses
  - _Requirements: 1.1, 1.5, 2.2_

- [ ] 5. Package `room-service`: room lifecycle, lobby, WebSocket broadcast, timer
  - Packages: `room-service`, `gateway`
  - Implement in-memory `RoomRegistry` (Map<roomCode, Room>) and `PlayerSessions` (Map<playerId, WebSocket>)
  - `createRoom`: generate unique 6-char uppercase alphanumeric code (retry up to 5 times on collision); persist Room in registry; return roomCode + inviteLink
  - `joinRoom`: enforce max 5 players and LOBBY state; reject duplicate nicknames; add player to room; broadcast `room:player_list` within 500ms
  - `setReady`: toggle ready state; broadcast `room:player_list` within 500ms; re-evaluate start eligibility
  - `startGame`: validate 2–5 players all ready; call `game-engine POST /internal/games/start`; transition room to SCENE_SELECTION
  - Host succession: when host disconnects in lobby, assign host to earliest-joined remaining player
  - In-game disconnect: mark player `connected: false`, retain avatar as silent NPC; broadcast updated player list; if only 1 player remains trigger TRAITOR_VICTORY/ALL_ABANDONED
  - Reconnect: restore player state from room snapshot when same JWT re-connects; send `room:full_state`
  - `discardRoom`: clear room from registry when player count reaches 0 or game ends
  - _Requirements: 1.2, 1.3, 1.4, 2.1, 2.3, 2.4, 3.1, 3.2, 3.3, 3.4, 3.5, 7.1, 7.2, 7.4, 12.1, 12.3, 12.4_

- [ ] 6. Package `game-engine`: role assignment and room state transitions
  - Packages: `game-engine`, `room-service`
  - Implement `POST /internal/games/start`: receive room + players + scene from `room-service`; call `rag-service GET /internal/scenes/candidates`; on candidates returned call `npc-service POST /internal/npc/scene-select` to pick one scene; on failure emit `scene-selection-failed` back to `room-service`
  - Assign roles: Fisher-Yates shuffle players; assign scene.roles[i] → players[i] (1-to-1); uniform-random designate one as traitor; deliver private role payloads per player via `room-service` session channels; deliver `PublicRole[]` (without isTraitor) to `npc-service`
  - Transition room to IN_GAME via `room-service POST /internal/rooms/:roomId/roles-assigned`
  - Abort and return room to LOBBY if scene role count < player count
  - _Requirements: 4.1, 4.2, 4.5, 5.1, 5.2, 5.3, 5.4, 5.5, 5.6_

- [ ] 7. Package `rag-service`: pgvector schema, retrieval endpoint, scene candidates
  - Packages: `rag-service`
  - Create PostgreSQL schema: `scenes` table with `id UUID`, `film_title TEXT`, `scene_title TEXT`, `role_count INT`, `metadata JSONB`, `embedding VECTOR(1536)`; create ivfflat index with lists=50
  - Implement `GET /internal/scenes/candidates?playerCount=<n>`: exact-match query first; if 0 results retry with `role_count BETWEEN n-1 AND n+1`; return max 10 scenes ordered by cosine similarity; respond within 1s in 95% of calls (measure with p95 logging)
  - Expose a typed `SceneRepository` class with `findCandidates(playerCount: number): Promise<Scene[]>`
  - _Requirements: 4.1, 4.3, 4.4, 4.5, 15.5_

- [ ] 8. Package `npc-service`: NPC response generation and scene selection
  - Packages: `npc-service`, `game-engine`, `rag-service`
  - Implement `POST /internal/npc/scene-select`: receive candidate scenes + player count; call GPT-4o to pick one; timeout 10s with 2 retries (backoff 1s, 2s); on failure emit `scene-selection-failed`
  - Implement `POST /internal/npc/respond`: build GPT-4o SYSTEM prompt (scene context, public roles, locale, no traitor identity); apply simulated delay 1000–2000ms before returning; timeout 10s; on failure return localised fallback string `"… [silenzio]"`
  - NPC prompt must never include `isTraitor` or `sabotagObjective`; enforce this with a type-level constraint (`PublicRole` type)
  - _Requirements: 4.2, 4.5, 6.1, 6.2, 6.3, 6.4, 10.2, 14.5_

- [ ] 9. Package `game-engine`: turn processing and game timer
  - Packages: `game-engine`, `room-service`
  - Implement `POST /internal/games/:roomId/chat`: validate message (connected player, length ≤ 500 chars); forward to `npc-service /respond`; broadcast player message + NPC response via `room-service` within 200ms of receipt
  - Implement `GameTimer`: start 15-minute countdown on IN_GAME entry; broadcast `room:timer_update` every 30s; on expiry transition to END_GAME with TRAITOR_VICTORY/TIMER_EXPIRED and trigger scoring
  - Stop timer when room transitions out of IN_GAME for any reason
  - _Requirements: 6.1, 6.5, 6.6, 7.1, 7.2, 7.3, 7.4_

- [ ] 10. Package `game-engine`: traitor vote, prop interactions, end-game
  - Packages: `game-engine`, `room-service`
  - Implement `POST /internal/games/:roomId/vote`: enforce one-vote-per-game invariant (reject if `traitorVote` already set); open 60s nomination window; broadcast `room:vote_started`; collect nominations; on majority or window expiry resolve vote; correct → END_GAME TRAITOR_DEFEATED; incorrect → game continues; expired → game continues; all three outcomes consume the one permitted vote
  - Implement prop interaction: receive `prop:interact` event from room-service; validate prop belongs to active scene; inject contextual chat action on behalf of player; record `PropInteraction` in GameSession
  - On END_GAME: call `npc-service POST /internal/npc/score`; broadcast `room:game_end` with outcome + scores + traitor identity; if scoring takes > 5s send `room:game_end` with placeholder scores then `room:scores_updated` when scores arrive (max 30s retry)
  - Scores must not appear in any broadcast before END_GAME (Property 8)
  - _Requirements: 8.6, 10.1, 11.1, 11.2, 11.3, 11.4, 11.5, 11.6, 13.4, 13.5_

- [ ] 11. Package `npc-service`: end-game scoring
  - Packages: `npc-service`, `game-engine`
  - Implement `POST /internal/npc/score`: build scoring prompt (full chat history, public roles, traitor identity, prop interactions, vote outcome, scoring rubric: 40pt role consistency + 30pt prop usage + 30pt traitor detection); call GPT-4o; timeout 10s with 2 retries; on failure return all-zero placeholder scores with a flag `partial: true`
  - Response schema: `{ scores: Array<{ playerId, score: 0–100, rationale: string }>, partial?: boolean }`
  - _Requirements: 13.1, 13.2, 13.3_

- [ ] 12. Package `transport-service`: WebTransport server, avatar positions, walkable validation
  - Packages: `transport-service`, `room-service`
  - Implement WebTransport server on HTTP/3; authenticate connections via JWT query param; associate session to roomId
  - On datagram receipt: deserialise MessagePack `AvatarPosition`; validate position is inside `walkablePolygon` using ray-casting point-in-polygon (O(n) on polygon vertices); if outside, discard and send error datagram to sender only; if valid, update last-known position and broadcast to all other sessions in the room
  - On session close: retain last-known position; include it in subsequent broadcasts as a static position until room ends
  - _Requirements: 8.3, 8.4, 9.1, 9.2, 9.3, 9.4_

- [ ] 13. Package `web-app`: Three.js isometric 3D view and avatar movement
  - Packages: `web-app`, `api-client`
  - Set up Three.js scene with orthographic camera at 45°/30° isometric angle; load scene background GLTF asset by `backgroundAssetKey`
  - Render player avatars as GLTF models at their `avatarPosition`; render props with distinct visual treatment; show only props belonging to the active scene
  - On floor click: raycast against y=0 plane to compute target Vector3; validate client-side that the point is within the walkable polygon; send position via `TransportClient.sendPosition()`; animate avatar locally with linear interpolation
  - On `onPositionUpdate`: update the corresponding avatar's position in the Three.js scene
  - On prop click: dispatch `prop:interact` event via `RoomClient.send()`
  - _Requirements: 8.1, 8.3, 8.4, 8.5, 8.6, 9.1_

- [ ] 14. Package `mobile-app`: Three.js 3D view in Expo WebView
  - Packages: `mobile-app`, `api-client`
  - Embed the web-app Three.js scene inside an Expo `WebView` component; pass room state (scene, players, props) into the WebView via `injectedJavaScript` on mount and updates
  - Bridge position updates from WebView to native via `window.postMessage`; forward to `TransportClient` from the React Native layer
  - Bridge prop-click events from WebView to native; forward to `RoomClient`
  - Isolate all bridge logic in a single `IsometricBridgeClient` class inside `api-client` mobile entry-point, so the WebView can be swapped for react-three-fiber without touching components
  - _Requirements: 8.2, 8.3, 8.4, 8.6_

- [ ] 15. Package `ingestion-service`: offline CLI for scene ingestion
  - Packages: `ingestion-service`, `rag-service`
  - Implement CLI entry point `cinerole-ingest --input <path>` (no HTTP server, no production traffic)
  - Parse and validate input JSON against `SceneDefinition` schema using zod; on validation failure print descriptive error and exit non-zero without writing to the database
  - Generate vector embedding for `narrativeContext + roles summary` using OpenAI `text-embedding-3-small`
  - `INSERT INTO scenes ... ON CONFLICT (id) DO UPDATE` to support re-ingestion
  - Print per-scene result (OK / SKIPPED / ERROR) to stdout; exit 0 only if all scenes succeeded
  - _Requirements: 15.1, 15.2, 15.3, 15.4, 15.5_

- [ ] 16. Package `web-app`: lobby, game, and score screens
  - Packages: `web-app`, `shared-ui`, `api-client`, `i18n`
  - Implement `LobbyScreen`: nickname input + room code input → call `HttpClient.join()`; display `PlayerList` with ready badges; host sees "Start Game" button enabled only when 2–5 players all ready; handle errors (room full, game in progress, nickname taken) via `ErrorBanner`
  - Implement room creation flow: `HttpClient.createRoom()` → display room code + copy invite link button
  - Implement `GameScreen`: chat panel (input + `ChatBubble` list); player list sidebar; timer display; "Vote" button (enabled once, opens nominee selector); prop interaction via click events from 3D view
  - Implement `ScoreScreen`: show `ScoreCard` for each player; reveal traitor identity; show game outcome; support placeholder state while scores load
  - Connect all screens to `RoomClient` events; use logic classes `RoomManager`, `GameStateManager`, `VoteManager` (framework-agnostic, no React imports)
  - All strings from `i18n`; no hardcoded text
  - _Requirements: 1.1, 2.1, 3.1, 3.2, 3.3, 6.5, 7.1, 7.2, 11.3, 13.4, 13.5, 14.1_

- [ ] 17. Package `mobile-app`: lobby, game, and score screens (React Native)
  - Packages: `mobile-app`, `shared-ui`, `api-client`, `i18n`
  - Mirror the screen structure of task 16 using React Native components; reuse all shared-ui components adapted to RN primitives
  - Replace web-specific input handling (click) with touch events; ensure tap targets meet minimum 44×44pt
  - Integrate `IsometricBridgeClient` from task 14 into `GameScreen`
  - All strings from `i18n`; no hardcoded text
  - _Requirements: 8.2, 14.1_

## Notes

- Tasks T1, T2, T3, and T15 have no inter-task dependencies and can be developed in parallel.
- T6 and T7 can be developed in parallel within wave 3/4 since rag-service has no dependency on game-engine.
- T12 (transport-service) is independent of the game logic chain and can be developed in parallel from wave 3.
- Each task should be committed with a property-based test covering the correctness property it implements (see design.md Correctness Properties).
- The `PublicRole` type constraint (no `isTraitor`, no `sabotagObjective`) must be enforced at compile time before T8 is merged; this is the implementation guard for Property 2.
- State machine transitions in room-service must be guarded by the finalità dell'end-game invariant (Property 5) from the moment T5 is implemented.
