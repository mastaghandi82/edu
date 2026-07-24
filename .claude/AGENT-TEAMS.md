# Agent Teams im edu-Projekt — Kurzanleitung

Interne Arbeitsnotiz (nicht Teil der veröffentlichten `site/`). Wie du das experimentelle
Feature **Agent Teams** in diesem Repo best-practice einsetzt.

## Was aktiv ist

- Schalter: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` unter `env` in
  `C:\Users\renep\.claude\settings.json` (global, gilt für alle Projekte).
- Anzeige-Modus: **`in-process`** (Default). Split-Panes (tmux/iTerm2) gibt es unter
  Windows/PowerShell nicht — `in-process` ist hier der einzig sinnvolle Modus. Alle Teammates
  laufen im selben Terminal, Navigation per Pfeiltasten.

## Was Agent Teams sind (Abgrenzung)

- **Agent Teams:** mehrere *echte* Claude-Code-Instanzen (Teammates) laufen parallel,
  koordiniert über die Lead-Session und eine **geteilte Task-Liste**; Teammates können
  **untereinander** kommunizieren.
- **Subagents** (`.claude/agents/*.md`, per Agent-Tool): laufen innerhalb *deiner* Session und
  berichten nur an den Main-Agent zurück — keine Inter-Agent-Kommunikation.
- Die hier angelegten Rollen (`didaktik-review`, `faktencheck`, `konsistenz-check`) sind
  Agent-Definitionen und lassen sich **beides**: als Subagent *und* als Teammate-Rolle nutzen.

## Spawnen (natürlichsprachig)

Beispiel-Prompt an die Lead-Session — reviewt eine Seite unter drei Blickwinkeln parallel:

```
Spawn 3 teammates, um site/claude-code.html und docs/claude-code/01-workflow.md zu reviewen:
- einen mit dem Agent-Typ didaktik-review (Einsteiger-Verständlichkeit)
- einen mit dem Agent-Typ faktencheck (Versionen, CLI-Flags, Zitate belegen)
- einen mit dem Agent-Typ konsistenz-check (site/ gegen docs/ abgleichen, Links, style.css)
Jeder gibt strukturierte Befunde mit Datei:Zeile zurück; ich wende die Fixes an.
```

Einzelnen Teammate ansprechen: „Sag dem faktencheck-Teammate, es soll auch die
gh-CLI-Version prüfen." (Jeder wird einzeln adressiert.)

## Steuern im Agent-Panel

- **↑ / ↓** — Teammate auswählen
- **Enter** — Transcript öffnen / direkt mit dem Teammate chatten
- **Escape** — Teammate unterbrechen
- **Ctrl+T** — Task-Liste ein-/ausblenden

## Best Practices

- **3–5 Teammates.** Token-Kosten skalieren linear mit der Teamgröße — größer lohnt selten.
- **Self-contained Tasks**, je 5–6 pro Teammate, mit klarem Deliverable.
- **Unterschiedliche Dateien pro Teammate** — verhindert Race Conditions bei gleichzeitigen
  Edits. Unsere drei Rollen sind bewusst read-only (sie melden, der Lead editiert).
- **Kontext im Spawn-Prompt mitgeben:** Teammates erben die Conversation-History des Lead
  **nicht** — alles Nötige in den Auftrag schreiben.
- **Regelmäßig Check-in**, nicht unbeaufsichtigt laufen lassen.

## Wann in edu sinnvoll — und wann nicht

- ✅ Sinnvoll: mehrere Themenseiten *gleichzeitig* entwerfen; eine Seite unter mehreren
  Blickwinkeln (Didaktik / Fakten / Konsistenz) *parallel* reviewen; breite Recherche.
- ❌ Wenig Nutzen: sequenzielle Prosa-Arbeit an *einer* Seite (der edu-Alltag) oder Edits an
  derselben Datei — hier überwiegt der Koordinations-Overhead.

## Grenzen (experimentell)

- Keine Session-Resumption (`/resume`, `/rewind`) mit in-process Teammates.
- Nur **ein** Team pro Session; keine verschachtelten Teams (Teammates spawnen keine eigenen).
- Shutdown kann verzögert sein (laufende Requests werden zuerst beendet).
- Lineare Token-Kosten — bewusst dosieren.

## Offizielle Doku

- Agent Teams: https://code.claude.com/docs/en/agent-teams.md
- Subagents (Vergleich): https://code.claude.com/docs/en/sub-agents.md
