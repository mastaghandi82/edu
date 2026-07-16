# edu — Lernprojekt Git & Claude Code

## Zweck

Lern- und Referenzprojekt. Ziel ist **Wissensaufbau**, kein Produkt: Der komplette Weg von
einer lokalen Datei bis zur live erreichbaren URL, und wie Claude Code dabei mitarbeitet.

Besonderheit: **Die deployte Website ist die Dokumentation selbst.** `site/index.html`
beschreibt den Prozess, mit dem sie deployt wird. Wer den Prozess ändert, muss die Doku
mitändern — beides ist dasselbe Artefakt.

## Struktur

- `site/index.html` — die Lern-Doku als Website. Einzeldatei, **kein Build-Schritt**,
  keine externen Abhängigkeiten. Nur dieser Ordner wird veröffentlicht.
- `docs/*.md` — die ausführlichen Notizen (Langform, Quelle für die Website).
- `.github/workflows/deploy.yml` — deployt `site/` bei jedem Push auf `main` nach GitHub Pages.

## Konventionen

- **Deploy-Trigger ist `main`.** Der Workflow lauscht auf `branches: [main]` — Branch-Namen
  nicht ändern, ohne den Workflow anzupassen.
- **Kein Build-Tool einführen**, ohne dass es dem Lernziel dient. Statisches HTML ist Absicht:
  weniger bewegliche Teile, mehr sichtbare Mechanik.
- **Nicht direkt auf `main` committen.** Der Zyklus Branch → PR → Merge ist hier selbst
  Lerninhalt, nicht nur Formalie.
- **Didaktik vor Kürze:** In `docs/` und `site/` zählt Verständlichkeit für Einsteiger.
  Fachbegriffe beim ersten Auftreten erklären, nicht voraussetzen.
- **Nichts behaupten, was nicht geprüft ist.** Versionsnummern, Action-Versionen und
  CLI-Flags vor dem Dokumentieren verifizieren — die Doku soll belastbar sein.

## Verifikation

Ein Deployment gilt erst als erfolgreich, wenn die **Live-URL im Browser die Seite zeigt**.
Ein grüner Actions-Lauf allein reicht nicht (klassischer Fall: Workflow grün, Seite 404,
weil die Pages-Quelle nicht auf "GitHub Actions" steht).

- Lauf beobachten: `gh run watch`
- Fehler eingrenzen: `gh run view --log-failed`
