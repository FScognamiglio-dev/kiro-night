# Design Document — CineRole Core

## Overview

CineRole è un'applicazione multiplayer di role-play ambientata in scene di film. L'architettura è a microservizi su Kubernetes, con due frontend (React web + React Native mobile) che comunicano via API Gateway. I protocolli real-time sono differenziati per natura del dato: WebSocket per la chat (messaggi ordinati, tolleranti a qualche decina di ms), WebTransport per il movimento degli avatar (latenza < 100 ms, perdita pacchetti accettabile).

---

## Architecture

### System Context

```
┌─────────────────────────────────────────────────────────────┐
│  Clients                                                     │
│  ┌──────────────┐    ┌──────────────────┐                   │
│  │   web-app    │    │   mobile-app     │                   │
│  │  React 18    │    │  React Native    │                   │
│  │  Three.js    │    │  Expo            │                   │
│  └──────┬───────┘    └────────┬─────────┘                   │
└─────────┼────────────────────┼─────────────────────────────┘
          │  HTTPS / WSS / H3  │
          ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│  gateway  (API Gateway)                                      │
│  • Routing  • Auth guest (JWT)  • Rate limiting             │
└──────┬──────────────┬──────────────────┬────────────────────┘
       │              │                  │
       ▼              ▼                  ▼
┌──────────────┐ ┌──────────────┐ ┌───────────────────┐
│ room-service │ │game-engine   │ │ transport-service │
│  WebSocket   │ │              │ │  WebTransport H3  │
└──────┬───────┘ └──────┬───────┘ └───────────────────┘
       │                │
       ▼                ▼
┌──────────────────────────────┐
│        npc-service           │
│  GPT-4o  •  scoring          │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│        rag-service           │
│  pgvector  •  retrieval      │
└──────────────────────────────┘
```

### Package Responsibilities (riepilogo)

| Package | Responsabilità primaria |
|---|---|
| `web-app` | SPA React 18: lobby, stanza, chat, vista 3D Three.js |
| `mobile-app` | React Native Expo: parità funzionale con web-app |
| `shared-ui` | Componenti presentazionali riusabili (nessuna logica di business) |
| `api-client` | Client tipizzato HTTP/WS/WebTransport; unico punto di accesso ai servizi |
| `i18n` | Bundle locale IT (default) + typed key catalogue |
| `gateway` | Routing, autenticazione guest JWT, rate limiting |
| `room-service` | Ciclo di vita stanza, lobby, WebSocket broadcast, timer |
| `game-engine` | Turni, assegnazione ruoli, voto traditore, transizioni di stato |
| `npc-service` | Chiamate GPT-4o, ritardo simulato, calcolo punteggio |
| `rag-service` | Retrieval scene con pgvector, selezione candidati |
| `transport-service` | WebTransport H3, posizioni avatar, broadcast in-room |
| `ingestion-service` | CLI offline: parsing scene, generazione embedding, scrittura pgvector |

---

## Data Models

### Scene

```typescript
interface Scene {
  id: string;                    // UUID
  filmTitle: string;
  sceneTitle: string;
  narrativeContext: string;      // testo per il prompt NPC
  roles: Role[];                 // lunghezza: 2–5
  props: Prop[];
  layoutData: SceneLayout;       // walkable area + spawn points
  embedding: number[];           // vettore pgvector (generato da ingestion)
}

interface Role {
  id: string;
  name: string;
  description: string;
  objectives: string[];
  isTraitor: false;             // tutte le role normali
}

interface TraitorRole extends Role {
  isTraitor: true;
  sabotagObjective: string;     // visibile solo al giocatore assegnato
}

interface Prop {
  id: string;
  name: string;
  description: string;
  interactionType: 'pickup' | 'examine' | 'use';
  position: Vector3;
}

interface SceneLayout {
  walkablePolygon: Vector3[];   // poligono convesso che delimita il movimento
  spawnPoints: Vector3[];        // uno per ruolo
  backgroundAssetKey: string;   // chiave per caricare l'asset 3D
}
```

### Room State Machine

```
LOBBY ──[start: 2≤players≤5, all ready]──► SCENE_SELECTION
  │                                               │
  │                                 [scene assigned, roles delivered]
  │                                               │
  │                                               ▼
  │                                           IN_GAME ──[timer=0 | vote correct]──► END_GAME
  │                                               │
  │                                    [all players left]
  │                                               │
  └──────────────────────────────────────────► DISCARDED
```

```typescript
type RoomState = 'LOBBY' | 'SCENE_SELECTION' | 'IN_GAME' | 'END_GAME' | 'DISCARDED';

interface Room {
  id: string;
  code: string;                  // 6 caratteri uppercase alfanumerici
  inviteLink: string;
  hostPlayerId: string;
  players: Map<string, Player>;  // playerId → Player
  state: RoomState;
  gameSession?: GameSession;     // presente solo in IN_GAME / END_GAME
  createdAt: number;             // epoch ms
}

interface Player {
  id: string;                    // JWT sub
  nickname: string;
  ready: boolean;
  connected: boolean;
  avatarPosition: Vector3;
  role?: AssignedRole;           // assegnato in SCENE_SELECTION
}

interface AssignedRole {
  roleId: string;
  isTraitor: boolean;
  sabotagObjective?: string;     // presente solo se isTraitor = true
}
```

### Game Session

```typescript
interface GameSession {
  sceneId: string;
  startedAt: number;
  timerDurationMs: number;       // 15 * 60 * 1000
  chatHistory: ChatMessage[];
  propInteractions: PropInteraction[];
  traitorVote?: TraitorVote;
  outcome?: GameOutcome;
}

interface ChatMessage {
  id: string;
  senderId: string;             // playerId o 'NPC'
  content: string;
  timestamp: number;
}

interface TraitorVote {
  initiatorId: string;
  nominations: Map<string, string>;  // votantId → nominatoId
  resolvedAt?: number;
  outcome?: 'CORRECT' | 'INCORRECT' | 'EXPIRED';
}

type GameOutcome =
  | { type: 'TRAITOR_DEFEATED'; winnerId: string }
  | { type: 'TRAITOR_VICTORY'; reason: 'TIMER_EXPIRED' | 'ALL_ABANDONED' };
```

---

## Component Design

### gateway

Implementato con un reverse proxy leggero (es. Fastify + http-proxy-middleware su Node.js). Responsabilità:

- **Auth guest**: riceve `{ nickname, roomCode }`, genera un JWT con `sub=playerId`, `roomId`, `nickname`, `exp=now+15min`. Firma con RS256 (chiave privata in secret Kubernetes).
- **Rate limiting**: sliding window in memoria (o Redis se multi-replica) — 10 req/min per IP sul path `/join`.
- **Routing**: forwarda le richieste ai microservizi downstream in base al path.
- **WebSocket upgrade**: proxya la connessione WS verso `room-service`.
- **WebTransport**: le connessioni H3 passano direttamente a `transport-service` (il gateway non può fare proxy trasparente di H3; i client si connettono direttamente su porta dedicata dopo aver ricevuto l'URL dal gateway).

### room-service

Gestisce il ciclo di vita delle stanze in memoria. Nessun database persistente.

**Struttura interna:**

```typescript
// RoomRegistry: Map<roomCode, Room>
// PlayerSessions: Map<playerId, WebSocket>

class RoomService {
  private rooms: Map<string, Room>;
  private sessions: Map<string, WebSocket>;

  createRoom(hostNickname: string, hostToken: JWTPayload): Room
  joinRoom(roomCode: string, playerToken: JWTPayload): Room
  setReady(roomCode: string, playerId: string, ready: boolean): void
  startGame(roomCode: string, hostId: string): void   // → emette evento a game-engine
  broadcast(roomCode: string, event: RoomEvent): void
  discardRoom(roomCode: string): void
}
```

**WebSocket events (server → client):**

| Event | Payload | Trigger |
|---|---|---|
| `room:player_list` | `Player[]` | join/leave/ready change |
| `room:game_starting` | `{ sceneId, roles }` | SCENE_SELECTION → IN_GAME |
| `room:chat_message` | `ChatMessage` | nuovo messaggio |
| `room:timer_update` | `{ remainingMs }` | ogni 30s |
| `room:vote_started` | `{ initiatorId, windowMs }` | voto avviato |
| `room:vote_result` | `TraitorVote` | voto risolto |
| `room:game_end` | `{ outcome, scores }` | fine partita |

**WebSocket events (client → server):**

| Event | Payload |
|---|---|
| `player:ready` | `{ ready: boolean }` |
| `player:chat` | `{ content: string }` |
| `player:vote` | `{ nomineeId: string }` |

### game-engine

Stateless rispetto alla persistenza: tutto lo stato è in-memory, ottenuto da `room-service` via chiamata interna (gRPC o HTTP interno, stesso cluster Kubernetes).

**Flusso di assegnazione ruoli:**

```
game-engine riceve: startGame(roomId, players[], scene)
  1. Shuffle players (Fisher-Yates)
  2. Assegna scene.roles[i] → players[i]  (1-to-1)
  3. Sceglie traitor = players[random(0, n-1)]
  4. Per ogni player:
       if player == traitor → invia { role, isTraitor: true, sabotagObjective }
       else                 → invia { role, isTraitor: false }
  5. Notifica room-service: rolesAssigned(roomId)
```

**Gestione turni:**

Il `game-engine` non impone un turno fisso (tutti i giocatori possono scrivere in chat). Ogni messaggio inviato via `player:chat` viene:
1. Validato (lunghezza max 500 caratteri, giocatore connesso)
2. Inoltrato a `npc-service` per generare la risposta NPC
3. Broadcastato a tutti tramite `room-service`

**Timer:**

```typescript
class GameTimer {
  private intervalId: NodeJS.Timeout;

  start(roomId: string, durationMs: number, onExpire: () => void): void {
    // broadcast ogni 30s
    // chiama onExpire a scadenza → game-engine transita in END_GAME
  }

  stop(): void { clearInterval(this.intervalId); }
}
```

### npc-service

Unico consumer delle API OpenAI. Costruisce il prompt con:

```
SYSTEM:
  Sei [NPC_NAME], il narratore della scena "[sceneTitle]" tratta da "[filmTitle]".
  Il tuo compito è orchestrare la scena mantenendo coerenza narrativa.
  I personaggi presenti sono: [role1.name: role1.description, ...].
  Non sai chi sia il traditore. Mantieni l'ambiguità drammatica.
  Rispondi sempre in [locale].

USER:
  [nickname] ([roleName]): [playerMessage]

ASSISTANT (history):
  [chatHistory ultimi N messaggi]
```

**Ritardo simulato:** prima di emettere la risposta, `npc-service` attende un delay casuale tra 1000ms e 2000ms (simulazione digitazione).

**Scoring prompt (invocato a end-game):**

```
Assegna un punteggio da 0 a 100 a ciascun giocatore secondo questi criteri:
  - Coerenza con il personaggio (40 pt)
  - Uso degli oggetti di scena (30 pt)
  - Partecipazione alla rilevazione del traditore (30 pt)
Il traditore era [traitorNickname] ([traitorRole]).
Ecco la cronologia completa della partita: [chatHistory].
Ecco le interazioni con gli oggetti: [propInteractions].
Esito del voto: [voteOutcome].
Rispondi con un JSON: { "scores": [{ "playerId": "...", "score": 0-100, "rationale": "..." }] }
```

**Circuit breaker:** se GPT-4o non risponde entro 10s, `npc-service` emette `scene-selection-failed` (durante selezione) o restituisce un messaggio di fallback NPC (durante il gameplay).

### rag-service

Tecnologia scelta: **pgvector** su PostgreSQL.

**Motivazione:** pgvector vive nello stesso cluster Kubernetes, elimina una dipendenza esterna (vs Pinecone), supporta SQL per query ibride (es. `WHERE role_count = $1`), e le dimensioni del dataset curato (centinaia di scene, non milioni) non richiedono un vector store distribuito.

**Schema:**

```sql
CREATE TABLE scenes (
  id          UUID PRIMARY KEY,
  film_title  TEXT NOT NULL,
  scene_title TEXT NOT NULL,
  role_count  INT  NOT NULL,
  metadata    JSONB NOT NULL,   -- scene completa serializzata
  embedding   VECTOR(1536)      -- text-embedding-3-small
);

CREATE INDEX ON scenes USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 50);
```

**Retrieval:**

```sql
SELECT id, metadata, embedding <=> $1 AS distance
FROM   scenes
WHERE  role_count = $2          -- exact match prima
ORDER  BY distance
LIMIT  10;
```

Se il risultato è vuoto, si ritenta con `role_count BETWEEN $2 - 1 AND $2 + 1`.

### transport-service

Server WebTransport (HTTP/3) su Node.js. Ogni Room ha un "room channel" in memoria.

```typescript
class TransportService {
  private rooms: Map<string, Set<WebTransportSession>>;

  onDatagram(session: WebTransportSession, data: AvatarPosition): void {
    // 1. Valida che position sia nel walkable polygon della scena
    // 2. Aggiorna last known position per il player
    // 3. Broadcast a tutti gli altri player nella room
  }
}

interface AvatarPosition {
  playerId: string;
  roomId: string;
  x: number;
  y: number;
  z: number;
  timestamp: number;
}
```

**Validazione walkable area:** point-in-polygon sul `walkablePolygon` della `SceneLayout`. Algoritmo ray-casting, O(n) sul numero di vertici del poligono (tipicamente < 20).

**Disconnessione:** quando una sessione WebTransport si chiude, `transport-service` mantiene l'ultima posizione nota e la include nei broadcast successivi come posizione statica.

### web-app / mobile-app

Architettura a layer:

```
┌────────────────────────────────────┐
│  Presentation Layer                │
│  (React components / RN components)│
│  shared-ui: LobbyScreen,           │
│  GameScreen, ScoreScreen           │
└────────────────┬───────────────────┘
                 │
┌────────────────▼───────────────────┐
│  Logic Layer (framework-agnostic)  │
│  RoomManager, GameStateManager,    │
│  AvatarController, VoteManager     │
└────────────────┬───────────────────┘
                 │
┌────────────────▼───────────────────┐
│  api-client                        │
│  RoomClient (WS), TransportClient  │
│  (WebTransport), HttpClient        │
└────────────────────────────────────┘
```

**Vista 3D isometrica (web-app):**

- Renderer: Three.js con proiezione ortografica isometrica (camera ruotata a 45° + inclinata a 30°).
- Asset: modelli GLTF per avatar e prop, caricati lazy in base alla `backgroundAssetKey` della scena.
- Avatar movement: su click sul pavimento, l'app calcola il punto 3D via raycasting sul piano y=0, invia la posizione a `transport-service`, e anima l'avatar localmente con interpolazione lineare (latenza percepita 0).
- Mobile: Three.js gira in un `<WebView>` Expo su Android/iOS. L'alternativa (react-three-fiber con Expo GL) è più elegante ma introduce dipendenza su `@react-three/fiber` non ancora stabile su Expo SDK 51+. La WebView garantisce parità di comportamento con il web senza fork del codice 3D.

**Decisione WebView vs react-three-fiber:**
- WebView: più semplice, codice 3D condiviso, ma overhead di bridge e UX leggermente peggiore su dispositivi medio-bassi.
- react-three-fiber nativamente: performance migliori su dispositivi recenti, ma maturità incerta su Expo e divergenza del codebase.
- **Scelta:** WebView per la prima versione, con path di migrazione a react-three-fiber quando l'ecosistema Expo lo supporta stabilmente.

### ingestion-service

CLI Node.js offline. Non espone endpoint HTTP.

```
$ cinerole-ingest --input scenes/godfather-I.json
```

**Flusso:**

```
1. Legge il file JSON (SceneDefinition)
2. Valida lo schema (zod)
3. Genera embedding con OpenAI text-embedding-3-small
4. INSERT INTO scenes (...) ON CONFLICT (id) DO UPDATE
5. Stampa OK / errori su stdout
```

**SceneDefinition (input format):**

```typescript
interface SceneDefinition {
  filmTitle: string;
  sceneTitle: string;
  narrativeContext: string;
  roles: Array<{
    name: string;
    description: string;
    objectives: string[];
    sabotagObjective?: string;   // presente solo per il ruolo che può essere assegnato come traitor
  }>;
  props: Array<{
    name: string;
    description: string;
    interactionType: 'pickup' | 'examine' | 'use';
    position: { x: number; y: number; z: number };
  }>;
  layoutData: {
    walkablePolygon: Array<{ x: number; y: number; z: number }>;
    spawnPoints: Array<{ x: number; y: number; z: number }>;
    backgroundAssetKey: string;
  };
}
```

---

## Communication Protocols

### Client ↔ room-service (WebSocket)

Formato messaggi: JSON. Autenticazione: il JWT è inviato nell'header `Authorization: Bearer <token>` durante l'handshake HTTP upgrade. `room-service` lo verifica con la chiave pubblica RS256.

### Client ↔ transport-service (WebTransport)

Formato messaggi: binario (Uint8Array serializzato con un codec compatto, es. MessagePack). La connessione WebTransport è autenticata con lo stesso JWT (passato come parametro query nell'URL di connessione, es. `https://transport.cinerole.app/room/:roomId?token=<jwt>`).

**Nota**: WebTransport over HTTP/3 richiede QUIC, disponibile su Chrome 97+, Firefox 114+, Safari 18+. Per i dispositivi che non supportano H3, `api-client` cade su WebSocket come fallback per il movimento (con latenza accettabilmente superiore).

### gateway ↔ microservizi (interno)

HTTP/1.1 interno via Kubernetes Service DNS. Non è necessario mTLS per la prima versione (tutti i pod sono nello stesso namespace con NetworkPolicy restrittiva).

### npc-service ↔ OpenAI

HTTPS REST verso `api.openai.com`. Timeout: 10s. Retry: 2 tentativi con backoff esponenziale (1s, 2s) prima di emettere fallback.

---

## Key Design Decisions

### 1. Stato in-memory vs database per le stanze

Le sessioni di gioco sono effimere per design. Usare un database (anche Redis) per lo stato delle stanze aggiunge latenza di scrittura su ogni evento (join, messaggio, posizione) senza beneficio: non c'è recovery da crash richiesto, e i dati vengono scartati a fine partita.

**Decisione:** stato in-memory in `room-service` e `game-engine`. Il trade-off accettato è che un restart del pod perde le stanze attive (accettabile per la v1; in futuro si può aggiungere Redis per HA).

### 2. pgvector vs Pinecone per il RAG

Pinecone offre scalabilità orizzontale automatica e SLA gestiti, ma introduce un servizio esterno, una dipendenza di rete, e un costo operativo. Il dataset di scene curate è nell'ordine delle centinaia di record: pgvector su PostgreSQL è ampiamente sufficiente e mantiene tutto nello stesso cluster.

**Decisione:** pgvector. Reversibile: se il dataset cresce oltre ~100k scene con query ad alto throughput, la migrazione a Pinecone è isolata a `rag-service`.

### 3. WebTransport con fallback WebSocket

WebTransport non è universalmente supportato (es. Safari < 18). Per non escludere utenti su browser/dispositivi meno recenti, `api-client` rileva il supporto a runtime e degrada a WebSocket per il canale di movimento.

**Decisione:** WebTransport come protocollo primario, WebSocket come fallback. L'SLA di 100ms è garantito su WebTransport; su WebSocket il target è best-effort.

### 4. NPC non conosce il traditore

Il prompt NPC non include mai l'identità del traditore. Questo è un vincolo di prodotto intenzionale per mantenere l'ambiguità drammatica, ma crea una complessità implementativa: `npc-service` riceve i ruoli da `game-engine` con il flag `isTraitor` sempre `false` per tutti i giocatori (il `game-engine` maschera l'informazione prima di passarla).

**Decisione:** `game-engine` è l'unica fonte di verità sull'identità del traditore. Espone due interfacce distinte: una per i client (ruolo privato completo) e una per `npc-service` (ruoli senza `isTraitor`).

### 5. Three.js in WebView su mobile

La duplicazione del renderer 3D è il maggiore rischio di divergenza tra web e mobile. La scelta della WebView elimina il fork, ma introduce latenza di bridge per eventi touch.

**Decisione:** WebView per v1, con un'interfaccia di messaggistica JS↔native (`window.postMessage` / `injectedJavaScript`) ben definita e isolata in `api-client` mobile, in modo che la migrazione a react-three-fiber sia circoscritta a quel layer.

---

## Correctness Properties

Le seguenti proprietà devono valere in ogni esecuzione corretta del sistema. Sono la base per i test property-based.

### Property 1: Unicità del ruolo

**Validates: Requirements 5.1, 5.2**

In ogni partita, ogni ruolo è assegnato a esattamente un giocatore e ogni giocatore ha esattamente un ruolo. Non esistono due `AssignedRole` con lo stesso `roleId` nella stessa `GameSession`.

### Property 2: Segretezza del traditore

**Validates: Requirements 5.3, 5.4, 10.1, 10.2, 10.3**

Nessun payload inviato a un giocatore non-traditore, o al `npc-service`, contiene `isTraitor: true` o un campo `sabotagObjective`. L'unico recipient legittimo è il player con `AssignedRole.isTraitor === true`.

### Property 3: Unicità del voto

**Validates: Requirements 11.2**

In ogni partita, `gameSession.traitorVote` può passare da `undefined` a un valore definito al più una volta. Qualsiasi tentativo successivo di iniziare un voto su una `GameSession` con `traitorVote` già definito è respinto.

### Property 4: Monotonia del timer

**Validates: Requirements 7.1, 7.2, 7.3**

`remainingMs` è strettamente decrescente nel tempo dopo l'avvio della partita. Non può aumentare né tornare al valore iniziale fintanto che la stanza è in stato `IN_GAME`.

### Property 5: Finalità dell'end-game

**Validates: Requirements 11.4, 11.5, 7.3**

Una stanza in stato `END_GAME` o `DISCARDED` non può mai transitare a `IN_GAME`, `SCENE_SELECTION`, o `LOBBY`. Le transizioni valide da `END_GAME` sono solo verso `DISCARDED`.

### Property 6: Capacità della stanza

**Validates: Requirements 1.2, 2.4**

Il numero di player con `connected: true` in una stanza non supera mai 5. Ogni tentativo di join su una stanza con 5 player connessi è respinto prima che lo stato venga modificato.

### Property 7: Invariante abbandono del traditore

**Validates: Requirements 10.4, 12.1**

Se il player con `isTraitor: true` si disconnette, `gameSession.traitorPlayerId` rimane invariato per tutta la durata della sessione. La partita continua; l'avatar del traditore diventa silente ma la sua identità è preservata per il calcolo finale.

### Property 8: Punteggi visibili solo a fine partita

**Validates: Requirements 13.4**

Nessun `PlayerScore` è incluso in alcun WebSocket broadcast prima che la stanza raggiunga lo stato `END_GAME`. L'evento `room:game_end` è l'unico evento che può trasportare punteggi.

---

## Deployment

```
Kubernetes Namespace: cinerole

Deployments:
  gateway            (1 replica, HPA min=1 max=3)
  room-service       (1 replica, HPA min=1 max=5)
  game-engine        (1 replica, HPA min=1 max=5)
  npc-service        (1 replica, HPA min=1 max=3)  — collo di bottiglia: OpenAI API
  rag-service        (1 replica, HPA min=1 max=2)
  transport-service  (1 replica, HPA min=1 max=5)
  postgres-pgvector  (StatefulSet, 1 replica, PVC 10Gi)

Secrets:
  openai-api-key     (montato come env var in npc-service e ingestion-service)
  jwt-keypair        (RS256 private key in gateway, public key in tutti i servizi)

Frontend:
  web-app  → S3 bucket + CloudFront CDN
  mobile-app → Expo build → App Store / Google Play
```

---

## Open Questions

1. **Asset 3D per le scene:** chi li produce? Modelli GLTF custom per ogni film richiedono effort artistico significativo. Alternativa: stile voxel con asset generici + colori/texture tematici che riducono il costo per scena.
2. **Walkable polygon:** deve essere definito manualmente nel `SceneDefinition` o calcolato automaticamente dall'asset 3D? Per v1, la definizione manuale è più controllata.
3. **NPC persona e nome:** ogni scena definisce un NPC con nome e persona, oppure esiste un NPC generico "narratore"? Attualmente il design assume un NPC generico; una persona per scena migliora l'immersione ma richiede un campo aggiuntivo in `SceneDefinition`.
4. **Concurrency del voto:** se due giocatori inviano `player:vote` in parallelo, il `game-engine` deve serializzare i voti. L'implementazione attuale li accoda in ordine di arrivo al WebSocket handler; verificare se serve un lock esplicito.

---

## Components and Interfaces

### gateway — Public API

```
POST /auth/join
  Body:    { nickname: string, roomCode: string }
  Returns: { token: string }            // JWT RS256, exp=15min
  Errors:  400 (validation), 404 (room not found), 409 (nickname taken),
           429 (rate limit), 403 (room full / game in progress)

POST /rooms
  Headers: Authorization: Bearer <token>
  Returns: { roomCode: string, inviteLink: string }

GET /rooms/:code
  Returns: { exists: boolean, playerCount: number, state: RoomState }
```

### room-service — Internal Interface

Chiamato da `game-engine` via HTTP interno (Kubernetes Service DNS):

```
POST /internal/rooms/:roomId/roles-assigned
  Body: { assignments: Array<{ playerId, roleId, isTraitor }> }

POST /internal/rooms/:roomId/game-end
  Body: { outcome: GameOutcome, scores: PlayerScore[] }

GET  /internal/rooms/:roomId
  Returns: Room (snapshot corrente)
```

### game-engine — Internal Interface

Chiamato da `room-service` al momento del `startGame`:

```
POST /internal/games/start
  Body: { roomId, players: Player[], scene: Scene }

POST /internal/games/:roomId/chat
  Body: { playerId, content: string }

POST /internal/games/:roomId/vote
  Body: { initiatorId, nomineeId: string }
```

### npc-service — Internal Interface

Chiamato da `game-engine`:

```
POST /internal/npc/respond
  Body: { roomId, scene: SceneContext, chatHistory: ChatMessage[],
          playerMessage: ChatMessage, roles: PublicRole[] }
  Returns: { npcMessage: string }         // dopo ritardo simulato

POST /internal/npc/score
  Body: { roomId, scene: SceneContext, chatHistory: ChatMessage[],
          roles: PublicRole[], traitorPlayerId: string,
          propInteractions: PropInteraction[], voteOutcome: string }
  Returns: { scores: PlayerScore[] }
```

`PublicRole` è `Role` senza il flag `isTraitor` né `sabotagObjective` — `game-engine` costruisce questo tipo prima di chiamare `npc-service`.

### rag-service — Internal Interface

```
GET /internal/scenes/candidates?playerCount=<n>
  Returns: { scenes: Scene[] }   // max 10, ordinati per cosine similarity
```

### transport-service — WebTransport Endpoint

```
wt://transport.cinerole.app/room/:roomId?token=<jwt>

Datagram in  (client → server): AvatarPosition (MessagePack binario)
Datagram out (server → client): AvatarPosition[] (tutti i player nella room)
```

### api-client — Interfacce esposte ai frontend

```typescript
interface RoomClient {
  connect(token: string): Promise<void>;
  on<E extends RoomEvent>(event: E['type'], handler: (e: E) => void): void;
  send(event: ClientEvent): void;
  disconnect(): void;
}

interface TransportClient {
  connect(roomId: string, token: string): Promise<void>;
  sendPosition(pos: Vector3): void;
  onPositionUpdate(handler: (update: AvatarPosition) => void): void;
  disconnect(): void;
}

interface HttpClient {
  join(nickname: string, roomCode: string): Promise<{ token: string }>;
  createRoom(token: string): Promise<{ roomCode: string; inviteLink: string }>;
}
```

---

## Error Handling

### Errori di rete e connessione

- **WebSocket disconnect:** `room-service` rileva la chiusura del socket. Se il player è in lobby, viene rimosso dalla lista e il client riceve `room:player_list` aggiornato. Se è in-game, l'avatar diventa silente (NPC) e gli altri ricevono `room:player_list` con `connected: false`.
- **WebTransport disconnect:** `transport-service` mantiene l'ultima posizione nota e continua a includerla nei broadcast. Nessuna azione su `room-service` (il canale di movimento è separato dal canale di gioco).
- **Reconnection:** il client si riconnette con lo stesso JWT (ancora valido entro 15min). `room-service` riconosce il `playerId` dal token, ripristina lo stato e invia un `room:full_state` snapshot al client riconnesso.

### Errori del layer LLM

- **GPT-4o timeout (> 10s) durante la risposta NPC:** `npc-service` restituisce un messaggio di fallback statico localizzato (es. "… [silenzio]") al `game-engine`, che lo broadcastsa come normale messaggio NPC. Il gioco continua.
- **GPT-4o timeout durante lo scoring:** `npc-service` restituisce punteggi placeholder (0 per tutti). `game-engine` invia la schermata finale con i placeholder e ritenta il calcolo in background; se il retry ha successo entro 30s, invia un evento `room:scores_updated`.
- **GPT-4o timeout durante la selezione scena:** `npc-service` emette `scene-selection-failed`; `room-service` riporta la stanza a `LOBBY` e notifica tutti i player.

### Errori di validazione

- Tutti i microservizi restituiscono errori strutturati: `{ code: string, message: string, details?: object }`.
- `gateway` traduce gli errori interni in HTTP standard (400, 403, 404, 429, 500) prima di rispondere ai client.

### Saturazione della stanza

- Tentativi di join su una stanza piena (5 player) o in corso vengono respinti da `room-service` con codice `ROOM_FULL` / `GAME_IN_PROGRESS` rispettivamente. `gateway` li espone come HTTP 403.

---

## Testing Strategy

### Property-Based Testing (PBT)

Le 8 correctness properties nella sezione dedicata sono il target principale del PBT. Per ciascuna si definisce un generatore di stati arbitrari di `Room` / `GameSession` e si verifica che la proprietà valga su 1000+ campioni.

Framework suggerito: **fast-check** (TypeScript, compatibile con Jest/Vitest).

Esempi:
- Property 2 (segretezza traditore): generare `GameSession` arbitrarie con `traitorPlayerId` random → verificare che nessun payload `npc-service` o broadcast generico contenga `isTraitor: true`.
- Property 5 (finalità end-game): generare sequenze di eventi arbitrari su una stanza in `END_GAME` → verificare che lo stato non torni mai a `IN_GAME`.

### Integration Tests

- `game-engine` + `npc-service` (mockato): verifica del flusso completo assegnazione ruoli → turno → fine partita.
- `room-service` + WebSocket client (ws): verifica dei broadcast entro le soglie temporali (500ms player_list, 200ms chat).
- `rag-service` + pgvector (test container): verifica del retrieval con dataset sintetico.

### Contract Tests

- `api-client` vs `gateway`: request/response schema validato con Zod su entrambi i lati.
- `npc-service` vs `game-engine`: `PublicRole[]` non deve mai contenere `isTraitor` né `sabotagObjective` — test di contratto esplicito.
