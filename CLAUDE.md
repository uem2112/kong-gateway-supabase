# Kong Gateway Supabase - Projektregeln

## IDENTITAET
- **Projekt:** Kong API Gateway - Zentraler Einstiegspunkt fuer Supabase + Custom Backends
- **Port:** 8000 (Proxy), 8001 (Admin, nur localhost)
- **Domain:** api.baucircle-beratung.de
- **Modus:** DB-less / Declarative Config

## TECH STACK
- Kong 3.5 (Alpine Docker)
- Declarative YAML Config (kong.yml)
- Traefik (Reverse Proxy, TLS/Let's Encrypt)
- Docker Compose + Coolify

## ROUTING (kong.yml)

| Route | Service | Backend | Port | Strip Path |
|-------|---------|---------|------|------------|
| /rest/v1/* | PostgREST | rest-sowwgwwc... | 3000 | Ja |
| /auth/v1/* | GoTrue | auth-sowwgwwc... | 8081 | Ja |
| /storage/v1/* | Storage | storage-sowwgwwc... | 5000 | Nein |
| /kaltakquise/* | Kaltakquise | kaltakquise-backend | 3350 | Ja |
| /sales/* | Sales Pipeline | sales-pipeline-backend | 3300 | Ja |

## ARCHITEKTUR-REGELN
- Alle Backends muessen ueber Kong geroutet werden
- `strip_path: true` bei allen Routes AUSSER Storage
- Admin API (8001) niemals oeffentlich exponieren
- CORS ist global konfiguriert - Aenderungen hier betreffen ALLE Services
- Neue Services: Route + Service in kong.yml hinzufuegen, Container neustarten

## CORS KONFIGURATION
Erlaubte Origins:
- baucircle-beratung.de (+ Subdomains)
- crm.planex-group.de
- lovable.dev / lovable.app / lovableproject.com
- localhost:3000, localhost:5173

## NETZWERKE
- `coolify` - Externes Netzwerk (alle Services)
- `sowwgwwc84coccwcwokwk8ss_default` - Supabase-internes Netzwerk

## WICHTIGE DATEIEN
- `docker-compose.yml` - Container-Konfiguration
- `kong.yml` - Aktive Routing-Config (PRODUKTIV)
- `kong.yml.example` - Template fuer neue Deployments
- `.env.example` - Environment Template

## HEALTH CHECK
- `kong health` alle 10s (Docker Health Check)
- Endpoints: /storage/v1/health, /auth/v1/health
