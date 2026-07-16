# Git-Grundlagen

> Notizen aus dem eigenen Durchlauf. Alles hier wurde praktisch gemacht, nicht abgeschrieben.

## Das mentale Modell: drei Zonen

Der Punkt, an dem Git für die meisten zuerst klickt. Eine Datei durchläuft drei Zustände:

```
Working Directory  ──git add──>  Staging Area  ──git commit──>  Repository
(deine Dateien)                  (Vormerkung)                   (Schnappschuss)
```

- **Working Directory** — der Ordner, in dem du arbeitest. Hier änderst du Dateien.
- **Staging Area** (auch *Index*) — die Vormerkliste: *was* soll in den nächsten Commit.
- **Repository** — die Kette dauerhafter Schnappschüsse (`.git/`).

**Warum die Staging Area?** Das ist der Schritt, den andere Systeme nicht haben, und er wirkt
zuerst überflüssig. Sein Nutzen: Du hast 10 Dateien geändert, aber nur 3 gehören inhaltlich
zusammen. Mit `git add` nimmst du gezielt diese 3 und committest sie als eine saubere Einheit.
Ein Commit soll **eine Änderung** sein, nicht "alles was gerade rumlag".

Wo liegt was:

```bash
git status            # Was ist geändert, was ist vorgemerkt?
git diff              # Working Directory vs. Staging  (was ist NICHT vorgemerkt)
git diff --staged     # Staging vs. letzter Commit     (was ist vorgemerkt)
```

## Ein Repo starten

```bash
git init -b main
```

**Das `-b main` ist nicht optional.** Ohne es nennt Git den ersten Branch `master` (sofern
`init.defaultBranch` nicht global gesetzt ist). GitHub und alle CI-Workflows erwarten heute
`main`. Unser Deploy-Workflow lauscht auf `branches: [main]` — mit `master` wäre er nie
ausgelöst worden. Ein stiller Fehlschlag: kein Fehler, nur passiert nichts.

Global ein für alle Mal festlegen:

```bash
git config --global init.defaultBranch main
```

## Identität

```bash
git config --global user.name "mastaghandi82"
git config --global user.email "188686737+mastaghandi82@users.noreply.github.com"
```

Ohne das schlägt der **erste Commit** fehl — nicht `git init`, nicht `git add`. Der Fehler
kommt also erst, wenn man schon glaubt, fertig zu sein.

**Zur E-Mail:** Sie steht dauerhaft und öffentlich in jedem Commit. In einem public Repo lesen
Spam-Crawler sie aus. GitHub stellt dafür eine `noreply`-Adresse bereit:

```
<user-id>+<username>@users.noreply.github.com
```

Die eigene ID findet man mit `gh api user --jq .id`. Commits werden damit auf GitHub trotzdem
korrekt dem Account zugeordnet — die Zuordnung läuft über die Adresse, nicht über den Namen.
Der Name ist frei wählbar.

`--global` schreibt nach `~/.gitconfig` (gilt überall). Ohne `--global` gilt es nur im
aktuellen Repo — praktisch, wenn beruflich und privat getrennte Adressen nötig sind.

## Dateien ignorieren — zwei verschiedene Wege

Das wird oft verwechselt, obwohl der Unterschied klar ist:

| | `.gitignore` | `.git/info/exclude` |
|---|---|---|
| Liegt | im Projekt | in `.git/` |
| Wird committet | **ja** | **nein**, nie |
| Aussage | "gehört in *niemandes* Repo" | "will *ich* hier nicht sehen" |
| Beispiel | `node_modules/`, `.env` | eigene Notizen, Scratch-Dateien |

Faustregel: Ist die Datei für **alle** irrelevant (Build-Artefakte, Secrets) → `.gitignore`.
Ist sie nur **dir** im Weg (persönliche Notizen wie `progress.md`) → `.git/info/exclude`.
Andere haben ihre eigenen; das gehört nicht ins geteilte Repo.

Prüfen, ob und **warum** eine Datei ignoriert wird — zeigt Datei und Zeilennummer der Regel:

```bash
git check-ignore -v .claude/progress.md
# .git/info/exclude:9:.claude/progress.md    .claude/progress.md
```

**Wichtige Falle:** Ignorieren wirkt nur auf **untracked** Dateien. Ist eine Datei schon
einmal committet, ignoriert Git sie weiter *nicht* — die Regel kommt zu spät. Dann braucht es
`git rm --cached <datei>`. Deshalb: `.gitignore` **vor** dem ersten Commit anlegen. Und bei
versehentlich committeten Secrets reicht das Entfernen nicht — sie bleiben in der Historie
und müssen als kompromittiert gelten.

`git status` fasst untracked Ordner zusammen (`?? .claude/`). Alle Dateien einzeln sehen:

```bash
git status --porcelain -uall
```

## Der Alltags-Zyklus

```bash
git switch -c mein-feature     # Branch anlegen + hinwechseln
# ... arbeiten ...
git add <dateien>              # vormerken
git commit -m "Beschreibung"   # festschreiben
git push -u origin mein-feature
```

`switch` ist der moderne Befehl zum Branch-Wechseln. Das ältere `checkout` kann das auch,
macht aber nebenbei noch drei andere Dinge — genau deshalb wurde es aufgeteilt in `switch`
(Branches) und `restore` (Dateien).

**Warum überhaupt Branches?** Damit `main` jederzeit deploybar bleibt. Auf einem Branch darf
es kaputt sein. Bei uns ist das direkt sichtbar: Jeder Push auf `main` löst ein Deployment aus
— ein kaputter Commit auf `main` ist sofort live.

Das `-u` beim ersten Push verknüpft lokalen und entfernten Branch. Danach reicht `git push`.

## Remotes

Ein *Remote* ist eine benannte Adresse einer Repo-Kopie. `origin` ist reine Konvention für
"da, wo es herkam" — kein Schlüsselwort.

```bash
git remote -v                  # welche Remotes gibt es?
```

## Nützlich zum Nachschauen

```bash
git log --oneline --graph      # Historie kompakt, mit Branch-Verlauf
git show <commit>              # was genau war in diesem Commit?
```

## Womit man sich wehtun kann

- `git push --force` — überschreibt die Historie auf dem Server. Arbeit anderer kann
  verschwinden. (`--force-with-lease` ist die entschärfte Variante.)
- `git reset --hard` — verwirft uncommittete Änderungen **ohne Rückfrage**. Was nie
  committet war, ist weg.

Beide stehen in diesem Projekt bewusst in der `deny`-Liste von `.claude/settings.json`.
Merksatz: **Was committet ist, ist fast immer rettbar. Was nie committet wurde, nicht.**
Im Zweifel: lieber ein Commit zu viel.
