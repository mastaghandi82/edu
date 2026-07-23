# Wiederherstellung & Rückgängig machen

> Notizen aus dem eigenen Durchlauf. Die Commit-Kette am Ende ist echt — hier im Repo
> nachvollziehbar, nicht erfunden.

Die Ausgangsfrage: *Ein Push auf `main` macht Ärger — wie komme ich zum letzten guten Stand
zurück?* Die Antwort hängt an **einer einzigen Unterscheidung**, und die ist der Kern dieser
Notiz.

## Das mentale Modell: zwei grundverschiedene Wege

Es gibt zwei Arten, „rückgängig" zu machen, und sie sind **nicht** austauschbar:

| | `git revert` | `git reset --hard` |
|---|---|---|
| Macht | einen **neuen** Commit, der das Gegenteil tut | schneidet die Historie **weg** |
| Historie | wird **länger** | wird **kürzer** (Commits verschwinden) |
| Sicher auf geteiltem `main`? | **ja** | **nein** — braucht Force-Push |
| Bild | Korrektur-Buchung | Zeitmaschine |

**Merksatz:** *Revert hängt an, Reset schneidet weg.* Weil Revert nie etwas entfernt, kannst du
beliebig oft hin- und herreverten, ohne je etwas zu verlieren.

## Die eine Frage, die alles entscheidet

**Ist der problematische Stand schon gepusht und hat ihn jemand anders gezogen?**

- **Ja / geteilter Branch (`main`):** → `revert`. Historie umzuschreiben, die andere schon
  haben, zieht ihnen den Boden weg. Der Gegen-Commit ist der einzig saubere Weg.
- **Nein / eigener Branch, den niemand gezogen hat:** → `reset --hard <guter-commit>` +
  `git push --force-with-lease`. Hier darfst du die Zeitmaschine nehmen.

Genau deshalb gilt hier die Konvention **„nicht direkt auf `main`"**: Auf einem Feature-Branch
bleibt dir der bequeme Reset-Weg; auf `main` bleibt nur der aufwendigere (aber sichere) Revert.

## Revert über die Web-GUI — Schritt für Schritt

Der ganze Weg geht ohne Terminal, wenn der Stand über einen **Pull Request** kam:

1. PR-Reiter öffnen. **Falle:** Er zeigt standardmäßig nur `is:open` — bei lauter gemergten
   PRs begrüßt dich „No open pull requests", und es sieht aus, als gäbe es keine. Auf
   **„Closed"** klicken (oder den Filter `is:open` löschen), dann tauchen sie auf.
2. Den gemergten PR öffnen und **ganz nach unten** scrollen, unter die Kommentare.
3. Im violetten Kasten *„Pull request successfully merged and closed"* sitzt rechts ein grauer
   **„Revert"**-Knopf — **unten im Merge-Kasten, nicht oben** an der Seite.
4. Klick erzeugt einen **neuen PR** „Revert: …". Erst wenn du *den* mergst, wird der Revert
   wirksam. Bis dahin ist nichts passiert.

### Die Lücke bei Direkt-Pushes

Den „Revert"-Knopf gibt es **nur am PR**, nicht an einem einzelnen Commit. Wer direkt auf
`main` gepusht hat (ohne PR), findet in der Web-GUI **keinen** Revert-Button an der
Commit-Seite. Zwei Auswege:

- **GitHub Desktop / VS-Code-Git:** Rechtsklick auf den Commit → *„Revert changes in commit"*.
- **Terminal:** `git revert <commit>` und pushen.

Das ist der handfeste Grund für „arbeite über PRs": Der PR **schenkt dir später den
Ein-Klick-Rückweg**, den ein Direkt-Push nicht hat.

## Der Reset-Weg (nur eigener Branch)

```bash
git reset --hard <guter-commit>      # lokal auf den letzten guten Stand
git push --force-with-lease          # Remote nachziehen
```

`--force-with-lease` statt `--force`: Es pusht nur, wenn sich der Remote seit deinem letzten
Fetch **nicht** verändert hat — so überschreibst du keine fremde Arbeit, die inzwischen
dazukam. Deshalb steht das nackte `--force` hier bewusst in der `deny`-Liste von
`.claude/settings.json`.

## Das Sicherheitsnetz: `reflog`

Selbst ein `reset --hard` verliert lokal fast nie etwas. `git reflog` protokolliert **jede**
Position, an der `HEAD` je war — auch die, die durch einen Reset „verschwunden" scheinen:

```bash
git reflog                    # jede frühere HEAD-Position, mit Kurz-SHA
git reset --hard HEAD@{2}     # zu einer davon zurück
```

Wichtig: Das reflog ist **lokal**. Der Server hat keins zum Durchblättern. Genau das ist ein
Argument für den lokalen Klon — die Web-GUI zeigt dieses Netz nicht.

## Durchgespielt an diesem Repo

Der Ernstfall wurde hier einmal trocken durchlaufen — die Kette steht so in der Historie:

```
934fcc3 Revert "Revert "progress.md … streichen""   ← 3. Schritt: Revert des Reverts
b017058 Merge pull request #12 …
8bbde57 Revert "progress.md … streichen"             ← 2. Schritt: Revert (Inhalt zurückgedreht)
904bf04 Merge pull request #7 …
a826262 progress.md aus den Claude-Code-Notizen streichen   ← 1. Original
```

Zwei Dinge, die dabei sichtbar wurden:

- **Revert ist inhaltlich exakt spiegelbildlich.** PR #7 änderte `+33 / −17` Zeilen, sein
  Revert `+17 / −33` — dieselben Dateien, Vorzeichen vertauscht. Prüfen lässt sich das nicht
  am Verlauf, sondern am Inhalt:

  ```bash
  git diff <original> <aktueller-stand> -- <datei>
  # leere Ausgabe = inhaltlich identisch mit dem Zielstand
  ```

  Nach dem Revert-des-Reverts war dieser Diff leer: `main` stand inhaltlich wieder **genau**
  auf dem gewollten Stand von PR #7 — nur mit drei zusätzlichen Commits, die die Reise
  dokumentieren.

- **Jeder dieser Merges löste den Deploy aus.** Ein Revert ist ein ganz normaler Push auf
  `main` — die Live-Seite zieht mit. „Rückgängig" heißt hier auch: geht wieder live.

## Fallen

- **„No open pull requests" heißt nicht „keine PRs".** Der Reiter filtert auf offene; gemergte
  liegen unter „Closed".
- **Der Revert-Knopf sitzt unten** im Merge-Kasten des PRs, nicht oben an der Seite.
- **Direkt-Pushes haben keinen Web-Revert** — nur PRs. Ein weiterer Grund für den PR-Zyklus.
- **Nur `reset` braucht einen Force-Push, `revert` nie.** Wer bei einem Revert zu `--force`
  greift, hat den falschen Weg gewählt.

## Was hängen bleiben soll

- **Die entscheidende Frage ist „schon geteilt?"** — sie wählt zwischen Revert und Reset.
- **Revert hängt an, Reset schneidet weg.** Auf `main` immer Revert.
- **Ein leerer `git diff` gegen den Zielstand ist der Beweis**, nicht ein Blick auf den
  Verlauf.
- **Was committet ist, ist fast immer rettbar** — `reflog` ist das lokale Netz.
