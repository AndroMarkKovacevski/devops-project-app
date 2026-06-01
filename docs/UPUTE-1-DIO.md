# Upute za 1. dio projekta — Lokalno razvojno okruženje

Projekt: Secure Event Ticketing Platform
Cilj 1. dijela: pokrenuti cijelu aplikaciju (5 servisa) lokalno, jednom naredbom, s hot-reloadom, trajnom bazom i odvojenim tajnama.

---

## 1. Što gradimo (pregled u 1 minuti)

Aplikacija ima 5 servisa. 


frontend = Web stranica za kupnju karata; Node.js / Express; 3000 
api = REST API (eventi, narudžbe, health);  Node.js / Express; 8080 
worker = Pozadinska obrada narudžbi iz reda; Node.js; nema port 
postgres = Baza podataka (trajna pohrana narudžbi); PostgreSQL 16; 5432 
redis = Red (queue) i cache; Redis 7; 6379 

### Kako podaci teku (arhitektura)

Tijek kupnje karte:
1. Browser učita frontend (port 3000).
2. Frontend mu kaže API adresu; browser dohvaća listu evenata s api (port 8080).
3. Klik na Purchase → api gurne narudžbu u Redis red (status queued).
4. worker stalno čeka na redu, uzme narudžbu, upiše je u Postgres (status processed).
5. api čita obrađene narudžbe iz Postgresa (/tickets/orders).


## 2. Struktura datoteka

Konačan raspored:


devops-project-app/
    compose.yaml              MOJE-dodano  (orkestracija svih servisa)
    .env.example              DOPUNJENO (dodan API_BASE_URL)
    .env                      kreirano lokalno, NE commitaš
    .gitignore                MOJE-dodano  (da .env i node_modules ne odu u git)
    docs/
      security/
        image-scan-report.md  izvještaj skeniranja slika
        trivy-api.txt         trivy zapisi od api
        trivy-frontend.txt    trivy zapisi od frontend
        trivy-worker.txt      trivy zapisi od worker
      UPUTE-1-DIO.md          MOJE-dodano  (ova uputa)
      UPUTE-2-dio.md          uputa za 2. dio projekta
    api/
      Containerfile           MOJE-dodano
      .dockerignore           MOJE-dodano
      package.json          
       src/server.js         
    worker/
      Containerfile           MOJE-dodano
      .dockerignore           MOJE-dodano
      package.json          
      src/worker.js         
    frontend/
      Containerfile           MOJE-dodano
      .dockerignore           MOJE-dodano
      package.json          
      src/...               
    infra/
      postgres/init.sql
    k8s
      00-namespace.yaml
      01-rbac.yamk
      02-configmap.yaml
      03-secret.yaml
      04-postgres.yaml
      05-redis.yaml
      06-api.yaml
      07-worker.yaml
      08-frontend.yaml
      09-networkpolicy.yaml  


---

## 3. Što sadrži svaka datoteka (objašnjeno)

### 3.1 Containerfile (api / worker / frontend)

Containerfile je recept za izgradnju kontejnerske slike. Naš je multi-stage — ima više faza:

- base — postavi radni direktorij i kopira package.json (radi cache-a slojeva).
- development — instalira sve ovisnosti uključujući nodemon i pokreće npm run dev → hot-reload. Ovo lokalni compose koristi.
- prod-deps — instalira samo produkcijske ovisnosti (--omit=dev).
- production — uzima samo te ovisnosti + izvorni kod → minimalna slika, pokreće se kao non-root korisnik (USER node). Ovo je slika za 2. dio (registry, skeniranje, Kubernetes).


- *multi-stage build* — odvojene faze za dev i prod.
- *minimalna runtime slika* — production faza nosi samo runtime ovisnosti, na node:20-alpine (mala Alpine baza).
- *non-root korisnik* — USER node (uid 1000), ne radi kao root.

worker/Containerfile nema EXPOSE jer worker ne sluša ni na jednom portu (pozadinski proces).

### 3.2 .dockerignore

Govori build procesu da ne kopira node_modules, .env, .git itd. u build kontekst. Posljedica: brže buildanje, manje slike i lokalni node_modules ne prebriše one iz slike.

### 3.3 compose.yaml

Glavna datoteka koja opisuje svih 5 servisa i kako se povezuju. Ključni dijelovi:

- build.target: development — gradi dev fazu sa hot-reloadom.
- env_file: .env — ubaci varijable okruženja u kontejnere.
- ports — objavi portove na host (3000:3000 = host:kontejner).
- volumes (bind mount ./api/src:/app/src) — hot-reload: promjena koda na disku rezultira: kontejner ga odmah vidi.
- volumes (named pgdata) — trajnost baze. Podaci prežive restart i compose down (ne prežive down -v).
- init.sql mount — pri prvom pokretanju baze kreira tablicu ticket_orders.
- healthcheck — kontejner javlja je li zdrav.
- depends_on ... condition: service_healthy — api i worker čekaju da baza i redis budu zdravi prije pokretanja (rješava utrku pri startu).
- restart: unless-stopped — auto-restart ako servis padne.

### 3.4 .env.example i .env

- .env.example je predložak koji se commita u git (bez pravih tajni).
- Dodan API_BASE_URL=http://api-ticketing.apps.ocp4.example.com jer browser API zove preko host porta.

### 3.5 .gitignore

Sprječava da u git odu: .env (secrets), node_modules/, logovi, OS smeće


## 4. Pokretanje (startup) — jedna naredba

Iz korijena projekta: 

# kreiranje slike i pokretanje cijelog stacka
podman compose up --build

""Za rad u pozadini (oslobodi terminal) dodaj -d:
# bash
podman compose up --build -d
podman compose ps          # pregled statusa
podman compose logs -f api # prati logove API-ja


Kad su api i frontend healthy, spremno je.

## 5. Validacija (dokaz da radi)

bash
# 1) API health (mora vratiti {status:ok,service:api})
curl http://localhost:8080/healthz

# 2) Readiness (provjerava bazu + redis -> {status:ready})
curl http://localhost:8080/readyz

# 3) Lista evenata
curl http://localhost:8080/events

# 4) Kupnja karte (gura narudžbu u red -> status 202, vrati orderId)
curl -X POST http://localhost:8080/tickets/purchase \
  -H Content-Type: application/json \
  -d '{eventId:evt-1001,customerEmail:student@example.com,quantity:2}'

# 5) Provjera obrađene narudžbe
curl http://localhost:8080/tickets/orders


UI test: otvori http://localhost:3000, odaberi event, klikni *Purchase*. U Output polju dobiješ orderId.

## 6. Zaustavljanje (shutdown) — i blast radius

bash
# Zaustavi i ukloni kontejnere + mrežu, ZADRŽI podatke baze:
podman compose down

# Pauziraj bez uklanjanja:
podman compose stop

# Ponovno pokreni zaustavljeno:
podman compose start

DESTRUKTIVNO:

```bash
podman compose down -v
```
-v briše i named volume pgdata → gube se sve narudžbe u bazi. 
Blast radius: samo lokalni razvojni podaci (nema produkcije), ali nije povratno. 
-v samo kad se namjerno čisti baza. Novu tablicu init.sql kreira pri sljedećem "up"

---


## 7. Troubleshooting (česti problemi)

Simptom Uzrok Rješenje 
---------
port is already allocated Port 3000/8080/5432; zauzet; Ugasi proces koji ga drži ili promijeni port u .env 

api se vrti u restart petlji; Redis/Postgres još nisu spremni; Normalno par sekundi; depends_on čeka health; Provjeri podman compose logs api 

/tickets/orders prazno; Worker nije obradio ili tablica ne postoji; Provjeri podman compose logs worker; ako tablica fali, baza je bila inicijalizirana bez init.sql → down -v pa up 

Promjena koda se ne vidi; Bind mount/SELinux Provjeri da editiraš datoteke u */src. Na Fedora/RHEL+Podman dodaj :Z na mount 

permission denied na init.sql (Podman/SELinux) SELinux labela U compose.yaml promijeni :ro u :ro,Z 

UI ne učita evente; Krivi API_BASE_URL Mora biti http://localhost:8080, a api mora objaviti port 8080 

npm ci greška u buildu; Nema package-lock.json; Koristi npm install (kao u priloženim datotekama) ili generiraj lockfile

### Korisne dijagnostičke naredbe
# bash
podman compose ps                 # status + health svih servisa
podman compose logs -f <servis>   # logovi (api, worker, frontend, postgres, redis)
podman compose exec api sh        # uđi u shell API kontejnera
podman compose exec postgres psql -U ticketing_user -d ticketing -c SELECT * FROM ticket_orders;
podman compose down && podman compose up --build   # čisti ponovni start (zadrži podatke)

