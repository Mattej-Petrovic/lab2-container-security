# 🔒 Lab 2 – Container Security

---

## Om projektet

Det här är mitt arbete från Lab 2, där fokus låg på containersäkerhet. Labben gick ut på att bygga en medvetet sårbar Docker-image, skanna den, härda den och jämföra resultaten. Utöver det genererade jag en SBOM och jobbade med OPA Gatekeeper-policies i Kubernetes.

---

## Steg för steg

**1. Sårbar image**
Byggde en enkel Flask-app med `python:3.8` som basimage och Flask 1.0.0. Skannade med Trivy och sparade resultatet i `scan-before.txt`.

**2. Härdad image**
Bytte till `python:3.12-slim-bookworm`, uppdaterade Flask till 3.0.0, la till en non-root-user, tog bort pip-cache och la till en healthcheck. Resultatet finns i `scan-after.txt`.

**3. SBOM**
Genererade en Software Bill of Materials i CycloneDX-format med Trivy (`sbom.json`).

**4. Gatekeeper-policies**
Deployade 5 policies i mitt namespace (`m4k-gang`) via Gatekeeper Lab och testade med dry-run mot en bad pod och en hardened pod.

---

## Verktyg

| Verktyg | Användning |
|---------|-----------|
| Docker | Bygga och köra container-images |
| Trivy | Sårbarhetsskanning och SBOM-generering |
| OPA Gatekeeper | Policy-enforcement i Kubernetes |

---

## Screenshots

| Screenshot | Beskrivning |
|-----------|-------------|
| [`trivy-before.png`](screenshots/trivy-before.png) | Skanningsresultat för den sårbara imagen |
| [`trivy-after.png`](screenshots/trivy-after.png) | Skanningsresultat efter härdning |
| [`gatekeeper-deny.png`](screenshots/gatekeeper-deny.png) | Bad pod nekad av Gatekeeper |
| [`gatekeeper-pass.png`](screenshots/gatekeeper-pass.png) | Hardened pod godkänd av Gatekeeper |

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
│   └── require-team-label.yaml
├── README.md
└── screenshots/
    ├── trivy-before.png
    ├── trivy-after.png
    ├── gatekeeper-deny.png
    └── gatekeeper-pass.png
```

---

## Reflektion

Den här labben har gett mig en bättre bild av vad containersäkerhet handlar om i praktiken. Innan tänkte jag att det mest handlade om att hålla sina images uppdaterade, men det är mycket mer än så. När jag skannade den sårbara imagen med Trivy blev det tydligt hur många kända sårbarheter som kan gömma sig i ett vanligt Python-basimage. Efter härdningen med nyare basimage, non-root-user, pinnade versioner och ett mindre image så sjönk antalet sårbarheter rejält.

SBOM var ett nytt koncept för mig, men det är lätt att förstå varför det behövs. Om en allvarlig sårbarhet upptäcks i ett bibliotek vill man snabbt kunna kolla om man faktiskt använder det, utan att behöva gå igenom varje image för hand. Det håller också på att bli ett krav genom EU Cyber Resilience Act, så det är något man behöver ha koll på.

Gatekeeper förändrar hur man jobbar med Kubernetes genom att säkerheten inte längre hänger på att varje utvecklare gör rätt manuellt. Policies fångar upp misstag automatiskt, till exempel om någon försöker köra en privilegierad container eller använda :latest-taggen. Det gör att säkerhet blir en del av systemet istället för något man hoppas att folk kommer ihåg.