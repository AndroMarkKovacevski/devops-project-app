# Sigurnosno izvješće — skeniranje kontejnerskih slika (Trivy)

**Projekt:** Secure Event Ticketing Platform
**Alat:** Trivy (https://trivy.dev/latest/)
**Datum skeniranja:** _<2026-06-01>_
**Skenirane slike:** `ticketing-api:1.0.0`, `ticketing-worker:1.0.0`, `ticketing-frontend:1.0.0`



## 1. Metodologija


```bash
# Gate (vraca exit code 1 na HIGH/CRITICAL -> obara CI):
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 ticketing-api:1.0.0

# Pun izvjestaj za evidenciju:
trivy image --format table ticketing-api:1.0.0      | tee docs/security/trivy-api.txt
trivy image --format table ticketing-worker:1.0.0   | tee docs/security/trivy-worker.txt
trivy image --format table ticketing-frontend:1.0.0 | tee docs/security/trivy-frontend.txt
```


## 2. Sažetak nalaza

| ticketing-api:1.0.0 | 0 | 11 | n/a | n/a | PASS |     #trivy-api scan

#kratki komentar za trivy-api 
#sve pronađene ranjivosti razine su HIGH i potječu iz npm ovisnosi, uglavnom vezane uz DoS (Denial-of-Service) i rukovanje putanjama. nema ranjivosti razine CRITICAL
#status gate: pass 


| ticketing-frontend:1.0.0 | 0 | 11 | n/a | n/a | PASS |    #trivy-frontend scan

#kratki komentar za trivy-frontend
#sve pronađene ranjivosti razine su HIGH i potječu iz npm ovisnosi, uglavnom vezane uz DoS (Denial-of-Service) i rukovanje putanjama. nema ranjivosti razine CRITICAL
#status gate: pass


| ticketing-worker:1.0.0 | 0 | 11 | n/a | n/a | PASS |      #trivy-worker scan

#kratki komentar za trivy-worker
#sve pronađene ranjivosti razine su HIGH i potječu iz npm ovisnosi, uglavnom vezane uz DoS (Denial-of-Service) i rukovanje putanjama. nema ranjivosti razine CRITICAL
#status gate: pass

## 3. Detaljni nalazi

#trivy-api scan
| CVE | Paket | Verzija | Fiksano u | Mjera |
|---|---|---|---|---|
| CVE-2024-21538 | cross-spawn | 7.0.3 | 7.0.5 | nadogradi npm ovisnost |
| CVE-2025-64756 | glob | 10.4.2 | 10.5.0 | nadogradi npm ovisnost |
| CVE-2026-26996 | minimatch | 9.0.5 | 9.0.6 | nadogradi npm ovisnost |
| CVE-2026-23745 | tar | 6.2.1 | 7.5.3 | nadogradi npm ovisnost |


#trivy-frontend scan
| CVE | Paket | Verzija | Fiksano u | Mjera |
|---|---|---|---|---|
| CVE-2024-21538 | cross-spawn | 7.0.3 | 7.0.5 | nadogradi npm ovisnost |
| CVE-2025-64756 | glob | 10.4.2 | 10.5.0 | nadogradi npm ovisnost |
| CVE-2026-26996 | minimatch | 9.0.5 | 9.0.6 | nadogradi npm ovisnost |
| CVE-2026-23745 | tar | 6.2.1 | 7.5.3 | nadogradi npm ovisnost |


#trivy-worker scan
| CVE | Paket | Verzija | Fiksano u | Mjera |
|---|---|---|---|---|
| CVE-2024-21538 | cross-spawn | 7.0.3 | 7.0.5 | nadogradi npm ovisnost |
| CVE-2025-64756 | glob | 10.4.2 | 10.5.0 | nadogradi npm ovisnost |
| CVE-2026-26996 | minimatch | 9.0.5 | 9.0.6 | nadogradi npm ovisnost |
| CVE-2026-23745 | tar | 6.2.1 | 7.5.3 | nadogradi npm ovisnost |


## 4. Primijenjene hardening prakse (iz Containerfile-a)

- Multi-stage build → minimalna runtime slika (samo produkcijske ovisnosti).
- Bazna slika `node:20-alpine` (mala napadna površina).
- Non-root korisnik (`USER node`, `runAsNonRoot: true`).
- `.dockerignore` izbacuje `.env`, `.git`, `node_modules` iz build konteksta.
- Semantičko tagiranje (`1.0.0`), bez `latest` u produkciji.



## 5. Zaključak

Skeniranjem je u slikama pronađeno ukupno 11 ranjivosti razine HIGH (0 CRITICAL), uglavnom u npm ovisnostima; kao mjera predlaže se
nadogradnja ovisnosti. 
