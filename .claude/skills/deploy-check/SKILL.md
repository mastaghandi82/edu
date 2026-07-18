---
description: Verifiziert ein GitHub-Pages-Deployment dieses Repos — Actions-Lauf
  beobachten, dann Live-URL prüfen. Nutzen nach jedem Merge auf main und bei Fragen
  wie "ist die Seite live?", "deploy prüfen", "hat das Deployment geklappt?".
argument-hint: "[commit-sha]"
allowed-tools: Bash(gh run list:*), Bash(gh run view:*), Bash(gh run watch:*), Bash(curl -sI:*), Bash(curl -s:*)
---

# Deployment verifizieren

Ein Deployment gilt erst als erfolgreich, wenn die **Live-URL die Änderung zeigt**.
Ein grüner Actions-Lauf allein reicht nicht (Klassiker: Workflow grün, Seite 404,
weil die Pages-Quelle nicht auf "GitHub Actions" steht).

Die `gh`-Befehle unten sind der Standardweg. In Umgebungen ohne `gh` (z. B. Claude Code
in der Cloud) dasselbe Ziel über die GitHub-API-Tools erreichen: Läufe listen, den
eigenen per SHA identifizieren, Status abfragen.

Falls ein Commit-SHA übergeben wurde, gilt der: $ARGUMENTS

## Schritte

1. **Den eigenen Lauf finden — nicht blind den neuesten nehmen:**

   ```
   gh run list --branch main --limit 5
   ```

   Den Lauf über den Commit-SHA des eigenen Merges identifizieren (`git log -1 origin/main`
   zum Vergleich). Es können parallel andere Läufe existieren.

2. **Lauf beobachten — ohne TTY zwingend mit ID:**

   ```
   gh run watch <run-id> --exit-status
   ```

   `--exit-status` sorgt dafür, dass ein fehlgeschlagener Lauf auch als Fehler endet.

3. **Bei Fehlschlag eingrenzen:**

   ```
   gh run view <run-id> --log-failed
   ```

4. **Live-URL prüfen — der eigentliche Beweis:**

   ```
   curl -sI https://mastaghandi82.github.io/edu/
   ```

   HTTP 200 erwartet. Danach den Inhalt der geänderten Seite abrufen und prüfen, dass
   die **konkrete Änderung** sichtbar ist (z. B. per `curl -s <seiten-url>` nach einem
   neuen Textbaustein greppen) — nicht nur, dass irgendeine Seite antwortet. GitHub
   Pages cached; wenige Minuten Verzögerung sind normal, erst danach ist es ein Fehler.

5. **Ergebnis melden:** Erst "deployt" sagen, wenn Schritt 4 die Änderung gezeigt hat.
   Wenn etwas offen blieb (z. B. Cache-Verzögerung), das ausdrücklich benennen.

## Bekannte Fallen

| Symptom | Ursache |
|---|---|
| Workflow grün, Seite 404 | Pages-Quelle nicht auf "GitHub Actions" gestellt |
| `gh run watch` hängt/scheitert | ohne TTY keine interaktive Auswahl — ID mitgeben |
| falscher Lauf beobachtet | neuesten statt eigenen Lauf genommen — SHA vergleichen |
| Änderung nicht sichtbar | Pages-Cache — kurz warten, dann erst als Fehler werten |
| `curl` auf Live-URL gibt 403 | Sandbox-Netzwerk-Policy blockt den Ausgang — kein Deploy-Fehler. Erst Proxy-Status prüfen, dann die Live-Prüfung ausdrücklich als offen melden und im Browser außerhalb erledigen lassen |
| `gh` nicht vorhanden | Cloud-/CI-Umgebung — GitHub-API-Tools nutzen (siehe oben) |
