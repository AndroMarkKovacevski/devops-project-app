# Runbook — Incidenti i troubleshooting (Secure Event Ticketing Platform)

Kratki operativni vodič za dijagnostiku i oporavak. Pokriva tri obavezna scenarija iz projekta (pad baze, loš image tag, neispravan secret) + opći postupak. Sve naredbe pretpostavljaju namespace `ticketing`

naredbe koje sluze za brisanje i slicno se izvode tek kad je potrebno i kada je developer siguran u što radi.

## Opći dijagnostički postupak 

# bash
kubectl -n ticketing get pods -o wide                  # tko nije Running/Ready?
kubectl -n ticketing describe pod <pod>                # sekcija Events na dnu = razlog
kubectl -n ticketing logs <pod> [-p]                   # -p = logovi prethodnog (srušenog) kontejnera
kubectl -n ticketing get events --sort-by=.lastTimestamp | tail -20

Tipična stanja i definicije:
- `CrashLoopBackOff` — kontejner se ruši u beskonačnoj petlji (pogrešna konfiguracija, nedostupna ovisnost).
- `ImagePullBackOff` / `ErrImagePull` — kriva/nepostojeća slika ili tag.
- `Pending` — nema resursa ili se ne može vezati.
- `0/1 Running` (nije "Ready") — ovisnost nije spremna

---

## Scenarij A — Pad baze (PostgreSQL nedostupan)

Simptomi: `/readyz` vraća 503; `api` podovi `0/2 Ready`; worker logovi prijavljuju greške upisa; UI kupnja prolazi (red prima) ali narudžbe ostaju `queued`.

Dijagnostika:
# bash
kubectl -n ticketing get pods -l app.kubernetes.io/name=postgres
kubectl -n ticketing logs deploy/postgres
kubectl -n ticketing describe pod -l app.kubernetes.io/name=postgres   # Events: PVC? OOMKilled?
kubectl -n ticketing exec deploy/postgres -- pg_isready -U ticketing_user -d ticketing


`Mogući uzroci i mjere:`
- `Pending`: provjeri `kubectl -n ticketing get pvc`. Ako nema StorageClass → postavi `storageClassName` u `04-postgres.yaml`.
- `OOMKilled`: podigni `limits.memory` za postgres pa `kubectl apply`.
- `Pao je pod, podaci na PVC-u`: Deployment ga automatski ponovno digne; podaci su sačuvani na `postgres-pvc` (zbog toga ide strategija Recreate).

`Validacija oporavka:`
# bash
kubectl -n ticketing rollout status deploy/postgres
curl http://$H/api/readyz     # ocekuje {"status":"ready"}

Red iz Redisa worker obradi automatski čim baza ponovno radi (zaostale `queued` narudžbe se procesiraju).

>NIKAD kao "rješenje" ne brisati `postgres-pvc` — to trajno gubi sve narudžbe. To je krajnja, nereverzibilna mjera samo uz svjesnu odluku.

---

## Scenarij B — Loš image tag (kriva/nepostojeća verzija slike)

`Simptomi:` nakon `set image` novi podovi su `ImagePullBackOff` ili `CrashLoopBackOff`; rollout zapeo.

`Dijagnostika:`
# bash
kubectl -n ticketing rollout status deploy/api          # zapinje
kubectl -n ticketing get pods -l app.kubernetes.io/name=api
kubectl -n ticketing describe pod <novi-api-pod>        # Events: "Failed to pull image ..."


`Mjera (rollback) — brzo i reverzibilno:`
# bash
kubectl -n ticketing rollout undo deploy/api            # vraća prethodnu (ispravnu) reviziju
kubectl -n ticketing rollout status deploy/api
kubectl -n ticketing rollout history deploy/api         # potvrdi reviziju

Zbog `maxUnavailable: 0` stari podovi su radili cijelo vrijeme → korisnici nisu osjetili prekid. To je ključna prednost rolling updatea: loša verzija nikad ne preuzme sav promet.

`Trajni popravak:` sagradi/učitaj ispravnu sliku s točnim tagom, pa `set image` na nju.



## Scenarij C — Neispravan secret (kriva lozinka baze)

`Simptomi:` `api`/`worker` u `CrashLoopBackOff` ili `0/2 Ready`; logovi: `password authentication failed for user "ticketing_user"`; `/readyz` = 503.

`Dijagnostika:`
# bash
kubectl -n ticketing logs deploy/api | grep -i password
kubectl -n ticketing get secret ticketing-secret -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d; echo
# Usporedi s onim sto baza ocekuje:
kubectl -n ticketing exec deploy/postgres -- printenv POSTGRES_PASSWORD


`Mjera:`
# bash
# Ispravi secret:
kubectl -n ticketing create secret generic ticketing-secret \
  --from-literal=POSTGRES_PASSWORD='ISPRAVNA_LOZINKA' \
  --dry-run=client -o yaml | kubectl apply -f -

# podovi ne pokupe novi secret automatski -> restart:
kubectl -n ticketing rollout restart deploy/api deploy/worker


`Caveat (blast radius):` ako je baza već inicijalizirana sa starom lozinkom, mijenjanje samo secreta neće promijeniti lozinku u bazi. Tada ili (a) postavi lozinku u bazi (`ALTER USER`), ili (b) ako su podaci nebitni (lab): obriši `postgres-pvc` i pusti reinit (DESTRUKTIVNO — gubiš podatke).
# bash
kubectl -n ticketing exec -it deploy/postgres -- \
  psql -U ticketing_user -d ticketing -c "ALTER USER ticketing_user WITH PASSWORD 'ISPRAVNA_LOZINKA';"

`Validacija:` `curl http://$H/api/readyz` → `{"status":"ready"}`.

---

## Brzi referentni popis

| Problem | Prva naredba | Vjerojatna mjera |

| Pod se ruši | `kubectl -n ticketing logs <pod> -p` | popravi config / ovisnost |

| Ne povlači sliku | `describe pod` → Events | `rollout undo` + ispravan tag |

| Nije Ready | provjeri `/readyz`, ovisnosti | čekaj DB/Redis ili popravi |

| Loša lozinka | usporedi secret vs baza | ispravi secret + `rollout restart` |

| Nema vanjskog pristupa | `get ingress`/`get route` | host/`API_BASE_URL`/ingress controller |

| NetworkPolicy blokira | `get netpol` + `exec ... nc` | uskladi from/podSelector |
