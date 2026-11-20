# Requirements Document

## Introduction

CineRole è un'applicazione web e mobile di role-play multiplayer ambientata in scene di film popolari. L'AI seleziona una scena da un dataset curato, assegna i ruoli ai giocatori (da 2 a 5), e orchestra la partita tramite un personaggio NPC. Uno dei giocatori è segretamente il traditore: deve sabotare la scena senza essere scoperto. La partita dura al massimo 15 minuti e si conclude prima se il traditore viene identificato. Al termine, l'NPC assegna un punteggio a ogni giocatore.

Questa spec copre l'intero prodotto CineRole nelle seguenti macro-aree:
1. Gestione stanze (creazione, join, lobby)
2. Selezione scena e assegnazione ruoli tramite AI/RAG
3. Gameplay testuale con NPC AI
4. Vista 3D isometrica (Three.js / React Native)
5. Meccanismo del traditore e voto collettivo
6. Punteggio finale
7. Internazionalizzazione

---

## Glossary

- **Guest**: giocatore che accede senza registrazione tramite nickname temporaneo.
- **Room**: stanza di gioco privata, identificata da un codice univoco e accessibile tramite link di invito.
- **Room_Service**: microservizio responsabile della gestione del ciclo di vita delle stanze, del join, dello stato della partita e del timer.
- **Game_Engine**: microservizio responsabile della logica di gioco: turni, voto traditore, punteggio finale.
- **NPC_Service**: microservizio che orchestra le chiamate LLM (GPT-4o), applica il ritardo simulato e calcola il punteggio.
- **RAG_Service**: microservizio per il retrieval delle scene dal dataset curato e la selezione dei ruoli.
- **Transport_Service**: server WebTransport per la trasmissione in real-time di posizione e movimento degli avatar.
- **Gateway**: API Gateway responsabile del routing, dell'autenticazione guest e del rate limiting.
- **Scene**: scena di un film popolare estratta dal dataset curato, con titolo, contesto narrativo, lista di ruoli e lista di oggetti contestuali.
- **Role**: personaggio assegnato a un giocatore per la durata della partita; include nome, descrizione e obiettivi narrativi.
- **Traitor_Role**: ruolo speciale assegnato in segreto a un giocatore; include istruzioni di sabotaggio non visibili agli altri giocatori e all'NPC.
- **NPC**: personaggio non giocante orchestrato dall'AI che guida la scena, risponde ai turni dei giocatori e mantiene la coerenza narrativa.
- **Turn**: unità di interazione in cui un giocatore invia un messaggio in chat e l'NPC risponde.
- **Traitor_Vote**: voto collettivo unico con cui i giocatori nominano il presunto traditore.
- **Score**: punteggio numerico assegnato dall'NPC a ogni giocatore al termine della partita.
- **Isometric_View**: vista 3D isometrica stile Habbo realizzata con Three.js (web) e renderer equivalente (mobile), che mostra gli avatar e gli oggetti della scena.
- **Avatar**: rappresentazione 3D del giocatore nella vista isometrica.
- **Prop**: oggetto contestuale della scena, cliccabile dall'utente per interagire con la narrativa.
- **Ingestion_Service**: tool offline che popola il dataset RAG con le scene curate.
- **i18n**: sistema di internazionalizzazione basato su i18next, con italiano come lingua di default.
- **WebTransport**: protocollo HTTP/3 usato per il movimento degli avatar in real-time.
- **WebSocket**: protocollo usato per la chat in tempo reale.

---

## Requirements

### Requirement 1: Accesso Guest alla Stanza

**User Story:** As a Guest, I want to enter a room using only a nickname, so that I can play without registering an account.

#### Acceptance Criteria

1. WHEN a Guest submits a nickname and a valid room code, THE Gateway SHALL authenticate the Guest and issue a session token scoped to that Room.
2. THE Room_Service SHALL reject join requests when the Room already contains 5 players, returning an error that informs the Guest the room is full.
3. THE Room_Service SHALL reject join requests when the Room's game is already in progress, returning an error that informs the Guest the game has started.
4. WHEN a Guest submits a nickname that is already taken in the same Room, THE Room_Service SHALL reject the request and prompt the Guest to choose a different nickname.
5. THE Gateway SHALL enforce a rate limit of 10 join attempts per IP address per minute; IF the limit is exceeded, THEN THE Gateway SHALL return an HTTP 429 response.

---

### Requirement 2: Creazione e Condivisione della Stanza

**User Story:** As a Guest, I want to create a private room and share it via code or invite link, so that I can play with specific people.

#### Acceptance Criteria

1. WHEN a Guest requests room creation, THE Room_Service SHALL generate a Room with a unique 6-character alphanumeric code and a shareable invite link.
2. THE Room_Service SHALL make the invite link resolvable: WHEN a Guest opens the invite link, THE Gateway SHALL redirect the Guest to the join flow pre-populated with the room code.
3. THE Room_Service SHALL persist the Room state in memory for the duration of the session; WHEN the game ends or all players leave, THE Room_Service SHALL discard the Room state entirely.
4. THE Room_Service SHALL prevent a Room from starting when fewer than 2 players are present, returning a descriptive error to the host.

---

### Requirement 3: Lobby e Stato dei Giocatori

**User Story:** As a Guest in the lobby, I want to see which players are connected and their readiness, so that the host can start the game when everyone is ready.

#### Acceptance Criteria

1. WHILE a Room is in lobby state, THE Room_Service SHALL broadcast player list updates to all connected Guests within 500ms of any join or leave event.
2. WHEN a Guest marks themselves as ready, THE Room_Service SHALL update the player's ready state and broadcast the updated list to all Guests in the Room.
3. WHEN all connected players are marked as ready and the player count is between 2 and 5, THE Room_Service SHALL allow the host to start the game.
4. WHEN a connected Guest disconnects during the lobby, THE Room_Service SHALL remove the Guest from the player list and broadcast the update.

---

### Requirement 4: Selezione della Scena tramite AI/RAG

**User Story:** As a player, I want the AI to automatically select a scene appropriate for the number of players, so that the game starts with a coherent narrative context.

#### Acceptance Criteria

1. WHEN a game starts, THE RAG_Service SHALL retrieve candidate Scenes from the dataset whose role count matches the number of players in the Room.
2. THE NPC_Service SHALL select one Scene from the candidates using GPT-4o, providing the scene title, narrative context, role list, and prop list to all players.
3. WHEN no Scene in the dataset matches the exact player count, THE RAG_Service SHALL retrieve Scenes with a role count within ±1 of the player count, and THE NPC_Service SHALL adapt the role list accordingly.
4. THE RAG_Service SHALL return candidate Scenes within 1 second of a retrieval request in 95% of calls.
5. WHEN the RAG_Service returns no candidates, THE NPC_Service SHALL return an error to the Game_Engine and THE Room_Service SHALL notify all players that scene selection failed.

---

### Requirement 5: Assegnazione dei Ruoli

**User Story:** As a player, I want to receive a unique role for the scene, so that I know my character and objectives.

#### Acceptance Criteria

1. WHEN a Scene is selected, THE Game_Engine SHALL assign one Role to each player; each Role SHALL be assigned to exactly one player.
2. THE Game_Engine SHALL assign the Traitor_Role to exactly one player, chosen at random.
3. THE Game_Engine SHALL deliver each player's Role privately via their session channel; other players and the NPC SHALL NOT receive the Traitor_Role identity.
4. THE Game_Engine SHALL deliver the Traitor_Role assignment with sabotage instructions visible only to the player who receives it.
5. WHEN role assignment is complete, THE Room_Service SHALL transition the Room state from lobby to in-game.

---

### Requirement 6: Gameplay Testuale con NPC AI

**User Story:** As a player, I want to interact with the scene through a text chat orchestrated by an AI NPC, so that the narrative progresses in a coherent and engaging way.

#### Acceptance Criteria

1. WHEN a player sends a chat message during a Turn, THE NPC_Service SHALL generate an NPC response using GPT-4o within 3 seconds in 95% of Turns (inclusive of simulated NPC typing delay).
2. THE NPC_Service SHALL include in the GPT-4o prompt: the Scene context, all Role descriptions (excluding Traitor_Role identity), the full chat history of the current game, and the player's message.
3. THE NPC_Service SHALL apply a simulated typing delay of 1–2 seconds before delivering the NPC response, so that the interaction feels natural.
4. THE NPC_Service SHALL maintain narrative consistency: WHEN the NPC responds, the response SHALL reference the Scene context and the player's Role.
5. WHILE the game is in progress, THE Room_Service SHALL broadcast each chat message (player and NPC) to all players in the Room within 200ms of receipt.
6. THE Game_Engine SHALL enforce a maximum game duration of 15 minutes; WHEN the timer expires, THE Game_Engine SHALL transition the Room to end-game state regardless of game outcome.

---

### Requirement 7: Timer di Partita

**User Story:** As a player, I want to see the remaining time during the game, so that I can pace my actions accordingly.

#### Acceptance Criteria

1. WHEN a game starts, THE Room_Service SHALL start a countdown timer of 15 minutes and broadcast the initial remaining time to all players.
2. WHILE a game is in progress, THE Room_Service SHALL broadcast a timer update to all players every 30 seconds.
3. WHEN the timer reaches zero, THE Game_Engine SHALL trigger end-game resolution with Traitor victory outcome.
4. WHEN the game ends before the timer expires, THE Room_Service SHALL stop the timer.

---

### Requirement 8: Vista 3D Isometrica

**User Story:** As a player, I want to see a 3D isometric view of the scene with my avatar and the scene's props, so that I have a visual context for the narrative.

#### Acceptance Criteria

1. WHEN a game starts, THE web-app SHALL render the Isometric_View using Three.js, displaying the scene's layout, all player Avatars, and all Props associated with the selected Scene.
2. WHEN a game starts, THE mobile-app SHALL render an equivalent Isometric_View using the React Native renderer, with feature parity to the web-app view.
3. WHEN a player moves their Avatar, THE web-app and mobile-app SHALL send the new position to the Transport_Service via WebTransport; THE Transport_Service SHALL broadcast the position update to all players in the Room within 100ms.
4. WHEN the Transport_Service broadcasts a position update, THE web-app and mobile-app SHALL update the corresponding Avatar position in the Isometric_View.
5. THE Isometric_View SHALL display only Props belonging to the currently active Scene; Props SHALL be visually distinct and identifiable.
6. WHEN a player clicks a Prop, THE web-app and mobile-app SHALL send a prop-interaction event to THE Game_Engine, which SHALL inject a contextual action into the chat on behalf of the player.

---

### Requirement 9: Movimento degli Avatar

**User Story:** As a player, I want to move my avatar in the 3D scene, so that I can position myself near props and other players.

#### Acceptance Criteria

1. WHEN a player triggers a movement input (click or drag on the scene), THE web-app or mobile-app SHALL compute the target position within the walkable area of the scene and send it to THE Transport_Service.
2. THE Transport_Service SHALL validate that the target position is within the scene's walkable bounds; IF the position is outside bounds, THEN THE Transport_Service SHALL ignore the update and return an error to the sender.
3. WHILE a game is in progress, THE Transport_Service SHALL accept position updates from all players and broadcast each update to all other players in the Room.
4. WHEN a player disconnects during the game, THE Transport_Service SHALL retain the last known position for that player's Avatar and broadcast it as a static position to remaining players.

---

### Requirement 10: Meccanismo del Traditore

**User Story:** As the Traitor, I want to receive secret sabotage instructions and act subtly to disrupt the scene, so that other players cannot easily identify me.

#### Acceptance Criteria

1. THE Game_Engine SHALL ensure that the Traitor_Role identity is never included in any payload delivered to the NPC_Service, other players, or any broadcast channel.
2. WHEN the Traitor player sends a chat message, THE NPC_Service SHALL process it using the same GPT-4o prompt structure as other players, with no information about the player's Traitor status.
3. THE Game_Engine SHALL make the Traitor's sabotage instructions available only in the private role delivery message to the Traitor player.
4. WHEN a player abandons the game, IF the abandoned player was the Traitor, THEN THE Game_Engine SHALL retain the Traitor status on the abandoned Avatar, and the game SHALL continue with the Avatar treated as a silent NPC.

---

### Requirement 11: Voto Collettivo sul Traditore

**User Story:** As a player, I want to vote for who I think the traitor is, so that the group can try to identify and stop the sabotage.

#### Acceptance Criteria

1. WHILE a game is in progress, THE Game_Engine SHALL allow players to initiate a Traitor_Vote; a vote SHALL require a majority of connected players to cast their nomination before it is resolved.
2. THE Game_Engine SHALL enforce that each Room may hold at most one Traitor_Vote per game; IF a vote has already occurred in the current game, THEN THE Game_Engine SHALL reject any further vote initiation.
3. WHEN a Traitor_Vote is initiated, THE Room_Service SHALL broadcast the vote prompt to all connected players and collect nominations within a 60-second window.
4. WHEN the Traitor_Vote resolves and the nominated player is the Traitor, THE Game_Engine SHALL transition the Room to end-game state with Traitor defeat outcome.
5. WHEN the Traitor_Vote resolves and the nominated player is not the Traitor, THE Game_Engine SHALL record the incorrect vote outcome and the game SHALL continue until the timer expires.
6. WHEN the 60-second vote window expires without a majority nomination, THE Game_Engine SHALL cancel the vote and resume the game; this SHALL count as the one permitted vote.

---

### Requirement 12: Gestione del Giocatore che Abbandona

**User Story:** As a remaining player, I want the game to continue if another player disconnects, so that an unexpected disconnection doesn't break the session.

#### Acceptance Criteria

1. WHEN a player disconnects during a game, THE Room_Service SHALL retain that player's Avatar in the scene as a silent NPC and broadcast the updated player list to all remaining players.
2. THE Game_Engine SHALL exclude a disconnected player from Turn eligibility and from Traitor_Vote resolution quorum calculations.
3. IF all players except one disconnect, THEN THE Game_Engine SHALL transition the Room to end-game state with an abandonment outcome.
4. WHEN a player reconnects within the same session using the same token, THE Room_Service SHALL restore the player's state including role, chat history, and avatar position.

---

### Requirement 13: Punteggio Finale

**User Story:** As a player, I want to see my score and that of all players at the end of the game, so that I know how well I played.

#### Acceptance Criteria

1. WHEN the game transitions to end-game state, THE NPC_Service SHALL request GPT-4o to compute a Score for each player based on three criteria: narrative role consistency, prop usage, and participation in traitor detection.
2. THE NPC_Service SHALL include in the scoring prompt: the full chat history, each player's Role, the Traitor_Role identity, all prop-interaction events, and the Traitor_Vote outcome.
3. THE NPC_Service SHALL deliver the Scores to THE Game_Engine within 5 seconds of end-game trigger; IF scoring exceeds 5 seconds, THEN THE Game_Engine SHALL display a placeholder and update scores when they arrive.
4. THE Game_Engine SHALL broadcast the final Score for all players to all connected players simultaneously at end-game; Scores SHALL NOT be visible to any player before end-game.
5. THE Game_Engine SHALL include in the end-game broadcast: each player's Score, the Traitor's identity, the game outcome (Traitor victory or Traitor defeat), and the selected Scene title.

---

### Requirement 14: Internazionalizzazione

**User Story:** As a player, I want the application UI to be in Italian by default and extensible to other languages, so that the product is accessible to its target audience and future markets.

#### Acceptance Criteria

1. THE web-app and mobile-app SHALL use i18next to load all user-facing strings from locale bundles; no string SHALL be hardcoded in components.
2. THE i18n package SHALL provide an Italian locale bundle as the default; WHEN a locale is not available, THE i18n package SHALL fall back to Italian.
3. WHERE an additional language locale bundle is provided, THE web-app and mobile-app SHALL load it without requiring a code change or redeployment.
4. THE i18n package SHALL expose a typed key catalogue so that missing or mistyped translation keys produce a compile-time error.
5. WHEN the NPC_Service generates an NPC response, THE NPC_Service SHALL include the target language in the GPT-4o prompt so that all AI-generated narrative text is delivered in the player's active locale.

---

### Requirement 15: Ingestion del Dataset Scene

**User Story:** As a content curator, I want to ingest curated film scenes into the RAG dataset, so that the game has a library of scenes to select from at runtime.

#### Acceptance Criteria

1. THE Ingestion_Service SHALL accept a structured scene definition as input, including: film title, scene title, narrative context, list of Roles (name, description, objectives), list of Props (name, description, interaction type), and scene layout data.
2. WHEN a scene definition is ingested, THE Ingestion_Service SHALL generate vector embeddings for the scene and store them in the RAG_Service's vector store.
3. WHEN a scene definition fails validation (missing required fields), THE Ingestion_Service SHALL return a descriptive validation error and SHALL NOT write to the vector store.
4. THE Ingestion_Service SHALL be runnable as an offline CLI tool and SHALL NOT expose any HTTP endpoint in production.
5. THE RAG_Service SHALL return ingested scenes in retrieval queries within 1 second of indexing completion.
