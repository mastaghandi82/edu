# Claude Code: Skills — Prozeduren als Dateien

> Vertiefung zum Skills-Abschnitt in `01-workflow.md`. Alles hier gegen die offizielle
> Doku verifiziert (code.claude.com/docs, Stand Juli 2026). Was nicht verifizierbar war,
> ist am Ende ausdrücklich benannt.

## Wofür Skills gut sind

Ein Skill ist eine **Prozedur als Datei**: eine Markdown-Anleitung, die Claude lädt, wenn —
und nur wenn — sie gebraucht wird. Das löst zwei Probleme auf einmal:

1. **Wiederholung.** Wer Claude dreimal erklärt hat, wie in diesem Projekt deployt wird,
   schreibt es beim vierten Mal als Skill auf. Ab dann kennt jede Session den Ablauf —
   auch die von Kolleg:innen, denn Projekt-Skills werden committet.
2. **Kontextkosten.** Die Alternative wäre, alles in die CLAUDE.md zu schreiben — aber die
   ist *immer* geladen und kostet bei jeder Anfrage. Ein Skill kostet fast nichts, solange
   er nicht gebraucht wird (Details unten).

Daraus folgt die Abgrenzung, die schon in `01-workflow.md` steht und hier der rote Faden ist:

> **Fakten → CLAUDE.md. Prozeduren → Skill.**

Konventionen, Build-Befehle, Projekteigenheiten: CLAUDE.md. Mehrstufige Abläufe,
Checklisten, Referenzmaterial, Workflows mit Nebenwirkungen: Skill.

## Skills und Slash-Commands sind zusammengeführt

Historische Fußnote, die Verwirrung erspart: Eigene Slash-Commands
(`.claude/commands/<name>.md`) und Skills waren früher getrennte Mechanismen. **Inzwischen
sind sie ein System.** Beide erzeugen `/<name>`, beide funktionieren gleich — alte
`commands`-Dateien laufen weiter, aber Skills sind der empfohlene Weg, weil sie mehr können:

- ein **Verzeichnis** statt einer Einzeldatei (Hilfsskripte, Referenzdateien),
- Frontmatter-Felder, die steuern, *wer* den Skill auslösen darf,
- **automatisches Laden** durch Claude, wenn die Beschreibung passt.

Der Skills-Abschnitt in `01-workflow.md` behandelt Commands und Skills noch als zwei
Bausteine — das war der Stand der damaligen Recherche. Diese Seite ist die aktuellere.

## Die Kernmechanik: Progressive Disclosure

Der Grund, warum Skills „fast nichts kosten": Sie werden **gestaffelt** geladen.

| Stufe | Was ist im Kontext | Kosten |
|---|---|---|
| immer | nur `name` + `description` | ~100 Tokens pro Skill |
| beim Auslösen | der komplette SKILL.md-Body | einmalig, bleibt für die Session |
| bei Bedarf | Zusatzdateien (`reference.md`, Skripte) | null, bis Claude sie liest |

Das erklärt zwei Dinge sofort:

- **Warum die `description` alles entscheidet.** Sie ist das Einzige, was Claude sieht,
  bevor der Skill geladen ist. Eine vage Beschreibung = ein Skill, der nie feuert.
- **Warum ein Skill ausführlich sein darf.** Der Body kostet erst beim Auslösen — anders
  als jede Zeile CLAUDE.md, die eine Dauersteuer ist.

Zwei Budget-Details für später (nicht fürs Verständnis nötig, aber gut zu wissen):
Die Skill-Liste (alle Beschreibungen zusammen) bekommt etwa **1 % des Kontextfensters**;
wird das knapp, kürzt Claude Code selten genutzte Beschreibungen. Pro Skill sind
`description` + `when_to_use` zusammen auf **1.536 Zeichen** begrenzt. Nach einer
Auto-Compaction werden bereits geladene Skills wieder angehängt — gedeckelt auf 5.000
Tokens pro Skill und 25.000 insgesamt.

## Wo Skills liegen

| Ebene | Pfad | Gilt für | Committen? |
|---|---|---|---|
| persönlich | `~/.claude/skills/<name>/SKILL.md` | dich, alle Projekte | nein (liegt außerhalb) |
| Projekt | `.claude/skills/<name>/SKILL.md` | alle im Repo | **ja** |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | wo das Plugin aktiv ist | – |
| Enterprise | verwaltete Einstellungen | ganze Organisation | – |

Bei Namenskonflikten gewinnt: Enterprise vor persönlich vor Projekt. Plugin-Skills haben
einen eigenen Namensraum (`plugin-name:skill-name`) und kollidieren nicht.

Praktisch relevant:

- **Monorepos:** Auch `.claude/skills/` in Unterverzeichnissen wird gefunden; solche Skills
  erscheinen zusätzlich unter qualifiziertem Namen wie `apps/web:deploy`.
- **Live-Reload:** Änderungen an `SKILL.md` wirken sofort, ohne Neustart der Session.

## Aufbau eines Skills

Minimal ist ein Skill ein Verzeichnis mit einer Datei:

```
deploy/
└── SKILL.md
```

Ausgebaut darf es ein kleines Paket sein:

```
deploy/
├── SKILL.md           # Pflicht: Übersicht + Anleitung
├── reference.md       # Detailwissen — wird erst bei Bedarf gelesen
├── examples.md        # Beispiele — dito
└── scripts/
    └── check.sh       # ausführbare Helfer
```

Faustregel aus der offiziellen Doku: **SKILL.md unter ~500 Zeilen** halten, Detailtiefe in
Zusatzdateien auslagern. Das ist Progressive Disclosure auf Dateiebene: SKILL.md sagt, *was*
es gibt und *wann* man wohin schaut; die Tiefe kostet erst beim Zugriff.

Ein einfaches, vollständiges Beispiel:

```yaml
---
description: Deployt site/ nach GitHub Pages und verifiziert die Live-URL.
  Nutzen bei "deploy", "veröffentlichen", "live stellen".
disable-model-invocation: true
---

1. Sicherstellen, dass auf einem Feature-Branch gearbeitet wird (nie direkt auf main).
2. PR erstellen, mergen — der Workflow deployt automatisch bei Push auf main.
3. Lauf beobachten: `gh run list` → ID holen → `gh run watch <id> --exit-status`.
4. Erst fertig melden, wenn die Live-URL im Browser die Änderung zeigt —
   ein grüner Actions-Lauf allein reicht nicht.
```

> **Lebendes Beispiel:** Dieses Repo enthält selbst einen Skill —
> `.claude/skills/deploy-check/SKILL.md` verifiziert Deployments nach den Regeln aus
> `docs/git/02-deployment.md`. Er wendet an, was diese Seite beschreibt: Trigger-Wörter in
> der description, `allowed-tools` für die Lesebefehle, Fallen-Tabelle im Body.
>
> Sein erster Ernstfall — die Verifikation seines eigenen Deployments — lieferte prompt
> zwei neue Fallen (unten in der Tabelle): Die Cloud-Session hatte weder `gh` noch freien
> Netzzugang. Der Skill funktionierte trotzdem, weil seine Schritte das *Ziel* nennen
> („den eigenen Lauf finden"), nicht nur das Werkzeug — die Prozedur ließ sich auf die
> GitHub-API-Tools übertragen. Die Live-Prüfung blieb offen und wurde ausdrücklich so
> gemeldet, statt den grünen Lauf als Beweis auszugeben.

## Das Frontmatter: die wichtigsten Felder

Alle Felder sind optional — sogar `name` (Standard: der Verzeichnisname). Aber ohne
`description` kann Claude den Skill nicht selbstständig finden.

| Feld | Bedeutung |
|---|---|
| `description` | *Was* der Skill tut und *wann* er gebraucht wird. **Der Auslöser.** |
| `when_to_use` | Zusätzliche Trigger-Hinweise; wird an die description angehängt |
| `disable-model-invocation` | `true` = nur manuell per `/<name>`, Claude lädt ihn nie von selbst |
| `user-invocable` | `false` = umgekehrt: nicht im `/`-Menü, nur Claude kann ihn nutzen |
| `argument-hint` | Autocomplete-Hinweis, z. B. `[issue-nummer]` |
| `arguments` | benannte Argumente für `$name`-Platzhalter |
| `allowed-tools` | Tools, die während des Skills ohne Rückfrage laufen dürfen |
| `disallowed-tools` | Tools, die während des Skills gesperrt sind |
| `model` / `effort` | Modell bzw. Denkaufwand nur für diesen Skill überschreiben |
| `context: fork` | Skill läuft in einem eigenen Subagent-Kontext (`agent` wählt welchen) |
| `paths` | Glob-Muster: Skill nur relevant, wenn passende Dateien berührt werden |
| `hooks` | Hooks, die nur für die Laufzeit dieses Skills gelten |
| `shell` | Shell für eingebettete Befehle: `bash` (Default) oder `powershell` |

Das Gespann `disable-model-invocation` / `user-invocable` ist die eigentliche
Design-Entscheidung pro Skill:

- **Workflow mit Nebenwirkungen** (deployt, pusht, löscht): `disable-model-invocation: true`.
  So bleibt der Mensch der Auslöser — Claude soll so etwas nicht „bei Gelegenheit" starten.
- **Reines Wissen** (Konventionen, Referenz): Auto-Invocation anlassen, damit es wirkt,
  ohne dass man dran denken muss.

## Auslösen und Argumente

Zwei Wege, ein Ergebnis:

1. **Automatisch** — Claude erkennt anhand der `description`, dass der Skill zur Aufgabe
   passt, und lädt ihn.
2. **Manuell** — `/skill-name argument1 "argument mit leerzeichen"` im Prompt.

Im Skill-Text stehen dafür Platzhalter zur Verfügung:

| Platzhalter | Bedeutung |
|---|---|
| `$ARGUMENTS` | alle Argumente als ein String |
| `$0`, `$1`, … | einzelne Argumente (0-basiert; Langform `$ARGUMENTS[0]`) |
| `$name` | benanntes Argument aus dem `arguments`-Frontmatter |
| `${CLAUDE_SKILL_DIR}` | Verzeichnis der SKILL.md — für Pfade zu Hilfsskripten |
| `${CLAUDE_PROJECT_DIR}` | Projektwurzel |

Mehrwortige Argumente brauchen Anführungszeichen; ein Literal-`$` wird mit `\$` escaped.

### Dynamischer Kontext: `` !`befehl` ``

Ein Skill kann Shell-Befehle einbetten, deren **Ausgabe** in den Text eingesetzt wird,
*bevor* Claude ihn sieht:

```markdown
Aktueller Stand:
!`git status --short`
```

Der Befehl läuft beim Laden des Skills; Claude bekommt nicht die Zeile, sondern das
Ergebnis. So startet eine Prozedur immer mit frischen Fakten statt mit Annahmen.
Stolperstein: Das `!` muss am Zeilenanfang oder nach einem Leerzeichen stehen —
`KEY=!`cmd`` wird *nicht* expandiert. Abschaltbar per
`"disableSkillShellExecution": true` in den Settings.

## Gute Skills schreiben

Destilliert aus der offiziellen Best-Practices-Doku plus dem, was sich hier im Projekt
bewährt hat:

1. **Die description ist das Produkt.** Sie muss sagen, *was* der Skill tut **und** *wann*
   er dran ist — mit den Wörtern, die in echten Aufgaben vorkommen („deploy",
   „veröffentlichen"), nicht mit Binnenjargon. Wichtigster Anwendungsfall zuerst.
2. **Eine Prozedur pro Skill.** Ein Skill „alles rund um Git" triggert nie richtig und
   lädt zu viel. Lieber drei kleine als einen breiten.
3. **Schreiben wie für neue Kolleg:innen:** nummerierte Schritte, konkrete Befehle,
   erwartete Ergebnisse, bekannte Fallen. Genau das, was man beim dritten Mal mündlich
   erklärt hätte.
4. **Nebenwirkungen absichern:** `disable-model-invocation: true` für alles, was nach
   außen wirkt.
5. **Testen wie ein Feature:** frische Session, Aufgabe stellen, die den Skill auslösen
   *sollte* — feuert er? Dann eine, die ihn *nicht* auslösen sollte — bleibt er still?
   Wer es gründlich mag: Anthropics offizielles Plugin `skill-creator`
   (`/plugin install skill-creator@claude-plugins-official`) kann Testfälle definieren,
   Läufe mit/ohne Skill vergleichen und die Trigger-Rate der description messen.

## Fallstricke

| Symptom | Ursache | Abhilfe |
|---|---|---|
| Skill feuert nie | description zu vage / falsche Wörter | Trigger-Wörter aufnehmen; manuell per `/<name>` gegentesten |
| Skill feuert dauernd | description zu breit | enger fassen oder `disable-model-invocation: true` |
| Skill lädt, aber ohne Beschreibung | YAML-Frontmatter fehlerhaft | mit `--debug` starten — Parse-Fehler werden angezeigt |
| Beschreibungen abgeschnitten | zu viele Skills fürs 1-%-Budget | `/doctor` prüfen; selten Genutztes deaktivieren |
| `` !`cmd` `` läuft nicht | `!` nicht am Zeilen-/Wortanfang | Position korrigieren |
| Skill „vergessen" nach langer Session | Compaction hat gekürzt | Skill einfach neu aufrufen |
| Skill nennt Werkzeuge, die die Umgebung nicht hat | lokal geschrieben, in Cloud/Sandbox ausgeführt (kein `gh`, kein freier Netzzugang) | Schritte ums *Ziel* formulieren, Befehl als Standardweg — dann ist der Ausweichweg erlaubt |
| Verifikationsschritt scheitert mit 403/Timeout | Sandbox-Netzwerk-Policy, nicht das Deployment | Ursache trennen (Proxy-Status prüfen); Offenes offen melden statt grün = fertig |

Und der klassische Denkfehler, gegen den diese Seite anschreibt: **alles in die CLAUDE.md
kippen.** Wenn Claude eine Regel ignoriert, ist die CLAUDE.md meist zu lang — die Antwort
ist nicht *mehr* Text dort, sondern auslagern in Skills.

## Über Claude Code hinaus (Portabilität)

Skills gibt es auch außerhalb von Claude Code — aber **sie synchronisieren nicht**:

- **Agent SDK** (eigene Agenten in Python/TypeScript): lädt dieselben Skills aus dem
  Dateisystem (`~/.claude/skills/`, `.claude/skills/`). Einschränkung: `allowed-tools`
  im Frontmatter wird dort **ignoriert** — Tool-Rechte steuert das SDK selbst.
- **claude.ai:** eigene Skills als ZIP-Upload, pro Benutzer.
- **Claude API:** eigene Skills per Upload (`/v1/skills`), workspace-weit; Laufzeit ohne
  Netzwerkzugriff.

Ein Skill ist also portables *Format*, aber kein geteilter *Speicher*: Wer denselben Skill
in Claude Code und auf claude.ai will, pflegt ihn zweimal.

## Nicht verifizierbar

Der Vollständigkeit halber, gemäß Projektregel („nichts behaupten, was nicht geprüft ist"):

- Eine **Obergrenze für die Anzahl** von Skills ist nirgends dokumentiert — praktisch
  begrenzt nur das Kontextbudget.
- Ein **Größenlimit für einzelne SKILL.md-Dateien** ist nicht dokumentiert (die ~500
  Zeilen sind Empfehlung, kein Limit).

---

## Die Kernidee

> **Ein Skill ist aufgeschriebene Erfahrung mit einem Auslöser dran.** Die description
> entscheidet, ob er gefunden wird; Progressive Disclosure sorgt dafür, dass Wissen erst
> kostet, wenn es gebraucht wird. Wer eine Erklärung zum dritten Mal tippt, hat einen
> Skill identifiziert.
