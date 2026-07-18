# Dotfiles: Konfiguration als Repo

> Drittes Thema im Wissensraum. Entstanden aus einer konkreten Frage: Sollte
> `~/.claude/` (persönliche CLAUDE.md, Skills) in ein Git-Repo, um Datenverlust
> vorzubeugen? Antwort: ja — und das Muster dahinter trägt weit über Claude Code hinaus.

## Warum überhaupt

Persönliche Konfiguration ist **destillierte Arbeitserfahrung**: die Shell-Aliase, die
sich über Jahre angesammelt haben, die Editor-Einstellungen, die CLAUDE.md mit den
Regeln, die aus echten Fehlern entstanden sind. Dieses Wissen liegt typischerweise an
genau einem Ort — auf einer Festplatte, unversioniert.

Damit gilt derselbe Satz wie für Chat-Verläufe: **Wissen, das nur an einem Ort
existiert, ist bereits verloren.** Der Plattentod macht es nur schlagartig sichtbar.

Ein Repo löst drei Probleme auf einmal:

1. **Backup** — der offensichtliche Teil.
2. **Sync** — neuer Rechner, gleiche Umgebung: klonen statt tagelang nachbauen.
3. **Das Warum** — „warum habe ich diese Regel im März geändert?" steht in der
   Commit-Message. Sonst nirgends.

## Der Begriff

*Dotfiles* heißen die Konfigurationsdateien im Home-Verzeichnis, weil sie mit einem
Punkt beginnen (`.gitconfig`, `.zshrc`, `.vimrc` — Unix blendet sie in Auflistungen
standardmäßig aus). „Dotfiles-Repo" ist der eingebürgerte Name für das Muster,
diese Dateien zu versionieren — auch wenn längst ganze Verzeichnisse wie
`~/.claude/` oder `~/.config/` dazugehören.

## Die entscheidende Sortierung: drei Kategorien

Der häufigste Fehler ist, *zu viel* zu versionieren. Im Home-Verzeichnis (und schon in
`~/.claude/` allein) liegen drei Sorten Daten, und nur eine gehört ins Repo:

| Kategorie | Beispiele | Ins Repo? |
|---|---|---|
| **Konfiguration & Wissen** | `CLAUDE.md`, `skills/`, `agents/`, `settings.json`, `.gitconfig`, `.zshrc` | **ja** |
| **Zustand & Verlauf** | Session-Verläufe, Caches, History-Dateien, `projects/` | nein — flüchtig, groß, teils sensibel |
| **Geheimnisse** | `.credentials.json`, Tokens, SSH-Keys, `.env` | **niemals** |

Zustand ist wertlos zu sichern (er veraltet in Minuten) und Geheimnisse sind gefährlich
zu sichern. Übrig bleibt genau das, was man nach einem Plattentod wirklich vermisst.

## Ansatz 1: Verzeichnis direkt als Repo (der Einstieg)

Für einen abgegrenzten Ordner wie `~/.claude/` der einfachste Weg: Das Verzeichnis
*ist* das Repo. Der Kern ist eine `.gitignore` nach dem **Allowlist-Prinzip** — alles
verboten, Gewolltes einzeln freigeschaltet:

```gitignore
/*
!.gitignore
!CLAUDE.md
!settings.json
!skills/
!agents/
```

Die erste Zeile ignoriert alles im Wurzelverzeichnis, jede `!`-Zeile schaltet gezielt
frei. Das ist die Secrets-Regel aus den Projektregeln in Aktion: **Denylists vergessen,
Allowlists nicht.** Wer stattdessen nur `.credentials.json` auf eine Verbotsliste
setzt, committet irgendwann die nächste Datei, an die niemand gedacht hat —
Verlaufsdaten, Caches, das nächste Token-File einer neuen Version.

Der Ablauf, mit dem Kontrollpunkt an der richtigen Stelle:

```bash
cd ~/.claude
git init
# .gitignore anlegen (Inhalt siehe oben)
git status        # ← DER Kontrollpunkt: tauchen NUR gewollte Dateien auf?
git add .
git commit -m "Claude-Konfiguration versionieren"
# privates Repo auf GitHub anlegen, dann:
git remote add origin git@github.com:<user>/claude-config.git
git push -u origin main
```

Das `git status` vor dem ersten `add` ist kein optionaler Schritt. Es ist der Moment,
in dem die Allowlist bewiesen wird — ein leerer Ignorierfehler zeigt sich hier, nicht
erst im Push.

## Ansatz 2: das ganze Home per Bare-Repo

Wenn die Dateien quer übers Home verstreut sind (`.gitconfig`, `.zshrc`, `.vimrc`,
`.config/…`), will man das Home-Verzeichnis *nicht* direkt zum Repo machen — sonst
meldet jedes `git status` in jedem Unterordner plötzlich das Dotfiles-Repo. Der
verbreitete Trick: ein **Bare-Repo** (ein Repo ohne eigenes Arbeitsverzeichnis) an
einem Seitenort, das Home als Arbeitsfläche nur auf Zuruf benutzt:

```bash
git init --bare ~/.dotfiles
alias config='git --git-dir=$HOME/.dotfiles --work-tree=$HOME'
config config status.showUntrackedFiles no
config add ~/.gitconfig ~/.zshrc
config commit -m "Shell- und Git-Konfiguration"
```

Danach verwaltet man die Dateien mit `config status`, `config add`, `config push` —
normales Git, nur unter anderem Namen. `showUntrackedFiles no` ist der Kniff, der
verhindert, dass jedes `config status` tausende ungetrackte Home-Dateien auflistet:
Hier wird nichts automatisch eingesammelt, jede Datei kommt einzeln per `add` dazu —
dasselbe Allowlist-Prinzip, nur prozedural statt per `.gitignore`.

## Ansatz 3: Werkzeuge

Es gibt Programme, die das Muster ausbauen — etwa GNU Stow (Symlink-Verwaltung: Repo
liegt an einem Ort, Symlinks zeigen vom Home dorthin) oder chezmoi (Templates für
Unterschiede zwischen Rechnern). Für den Einstieg braucht es sie nicht; hier nur als
Zeiger benannt und bewusst nicht vertieft — beide wurden für diese Notizen nicht
selbst durchlaufen, und Ungeprüftes wird hier nicht als Anleitung ausgegeben.

## Geheimnisse: die drei Verteidigungslinien

1. **Allowlist statt Denylist** (siehe oben) — strukturell verhindern, dass
   Unbedachtes hineinrutscht.
2. **Vor dem ersten Commit lesen, was committet wird.** Konkreter Fall
   `~/.claude/settings.json`: Dort kann ein `env`-Block mit API-Keys stehen. Dann
   entweder die Keys auslagern oder die Datei nicht committen. `git diff --cached`
   vor jedem Commit zeigt, was wirklich hineingeht.
3. **Privates Repo — aber „privat" ist keine Sicherheitsmaßnahme.** Es schützt vor
   fremden Blicken, nicht vor dem eigenen versehentlichen Öffentlichmachen später
   (Repo-Sichtbarkeit umgestellt, Fork, Kopie). Die Regel bleibt: Geheimnisse gehören
   *gar nicht* hinein, auch nicht in private Repos.

Und wenn doch eines hineingeraten und gepusht ist: **rotieren, nicht nur löschen.**
Ein aus der Historie entferntes Secret kann längst kopiert sein — der einzige sichere
Zustand ist, dass das alte Token ungültig ist. History-Umschreiben ist Kosmetik,
Rotation ist die Maßnahme.

## Der Restore-Test

Projektregel: **Ein Backup, das nie zurückgespielt wurde, ist eine Vermutung.** Bei
Dotfiles ist der Test so billig, dass es keine Ausrede gibt — in ein Testverzeichnis
(oder auf den nächsten frischen Rechner) klonen und prüfen, ob die Umgebung
funktioniert. Für `~/.claude/`: Sieht eine neue Claude-Code-Session die Skills und
die CLAUDE.md? Erst danach ist es ein Backup.

## Angewandt auf `~/.claude/`

Die Sortierung von oben, konkret:

| Datei/Ordner | Kategorie | Ins Repo? |
|---|---|---|
| `CLAUDE.md` | Wissen | ja |
| `skills/`, `agents/` | Wissen | ja |
| `settings.json` | Konfiguration | ja — nach Secret-Check |
| `settings.local.json` | per Definition lokal | nein |
| `projects/`, Session-Daten | Zustand | nein |
| `.credentials.json` | Geheimnis | **niemals** |

---

## Die Kernidee

> **Konfiguration ist Wissen, und Wissen braucht einen Ort, der einen Plattentod
> überlebt.** Versioniert wird per Allowlist — nicht, weil Git es verlangt, sondern
> weil die Verbotsliste immer eine Datei zu spät kommt. Und ein Backup ist erst dann
> eines, wenn es einmal zurückgespielt wurde.
