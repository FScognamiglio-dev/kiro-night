---
inclusion: always
---
# Structure

## Packages
| Package | Responsibility | May depend on |
|---|---|---|
| `web-app` | React SPA: lobby, stanza, vista 3D (Three.js), chat UI | `shared-ui`, `i18n`, `api-client` |
| `mobile-app` | React Native: parita funzionale con web-app | `shared-ui`, `i18n`, `api-client` |
| `shared-ui` | Componenti UI condivisi tra web e mobile | `i18n` |
| `api-client` | Client HTTP/WS/WebTransport verso i microservizi | — |
| `i18n` | Bundle traduzioni e utility di localizzazione | — |
| `gateway` | API Gateway: routing, auth guest, rate limiting | — |
| `room-service` | Gestione stanze: creazione, join, stato partita, timer | `gateway` |
| `game-engine` | Logica di gioco: turni, voto traditore, punteggio finale | `room-service` |
| `npc-service` | Orchestrazione NPC: chiamate LLM, ritardo simulato, scoring | `game-engine`, `rag-service` |
| `rag-service` | Retrieval delle scene da dataset curato, selezione ruoli | — |
| `ingestion-service` | Motore di ingestion offline per popolare il dataset RAG | `rag-service` |
| `transport-service` | WebTransport server per posizione/movimento avatar in real-time | `room-service` |

## Rules
- Ogni task tocca al piu 3 package dalla tabella sopra, nominati esplicitamente.
- Dipendenze non elencate in "May depend on" sono vietate.
- Un nuovo package richiede di aggiornare questa tabella prima di qualsiasi implementazione.
- `shared-ui` non contiene logica di business: solo componenti presentazionali.
- `ingestion-service` e un tool offline; non riceve traffico di produzione.

## Stack
- **Frontend web**: React 18, Three.js, i18next
- **Frontend mobile**: React Native (Expo), i18next
- **Backend**: Node.js (TypeScript), microservizi deployati su Kubernetes
- **Real-time movimento**: WebTransport (HTTP/3)
- **Real-time chat**: WebSocket
- **LLM**: OpenAI GPT-4o (via API REST)
- **RAG**: vettoriale (tecnologia da definire in design — es. pgvector o Pinecone)
- **Deploy frontend**: Object storage S3-compatible (es. AWS S3 + CloudFront)
- **Deploy backend**: Kubernetes (ogni microservizio e un Deployment separato)
