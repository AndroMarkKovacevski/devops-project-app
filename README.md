# Secure Event Ticketing Platform — DevSecOps projekt

Višeslojna aplikacija isporučena kroz cijeli DevOps/DevSecOps ciklus: lokalni razvoj kroz kontejnere (Docker/Podman Compose) i produkcijska orkestracija na Kubernetes/OpenShift okruženju.

Projekt za kolegij *Uvod u DevOps – DevSecOps*, Sveučilište Algebra Bernays.

## Servisi

| Servis | Uloga | Tehnologija | Port |
|---|---|---|---|
| frontend | Web sučelje za kupnju karata | Node.js / Express | 3000 |
| api | REST API (eventi, narudžbe, health) | Node.js / Express | 8080 |
| worker | Pozadinska obrada narudžbi iz reda | Node.js | — |
| postgres | Trajna pohrana narudžbi | PostgreSQL 16 | 5432 |
| redis | Red (queue) i cache | Redis 7 | 6379 |

Tok kupnje: browser → frontend → api → (red u Redisu) → worker → upis u Postgres.

## Struktura repozitorija

```
.
├── api/ worker/ frontend/   # izvorni kod servisa + Containerfile + .dockerignore
├── infra/postgres/init.sql  # inicijalizacija baze
├── compose.yaml             # 1. dio — lokalni razvoj (sve jednom naredbom)
├── .env.example             # primjer konfiguracije (kopiraj u .env)
├── k8s/                     # 2. dio — Kubernetes/OpenShift manifesti
├── .github/workflows/ci.yaml# (bonus) CI: build + Trivy + push
└── docs/                    # upute, runbook, sigurnosno izvješće, izvještaj
```

## Brzi start — 1. dio (lokalno)

```bash
cp .env.example .env
docker compose up --build        # ili: podman compose up --build
```
Validacija:
```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
curl http://localhost:8080/events
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
curl http://localhost:8080/tickets/orders     # narudžba sa "status":"processed"
```
UI: otvori `http://localhost:3000`. Detaljno: [`docs/UPUTE-1-DIO.md`](docs/UPUTE-1-DIO.md).

## Brzi start — 2. dio (produkcija)

- Kubernetes (npr. CentOS 9): [`docs/UPUTE-2-DIO.md`](docs/UPUTE-2-DIO.md)
- OpenShift (DO180, samo `oc`/`podman`): [`docs/UPUTE-2-DIO-OPENSHIFT.md`](docs/UPUTE-2-DIO-OPENSHIFT.md)

## Sigurnosni elementi

Multi-stage build i non-root runtime, odvojeni Secret/ConfigMap (bez tajni u kodu), liveness/readiness probe, resource requests/limits, ServiceAccount + RBAC, NetworkPolicy segmentacija, te skeniranje slika alatom Trivy ([`docs/security/image-scan-report.md`](docs/security/image-scan-report.md)).

## Dokumentacija

- [`docs/UPUTE-1-DIO.md`](docs/UPUTE-1-DIO.md) — lokalno okruženje
- [`docs/UPUTE-2-DIO.md`](docs/UPUTE-2-DIO.md) / [`docs/UPUTE-2-DIO-OPENSHIFT.md`](docs/UPUTE-2-DIO-OPENSHIFT.md) — produkcija
- [`docs/RUNBOOK.md`](docs/RUNBOOK.md) — incidenti i troubleshooting
- [`docs/security/image-scan-report.md`](docs/security/image-scan-report.md) — sigurnosno izvješće
- [`docs/Izvjestaj-PRIMJER.docx`](docs/Izvjestaj-PRIMJER.docx) — predložak završnog izvještaja
