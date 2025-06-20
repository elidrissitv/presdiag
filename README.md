# Architecture Technique Détailée de la Plate-forme ENT

## 1. Vue d'ensemble

La plate-forme adopte un modèle **microservices** déployé via Docker Compose et orchestré par **Traefik**. Chaque domaine fonctionnel (authentification, gestion des cours, notes, messagerie, chatbot, etc.) est encapsulé dans un service isolé, communiquant principalement en **REST / JSON** et, pour l'asynchrone, via **RabbitMQ**.

- Sécurité : Keycloak (OIDC) assure l'authentification et la délégation des rôles. Les services Python valident les JWT via un _Auth Service_ dédié.
- Bases de données : Cassandra (données métier), Postgres (Keycloak), Qdrant (vecteurs IA), MinIO (objets), RabbitMQ (broker).
- Observabilité : Prometheus scrape les métriques exposées par chaque service FastAPI ; Grafana propose des tableaux de bord prêts-à-l'emploi.

## 2. Microservices Backend

| Service              | Port conteneur / Hôte | Stack                        | Rôle principal                       | Endpoints racine (Traefik) |
| -------------------- | --------------------- | ---------------------------- | ------------------------------------ | -------------------------- |
| auth-service         | 8000→8000             | FastAPI, PyJWT               | Validation des tokens, JWKS public   | `/api/auth`, `/test-jwks`  |
| user-service         | 8001→8001             | FastAPI, Cassandra           | CRUD utilisateurs & rôles            | `/api/users`               |
| service-cours        | 8002→8002             | FastAPI, Cassandra, MinIO    | Gestion & dépôt des cours            | `/api/cours`               |
| service-fichiers     | 8003→8003             | FastAPI, MinIO               | Serveur de fichiers sécurisés        | `/api/fichiers`            |
| notification-service | 8004→8004             | FastAPI, RabbitMQ, Cassandra | Push notifications (notes, support…) | `/api/notifications`       |
| calendar-service     | 8004→8005 (host)      | FastAPI, Cassandra           | Calendrier académique                | `/api/calendar`            |
| notes-service        | 8006→8006             | FastAPI, Cassandra           | Gestion des notes                    | `/api/notes`               |
| support-service      | 8008→8008             | FastAPI, Cassandra, RabbitMQ | Tickets support & FAQ                | `/api/support`             |
| chatbot-service      | 8082→8082             | FastAPI, Qdrant, Ollama      | IA conversationnelle, RAG            | `/chatbot`, `/docs`        |
| massagerie-service   | 8010→8010             | FastAPI, Cassandra, RabbitMQ | Messagerie interne                   | `/api/messages`            |
| calendar-service     | 8004→8005             | FastAPI, Cassandra           | Événements & emplois du temps        | `/api/calendar`            |
| Traefik              | 80 / 443 / 8080       | Go                           | API Gateway & TLS termination        | `/dashboard`               |
| RabbitMQ             | 5672 / 15672          | Erlang                       | Broker AMQP ; échanges asynchrones   | `/` (UI 15672)             |
| MinIO                | 9000 / 9001           | Go                           | Stockage objets S3                   | `/`                        |
| Qdrant               | 6333                  | Rust                         | Base vecteurs pour RAG               | `/collections`             |

Chaque service expose également `/health`, `/metrics/<service>` pour le monitoring.

## 3. Modèle de données

1. **Cassandra – keyspace `ent_keyspace`** :
   - `utilisateurs(id,…,role)`
   - `cours`, `consultation_cours`
   - `interactions_chatbot`
   - `notifications`
   - `calendar_events`
   - `messages`
   - `notes`
   - `support_requests`
2. **Postgres** : schéma interne Keycloak (utilisateurs, rôles, sessions).
3. **Qdrant** : collection `documents` (vectors, payload).
4. **MinIO** : bucket `cours` pour fichiers PDF/vidéos.

## 4. Communications inter-services

- **REST** synchrones via Traefik (port 80) : tous les appels `/api/<resource>`.
- **RabbitMQ** : notifications, messagerie interne, files `emails`, `support`, etc.
- **HTTP Streaming / WebSocket** (optionnel – non implémenté actuellement).

```graph TD
    frontend -- JWT --> traefik --> auth-service;
    frontend --> traefik --> user-service;
    notes-service -- REST --> notification-service;
    notes-service -- gRPC? (non) --> cassandra;
    chatbot-service -- Vector search --> qdrant;
    service-cours -- Upload --> minio;
    support-service -- AMQP --> rabbitmq;
```

## 5. Frontend `Portail`

- **React (+ CRA & CRACO)**, Tailwind/Custom CSS.
- Authentification : `keycloak-js`, stockage token en mémoire + refresh automatique.
- Structure des pages :
  1. `/login` – formulaire de connexion Keycloak.
  2. `/portail` – tableau de bord : services disponibles.
  3. `/cours` + `CoursModal` – liste & téléchargement des cours.
  4. `/chat` (`GlobalChat`) – chatbot IA.
  5. `/notes` – consultation des notes & formulaire enseignant.
  6. `/calendar` – calendrier intégré.
  7. `/messagerie` – messagerie interne.
  8. `/support` – création tickets.
  9. `/admin` – gestion utilisateurs (rôle `admin`).

`PrivateRoute` et `AdminRoute` protègent les pages en fonction des rôles (`etudiant`, `enseignant`, `admin`).

## 6. Sécurité & authentification

1. **Keycloak** gère : SSO, OAuth2/OIDC, rôles.
2. Les microservices valident le token via JWKS (cache) et vérifient le rôle.
3. Traefik peut être configuré avec HTTPS (certificats Let's Encrypt) ; ici, l'entrée `websecure` (:443) est prête.
4. MinIO, RabbitMQ, Prom/Grafana accessibles uniquement via réseau interne `ent_network` (sauf ports UI mappés).

## 7. Routage Traefik

- Détection automatique via labels Docker.
- Entrypoints : `web` (:80) et `websecure` (:443).
- Middlewares par défaut : `global-compress`, `security-headers`.
- Règles :`PathPrefix(/api/<service>)` → service correspondant.
- Tableau d'exemple :
  - `/api/notes` → notes-service (port 8006).
  - `/metrics/notes` → notes-service (port 8006) après `notes-metrics-strip`.
- Dashboard accessible sous `/dashboard` (protégé à activer en prod).
- Possibilité d'activer **Let's Encrypt** via `certificatesResolvers` pour HTTPS automatique.

## 8. Observabilité

- **Prometheus** scrape `:800X/metrics` + métriques Traefik.
- **Grafana** importe automatiquement un dashboard `grafana/provisioning/dashboards/microservices.json` (latence, taux d'erreurs, CPU…).
- Chaque service incrémente des counters via décorateur `@track_metrics`.

## 9. Déploiement & Exécution

1. `docker compose up -d --build` : construit images et crée réseau `ent_network`.
2. **Healthchecks** : Cassandra, Keycloak, RabbitMQ, services FastAPI.
3. **Init jobs** :
   - `cassandra-init` exécute `init.cql`.
   - `keycloak-init` configure realm / clients.
4. **Volumes** : persistance données (`cassandra_data`, `postgres_data`, `minio_data`, `qdrant_data`, `ollama_data`).
5. **Restart policy** : `unless-stopped` pour bases & chatbot, `on-failure` pour init jobs.

## 10. Répartition des responsabilités

Auteurs non explicitement indiqués dans le dépôt. Ajouter ce tableau manuellement le cas échéant :
| Domaine | Référent | Email |
|---------|----------|-------|
| Auth & Utilisateurs | – | – |
| Cours & Fichiers | – | – |
| IA / Chatbot | – | – |

## 11. Plan de rapport technique complet

1. Introduction & objectifs.
2. Architecture globale (diagrammes, microservices).
3. Sécurité & gouvernance des accès.
4. Détails Backend : par service (code, schémas de BDD, tests).
5. Détails Frontend : UX, flux d'auth, state management.
6. Infrastructure : Traefik, Docker / CI, réseaux, déploiement.
7. Observabilité & performances.
8. Plan de montée en charge & scalabilité.
9. Conclusion & perspectives (CI/CD, Kubernetes, IA générative…).

---

_Document généré automatiquement – mise à jour recommandée après chaque évolution majeure._
