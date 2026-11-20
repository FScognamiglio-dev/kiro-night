---
inclusion: always
---
# Product

## Goal
CineRole e un'applicazione web e mobile di role-play multiplayer ambientata in scene di film popolari.
L'AI seleziona una scena da un dataset curato, assegna i ruoli ai giocatori (max 5), e orchestra
la partita tramite un personaggio NPC. Uno dei giocatori e segretamente il traditore: deve sabotare
la scena senza essere scoperto. La partita dura al massimo 15 minuti e si conclude prima se il
traditore viene identificato. Al termine, l'NPC assegna un punteggio a ciascun giocatore.

## Users
- **Giocatore guest**: entra con un nickname temporaneo, gioca senza registrazione, non ha dati persistenti.

## In scope
- Stanze private accessibili tramite codice o link di invito
- Selezione automatica della scena e assegnazione dei ruoli da parte dell'AI
- Gameplay testuale (chat) con NPC AI che orchestra la scena
- Vista 3D isometrica stile Habbo con avatar e oggetti cliccabili, realizzata con Three.js
- Oggetti contestuali al film selezionato, utilizzabili dai giocatori tramite click
- Meccanismo del traditore: un giocatore riceve il ruolo in segreto; l'NPC non conosce la sua identita
- Voto collettivo unico per rivelare il traditore
- Punteggio per ogni giocatore calcolato dall'NPC su criteri definiti nel prompt (coerenza con il
  personaggio, uso degli oggetti, partecipazione alla rilevazione del traditore), mostrato solo al termine
- Internazionalizzazione (i18n) con italiano come lingua default, architettura pronta per altre lingue
- Parita funzionale completa tra web (React) e mobile (React Native)
- Motore di ingestion per il dataset curato delle scene (RAG)

## Out of scope
- Registrazione utenti e account persistenti
- Storage delle sessioni di gioco (le partite sono effimere)
- Comunicazione vocale
- Stanze pubbliche
- Moderazione automatica dei contenuti testuali
- Statistiche storiche per giocatore

## Success criteria
- Una stanza con 2-5 giocatori puo completare una partita end-to-end senza errori bloccanti
- L'NPC risponde entro 3 secondi (incluso ritardo simulato) nel 95% dei turni
- Il movimento degli avatar nella scena 3D ha latenza percepita inferiore a 100 ms
- La vista 3D e funzionante su browser desktop e su app mobile

## Decided behaviors
- Giocatori < 2: la stanza non puo avviare la partita
- Giocatori > 5: accesso alla stanza negato
- Voto sul traditore sbagliato: la partita continua fino allo scadere del tempo (voto non ripetibile)
- Tempo scaduto senza rilevazione: vittoria del traditore, punteggi mostrati
- Giocatore che abbandona: il suo avatar rimane come NPC silente per non rompere la scena
- NPC non conosce l'identita del traditore: mantiene l'ambiguita drammatica

## Constraints
- LLM: OpenAI GPT-4o per NPC e selezione scene
- Lingua default: italiano; architettura i18n per estensioni future
- Sessioni effimere: nessun dato di gioco persistito dopo la fine della partita
- Nessuna dipendenza runtime da fonti esterne per le scene (dataset curato locale via RAG)
- Movimento avatar via WebTransport (HTTP/3), chat via WebSocket
- Frontend deployato su object storage S3-compatible; microservizi su Kubernetes

## Assumptions
- Il dataset iniziale delle scene sara popolato manualmente prima del lancio (ingestion usata offline)
- I criteri di punteggio nel prompt NPC saranno definiti in dettaglio nella fase di design
- La gestione di abusi testuali e fuori scope per la prima versione
