# Kong API Gateway for Supabase

Production-ready Kong Gateway für Self-Hosted Supabase in Coolify/Traefik Umgebungen.

## Übersicht

Dieses Repository enthält eine Kong Gateway Konfiguration, die einen **einheitlichen API Endpoint** für alle Supabase Services bereitstellt (PostgREST, GoTrue, Storage, Realtime, Functions).

### Problem ohne Kong

```
❌ Direkter Zugriff auf einzelne Services:
- http://rest:3000       → PostgREST
- http://auth:8081       → GoTrue
- http://storage:5000    → Storage

Probleme:
- Kein einheitlicher Endpoint
- Storage Uploads fehlschlagen mit "Route not found"
- Nicht wie managed Supabase (supabase.co/project-ref)
```

### Lösung mit Kong

```
✅ Einheitlicher Gateway Endpoint:
https://api.your-domain.com

Routes:
- /rest/v1/*     → PostgREST (rest:3000)
- /auth/v1/*     → GoTrue (auth:8081)
- /storage/v1/*  → Storage (storage:5000)
- /realtime/v1/* → Realtime (realtime:4000) [optional]
- /functions/v1/* → Functions (functions:9000) [optional]
```

---

## Features

✅ **Einheitlicher API Endpoint** wie managed Supabase
✅ **DB-less Mode** (declarative config) - keine Kong Database nötig
✅ **Traefik Integration** - TLS/HTTPS über Let's Encrypt
✅ **CORS konfiguriert** - Ready für Browser-Apps
✅ **Production Ready** - Health Checks, Logging, Restart Policy
✅ **Template-Ready** - Einfach für mehrere Kunden duplizierbar

---

## Voraussetzungen

- Docker & Docker Compose
- Coolify mit Traefik
- Selbst-gehostete Supabase Services (rest, auth, storage, etc.)
- Alle Services im gleichen Docker Network (`coolify`)

---

## Quick Start

### 1. Repository clonen

```bash
git clone https://github.com/your-org/kong-gateway-supabase.git
cd kong-gateway-supabase
```

### 2. Supabase Container-Namen herausfinden

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}" | grep -E "(rest|auth|storage)"
```

Beispiel-Output:
```
rest-sowwgwwc84coccwcwokwk8ss      127.0.0.1:3000->3000/tcp
auth-sowwgwwc84coccwcwokwk8ss      127.0.0.1:8081->8081/tcp
storage-sowwgwwc84coccwcwokwk8ss   127.0.0.1:5000->5000/tcp
```

### 3. kong.yml konfigurieren

**Option A: Von Template kopieren**

```bash
cp kong.yml.example kong.yml
nano kong.yml
```

**Option B: Template bereits angepasst nutzen** (baucircle-beratung.de)

Die `kong.yml` ist bereits mit den Container-Namen konfiguriert. Prüfe sie:

```bash
cat kong.yml
```

**Ersetze folgende Werte:**

```yaml
services:
  - name: postgrest-service
    url: http://rest-YOUR-CONTAINER-NAME:3000  # ← Anpassen!

  - name: gotrue-service
    url: http://auth-YOUR-CONTAINER-NAME:8081  # ← Anpassen!

  - name: storage-service
    url: http://storage-YOUR-CONTAINER-NAME:5000  # ← Anpassen!

plugins:
  - name: cors
    config:
      origins:
        - "https://your-domain.com"  # ← Deine Frontend Domain
```

### 4. Environment konfigurieren

```bash
cp .env.example .env
nano .env
```

Setze deine Gateway Domain:

```bash
GATEWAY_DOMAIN=api.your-domain.com
```

### 5. DNS Eintrag erstellen

Erstelle einen A-Record:

```
api.your-domain.com  →  [Server-IP]
```

Verifiziere:

```bash
dig api.your-domain.com +short
```

### 6. Deployment

**Option A: Via Coolify (empfohlen)**

1. Coolify öffnen → Neues Projekt erstellen: "Kong Gateway"
2. "+ Add Resource" → "New Service" → "Docker Compose"
3. Git Repository: Dieses Repo
4. Branch: `main`
5. Docker Compose Path: `docker-compose.yml`
6. **Deploy!**

**Option B: Manuell per CLI**

```bash
# Auf Server
mkdir -p /opt/kong-gateway
cd /opt/kong-gateway

# Dateien hochladen
scp docker-compose.yml kong.yml .env root@your-server:/opt/kong-gateway/

# Starten
docker compose up -d

# Logs prüfen
docker logs -f supabase-kong-gateway
```

### 7. Verify Deployment

**Erwartete Logs:**

```
✓ declarative config loaded
✓ starting proxy
```

**Test Endpoints:**

```bash
# Gateway erreichbar
curl -I https://api.your-domain.com

# PostgREST Route
curl https://api.your-domain.com/rest/v1/

# GoTrue Health
curl https://api.your-domain.com/auth/v1/health

# Storage Health
curl https://api.your-domain.com/storage/v1/health
```

**Alle sollten 200 oder 401 zurückgeben (nicht 404 oder 502)!**

---

## Backend Integration

### ENV Variablen aktualisieren

**Vorher (❌ falsch):**

```bash
SUPABASE_URL=http://rest-project:3000
SUPABASE_STORAGE_URL=http://storage-project:5000
```

**Nachher (✅ korrekt):**

```bash
# Ein einziger Gateway Endpoint
SUPABASE_URL=https://api.your-domain.com
SUPABASE_SERVICE_KEY=your-service-key
SUPABASE_STORAGE_BUCKET=your-bucket

# SUPABASE_STORAGE_URL ist NICHT mehr nötig!
```

### Backend neu starten

```bash
# In Coolify: Restart Service
# Oder per CLI:
docker restart your-backend-container
```

---

## Konfiguration

### Routing-Details

| Service | Port | Kong Route | Strip Path? | Warum? |
|---------|------|------------|-------------|---------|
| PostgREST | 3000 | `/rest/v1` | ✅ Ja | PostgREST erwartet `/` |
| GoTrue | 8081 | `/auth/v1` | ✅ Ja | GoTrue erwartet `/` |
| Storage | 5000 | `/storage/v1` | ❌ Nein | Storage erwartet `/storage/v1` |
| Realtime | 4000 | `/realtime/v1` | ❌ Nein | Realtime erwartet `/realtime/v1` |
| Functions | 9000 | `/functions/v1` | ❌ Nein | Functions erwartet `/functions/v1` |

### CORS Origins anpassen

In `kong.yml`:

```yaml
plugins:
  - name: cors
    config:
      origins:
        - "https://your-domain.com"           # Frontend
        - "https://app.your-domain.com"       # App
        - "http://localhost:3000"             # Local Dev
        - "http://localhost:5173"             # Vite
```

### Optionale Services aktivieren

**Realtime:**

```yaml
- name: realtime-service
  url: http://realtime-YOUR-CONTAINER:4000
  routes:
    - name: realtime-route
      paths:
        - /realtime/v1
      strip_path: false
```

**Edge Functions:**

```yaml
- name: functions-service
  url: http://functions-YOUR-CONTAINER:9000
  routes:
    - name: functions-route
      paths:
        - /functions/v1
      strip_path: false
```

---

## Troubleshooting

### Problem: "Route not found"

**Symptom:**

```json
{"message": "Route POST:/storage/v1/object/... not found"}
```

**Lösungen:**

1. **Falsche Container-Namen in kong.yml**

```bash
# Prüfe echte Namen
docker ps --format "{{.Names}}" | grep storage

# Aktualisiere kong.yml mit echtem Namen
# url: http://storage-ECHTER-NAME:5000

# Restart Kong
docker restart supabase-kong-gateway
```

2. **Kong kann Services nicht erreichen**

```bash
# Test von Kong aus
docker exec supabase-kong-gateway ping storage-container-name

# Prüfe Network
docker network inspect coolify | grep -A 5 kong
docker network inspect coolify | grep -A 5 storage
```

3. **Falsches strip_path**

```yaml
# Storage: strip_path MUSS false sein!
- name: storage-service
  routes:
    - strip_path: false  # ← WICHTIG!
```

### Problem: 502 Bad Gateway

**Ursachen:**

1. Supabase Service ist down

```bash
docker ps | grep storage
docker logs storage-container-name
```

2. Falscher Port in kong.yml

```yaml
# PostgREST: Port 3000 (nicht 3001!)
# GoTrue: Port 8081 (nicht 9999!)
# Storage: Port 5000 (nicht 5001!)
```

3. Nicht im gleichen Network

```bash
docker network inspect coolify | grep -E "(kong|rest|auth|storage)"
```

### Problem: CORS Errors

**Symptom:**

```
Access to fetch at 'https://api.../storage/v1/...' has been blocked by CORS policy
```

**Lösung:**

1. Füge Frontend Domain zu CORS Origins hinzu:

```yaml
plugins:
  - name: cors
    config:
      origins:
        - "https://your-frontend-domain.com"  # ← Hinzufügen!
```

2. Restart Kong:

```bash
docker restart supabase-kong-gateway
```

3. Test CORS:

```bash
curl -I -X OPTIONS \
  -H "Origin: https://your-frontend.com" \
  https://api.your-domain.com/storage/v1/health

# Sollte enthalten:
# Access-Control-Allow-Origin: https://your-frontend.com
```

### Problem: Kong startet nicht

**Logs prüfen:**

```bash
docker logs supabase-kong-gateway
```

**Häufige Fehler:**

1. **"failed to load declarative config"** - Syntax-Fehler in kong.yml

```bash
# Prüfe Syntax
docker exec supabase-kong-gateway kong config parse /kong/declarative/kong.yml
```

2. **"No such file"** - Volume nicht gemountet

```bash
# Prüfe Mounts
docker inspect supabase-kong-gateway | grep -A 5 Mounts
```

3. **"network not found"** - Coolify Network fehlt

```bash
# Prüfe Network
docker network ls | grep coolify
```

---

## Monitoring & Logs

### Kong Logs live verfolgen

```bash
docker logs -f supabase-kong-gateway
```

### Nur Errors anzeigen

```bash
docker logs supabase-kong-gateway 2>&1 | grep -i error
```

### Kong Admin API (nur intern!)

```bash
# Health Status
curl http://localhost:8001/status

# Alle Routes anzeigen
curl http://localhost:8001/routes | jq

# Alle Services anzeigen
curl http://localhost:8001/services | jq
```

⚠️ **WICHTIG:** Admin API (Port 8001) ist nur auf localhost erreichbar!

---

## Production Best Practices

### Security

✅ Admin API nur localhost (✓ bereits konfiguriert)
✅ Kong läuft ohne Root (Alpine Image)
✅ Kein direkter Port-Expose (nur via Traefik)
✅ CORS richtig konfiguriert

### Monitoring

```bash
# Health Check
curl https://api.your-domain.com/storage/v1/health

# Logs überwachen
docker logs -f supabase-kong-gateway | grep -v "GET /storage/v1/health"
```

### Backup

```bash
# Kong Config ist in Git ✓
# Bei Änderungen:
git add kong.yml
git commit -m "chore: Update container names"
git push
```

### Updates

```bash
# Kong Image updaten
docker compose pull
docker compose up -d

# Logs prüfen
docker logs supabase-kong-gateway
```

---

## Skalierung für mehrere Kunden

### Template-Struktur

```bash
# kunde1
api.kunde1.com → Kong Gateway (eigene Instanz)

# kunde2
api.kunde2.com → Kong Gateway (eigene Instanz)
```

### Deployment Script

```bash
#!/bin/bash
CUSTOMER=$1
DOMAIN=$2

# Clone template
git clone https://github.com/your-org/kong-gateway-supabase.git $CUSTOMER-kong
cd $CUSTOMER-kong

# Update config
sed -i "s/api.example.com/$DOMAIN/g" .env
sed -i "s/YOUR-CONTAINER/$CUSTOMER-supabase/g" kong.yml

# Deploy
docker compose up -d
```

---

## Support

### Logs analysieren

```bash
docker logs supabase-kong-gateway
```

### Network prüfen

```bash
docker network inspect coolify
```

### Container prüfen

```bash
docker ps | grep -E "(kong|rest|auth|storage)"
```

### Kong Config validieren

```bash
docker exec supabase-kong-gateway kong config parse /kong/declarative/kong.yml
```

---

## Changelog

### v1.0.0 (2026-01-23)

- Initial Release
- Kong 3.5 Alpine
- Declarative Config
- Traefik Integration
- CORS Plugin
- Health Checks
- Production Ready

---

## License

MIT

---

## Author

Created for Baucircle Beratung GmbH

---

## Links

- [Kong Documentation](https://docs.konghq.com/)
- [Supabase Self-Hosting](https://supabase.com/docs/guides/self-hosting)
- [Coolify Documentation](https://coolify.io/docs)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
