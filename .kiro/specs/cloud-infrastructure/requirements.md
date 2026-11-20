# Requirements Document

## Introduction

Questo documento definisce i requisiti infrastrutturali per il deployment di CineRole su Kubernetes.
CineRole è un'applicazione web e mobile di role-play multiplayer; il backend è composto da
microservizi (gateway, room-service, game-engine, npc-service, rag-service, transport-service)
deployati come Deployment Kubernetes separati. Il documento copre la topologia del cluster,
l'esposizione pubblica, il routing interno, l'isolamento di rete e la scalabilità orizzontale.

## Glossary

- **Cluster**: l'insieme delle VM Kubernetes che ospita tutti i microservizi di CineRole.
- **Control_Plane_Node**: VM Kubernetes che esegue i componenti di controllo (API Server, etcd, Scheduler, Controller Manager).
- **Worker_Node**: VM Kubernetes che esegue i Pod applicativi.
- **etcd**: datastore distribuito che mantiene lo stato del cluster; richiede quorum a maggioranza (⌊n/2⌋+1 su n nodi).
- **Cloud_Load_Balancer**: load balancer gestito dal provider cloud (es. AWS ALB, GCP Load Balancer, Azure LB) che costituisce il solo punto di esposizione pubblica del cluster.
- **Ingress_Controller**: componente interno al cluster (es. NGINX Ingress, Traefik) che riceve il traffico dal Cloud_Load_Balancer e lo instrada ai Service Kubernetes interni.
- **HPA**: Horizontal Pod Autoscaler Kubernetes; scala orizzontalmente il numero di replica di un Deployment in base a metriche di carico.
- **NetworkPolicy**: risorsa Kubernetes che definisce regole di isolamento del traffico tra Pod e namespace.
- **Namespace**: unità di isolamento logico all'interno del cluster Kubernetes.
- **Service**: risorsa Kubernetes che espone un insieme di Pod all'interno del cluster tramite un indirizzo stabile.
- **ClusterIP**: tipo di Service Kubernetes accessibile solo dall'interno del cluster; non espone porte pubbliche.

---

## Requirements

### Requirement 1: Topologia del Cluster Kubernetes

**User Story:** Come operatore dell'infrastruttura, voglio un cluster Kubernetes ad alta disponibilità
con control plane ridondato, in modo che un singolo guasto a un nodo non interrompa il servizio.

#### Criteri di Accettazione

1. THE Cluster SHALL essere composto da esattamente 6 VM: 3 Control_Plane_Node e 3 Worker_Node come configurazione baseline.
2. WHEN un Control_Plane_Node diventa non raggiungibile, THE Cluster SHALL mantenere il quorum etcd con i 2 Control_Plane_Node rimanenti e continuare a servire le richieste senza interruzione.
3. THE etcd SHALL operare in modalità HA distribuita sui 3 Control_Plane_Node con quorum a maggioranza (almeno 2 su 3 nodi attivi).
4. WHEN il carico sui Worker_Node supera le soglie configurate nell'HPA, THE HPA SHALL scalare orizzontalmente i Pod dei microservizi sui Worker_Node esistenti senza modificare il numero di Control_Plane_Node.
5. IF un Worker_Node diventa non raggiungibile, THEN THE Cluster SHALL rischedulare i Pod ospitati su quel nodo sui Worker_Node rimanenti entro il timeout di rilevazione configurato.
6. THE Control_Plane_Node SHALL essere dedicati esclusivamente ai componenti di controllo del cluster; nessun Pod applicativo SHALL essere schedulato su di essi.

---

### Requirement 2: Esposizione Pubblica tramite Cloud Load Balancer

**User Story:** Come operatore dell'infrastruttura, voglio che il cluster sia raggiungibile dall'esterno
attraverso un unico Cloud_Load_Balancer gestito dal provider, in modo da centralizzare terminazione TLS,
healthcheck e protezione DDoS fuori dal cluster.

#### Criteri di Accettazione

1. THE Cloud_Load_Balancer SHALL costituire l'unico punto di ingresso pubblico per tutto il traffico HTTP, HTTPS e WebSocket diretto ai microservizi di CineRole.
2. WHEN una richiesta HTTPS raggiunge il Cloud_Load_Balancer, THE Cloud_Load_Balancer SHALL terminare TLS e inoltrare il traffico in chiaro (o con TLS interno) all'Ingress_Controller.
3. THE Cloud_Load_Balancer SHALL eseguire healthcheck periodici sull'Ingress_Controller e rimuovere dal pool un nodo che non supera il healthcheck entro 2 controlli consecutivi falliti.
4. IF il Cloud_Load_Balancer non riceve risposta dall'Ingress_Controller entro 30 secondi, THEN THE Cloud_Load_Balancer SHALL restituire al client una risposta HTTP 502 con messaggio di errore leggibile.
5. THE Cloud_Load_Balancer SHALL supportare WebSocket keep-alive per il traffico di chat in modo da non interrompere connessioni attive per timeout inattività.
6. THE Cloud_Load_Balancer SHALL supportare connessioni HTTP/3 (QUIC) per il traffico WebTransport del transport-service dedicato al movimento degli avatar.

---

### Requirement 3: Ingress Controller e Routing Interno

**User Story:** Come operatore dell'infrastruttura, voglio un Ingress_Controller interno al cluster che
instrada il traffico ai Service corretti, in modo che ogni microservizio non debba esporre direttamente
una porta pubblica.

#### Criteri di Accettazione

1. THE Ingress_Controller SHALL essere deployato all'interno del cluster come Deployment Kubernetes e ricevere il traffico esclusivamente dal Cloud_Load_Balancer tramite un Service di tipo LoadBalancer o NodePort interno.
2. THE Ingress_Controller SHALL instradare le richieste ai Service Kubernetes interni in base a regole Ingress definite per hostname e path prefix (es. `/api/room` → room-service, `/api/game` → game-engine).
3. WHEN viene deployata una nuova versione di una regola Ingress, THE Ingress_Controller SHALL applicare la nuova configurazione di routing senza interrompere le connessioni WebSocket attive.
4. THE Ingress_Controller SHALL supportare il rate limiting per path, configurabile tramite annotazioni Ingress, con un limite default di 100 richieste al secondo per IP sorgente.
5. IF una richiesta è destinata a un path non definito nelle regole Ingress, THEN THE Ingress_Controller SHALL rispondere con HTTP 404 senza esporre dettagli interni del cluster.
6. THE Ingress_Controller SHALL aggiungere gli header di sicurezza HTTP (Strict-Transport-Security, X-Content-Type-Options, X-Frame-Options) a tutte le risposte uscenti.

---

### Requirement 4: Deployment dei Microservizi

**User Story:** Come sviluppatore, voglio che ogni microservizio sia deployato come Deployment
Kubernetes indipendente, in modo da poter rilasciare, scalare e riavviare ogni componente
senza impatto sugli altri.

#### Criteri di Accettazione

1. THE Cluster SHALL deployare ciascuno dei seguenti microservizi come Deployment Kubernetes separato: gateway, room-service, game-engine, npc-service, rag-service, transport-service.
2. WHEN viene rilasciata una nuova immagine di un microservizio, THE Cluster SHALL eseguire un rolling update che mantiene attiva almeno una replica del vecchio Pod fino alla disponibilità della nuova replica.
3. THE Cluster SHALL definire resource requests e limits (CPU e memoria) per ogni Deployment in modo che un singolo microservizio non consumi l'intero Worker_Node.
4. WHEN un Pod di un microservizio termina con exit code diverso da 0, THE Cluster SHALL riavviarlo automaticamente secondo la restart policy configurata (Always), fino al limite di backoff esponenziale.
5. IF la probe di readiness di un Pod non risponde entro il timeout configurato, THEN THE Cluster SHALL rimuovere il Pod dall'endpoint del Service corrispondente finché la probe non torna positiva.

---

### Requirement 5: Scalabilità Orizzontale con HPA

**User Story:** Come operatore, voglio che i microservizi con carico variabile scalino automaticamente
in orizzontale, in modo da gestire i picchi di traffico senza provisioning manuale.

#### Criteri di Accettazione

1. THE HPA SHALL essere configurato per gateway, room-service, game-engine e npc-service con soglia di scale-out all'80% di utilizzo CPU medio.
2. WHEN l'utilizzo CPU medio di un Deployment supera la soglia configurata per almeno 60 secondi consecutivi, THE HPA SHALL aggiungere una replica entro 90 secondi.
3. WHEN l'utilizzo CPU medio di un Deployment scende sotto il 40% per almeno 5 minuti consecutivi, THE HPA SHALL rimuovere una replica rispettando il numero minimo configurato.
4. THE HPA SHALL rispettare un numero minimo di 2 repliche per gateway e room-service per garantire la disponibilità durante lo scale-in.
5. WHERE l'infrastruttura supporta metriche custom, THE HPA SHALL poter essere configurato anche su metriche applicative (es. numero di stanze attive per room-service) in aggiunta alle metriche CPU.

---

### Requirement 6: Isolamento di Rete tra Namespace

**User Story:** Come operatore della sicurezza, voglio che i microservizi siano isolati tramite
NetworkPolicy Kubernetes, in modo che un componente compromesso non possa raggiungere
arbitrariamente gli altri.

#### Criteri di Accettazione

1. THE Cluster SHALL organizzare i microservizi in almeno due Namespace distinti: uno per i servizi esposti (gateway, Ingress_Controller) e uno per i servizi interni (room-service, game-engine, npc-service, rag-service, transport-service).
2. THE NetworkPolicy SHALL negare per default tutto il traffico ingress e egress tra Namespace non esplicitamente autorizzati (default-deny).
3. WHEN un microservizio interno riceve una richiesta da un namespace non autorizzato, THE NetworkPolicy SHALL bloccare la connessione prima che raggiunga il Pod di destinazione.
4. THE NetworkPolicy SHALL autorizzare esplicitamente solo i flussi necessari: Ingress_Controller → gateway, gateway → room-service, gateway → game-engine, npc-service → rag-service, game-engine → npc-service, transport-service → room-service.
5. IF un Pod tenta di aprire una connessione verso l'API Server Kubernetes da un namespace applicativo, THEN THE NetworkPolicy SHALL bloccare la connessione, eccetto per i Service Account con permessi RBAC espliciti.
6. THE Cluster SHALL applicare le NetworkPolicy tramite un CNI plugin che supporta l'enforcement (es. Calico, Cilium); un CNI senza enforcement non è ammesso in produzione.

---

### Requirement 7 — Nessuna Esposizione Diretta dei Service Interni

**User Story:** Come operatore della sicurezza, voglio che nessun microservizio interno esponga
direttamente una porta pubblica, in modo che l'unico percorso di accesso esterno passi
per Cloud_Load_Balancer e Ingress_Controller.

#### Criteri di Accettazione

1. THE Cluster SHALL configurare tutti i Service dei microservizi interni (room-service, game-engine, npc-service, rag-service, transport-service) come tipo ClusterIP, senza porte esposte all'esterno del cluster.
2. WHEN viene deployato un nuovo microservizio, THE Cluster SHALL verificare tramite admission webhook o policy (es. OPA/Gatekeeper) che il Service associato non sia di tipo NodePort o LoadBalancer, a meno di autorizzazione esplicita.
3. THE gateway SHALL essere l'unico microservizio raggiungibile dall'Ingress_Controller; tutti gli altri microservizi SHALL essere raggiungibili solo tramite chiamate intra-cluster dal gateway o da altri microservizi autorizzati.
4. IF viene rilevato un Service di tipo NodePort o LoadBalancer non autorizzato nel cluster, THEN THE Cluster SHALL generare un alert verso il sistema di monitoraggio configurato.

---

### Requirement 8 — Gestione dei Certificati TLS

**User Story:** Come operatore, voglio che i certificati TLS siano gestiti automaticamente,
in modo da evitare interruzioni del servizio per scadenza manuale dei certificati.

#### Criteri di Accettazione

1. THE Cluster SHALL utilizzare cert-manager per il provisioning e il rinnovo automatico dei certificati TLS utilizzati dall'Ingress_Controller per la comunicazione interna cluster→LB.
2. WHEN un certificato TLS è a meno di 30 giorni dalla scadenza, THE cert-manager SHALL avviare automaticamente il processo di rinnovo senza interruzione del servizio.
3. IF il rinnovo automatico di un certificato fallisce, THEN THE Cluster SHALL generare un alert con severità critica verso il sistema di monitoraggio configurato.

---

### Requirement 9 — Monitoraggio e Healthcheck

**User Story:** Come operatore, voglio che lo stato di salute del cluster e dei microservizi sia
osservabile in tempo reale, in modo da rilevare e risolvere anomalie prima che impattino gli utenti.

#### Criteri di Accettazione

1. THE Cluster SHALL esporre metriche Prometheus per ogni Deployment (CPU, memoria, numero di repliche, latenza delle richieste) tramite endpoint `/metrics` accessibili solo dall'interno del cluster.
2. WHEN un Deployment scende a 0 repliche disponibili, THE Cluster SHALL generare un alert entro 60 secondi verso il sistema di alerting configurato.
3. THE Ingress_Controller SHALL registrare log strutturati (JSON) per ogni richiesta ricevuta, includendo: timestamp, path, metodo HTTP, status code, latenza, IP sorgente anonimizzato.
4. WHILE una partita è attiva, THE npc-service SHALL rispettare il vincolo di latenza: il 95% delle risposte NPC SHALL essere prodotto entro 3 secondi dalla ricezione della richiesta.

---

### Requirement 10 — Continuità operativa e Disaster Recovery

**User Story:** Come operatore, voglio che il cluster sia resiliente ai guasti parziali,
in modo da garantire la continuità delle partite in corso.

#### Criteri di Accettazione

1. WHEN un singolo Worker_Node subisce un guasto, THE Cluster SHALL rischedulare i Pod impattati sui Worker_Node rimanenti senza richiedere intervento manuale, entro il timeout di rilevazione di node-not-ready (default: 5 minuti).
2. WHEN un singolo Control_Plane_Node subisce un guasto, THE Cluster SHALL mantenere il quorum etcd con i 2 nodi rimanenti e continuare a processare le richieste all'API Server senza interruzione.
3. THE etcd SHALL eseguire snapshot automatici con frequenza configurabile (default: ogni 6 ore) e conservare gli ultimi 5 snapshot su storage persistente esterno al cluster.
4. IF tutti e 3 i Control_Plane_Node diventano simultaneamente non raggiungibili, THEN THE Cluster SHALL interrompere l'accettazione di nuove richieste di modifica allo stato del cluster (write quorum perso) mantenendo i Pod esistenti in esecuzione fino al ripristino.

---

### Requirement 11 — Storage Condiviso su Bucket S3-Compatible

**User Story:** Come sviluppatore di microservizi, voglio uno storage condiviso accessibile da tutti i componenti del cluster, in modo che possano leggere e scrivere asset statici (immagini, file di configurazione, risorse condivise) senza dipendenze dirette tra servizi.

#### Criteri di Accettazione

1. THE Cluster SHALL avere accesso a un bucket su storage S3-compatible dedicato alla condivisione di asset statici tra i microservizi, distinto dal bucket S3 usato per il frontend statico.
2. THE bucket condiviso SHALL essere accessibile in lettura e scrittura da tutti i microservizi del cluster tramite credenziali iniettate come Kubernetes Secret (access key e secret key mai in chiaro nei manifesti o nelle immagini).
3. WHEN un microservizio scrive un asset nel bucket condiviso, THE bucket SHALL rendere l'asset disponibile in lettura agli altri microservizi entro la consistenza eventuale garantita dal provider S3-compatible scelto.
4. THE Cluster SHALL montare le credenziali del bucket come Secret Kubernetes nei Pod dei microservizi che ne hanno bisogno, senza esporre le credenziali come variabili d'ambiente in chiaro nei log o negli eventi Kubernetes.
5. IF la connessione al bucket condiviso non è raggiungibile da un microservizio, THEN THE microservizio SHALL registrare un errore strutturato nel log e restituire un codice di errore appropriato al chiamante, senza andare in crash.
6. THE bucket condiviso SHALL avere versioning abilitato, in modo da consentire il recupero di versioni precedenti degli asset in caso di sovrascrittura accidentale.
