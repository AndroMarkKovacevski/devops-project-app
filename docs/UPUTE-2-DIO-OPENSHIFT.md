# Deploy na OpenShift (DO180) — samo `oc` i `podman`

## Pregled koraka

1. Projekt (namespace)
2. Build slika (Metoda A: `oc new-build`  **ili**  Metoda B: `podman build` + push)
3. SCC dozvola (da slike rade na OpenShiftu)
4. Konfiguracija + tajna
5. Baza, Redis, pa aplikacija
6. Mreža (NetworkPolicy) + vanjski pristup (Route)
7. Validacija
8. (odvojeno) Trivy sigurnosni artefakt
9. Rolling update / rollback



## 1. Projekt

```bash
oc new-project ticketing
# ako vec postoji:  oc project ticketing
```

## 2. Build slika

OpenShift ne povlači lokalne `podman` slike automatski — sliku treba dobiti u klaster. Imaš dvije metode; odaberi jednu.

### Metoda A — build unutar klastera (`oc new-build`)  ✅ preporučeno
Ne treba ti vanjski registry ni izlaganje internog. OpenShift sam sagradi i spremi sliku u interni registry kao ImageStream.

```bash
for s in api worker frontend; do
  # 1) kreiraj build (docker strategija = gradi po Containerfile-u)
  oc new-build --name=ticketing-$s --binary --strategy=docker -n ticketing

  # 2) reci buildu da je datoteka "Containerfile" (a ne "Dockerfile")
  oc -n ticketing patch bc/ticketing-$s --type=merge \
    -p '{"spec":{"strategy":{"dockerStrategy":{"dockerfilePath":"Containerfile"}}}}'

  # 3) pošalji sadržaj foldera servisa i pokreni build (gradi zadnju fazu = production)
  oc start-build ticketing-$s --from-dir=./$s --follow -n ticketing

  # 4) označi semantičkim tagom (umjesto 'latest')
  oc tag ticketing-$s:latest ticketing-$s:1.0.0 -n ticketing
done
```

Slike su sada u internom registryju na putanji:
`image-registry.openshift-image-registry.svc:5000/ticketing/ticketing-<servis>:1.0.0`

Poveži manifeste s tom putanjom (uređuje `image:` polja jednom naredbom):
```bash
REG=image-registry.openshift-image-registry.svc:5000/ticketing
sed -i "s#image: ticketing-api:1.0.0#image: $REG/ticketing-api:1.0.0#"           k8s/06-api.yaml
sed -i "s#image: ticketing-worker:1.0.0#image: $REG/ticketing-worker:1.0.0#"     k8s/07-worker.yaml
sed -i "s#image: ticketing-frontend:1.0.0#image: $REG/ticketing-frontend:1.0.0#" k8s/08-frontend.yaml
```

### Metoda B — `podman build` + push u interni registry
Ako želiš graditi lokalno podmanom. Treba izložen interni registry (zahtijeva cluster-admin; na CRC `kubeadmin` ga ima).

```bash
# (jednom) izloži interni registry rutom:
oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge \
  -p '{"spec":{"defaultRoute":true}}'
REGHOST=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')

# login podmanom (token tvog korisnika):
podman login -u $(oc whoami) -p $(oc whoami -t) $REGHOST --tls-verify=false

# build PRODUKCIJSKE faze, tag i push za svaki servis:
for s in api worker frontend; do
  podman build --target production -t $REGHOST/ticketing/ticketing-$s:1.0.0 ./$s
  podman push --tls-verify=false $REGHOST/ticketing/ticketing-$s:1.0.0
done
```
Zatim isti `sed` kao u Metodi A (slike su na istoj internoj putanji).



## 3. SCC dozvola (da slike rade pod OpenShiftom)

OpenShift po defaultu pokreće kontejnere s nasumičnim UID-om (restricted SCC). Službene `postgres`/`redis`/`node` slike to ne vole. Najjednostavnije rješenje za lab: dopusti `anyuid` našem ServiceAccountu.

```bash
# Prvo kreiraj ServiceAccount (iz RBAC manifesta):
oc apply -f k8s/01-rbac.yaml

# Dopusti anyuid (treba cluster-admin; kubeadmin na CRC ga ima):
oc adm policy add-scc-to-user anyuid -z ticketing-sa -n ticketing
```

> **Što ovo znači / sigurnosni kompromis:** `anyuid` dopušta kontejneru da radi pod UID-om koji slika traži (npr. postgres kao 999, node kao 1000). To je standardna lab-praksa, ali je manje restriktivno od "arbitrary UID". Za stroži (sigurniji) pristup vidi poglavlje 10 — ondje ne koristiš `anyuid`, nego prilagodiš slike/securityContext. Na obrani spomeni da si svjesno odabrao jednostavniji put i koji je trade-off.



## 4. Konfiguracija + tajna

```bash
# Tajnu kreiraj naredbom (ne ide u git):
oc -n ticketing create secret generic ticketing-secret \
  --from-literal=POSTGRES_PASSWORD='JakaLozinka123!'

# ConfigMap (API_BASE_URL ćemo ispraviti u koraku 6, kad znamo Route host):
oc apply -f k8s/02-configmap.yaml
```



## 5. Baza, Redis, aplikacija

```bash
oc apply -f k8s/04-postgres.yaml
oc apply -f k8s/05-redis.yaml
oc apply -f k8s/06-api.yaml
oc apply -f k8s/07-worker.yaml
oc apply -f k8s/08-frontend.yaml

# Prati da sve dođe u Running/Ready:
oc -n ticketing get pods -w
```



## 6. Mreža + vanjski pristup (Route)

```bash
# NetworkPolicy segmentacija (radi ako CNI podržava; OpenShift OVN/SDN podržava):
oc apply -f k8s/09-networkpolicy.yaml

# Vanjski pristup preko Routea — najlakše s 'oc expose' (auto host):
oc -n ticketing expose service frontend
oc -n ticketing expose service api

# Pročitaj dodijeljene hostove:
oc -n ticketing get route
```

Pretpostavimo da `oc get route` pokaže npr. `api-ticketing.apps-crc.testing`. Tada postavi `API_BASE_URL` na host API rute:

```bash
API_HOST=$(oc -n ticketing get route api -o jsonpath='{.spec.host}')
oc -n ticketing patch configmap ticketing-config --type=merge \
  -p "{\"data\":{\"API_BASE_URL\":\"http://$API_HOST\"}}"

# Frontend mora pokupiti novi API_BASE_URL:
oc -n ticketing rollout restart deploy/frontend
```

> Ovdje su frontend i api na **različitim** hostovima (svaki svoja ruta), pa `API_BASE_URL` pokazuje ravno na API host (bez `/api`). API je CORS-otvoren (`Access-Control-Allow-Origin: *`) pa cross-host poziv radi.
>
> Ako želiš fiksne hostove / path routing na istom hostu, umjesto `oc expose` koristi `k8s/route-openshift.yaml` (uredi `host:` na svoju apps domenu, koja se dobije s `oc get ingresses.config/cluster -o jsonpath='{.spec.domain}'`).



## 7. Validacija

```bash
FE_HOST=$(oc -n ticketing get route frontend -o jsonpath='{.spec.host}')
API_HOST=$(oc -n ticketing get route api -o jsonpath='{.spec.host}')

curl http://$API_HOST/healthz          # {"status":"ok","service":"api"}
curl http://$API_HOST/readyz           # {"status":"ready"}
curl http://$API_HOST/events
curl -X POST http://$API_HOST/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
sleep 1
curl http://$API_HOST/tickets/orders   # narudžba sa "status":"processed"

echo "UI: http://$FE_HOST"             # otvori u browseru
```
`processed` dokazuje lanac api → redis → worker → postgres unutar klastera.



## 8. Trivy — sigurnosni artefakt (ODVOJENO od deploya)

Ovo nije dio deploya i ne miješa se s `oc`. Skeniraš slike podmanom-sagrađene ili one iz registryja, jednom, i spremiš ispis. Projekt to traži (str. 9 "Sigurnosno izvješće", ishod I2 "Skeniranje ranjivosti").

```bash
# Build lokalno podmanom samo za skeniranje (ako nisi išao Metodom B):
podman build --target production -t ticketing-api:1.0.0 ./api

# Skeniraj i spremi ispis u repo:
trivy image ticketing-api:1.0.0 | tee docs/security/trivy-api.txt
```
Brojeve i mjere prepiši u `docs/security/image-scan-report.md` (predložak je u repou). Ako nemaš Trivy instaliran, on je jedan binarni alat — upute na `https://trivy.dev/latest/`. Nije nužan da deploy radi, nego da imaš traženi artefakt.



## 9. Rolling update i rollback

```bash
# Nova verzija: rebuild + novi tag
oc start-build ticketing-api --from-dir=./api --follow -n ticketing
oc tag ticketing-api:latest ticketing-api:1.0.1 -n ticketing

REG=image-registry.openshift-image-registry.svc:5000/ticketing
oc -n ticketing set image deploy/api api=$REG/ticketing-api:1.0.1
oc -n ticketing rollout status deploy/api
oc -n ticketing rollout history deploy/api

# Rollback:
oc -n ticketing rollout undo deploy/api
```
Jer api ima `maxUnavailable: 0`, tijekom updatea nema prekida usluge.



## 10. (Opcionalno) Stroži pristup bez `anyuid`

Ako želiš bodove za pravi least-privilege i ne koristiti `anyuid`:

- **Node servisi** (api/worker/frontend): ostavi `runAsNonRoot: true`, ali makni fiksni UID i daj pisivi HOME:
  ```bash
  for d in api worker frontend; do
    oc -n ticketing patch deploy/$d --type=json \
      -p='[{"op":"remove","path":"/spec/template/spec/securityContext/runAsUser"},
           {"op":"remove","path":"/spec/template/spec/securityContext/runAsGroup"}]'
    oc -n ticketing set env deploy/$d HOME=/tmp
  done
  ```
- **Postgres**: pod restricted SCC službena `postgres` slika zna pasti. Koristi OpenShift-kompatibilnu sliku (npr. iz kataloga `oc new-app postgresql-persistent ...`) — ali pazi, ona koristi `POSTGRESQL_*` varijable umjesto `POSTGRES_*`, pa bi trebalo uskladiti ConfigMap. Za lab je `anyuid` (poglavlje 3) jednostavniji i prihvatljiv.



## 11. Čišćenje

```bash
oc delete project ticketing     # briše SVE u projektu (DESTRUKTIVNO)
```



## Brzi referentni popis (oc)

```bash
oc -n ticketing get all
oc -n ticketing logs deploy/worker -f
oc -n ticketing describe pod <pod>            # Events: razlog pada / probe
oc -n ticketing get events --sort-by=.lastTimestamp
oc -n ticketing rsh deploy/api                # shell u podu (oc ekvivalent exec -it)
oc -n ticketing rsh deploy/postgres psql -U ticketing_user -d ticketing -c "SELECT count(*) FROM ticket_orders;"
oc -n ticketing get route,netpol,secret,configmap
```
