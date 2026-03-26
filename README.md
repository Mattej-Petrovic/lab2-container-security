# Lab 2 – Container Security

## Om projektet

Det här projektet handlar om containersäkerhet i praktiken. Arbetet gick ut på att bygga en medvetet sårbar Docker-image, skanna den med Trivy, härda den och jämföra resultaten. Utöver det genererades en SBOM, imagen signerades med Cosign och tre egna OPA Gatekeeper-policies deployades i Kubernetes-namespacet `m4k-gang`.

---

## Steg för steg

**1. Sårbar image**
Byggde en enkel Flask-app med `python:3.8` som basimage och Flask 1.0.0. Skannade med Trivy och sparade resultatet i `scan-before.txt`.

**2. Härdad image**
Bytte till `python:3.12-slim-bookworm`, uppdaterade Flask till 3.0.0, lade till en non-root-user, tog bort pip-cache och lade till en healthcheck. Resultatet finns i `scan-after.txt`.

**3. SBOM**
Genererade en Software Bill of Materials i CycloneDX-format med Trivy (`sbom.json`).

**4. OPA Gatekeeper-policies**
Deployade tre egna policies i namespacet `m4k-gang`:
- `require-team-label` — alla pods måste ha en `team`-label
- `block-latest-tag` — containers får inte använda `:latest`-taggen
- `require-resource-limits` — alla containers måste ha CPU- och minneslimiter satta

**5. Cosign — image-signering**
Imagen signerades med Cosign via Sigstore och är verifierbar i det publika transparency log.

---

## Verktyg

| Verktyg | Användning |
|---------|-----------|
| Docker | Bygga och köra container-images |
| Trivy | Sårbarhetsskanning och SBOM-generering |
| OPA Gatekeeper | Policy-enforcement i Kubernetes |
| Cosign | Kryptografisk signering av container-image |

---

## Cosign — signerad image

Imagen är signerad med Cosign via Sigstore och publik nyckelinfrastruktur:
```bash
cosign sign --yes mattejpetrovic/my-app:hardened
```

Verifiera signaturen:
```bash
cosign verify mattejpetrovic/my-app:hardened \
  --certificate-identity-regexp=".*" \
  --certificate-oidc-issuer-regexp=".*"
```

Image digest: `sha256:904ac73fbc35e0df97cd72a130348a782b536239d53c032114abadba38a3ef75`

---

## Säkerhetsstrategi

Säkerhetsarbetet följer en skiktad approach där varje lager adresserar olika delar av hotbilden.

**Image-härdning** är första försvarslinjen. Genom att byta från `python:3.8` till `python:3.12-slim-bookworm` reducerades antalet sårbarheter drastiskt. Slim-varianten innehåller bara det som krävs för att köra applikationen, vilket minimerar attackytan. Containers körs som non-root-användare, har pinnade beroenden och en healthcheck som säkerställer att trasiga containers upptäcks snabbt.

**SBOM** ger full synlighet i vad imagen innehåller. Med en SBOM i CycloneDX-format går det snabbt att avgöra om en nyupptäckt sårbarhet påverkar systemet utan att inspektera varje image manuellt. Detta är också ett kommande krav under EU Cyber Resilience Act.

**Image-signering med Cosign** säkerställer att imagen som körs i produktion är identisk med den som byggdes och granskades. Signaturen registreras i Sigstores publika transparency log, vilket gör det omöjligt att manipulera imagen utan att signaturen bryts.

**OPA Gatekeeper** flyttar säkerhetskontroller från individuella utvecklare till systemet. Policies som blockerar latest-taggar, kräver resurslimiter och kräver labels körs automatiskt vid varje deploy — misstag fastnar i admission control innan de når klustret.

---

## Screenshots

| Screenshot | Beskrivning |
|-----------|-------------|
| [`trivy-before.png`](screenshots/trivy-before.png) | Skanningsresultat för den sårbara imagen |
| [`trivy-after.png`](screenshots/trivy-after.png) | Skanningsresultat efter härdning |
| [`gatekeeper-deny.png`](screenshots/gatekeeper-deny.png) | Bad pod nekad av Gatekeeper |
| [`gatekeeper-pass.png`](screenshots/gatekeeper-pass.png) | Hardened pod godkänd av Gatekeeper |
| [`cosign-verify.png`](screenshots/cosign-verify.png) | Cosign-verifiering i terminalen |

---

## Repostruktur
```
lab2-container-security/
├── Dockerfile.vulnerable
├── Dockerfile.hardened
├── app.py
├── requirements.txt
├── sbom.json
├── scan-before.txt
├── scan-after.txt
├── policies/
│   ├── require-labels-template.yaml
│   ├── require-team-label.yaml
│   ├── block-latest-tag.yaml
│   └── require-resource-limits.yaml
├── README.md
└── screenshots/
    ├── trivy-before.png
    ├── trivy-after.png
    ├── gatekeeper-deny.png
    ├── gatekeeper-pass.png
    └── cosign-verify.png
```

---

## Reflektion

Den här labben har gett mig en bättre bild av vad containersäkerhet handlar om i praktiken. Innan tänkte jag att det mest handlade om att hålla sina images uppdaterade, men det är mycket mer än så. När jag skannade den sårbara imagen med Trivy blev det tydligt hur många kända sårbarheter som kan gömma sig i ett vanligt Python-basimage. Efter härdningen med nyare basimage, non-root-user, pinnade versioner och en mindre image sjönk antalet sårbarheter rejält.

SBOM var ett nytt koncept för mig, men det är lätt att förstå varför det behövs. Om en allvarlig sårbarhet upptäcks i ett bibliotek vill man snabbt kunna kolla om man faktiskt använder det, utan att behöva gå igenom varje image för hand. Det håller också på att bli ett krav genom EU Cyber Resilience Act.

Gatekeeper förändrar hur man jobbar med Kubernetes genom att säkerheten inte längre hänger på att varje utvecklare gör rätt manuellt. Policies fångar upp misstag automatiskt — om någon försöker köra en privilegierad container eller använda `:latest`-taggen stoppas det direkt i admission control. Det gör att säkerhet blir en del av systemet istället för något man hoppas att folk kommer ihåg.

Cosign adderar ytterligare ett lager genom att kryptografiskt bevisa att imagen inte manipulerats efter att den byggdes. I en verklig produktionsmiljö är det avgörande för att kunna lita på vad som faktiskt körs i klustret.