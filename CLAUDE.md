# edu — Wissensraum

## Zweck

Lern- und Referenzprojekt für technische Themen rund um Entwicklung, Werkzeuge und Betrieb.
Ziel ist **Wissensaufbau**, kein Produkt: belastbare, nachlesbare Notizen zu Themen, die
einmal verstanden dauerhaft tragen.

Erstes Thema war Git & Deployment. Weitere kommen dazu — die Struktur ist darauf ausgelegt.

Besonderheit: **Die deployte Website ist die Dokumentation selbst.** `site/` beschreibt unter
anderem den Prozess, mit dem es deployt wird. Wer den Prozess ändert, ändert die Doku mit.

## Struktur

- `site/` — die veröffentlichte Fassung. `index.html` ist die Übersicht, je Thema eine Seite.
  Einzelne HTML-Dateien plus ein gemeinsames `style.css`, **kein Build-Schritt**, keine
  externen Abhängigkeiten. Nur dieser Ordner geht live.
- `docs/<thema>/` — die Langform zum Nachschlagen, Quelle für die Website.
  Aktuell: `docs/git/`, `docs/claude-code/`.
- `.github/workflows/deploy.yml` — deployt `site/` bei jedem Push auf `main`.

Neues Thema = neuer Ordner unter `docs/`, neue Seite in `site/`, Eintrag auf der Übersicht.

## Konventionen

- **Deploy-Trigger ist `main`.** Der Workflow lauscht auf `branches: [main]` — Branch-Namen
  nicht ändern, ohne den Workflow anzupassen.
- **Kein Build-Tool einführen**, ohne dass es dem Lernziel dient. Statisches HTML ist Absicht:
  weniger bewegliche Teile, mehr sichtbare Mechanik.
- **Nicht direkt auf `main` committen.** Der Zyklus Branch → PR → Merge ist hier selbst
  Lerninhalt.
- **Didaktik vor Kürze:** Verständlichkeit für Einsteiger zählt. Fachbegriffe beim ersten
  Auftreten erklären, nicht voraussetzen.
- **Nichts behaupten, was nicht geprüft ist.** Versionsnummern, CLI-Flags und Zitate aus
  Nutzungsbedingungen vor dem Dokumentieren verifizieren — die Doku soll belastbar sein.
  Wo etwas unklar blieb, das offen benennen statt plausibel zu raten.
- **Fehler gehören dazu.** Was im eigenen Durchlauf schiefging, wird dokumentiert, nicht
  geglättet — die Fallen sind der wertvollste Teil der Notizen.

## Verifikation

Ein Deployment gilt erst als erfolgreich, wenn die **Live-URL im Browser die Seite zeigt**.
Ein grüner Actions-Lauf allein reicht nicht (klassischer Fall: Workflow grün, Seite 404,
weil die Pages-Quelle nicht auf "GitHub Actions" steht).

- Lauf beobachten: `gh run list` → ID holen → `gh run watch <id> --exit-status`
  (ohne TTY braucht `watch` zwingend eine ID — siehe `docs/git/02-deployment.md`)
- Fehler eingrenzen: `gh run view --log-failed`
