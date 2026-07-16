# Deployment: GitHub Pages via Actions

> Wie aus einem `git push` eine live erreichbare URL wird.

## Das Prinzip

**Continuous Deployment**: Niemand lädt Dateien irgendwo hoch. Du pusht nach `main` — eine
Maschine bei GitHub startet, holt das Repo, packt einen Ordner ein und veröffentlicht ihn.
Der Push ist der einzige Auslöser.

```
git push origin main
   └─> GitHub Actions startet einen Runner (frische Ubuntu-VM)
        └─> checkout: Repo holen
             └─> upload-pages-artifact: Ordner site/ als Artefakt packen
                  └─> deploy-pages: Artefakt veröffentlichen
                       └─> https://mastaghandi82.github.io/edu/
```

Der Wert liegt in der **Reproduzierbarkeit**: Die VM ist jedes Mal frisch. Es gibt kein "läuft
bei mir" — nur das, was im Repo steht, existiert. Vergessene lokale Datei = sofort sichtbarer
Fehler statt stiller Zeitbombe.

## Zwei Dinge müssen stimmen

Der wichtigste Punkt überhaupt, weil hier fast jeder beim ersten Mal hängen bleibt:

1. **Der Workflow** (`.github/workflows/deploy.yml`) — im Repo, macht die Arbeit.
2. **Die Pages-Quelle** — eine Einstellung *im Browser*:
   `Settings → Pages → Build and deployment → Source → GitHub Actions`

**Der Workflow allein aktiviert Pages nicht.** Fehlt Schritt 2, läuft der Workflow grün durch
und die Seite bleibt trotzdem 404. Ein grüner Haken ist deshalb *kein* Beweis — nur die
aufgerufene URL ist einer.

## Der Workflow, Zeile für Zeile

```yaml
name: Deploy static content to Pages

on:
  push:
    branches: [main]      # Auslöser: Push auf main
  workflow_dispatch:      # zusätzlich: manuell per Knopfdruck

permissions:
  contents: read          # checkout darf das Repo lesen
  pages: write            # darf auf Pages deployen
  id-token: write         # darf ein OIDC-Token anfordern

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/configure-pages@v6
      - uses: actions/upload-pages-artifact@v5
        with:
          path: ./site    # NUR dieser Ordner wird veröffentlicht
      - id: deployment
        uses: actions/deploy-pages@v5
```

### `permissions`

Diese drei **ersetzen die Standard-Rechte komplett** — das ist Least-Privilege: Der Job kann
nur, was hier steht. Schreibt man den Block, muss man vollständig sein; `contents: read` etwa
fehlt sonst dem `checkout`-Schritt.

`id-token: write` ist das unintuitivste. Es erlaubt, ein kurzlebiges OIDC-Token anzufordern,
mit dem Pages verifiziert, dass das Deployment aus legitimer Quelle stammt (Claims über Repo
und Branch). Kein dauerhaftes Secret im Spiel — das Token lebt nur für diesen Lauf.

### `concurrency`

```yaml
group: "pages"
cancel-in-progress: false
```

Nur ein Deployment gleichzeitig. `false` ist bewusst: Ein **laufendes** Deployment wird nie
abgebrochen. Bei CI-Tests wäre `true` richtig (veraltete Läufe abwürgen spart Zeit) — bei
einem Produktiv-Deploy hinterlässt ein Abbruch mittendrin womöglich einen halben Stand.

### `path: ./site`

Nur `site/` geht live, nicht das ganze Repo. `docs/`, `CLAUDE.md`, `.claude/` bleiben im Repo
sichtbar, aber unveröffentlicht. Zeigt der Pfad auf nichts, schlägt der Schritt hart fehl —
das ist gut, ein stiller Fehlschlag wäre schlimmer.

## Action-Versionen

Gegen die GitHub-API geprüft (Stand Juli 2026), **nicht** aus dem Gedächtnis:

| Action | Aktuell |
|---|---|
| `actions/checkout` | v7 |
| `actions/configure-pages` | v6 |
| `actions/upload-pages-artifact` | v5 |
| `actions/deploy-pages` | v5 |

**Kurioses Detail:** GitHubs *eigene* Starter-Vorlage hinkt dem hinterher und zeigt einen
älteren Mix (checkout@v4, configure-pages@v5, upload-pages-artifact@v3, deploy-pages@v5).
Falls die aktuellen Majors Ärger machen, ist der Starter-Mix der von GitHub selbst getestete
Rückfallweg.

Lehre daraus: **Versionsnummern nachschlagen, nicht erinnern.** Doku veraltet — auch offizielle.

`@v7` ist ein *floating tag*: zeigt immer auf die neueste v7.x. Bequem, aber nicht
bit-reproduzierbar. Sicherheitskritische Projekte pinnen den vollen Commit-SHA.

## Voraussetzungen

- **Free-Plan ⇒ Repo muss public sein.** Pages aus privaten Repos gibt es erst ab Pro/Team.
- **Die veröffentlichte Seite ist immer öffentlich** — auch bei privatem Repo. Pages ist
  Hosting, kein Zugriffsschutz. Nie etwas Vertrauliches nach `site/` legen.
- Limits: 1 GB Seite, 10 min Deploy-Timeout, 100 GB/Monat Bandbreite (weich).

## URL-Schema

- **Projekt-Site** (unser Fall): `https://<user>.github.io/<repo>/` → `.../edu/`
- **User-Site**: Repo muss exakt `<user>.github.io` heißen → `https://<user>.github.io/`

Max. eine User-Site pro Account, max. eine Pages-Site pro Repo.

## Läufe beobachten

```bash
gh run watch              # laufenden Job live im Terminal
gh run list               # letzte Läufe + Status
gh run view --log-failed  # NUR die fehlgeschlagenen Schritte
```

`--log-failed` ist der Zeitsparer: CI-Logs sind tausende Zeilen, davon interessieren fünf.

## Typische Fehlerbilder

| Symptom | Ursache |
|---|---|
| Workflow grün, Seite 404 | Pages-Quelle nicht auf "GitHub Actions" gestellt |
| Push abgelehnt bei `.github/workflows/` | Token fehlt der `workflow`-Scope |
| Workflow läuft gar nicht | Branch heißt `master`, Trigger lauscht auf `main` |
| 404 trotz Deploy | keine `index.html` im Wurzelverzeichnis von `path` |
| Deploy hängt auf "Waiting" | Environment `github-pages` hat Protection Rules |
| Ordner mit `_` fehlen | Jekyll-Verarbeitung — `.nojekyll` anlegen |
| Versteckte Dateien fehlen | `upload-pages-artifact` ignoriert sie seit v4 → `include-hidden-files: true` |

## Übertragbar auf andere Projekte

Das Muster ist überall gleich, nur die Mitte ändert sich:

1. Auslöser (Push/PR/Zeitplan)
2. Rechte minimal deklarieren
3. Code holen
4. **Bauen** (bei uns: nichts, es ist statisch)
5. Ergebnis veröffentlichen

Bei einer Vite-App käme zwischen Schritt 3 und 5 `npm ci && npm run build`, und `path` zeigte
auf `dist/`. Das Gerüst bleibt identisch.
