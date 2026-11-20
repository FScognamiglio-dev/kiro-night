# Design Document — Cloud Infrastructure CineRole

## Overview

Questo documento descrive il design tecnico dell'infrastruttura Kubernetes per CineRole, un'applicazione
di role-play multiplayer a sessioni effimere. L'infrastruttura deve soddisfare tre macro-obiettivi:

1. **Alta disponibilità** — nessun single point of failure nel control plane né nei servizi applicativi esposti.
2. **Isolamento di sicurezza** — solo il Cloud Load Balancer raggiunge il cluster dall'esterno; i microservizi interni comunicano solo secondo flussi esplicitamente autorizzati.
3. **Scalabilità orizzontale automatica** — i servizi con carico variabile scalano senza intervento manuale.

Il backend è composto da sei microservizi (`gateway`, `room-service`, `game-engine`, `npc-service`,
`rag-service`, `transport-service`) deployati come Deployment Kubernetes indipendenti su un cluster
a sei nodi (3 control plane + 3 worker).

---

## Architecture

### Topologia del Cluster

```
┌─────────────────────────────────────────────────────────────────────┐
│  INTERNET                                                           │
│                        Client (browser / app)                       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ HTTPS / WSS / HTTP3
                                ▼
┌───────────────────────────────────────────────────────────────────┐
│  CLOUD PROVIDER                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Cloud Load Balancer (ALB / GCP LB / Azure LB)             │  │
│  │  • Terminazione TLS                                         │  │
│  │  • Healthcheck Ingress Controller                           │  │
│  │  • WebSocket keep-alive                                     │  │
│  │  • HTTP/3 (QUIC) per WebTransport                           │  │
│  └───────────────────────┬─────────────────────────────────────┘  │
└──────────────────────────┼────────────────────────────────────────┘
                           │ HTTP (plain) / HTTP/2 interno
                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  KUBERNETES CLUSTER                                                          │
│                                                                              │
│  ┌─────────────────── Namespace: cinlerole-exposed ────────────────────┐    │
│  │                                                                      │    │
│  │  ┌────────────────────────────────────────────────────────────────┐ │    │
│  │  │  Ingress Controller (NGINX)  — Service LoadBalancer/NodePort   │ │    │
│  │  │  • Routing per hostname/path                                   │ │    │
│  │  │  • Rate limiting (100 req/s per IP, configurabile)             │ │    │
│  │  │  • Security headers (HSTS, X-Frame-Options, X-Content-Type)    │ │    │
│  │  └────────────────────────────┬───────────────────────────────────┘ │    │
│  │                               │                                      │    │
│  │  ┌────────────────────────────▼───────────────────────────────────┐ │    │
│  │  │  gateway  (Deployment, min 2 repliche, HPA 80% CPU)            │ │    │
│  │  └────────────────────────────────────────────────────────────────┘ │    │
│  └───────────────────────────────┬──────────────────────────────────────┘    │
│                                  │ ClusterIP (flussi autorizzati)             │
│  ┌─────────────────── Namespace: cinerole-internal ────────────────────┐     │
│  │                                                                      │     │
│  │  room-service (HPA)   game-engine (HPA)   npc-service (HPA)         │     │
│  │       │                     │                   │                   │     │
│  │       └─────────────────────┘                   │                   │     │
│  │                                           rag-service                │     │
│  │  transport-service                                                   │     │
│  │                                                                      │     │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  Control Plane (3 nodi dedicati)                                       │  │
│  │  CP-1  ◄──► CP-2  ◄──► CP-3   (etcd HA, quorum 2/3)                  │  │
│  │  Taint: node-role.kubernetes.io/control-plane:NoSchedule              │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  Worker Nodes: WN-1, WN-2, WN-3  (Pod applicativi e Ingress)          │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
                           │
                           │ S3-compatible API
                    ┌──────▼──────────────────┐
                    │  Bucket condiviso        │
                    │  (asset statici, versio- │
                    │   ning abilitato)        │
                    └──────────────────────────┘
                    ┌──────────────────────────┐
                    │  Bucket frontend         │
                    │  (React/RN static files) │
                    └──────────────────────────┘
```

### Flussi di Traffico Principali

| Flusso | Protocollo | Percorso |
|--------|-----------|---------|
| Chat (WebSocket) | WSS → WS interno | Client → LB → Ingress → gateway → room-service |
| Movimento avatar | HTTP/3 WebTransport | Client → LB (QUIC) → Ingress → gateway → transport-service |
| NPC response | HTTPS | Client → LB → Ingress → gateway → game-engine → npc-service → rag-service |
| Asset statici frontend | HTTPS | Client → CDN / LB → Bucket frontend |
| Asset condivisi | S3 API | microservizi → Bucket condiviso (credenziali da Secret K8s) |

---

## Components and Interfaces

### 2.1 Control Plane (3 nodi)

Ogni Control_Plane_Node esegue: `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `etcd`.

**Configurazione etcd HA:**
- Cluster etcd a 3 membri distribuiti sui 3 nodi; quorum a maggioranza (⌊3/2⌋+1 = 2 nodi).
- Snapshot automatici ogni 6 ore verso bucket S3-compatible esterno; retention degli ultimi 5 snapshot.
- Comunicazione tra peer etcd cifrata con TLS mTLS generato da cert-manager.

**Isolamento dal workload applicativo:**
- Taint `node-role.kubernetes.io/control-plane:NoSchedule` su tutti e 3 i nodi di control plane.
- Nessun Pod applicativo può essere schedulato su questi nodi (Requisito 1.6).

### 2.2 Worker Nodes (3 nodi)

Eseguono i Pod applicativi, l'Ingress Controller e il DaemonSet del CNI plugin.

**Distribuzione dei Pod:**
- PodAntiAffinity `preferredDuringSchedulingIgnoredDuringExecution` per gateway e room-service: le repliche vengono distribuite su nodi distinti dove possibile.

### 2.3 Ingress Controller (NGINX Ingress)

**Scelta tecnologica:** NGINX Ingress Controller viene preferito a Traefik perché:
- Supporto nativo a WebSocket upgrades senza configurazione aggiuntiva.
- Rate limiting configurabile per annotazioni (via `nginx.ingress.kubernetes.io/limit-rps`).
- Supporto HTTP/3 tramite modulo QUIC di NGINX (versione 1.25+), necessario per WebTransport.
- Ecosistema maturo con supporto cert-manager tramite `cert-manager.io/cluster-issuer`.

**Tabella di routing Ingress:**

| Path prefix | Backend Service | Porta |
|-------------|----------------|-------|
| `/api/room` | room-service | 8080 |
| `/api/game` | game-engine | 8080 |
| `/api/npc` | npc-service | 8080 |
| `/api/rag` | rag-service | 8080 |
| `/transport` | transport-service | 9000 (HTTP/3) |
| `/` (default) | gateway | 8080 |

**Security headers aggiunti a tutte le risposte:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
```

### 2.4 Cloud Load Balancer

Il LB è gestito dal provider cloud e non fa parte del cluster. Responsabilità:

- Terminazione TLS (certificato wildcard o SAN gestito dal provider o da cert-manager tramite DNS-01).
- Healthcheck HTTP verso l'Ingress Controller (`GET /healthz`); rimozione dal pool dopo 2 fallimenti consecutivi.
- HTTP 502 se nessun backend risponde entro 30 secondi.
- WebSocket keep-alive abilitato (idle timeout ≥ 3600 s).
- HTTP/3 (QUIC) abilitato sulla porta 443 UDP per WebTransport.

### 2.5 Microservizi

Tutti deployati nel namespace `cinerole-internal` come Deployment Kubernetes con Service ClusterIP.

| Microservizio | Repliche baseline | HPA | Min repliche | Max repliche |
|---------------|-------------------|-----|-------------|-------------|
| gateway | 2 | Sì (80% CPU) | 2 | 10 |
| room-service | 2 | Sì (80% CPU) | 2 | 10 |
| game-engine | 1 | Sì (80% CPU) | 1 | 8 |
| npc-service | 1 | Sì (80% CPU) | 1 | 8 |
| rag-service | 1 | No | 1 | — |
| transport-service | 1 | No | 1 | — |

**Resource requests/limits indicativi (da calibrare con load test):**

| Microservizio | CPU request | CPU limit | Memory request | Memory limit |
|---------------|------------|-----------|---------------|-------------|
| gateway | 100m | 500m | 128Mi | 512Mi |
| room-service | 200m | 1000m | 256Mi | 1Gi |
| game-engine | 200m | 1000m | 256Mi | 1Gi |
| npc-service | 500m | 2000m | 512Mi | 2Gi |
| rag-service | 500m | 2000m | 1Gi | 4Gi |
| transport-service | 200m | 1000m | 256Mi | 1Gi |

> **Nota:** `npc-service` e `rag-service` hanno limiti più alti perché eseguono inferenza con GPT-4o e ricerca vettoriale; i valori vanno rifiniti dopo il primo profiling in staging.

### 2.6 CNI Plugin — Cilium

**Scelta tecnologica:** Cilium viene preferito a Calico perché:
- Enforcement NetworkPolicy a livello eBPF con visibilità L7 (HTTP, gRPC).
- Supporto nativo a Kubernetes NetworkPolicy standard e CiliumNetworkPolicy estese.
- Hubble integrato per observability del traffico di rete (audit trail dei flussi).
- Prerequisito Requisito 6.6: il CNI deve supportare l'enforcement effettivo.

### 2.7 cert-manager

Gestisce il ciclo di vita dei certificati TLS interni al cluster (Ingress, etcd mTLS).

- `ClusterIssuer` Let's Encrypt (produzione: ACME DNS-01; staging: ACME HTTP-01).
- Rinnovo automatico avviato 30 giorni prima della scadenza (Requisito 8.2).
- Alert Prometheus `certmanager_certificate_expiry_seconds < 2592000` (30 gg) a severità critica (Requisito 8.3).

### 2.8 OPA/Gatekeeper — Admission Webhook

Applica policy di sicurezza a tempo di admission:

- **Policy `no-public-service`:** rifiuta qualsiasi `Service` di tipo `NodePort` o `LoadBalancer` nei namespace `cinerole-exposed` e `cinerole-internal` senza annotazione `cinerole.io/public-service: "true"`.
- **Policy `require-resource-limits`:** rifiuta Pod senza `resources.limits` definiti.
- **Policy `no-privileged-containers`:** rifiuta container con `securityContext.privileged: true`.

### 2.9 Monitoring Stack

- **Prometheus** (namespace `monitoring`): scraping `/metrics` da tutti i Deployment, Ingress Controller, etcd, Node Exporter, cert-manager.
- **Alertmanager**: routing degli alert verso il canale configurato (Slack / PagerDuty / email).
- **Grafana**: dashboard per ogni microservizio (latenza p95, replica count, CPU/memory).

**Alert critici definiti:**

| Alert | Condizione | Severità |
|-------|-----------|---------|
| `DeploymentDown` | `kube_deployment_status_replicas_available == 0` per > 60 s | critical |
| `CertExpiryNear` | `certmanager_certificate_expiry_seconds < 2592000` | critical |
| `UnauthorizedService` | Service `NodePort`/`LoadBalancer` rilevato senza annotazione | critical |
| `NpcLatencyHigh` | p95 latenza npc-service > 3 s per > 5 min | warning |
| `EtcdSnapshotFailed` | Job snapshot etcd fallito | critical |

---

## Data Models

### 3.1 Struttura dei Namespace

```
cinerole-exposed
  ├── Deployment: ingress-nginx-controller
  └── Service: ingress-nginx-controller (LoadBalancer — unica eccezione)

cinerole-internal
  ├── Deployment: gateway
  ├── Deployment: room-service
  ├── Deployment: game-engine
  ├── Deployment: npc-service
  ├── Deployment: rag-service
  ├── Deployment: transport-service
  └── Secret: s3-shared-credentials

monitoring
  ├── Deployment: prometheus
  ├── Deployment: alertmanager
  └── Deployment: grafana

cert-manager
  └── Deployment: cert-manager (+ webhook + cainjector)
```

### 3.2 NetworkPolicy — Matrice dei Flussi Autorizzati

La policy predefinita è **default-deny ingress+egress** su tutti i namespace applicativi.

```
INGRESS CONTROLLER (cinerole-exposed)
  → gateway (cinerole-internal) [porta 8080]

gateway (cinerole-internal)
  → room-service      [porta 8080]
  → game-engine       [porta 8080]
  → transport-service [porta 9000]

game-engine (cinerole-internal)
  → npc-service       [porta 8080]

npc-service (cinerole-internal)
  → rag-service       [porta 8080]

transport-service (cinerole-internal)
  → room-service      [porta 8080]

TUTTI i namespace applicativi
  → DNS (kube-dns, porta 53 UDP/TCP)   [necessario per service discovery]
  → S3 endpoint esterno                [porta 443, solo dai Pod che montano il Secret]

monitoring
  → /metrics su tutti i namespace      [porta 9090 / path /metrics]
```

Ogni flusso non elencato è bloccato da Cilium prima di raggiungere il Pod di destinazione (Requisito 6.3).

### 3.3 Secret Kubernetes per Credenziali S3

```yaml
# Template — valori reali iniettati da CI/CD (es. Vault / AWS Secrets Manager)
apiVersion: v1
kind: Secret
metadata:
  name: s3-shared-credentials
  namespace: cinerole-internal
type: Opaque
data:
  AWS_ACCESS_KEY_ID: <base64>       # mai in chiaro nei manifesti
  AWS_SECRET_ACCESS_KEY: <base64>
  S3_ENDPOINT: <base64>
  S3_BUCKET_SHARED: <base64>
```

Il Secret viene montato come volume nel Pod (non come variabile d'ambiente leggibile nei log):

```yaml
volumeMounts:
  - name: s3-creds
    mountPath: /run/secrets/s3
    readOnly: true
volumes:
  - name: s3-creds
    secret:
      secretName: s3-shared-credentials
```

### 3.4 HPA — Configurazione Tipo

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gateway-hpa
  namespace: cinerole-internal
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gateway
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60      # attende 60 s prima di scale-out
      policies:
        - type: Pods
          value: 1
          periodSeconds: 90               # aggiunge max 1 replica ogni 90 s
    scaleDown:
      stabilizationWindowSeconds: 300     # attende 5 min prima di scale-in
```

---

## Error Handling

### 5.1 Failure di un Worker Node

**Scenario:** WN-X diventa `NotReady`.

1. `node-lifecycle-controller` rileva il nodo non raggiungibile dopo il timeout `node-not-ready` (default 5 min, configurabile con `--node-monitor-grace-period`).
2. I Pod sul nodo vengono marcati per eviction.
3. Il scheduler rischedulara i Pod sui Worker Node rimanenti (WN-Y, WN-Z).
4. Se le repliche disponibili di un Deployment scendono a 0, scatta l'alert `DeploymentDown` entro 60 s.
5. Nessun intervento manuale richiesto (Requisito 10.1).

**Decisione di design:** PodDisruptionBudget (PDB) con `minAvailable: 1` per gateway e room-service garantisce che durante l'eviction non vengano rimossi tutti i Pod contemporaneamente.

### 5.2 Failure di un Control Plane Node

**Scenario:** CP-X diventa non raggiungibile.

1. etcd mantiene il quorum con i 2 nodi rimanenti (Requisito 1.2 / 10.2).
2. `kube-apiserver` continua a rispondere sui nodi CP-Y e CP-Z (configurazione HA con VIP o load balancer interno per l'API Server).
3. I Pod esistenti continuano a girare senza interruzione.
4. Nuove richieste di scaling / deploy continuano a funzionare.

**Decisione di design:** il Cloud Load Balancer interno (o un VIP keepalived) distribuisce le chiamate all'API Server sui 3 control plane; in caso di failure di uno, il traffico viene reindirizzato automaticamente.

### 5.3 Failure Totale del Control Plane (caso estremo)

Se tutti e 3 i Control Plane Node diventano simultaneamente non raggiungibili (Requisito 10.4):
- etcd perde il write quorum: nessuna modifica allo stato del cluster viene accettata.
- I Pod **esistenti** continuano a girare perché i kubelet funzionano in modo autonomo per i Pod già schedulati.
- Nessuna nuova richiesta di scaling, deploy o riconfigurazione viene processata fino al ripristino.

### 5.4 Rinnovo Certificato Fallito

1. cert-manager tenta il rinnovo 30 giorni prima della scadenza.
2. Se il rinnovo fallisce (es. DNS-01 challenge non risolta), Alertmanager emette alert `CertExpiryNear` a severità critica.
3. L'operatore ha 30 giorni per intervenire prima che il certificato scada.
4. Il servizio rimane attivo durante il tentativo di rinnovo (il vecchio certificato è ancora valido).

### 5.5 S3 Condiviso Non Raggiungibile

Ogni microservizio che usa il bucket condiviso deve implementare il seguente comportamento (Requisito 11.5):
- Tentativo con retry esponenziale (max 3 tentativi, backoff: 1s, 2s, 4s).
- Dopo 3 tentativi falliti: log strutturato `{"level":"error","event":"s3_unavailable","service":"<nome>"}`.
- Risposta al chiamante con HTTP 503 `{"error": "storage_unavailable"}` — il microservizio **non** va in crash.
- Il circuit breaker (implementato a livello applicativo, es. con la libreria `resilience4j` o `opossum`) apre dopo 5 failure consecutive per evitare la cascata di errori.

### 5.6 Service Non Autorizzato Rilevato

1. OPA/Gatekeeper blocca la creazione del Service a tempo di admission con `400 Admission webhook denied`.
2. Se un Service non autorizzato fosse già presente (es. creato prima dell'installazione di Gatekeeper), Prometheus scraping della Gatekeeper audit policy genera l'alert `UnauthorizedService` verso Alertmanager (Requisito 7.4).

---

## Correctness Properties

> **Property-Based Testing non applicabile.** Questa feature riguarda esclusivamente
> configurazione infrastrutturale (manifesti Kubernetes, NetworkPolicy, HPA, cert-manager,
> OPA policies). Non esistono pure function con input/output testabili in modo universale:
> il comportamento è dichiarativo e deterministico, non parametrico su un grande spazio di input.
> La copertura della correttezza è garantita da snapshot tests, policy compliance checks e
> integration tests descritti nella sezione Testing Strategy.

### Property 1: N/A — Infrastructure as Code

Nessuna proprietà PBT applicabile. Vedere Testing Strategy per la strategia di verifica alternativa.

**Validates: Requirements 1.1, 2.1, 3.1, 4.1, 5.1, 6.1, 7.1, 8.1, 9.1, 10.1, 11.1**

---

## Testing Strategy

> **Nota sul Property-Based Testing:** questa feature riguarda esclusivamente configurazione
> infrastrutturale (manifesti Kubernetes, NetworkPolicy, HPA, cert-manager). Non esistono
> pure function con input/output sufficientemente variegati da giustificare PBT. La strategia
> di test è basata su snapshot tests, policy compliance checks e integration tests.

### 6.1 Snapshot Tests (manifesti Kubernetes)

Ogni manifesto Kubernetes (Deployment, Service, NetworkPolicy, HPA, Ingress) viene validato con:

- **`kubeconform`** (o `kubeval`): validazione dello schema Kubernetes rispetto alla versione target.
- **`kustomize build | kubeconform`**: verifica che la pipeline Kustomize produca manifesti validi prima di ogni deploy.
- Snapshot dei manifesti sintetizzati archiviati in CI: le modifiche strutturali inattese producono un diff visibile nel PR.

### 6.2 Policy Compliance Tests

- **OPA Conftest**: suite di test Rego che verificano:
  - Nessun Service di tipo `NodePort`/`LoadBalancer` nei namespace applicativi senza annotazione.
  - Tutti i Deployment hanno `resources.limits` definiti.
  - Nessun container con `privileged: true`.
  - Tutti i Pod hanno liveness e readiness probe configurate.
- I test Conftest vengono eseguiti nel job CI `policy-check` prima del deploy in staging.

### 6.3 Integration Tests (staging)

Ambiente staging: cluster Kubernetes di dimensioni ridotte (1 control plane, 2 worker), identico per configurazione al cluster di produzione.

| Test | Verifica | Frequenza |
|------|---------|----------|
| Healthcheck end-to-end | LB → Ingress → gateway risponde HTTP 200 | ogni deploy |
| WebSocket upgrade | Connessione WSS persistente per 60 s senza interruzione | ogni deploy |
| WebTransport | Connessione HTTP/3 verso transport-service stabilita | ogni deploy |
| HPA scale-out | Inject carico CPU >80% per 90 s → replica count cresce | nightly |
| NetworkPolicy denial | Pod non autorizzato non riesce a raggiungere room-service | ogni deploy |
| Cert renewal simulation | `kubectl cert-manager renew` → certificato aggiornato senza downtime | settimanale |
| etcd snapshot | Job snapshot eseguito → oggetto presente nel bucket S3 | nightly |
| S3 unavailable | Microservizio risponde HTTP 503 invece di crashare | ogni deploy |

### 6.4 Chaos Tests (produzione controllata)

Dopo la stabilizzazione in staging, si eseguono periodicamente (mensile):

- **Node failure drill:** spegnimento forzato di un Worker Node → verifica rischedulazione Pod entro 5 min.
- **Control plane failure drill:** spegnimento di un Control Plane Node → verifica continuità API Server.
- Strumento consigliato: **Chaos Mesh** (open source, integrato con Kubernetes).

### 6.5 CI/CD Pipeline

```
PR aperto
  └─ [job: lint]           kubeconform + kustomize build
  └─ [job: policy-check]   conftest run ./manifests/...
  └─ [job: unit-tests]     test unitari microservizi

Merge su main
  └─ [job: build-images]   build + push immagini Docker
  └─ [job: deploy-staging] kubectl apply su cluster staging (automatico)
  └─ [job: integration-tests] suite §6.3

Tag release
  └─ [approvazione manuale operatore]
  └─ [job: deploy-prod]    kubectl apply su cluster produzione (rolling update)
```

---

## Appendice: Decisioni Tecnologiche Chiave

| Decisione | Scelta | Alternativa scartata | Motivazione |
|-----------|--------|---------------------|-------------|
| CNI Plugin | Cilium | Calico | eBPF enforcement, Hubble observability, L7 policy |
| Ingress Controller | NGINX Ingress | Traefik | HTTP/3 nativo, WebSocket upgrade stabile, rate-limiting per annotazioni |
| Admission Control | OPA/Gatekeeper | Kyverno | Maturità, ecosistema Rego già conosciuto dal team |
| TLS Management | cert-manager | Gestione manuale | Rinnovo automatico, integrazione nativa con Ingress NGINX |
| Monitoring | Prometheus + Grafana | Datadog | Costo zero, stack open source standard per K8s |
| Chaos Testing | Chaos Mesh | LitmusChaos | Interfaccia web integrata, supporto nativo Kubernetes CRD |
