# Architecture technique — Application web de tracking basée sur numéro de téléphone

## 1) Vision produit

Construire une plateforme de suivi de localisation **avec consentement explicite**, pilotée depuis un tableau de bord web/mobile, permettant :

- le partage de localisation en temps réel ou périodique ;
- la consultation d'un historique limité dans le temps ;
- des alertes d'entrée/sortie de zones (geofencing) ;
- une exploitation conforme aux exigences de sécurité et de conformité légale.

> **Important** : un numéro de téléphone seul ne permet pas de localiser un utilisateur en continu de façon fiable et légale sans son implication active. La solution repose donc sur une application/mobile web app installée par l'utilisateur, liée à son numéro après vérification OTP.

---

## 2) Principes de conformité (privacy by design)

1. **Consentement explicite préalable**
   - Écran de consentement clair (finalités, fréquence, durée de conservation, partage des données).
   - Consentement versionné et journalisé (timestamp, version CGU, portée).
   - Révocation à tout moment depuis l'application.

2. **Minimisation des données**
   - Collecter uniquement : latitude, longitude, précision, timestamp, batterie, device_id pseudonymisé.
   - Pas de collecte de données non nécessaires (contacts, SMS, etc.).

3. **Durée de rétention configurable**
   - TTL configurable par tenant/politique (ex. 7/30/90 jours).
   - Purge automatique (jobs planifiés + politique DB de partition/expiration).

4. **Transparence et droits utilisateur**
   - Export de données (format JSON/CSV).
   - Suppression de compte et effacement des traces (hors obligations légales).
   - Journal d'accès administrateur.

5. **Cadre légal**
   - RGPD (UE) : base légale, DPA, registre des traitements, DPIA recommandé.
   - Politique de confidentialité explicite et mécanisme de preuve de consentement.

---

## 3) Architecture globale

## 3.1 Composants

1. **Client mobile / PWA**
   - Capture GPS (foreground/background selon OS).
   - Envoi périodique ou streaming par intervalle adaptatif.
   - Gestion locale du consentement et des permissions système.

2. **API Gateway (REST + WebSocket/SSE)**
   - Authentification JWT/OAuth2.
   - Rate limiting, WAF, validation schéma.
   - Routage vers services backend.

3. **Service Auth & Identity**
   - Vérification numéro via OTP (SMS provider).
   - Gestion comptes, rôles (admin, superviseur, utilisateur).
   - Rotation et révocation des tokens.

4. **Service Location Ingestion**
   - Endpoint sécurisé `/v1/locations`.
   - Validation (timestamp anti-rejeu, précision minimum, signature device).
   - Publication en file de messages (Kafka/RabbitMQ/SQS).

5. **Service Geofencing**
   - Évalue les événements de position vs zones définies.
   - Génère événements `ENTER`, `EXIT`, `DWELL`.
   - Déduplication et anti-spam d'alertes.

6. **Service Notifications**
   - Push (FCM/APNS), email, webhook.
   - Stratégie de retry + DLQ (dead-letter queue).

7. **Service History & Query**
   - Requêtes temporelles et spatiales.
   - Agrégations pour dashboard (trajets, distance, heatmap).

8. **Datastores**
   - **PostgreSQL + PostGIS** : positions, zones, comptes, consentements.
   - **Redis** : cache, sessions, présence temps réel.
   - **Object storage** (optionnel) : exports, rapports.

9. **Backoffice Dashboard (web)**
   - Carte temps réel, timeline, gestion des zones.
   - Paramètres de politique (rétention, fréquence, notification).

10. **Observabilité & sécurité**
   - Logs structurés, métriques, traces distribuées.
   - SIEM + alertes sécurité + audit trails.

## 3.2 Flux nominal

1. L'utilisateur crée un compte avec son numéro.
2. OTP validé, consentement explicite accepté.
3. L'app collecte la position (selon fréquence autorisée).
4. API reçoit, chiffre/valide, envoie au bus.
5. Worker geofence détecte entrée/sortie, publie événement.
6. Notification envoyée selon règles.
7. Dashboard lit données historisées/temps réel.

---

## 4) Modèle de données (simplifié)

## 4.1 Tables clés

- `users(id, phone_e164, status, created_at)`
- `devices(id, user_id, platform, public_key, last_seen_at)`
- `consents(id, user_id, scope, granted_at, revoked_at, policy_version)`
- `locations(id, user_id, device_id, ts, geom, accuracy_m, speed_mps, battery_pct, source)`
- `geofences(id, tenant_id, name, geom_polygon, active, created_by)`
- `geofence_events(id, geofence_id, user_id, event_type, ts, location_id)`
- `notification_rules(id, tenant_id, channel, trigger, cooldown_sec, enabled)`
- `audit_logs(id, actor_id, action, target_type, target_id, ts, metadata)`

## 4.2 Indexation recommandée

- `locations(user_id, ts DESC)`
- index spatial PostGIS sur `locations.geom` et `geofences.geom_polygon`
- partitionnement temporel mensuel/hebdomadaire selon volumétrie

---

## 5) APIs REST proposées

### Auth & consentement

- `POST /v1/auth/request-otp`
- `POST /v1/auth/verify-otp`
- `POST /v1/consents/grant`
- `POST /v1/consents/revoke`
- `GET /v1/consents/me`

### Localisation

- `POST /v1/locations` (batch support)
- `GET /v1/locations/latest?user_id=...`
- `GET /v1/locations/history?user_id=...&from=...&to=...`

### Geofencing

- `POST /v1/geofences`
- `GET /v1/geofences`
- `PUT /v1/geofences/{id}`
- `DELETE /v1/geofences/{id}`
- `GET /v1/geofences/events?from=...&to=...`

### Notifications

- `POST /v1/notification-rules`
- `GET /v1/notifications/events`

### Administration & conformité

- `GET /v1/audit-logs`
- `POST /v1/privacy/export`
- `DELETE /v1/privacy/account`

---

## 6) Sécurité technique

1. **Chiffrement en transit**
   - TLS 1.2+ partout, HSTS, pinning mobile (si possible).

2. **Chiffrement au repos**
   - Chiffrement volume + colonnes sensibles (numéro tel tokenisé/chiffré).
   - KMS/HSM pour gestion des clés et rotation planifiée.

3. **Contrôle d'accès**
   - RBAC/ABAC côté API.
   - Segmentation multi-tenant stricte.
   - Principe du moindre privilège (IAM).

4. **Protection API**
   - JWT courts + refresh token rotatif.
   - Anti brute-force OTP, quotas par IP/device.
   - Validation stricte payload + signature horodatée.

5. **Audit & détection**
   - Journalisation immuable des accès sensibles.
   - Détection anomalies (positions impossibles, spoofing GPS).

---

## 7) Scalabilité & résilience

- Architecture stateless pour services API (horizontal scaling).
- File de messages pour découpler ingestion et traitements.
- Workers autoscalés selon backlog.
- Read replicas PostgreSQL pour requêtes dashboard.
- Circuit breakers, retries exponentiels, idempotence des événements.
- Multi-AZ recommandé ; RPO/RTO définis selon SLA.

---

## 8) Tableau de bord (web/mobile)

### Écrans principaux

1. **Vue temps réel**
   - Carte avec position actuelle, statut (en ligne/hors ligne), précision GPS.

2. **Historique**
   - Filtre temporel + lecture de trajet.
   - Limite explicite par politique de rétention.

3. **Geofences**
   - Création zones (cercle/polygone), activation/désactivation.

4. **Alertes**
   - Flux d'événements + accusé de lecture + export.

5. **Conformité / Paramètres**
   - Consentements, logs d'accès, suppression/export des données.

---

## 9) Stratégie de déploiement (référence)

- **Frontend** : React/Next.js (dashboard web), app mobile (React Native/Flutter) ou PWA avancée.
- **Backend** : Node.js/NestJS ou Go (API + workers).
- **DB** : PostgreSQL + PostGIS.
- **Infra** : Kubernetes + Ingress + cert-manager + secrets manager.
- **CI/CD** : tests auto, scans SAST/DAST, déploiement progressif (canary).

---

## 10) Roadmap de livraison

### Phase 1 — MVP (6–8 semaines)

- OTP + consentement explicite
- Envoi localisation périodique
- Carte temps réel + historique 7 jours
- Geofence simple (entrée/sortie)

### Phase 2 — Production readiness

- Hardening sécurité, audit complet, exports RGPD
- Scalabilité (queue + workers + réplication)
- Alerting avancé, monitoring SLO/SLA

### Phase 3 — Avancé

- Détection d'anomalies / fraude GPS
- Optimisation batterie intelligente
- Reporting BI multi-tenant

---

## 11) KPI produit / exploitation

- Latence d'affichage position temps réel (P95)
- Taux de livraison des notifications
- Taux d'échec ingestion
- Précision moyenne GPS
- Taux de consentement actif
- Respect SLA purge données (TTL)

---

## 12) Risques & mitigations

- **Risque légal** : usage sans consentement -> blocage fonctionnel sans consentement valide.
- **Risque batterie** : tracking trop fréquent -> fréquence adaptive + mode économie.
- **Risque faux positifs geofence** : jitter GPS -> seuil de confiance + dwell time.
- **Risque charge** : pics d'ingestion -> queue + autoscaling + backpressure.

---

## 13) Conclusion

Cette architecture permet de livrer une application de tracking exploitable en production, **centrée sur le consentement**, techniquement scalable et conforme aux exigences de sécurité (chiffrement transit/repos) et de gouvernance des données.
