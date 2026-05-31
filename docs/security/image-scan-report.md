# Sigurnosno izvješće — skeniranje kontejnerskih slika (Trivy)

**Projekt:** Secure Event Ticketing Platform
**Alat:** Trivy (https://trivy.dev/latest/)
**Datum skeniranja:** _<UPISATI>_
**Skenirane slike:** `ticketing-api:1.0.0`, `ticketing-worker:1.0.0`, `ticketing-frontend:1.0.0`

> Ovo je predložak. Pokreni skeniranje (naredbe niže), pa upiši stvarne rezultate i priloži screenshot/izvod.

---

## 1. Metodologija

Skeniranje se izvodi **prije deploya** (shift-left princip) na svakoj slici. Quality gate: build/CI **pukne** ako se nađe ranjivost razine `HIGH` ili `CRITICAL` koja ima dostupan popravak.

```bash
# Gate (vraca exit code 1 na HIGH/CRITICAL -> obara CI):
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ticketing-api:1.0.0

# Pun izvjestaj za evidenciju:
trivy image --format table ticketing-api:1.0.0      | tee docs/security/trivy-api.txt
trivy image --format table ticketing-worker:1.0.0   | tee docs/security/trivy-worker.txt
trivy image --format table ticketing-frontend:1.0.0 | tee docs/security/trivy-frontend.txt
```

---

## 2. Sažetak nalaza

| Slika | CRITICAL | HIGH | MEDIUM | LOW | Status gate |
|---|---|---|---|---|---|
| ticketing-api:1.0.0 | _0_ | _0_ | _?_ | _?_ | _PASS/FAIL_ |
| ticketing-worker:1.0.0 | _0_ | _0_ | _?_ | _?_ | _PASS/FAIL_ |
| ticketing-frontend:1.0.0 | _0_ | _0_ | _?_ | _?_ | _PASS/FAIL_ |

_(Popuni stvarnim brojevima iz ispisa.)_

---

## 3. Detaljni nalazi i korektivne mjere

Za svaku nađenu HIGH/CRITICAL ranjivost upiši:

| CVE | Paket | Verzija | Fiksano u | Mjera |
|---|---|---|---|---|
| _CVE-..._ | _npr. openssl_ | _..._ | _..._ | _nadogradi baznu sliku / paket_ |

Tipične mjere:
- **Nadogradi baznu sliku** (`node:20-alpine` → najnoviji patch) i ponovno sagradi.
- **Nadogradi npm ovisnosti** (`npm audit fix`, podigni verzije u `package.json`).
- **`--ignore-unfixed`** za ranjivosti bez dostupnog popravka (dokumentiraj odluku).

---

## 4. Primijenjene hardening prakse (iz Containerfile-a)

- Multi-stage build → minimalna runtime slika (samo produkcijske ovisnosti).
- Bazna slika `node:20-alpine` (mala napadna površina).
- Non-root korisnik (`USER node`, `runAsNonRoot: true`).
- `.dockerignore` izbacuje `.env`, `.git`, `node_modules` iz build konteksta.
- Semantičko tagiranje (`1.0.0`), bez `latest` u produkciji.

---

## 5. Zaključak

_<Kratko: jesu li slike prošle gate, koje su mjere poduzete, preostali rizik.>_
