# Claude Code: Arbeitsweise verstehen

> Die Bausteine, die dauerhaft tragen — nicht die Feature-Liste. Alles hier gegen die
> offizielle Doku verifiziert.

## Das Grundproblem: Kontext ist endlich

Fast jede Design-Entscheidung in Claude Code lässt sich hierauf zurückführen. Das
Kontextfenster ist der Arbeitsspeicher: alles, was Claude gerade "sieht". Es ist begrenzt,
und **alles darin kostet — bei jeder einzelnen Anfrage neu**.

Daraus folgt die zentrale Frage bei jedem Baustein unten: *Muss das immer geladen sein, oder
nur wenn es gebraucht wird?* Wer das versteht, trifft die Konfigurationsentscheidungen von
selbst richtig.

## Die Bausteine im Überblick

| Baustein | Wann geladen | Wofür |
|---|---|---|
| `CLAUDE.md` | **immer** | Fakten übers Projekt |
| Skills | nur bei Bedarf | Prozeduren |
| Slash-Commands | bei Aufruf | Abkürzungen |
| Subagents | eigener Kontext | Teilaufgaben auslagern |
| Hooks | Automatik | Determinismus erzwingen |

---

## 1. CLAUDE.md — die Projekt-Fakten

Wird bei **jeder** Anfrage mitgeladen. Mehrere Ebenen greifen zusammen:

| Datei | Gilt für | Committet? |
|---|---|---|
| `~/.claude/CLAUDE.md` | dich, überall | nein (liegt außerhalb) |
| `<projekt>/CLAUDE.md` | alle im Repo | **ja** |
| `<projekt>/CLAUDE.local.md` | nur dich, dies Repo | nein (auto-ignoriert) |

Bei Konflikt gewinnt die speziellere Ebene. Unterverzeichnisse können eigene haben — die
werden erst geladen, wenn dort gearbeitet wird (Monorepos).

**Hinein gehört:** Build-Befehle, Konventionen, Projekt-Eigenheiten, Fallstricke.
**Nicht hinein:** mehrstufige Prozeduren (→ Skills). Und nichts, was aus dem Code selbst
ablesbar ist — Claude kann lesen.

Faustregel: **unter ~200 Zeilen.** Eine aufgeblähte CLAUDE.md ist eine Dauersteuer auf jede
Anfrage.

`/init` erzeugt einen ersten Entwurf aus der Codebase.

## 2. Permissions — was ohne Rückfrage laufen darf

Der Punkt, der am meisten Alltagsreibung erzeugt oder eben nicht.

| Datei | Gilt für | Committen? |
|---|---|---|
| `~/.claude/settings.json` | dich, alle Projekte | – |
| `.claude/settings.json` | **alle im Repo** | **ja** |
| `.claude/settings.local.json` | nur dich, dies Repo | **nein** (gehört in `.gitignore`) |

Vorrang: local → project → user (spezifischer schlägt allgemeiner).

Drei Listen, und die Aufteilung ist die eigentliche Lehre:

```jsonc
{
  "permissions": {
    "allow": ["Bash(git status:*)", "Bash(git log:*)"],   // Lesen: keine Rückfrage
    "ask":   ["Bash(git push:*)", "Bash(gh pr merge:*)"], // Wirkung nach außen: fragen
    "deny":  ["Bash(git push --force:*)", "Read(./.env)"] // Unumkehrbar/geheim: nie
  }
}
```

**Merksatz: `allow` fürs Lesen, `ask` für Wirkung nach außen, `deny` für Unumkehrbares.**

Die Syntax `Bash(git log:*)` ist Präfix-Matching — der Doppelpunkt trennt Befehl und
Argumente. `deny` schlägt immer `allow`.

Der Sinn ist nicht Bequemlichkeit: Wer *alles* durchwinkt, liest die Rückfragen irgendwann
nicht mehr — und übersieht die eine, die zählte. Rückfragen sparsam einsetzen heißt, dass sie
wieder Signal tragen.

### Permission-Modi (Shift+Tab wechselt)

- `default` — fragt bei Edits und Shell-Befehlen
- `plan` — **nur lesen**, keine Änderungen. Für Recherche und Planung
- `acceptEdits` — Datei-Edits ohne Rückfrage
- `bypassPermissions` — keine Rückfragen. Nur in Container/Sandbox vertretbar

`plan` ist der unterschätzteste: erst verstehen lassen, dann bauen. Fehler im Plan sind
billig, Fehler im Code teuer.

## 3. Slash-Commands

Eingebaut: `/init`, `/clear`, `/compact`, `/context`, `/plan`, `/code-review`,
`/permissions`, `/doctor`, `/model`, `/diff`, `/mcp`.

Eigene: `.claude/commands/<name>.md` → aufrufbar als `/<name>`. Markdown mit optionalem
Frontmatter (`description`, `allowed-tools`), Argumente via `$ARGUMENTS`.

## 4. Skills — Prozeduren, die nur bei Bedarf laden

`.claude/skills/<name>/SKILL.md` (Projekt) oder `~/.claude/skills/` (überall).

**Der Unterschied zur CLAUDE.md ist der ganze Punkt:** CLAUDE.md ist *immer* geladen. Ein
Skill wird nur geladen, wenn er relevant ist. Deshalb darf er ausführlich sein, ohne zu
kosten.

> **Fakten → CLAUDE.md. Prozeduren → Skill.**

Ein Skill kann ein ganzes Verzeichnis sein (Hilfsskripte, Referenzdateien). Das `description`-
Feld entscheidet, ob Claude ihn findet — es ist kein Kommentar, sondern der Auslöser.

> **Vertiefung: `02-skills.md`.** Dort auch der aktuellere Stand: Slash-Commands (Abschnitt 3)
> und Skills sind inzwischen zu *einem* System zusammengeführt.

## 5. Subagents — eigener Kontext für Teilaufgaben

`.claude/agents/<name>.md` (Projekt) oder `~/.claude/agents/<name>.md` (überall).

⚠️ **Flache `.md`-Datei** — *nicht* `<name>/AGENT.md`. (Genau hier lag eine erste Recherche
falsch; das Verzeichnis-Format existiert nicht.)

```yaml
---
name: quick-reviewer          # PFLICHT: kleingeschrieben, Bindestriche
description: Scans code and suggests readability improvements   # PFLICHT: der Auslöser
tools: Read, Grep             # optional — hier: Leserechte, kein Schreiben
model: haiku                  # optional: sonnet|opus|haiku|fable|inherit
---

You are a code improvement specialist. For each issue, explain the problem
and show an improved version. Keep feedback concise and actionable.
```

Der Markdown-Body ist der System-Prompt. Weitere optionale Felder: `disallowedTools`,
`permissionMode`, `maxTurns`, `skills`, `effort`, `isolation: worktree`, `color`.

Aufruf: automatisch (Claude entscheidet per `description`), per Nennung im Prompt, garantiert
per `@agent-<name>`, oder via `claude --agent <name>`.

**Warum das nützt:** Ein Subagent hat ein *eigenes* Kontextfenster. Eine breite Suche liest
50 Dateien — im Hauptkontext wäre der danach voll. Als Subagent kommt nur das Ergebnis zurück.

> Die Recherche für dieses Projekt lief genau so: zwei Subagents parallel (Claude-Code-Doku
> und GitHub-Pages-Doku). Zurück kamen zwei Zusammenfassungen statt dutzender Doku-Seiten.

## 6. Hooks — Automatik, die nicht vergessen wird

In `settings.json` unter `"hooks"`. Wichtig: **Die Umgebung führt sie aus, nicht Claude.**
Das ist der Unterschied zwischen "soll immer passieren" und "passiert immer".

Events u.a.: `SessionStart`, `UserPromptSubmit`, `PreToolUse` (kann blockieren!),
`PostToolUse`, `Stop`, `SessionEnd`, `PreCompact`.

Typisch: Formatter nach jedem Edit (`PostToolUse`), Kontext beim Start laden
(`SessionStart` — genau das speist die `progress.md` unten ein), geschützte Dateien blocken
(`PreToolUse`).

Faustregel: **Bitte an Claude = kann übersehen werden. Hook = passiert.** Was wirklich immer
gelten muss, gehört in einen Hook, nicht in eine Anweisung.

## 7. Kontext-Management

- `/context` — Auslastung ansehen. Bei Trägheit zuerst hier schauen
- `/compact [fokus]` — Verlauf zusammenfassen, Platz schaffen. Optional mit Fokus
- `/clear` — frische Session. CLAUDE.md bleibt, Verlauf ist weg
- **Auto-Compact** — ab ~85% automatisch

Faustregel: **Neue Aufgabe → `/clear`.** Kontext aus einer erledigten Aufgabe hilft der
nächsten nicht, kostet aber weiter.

## 8. Terminal vs. IDE

| | Terminal | VS-Code-Extension |
|---|---|---|
| Diff-Ansicht | – | ✓ nativ |
| Selection-Kontext | – | ✓ `Alt+K` |
| `!befehl` direkt | ✓ | – |
| Tab-Completion | ✓ | – |
| Diagnostics | – | ✓ (Problems-Panel) |

`!befehl` im Terminal führt ihn aus und legt die Ausgabe in den Chat — ideal für
interaktives wie `gh auth login`, das Claude nicht selbst bedienen kann.

## 9. GitHub-Integration

Claude sieht Branch, uncommittete Änderungen und letzte Commits von selbst. Mit `gh` kommen
PRs, CI-Logs und Issues dazu.

`/install-github-app` richtet die Claude-GitHub-App fürs Repo ein (`@claude`-Erwähnungen in
Issues/PRs, optional Actions-Workflows). **Braucht Repo-Admin-Rechte.** Für rein lokales
Arbeiten nicht nötig — es ist eher ein Team-Feature.

## 10. Projektverlauf über Sitzungen

`/clear` und Compaction löschen den Verlauf — das Projektwissen darf nicht mitsterben.
Lösung: `.claude/progress.md`, per `SessionStart`-Hook beim Start eingespeist.

Zwei Zonen:
- **Kopf** (wird überschrieben): Ziel, Stand, nächste Schritte, offene Fragen
- **Verlauf** (nur anhängen, neueste unten): `## YYYY-MM-DD — Titel` mit *was* und *warum so*

Das *Warum* ist der eigentliche Wert. Was gemacht wurde, steht in der Git-Historie. Warum es
so und nicht anders entschieden wurde, steht **nirgends** — außer hier.

Nicht committen: persönlicher Arbeitsstand → `.git/info/exclude`.

---

## Die Kernidee

Der rote Faden hinter allem:

> **Kontext ist endlich. Lade Fakten immer (CLAUDE.md), Prozeduren bei Bedarf (Skills),
> lagere Breitensuche aus (Subagents), erzwinge Unverzichtbares deterministisch (Hooks).**

Das gilt unabhängig von Version und Projekt.
