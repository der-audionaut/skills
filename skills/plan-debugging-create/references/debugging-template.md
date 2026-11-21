# Template for the debugging plan document

The header block keys `Claude-Session` and `Status`, the table headers of `## Arbeitsschritte` and `## Parallel abarbeitbar`, `## Schritte` with `### Schritt N — <Titel>` and the note on progress marking are taken verbatim from `skills/plan-create/references/plan-template.md` and are brought in line here whenever the marking logic of `/plan-coding` changes there — only then do `/plan-review`, `/plan-lint` and `/plan-coding` find the same anchors in a debugging plan as in a feature plan. `## Ursachenanalyse` is the anchor by which review and lint recognize a debugging plan; it is therefore never left out, not even when the cause stays a suspicion. In another output language it is translated like every other heading — review and lint recognize it by its meaning.

Structure and order are binding. Sections the bug does not touch you leave out instead of leaving them empty — an empty section is dead knowledge before the plan has even been touched. Do **not** create history sections; they come into being when `/plan-review`, `/plan-coding` or `/plan-lint` run for the first time.

Replace placeholders in angle brackets; do not carry over the comments in parentheses.

---

````markdown
---
Debugging: <kurzer Titel>
Fehlerbild: <ein Satz: was falsch passiert, aus Sicht dessen, der es bemerkt>
Erstellt: <YYYY-MM-DD>
Claude-Session: <Session-ID>
Status: Entwurf
---

# <Debugging-Titel>

<Zwei, drei Sätze: welches Symptom, welche Grundursache mit Einstufung `belegt` oder `Verdacht`, welche Lösung. Behoben wird genau dieses fehlerhafte Verhalten — das steht hier einmal und gilt für jeden Schritt.>

## Arbeitsschritte

| Nr | Schritt | Ergebnis | Abhängig von | Welle |
|----|---------|----------|--------------|-------|
| 1 | <Fix an der Grundursache> | <Regressionstest grün, Symptom tritt nicht mehr auf> | — | W1 |
| 2 | <Gleichartige Stelle in Modul B> | <…> | 1 | W2 |
| 3 | <Randfall-Test R1 — ein Fall, den der Ist-Stand schon richtig behandelt> | <…> | 1 | W2 |

## Parallel abarbeitbar

| Welle | Schritte | Voraussetzung | Berührungspunkte |
|-------|----------|---------------|------------------|
| W1 | 1 | — | — |
| W2 | 2, 3 | W1 abgeschlossen | keine gemeinsamen Dateien — sonst gehört Schritt 3 in eine eigene Welle |

<Ein Satz zur Einordnung: was sich dadurch tatsächlich verkürzt — oder dass keine echte Parallelität besteht.>

## Ziel

<Was gilt als erreicht? Prüfbar formuliert: welches Symptom unter welcher Eingabe nicht mehr auftritt, welche Regressionstests grün sind, welche Randfälle abgedeckt sind. Bei `Verdacht`: welche Hypothese die Diagnose bestätigt oder verwirft.>

## Nicht-Ziele

- <was ausdrücklich nicht dazugehört — darunter Refactorings und Optimierungen, die beim Lesen auffielen, als `<Datei:Zeile> — <Befund> — siehe /plan-refactoring-create` bzw. `/plan-optimization-create`>

## Funktionsweise

### Soll-Verhalten
<Was der Code tun soll, und woher das bekannt ist: Test, Doku, Ticket, Namensgebung. Wo das Soll uneindeutig ist, steht es als Annahme und als offene Frage.>

### Pfad
<Der betroffene Pfad vom Einstieg bis zum Symptom als Kette von Datei:Zeile, mit dem, was an jeder Station mit den Daten passiert.>

### Build und Tests
<Die dokumentierten Befehle und das Ergebnis des Laufs zum Zeitpunkt der Analyse: grün, rot mit den roten Tests, „unbekannt", wenn kein Befehl dokumentiert ist, oder „nicht ausgeführt — lädt nicht versionierte Konfiguration (<Datei oder Variablen>)".>

### Nicht gelesen
<Was ungelesen blieb und warum.>

## Fehlerbild und Reproduktion

- **Symptom:** <was falsch passiert — Ausgabe, Fehlermeldung, Zustand; Auszüge aus Logs und Stacktraces nur so weit, wie sie den Fehler belegen, personenbezogene Werte, Tokens und Zugangsdaten als `<redacted>`>
- **Umgebung:** <Version, Konfiguration, Datenlage, soweit bekannt; Unbekanntes als „unbekannt">
- **Reproduktion:** <Befehl oder Test; ein eigenes Skript mit seinem Ablageort außerhalb des Repositorys>
- **Ergebnis:** reproduziert | nicht reproduziert — <Ausgabe-Auszug, oder warum nicht>
- **Diagnoseergebnis:** <nur im Folgeplan: Vorgängerplan, was dessen Diagnose beobachtet hat, welche `H` es bestätigt oder verwirft — Werte geschwärzt>

## Ursachenanalyse

### Hypothesen

#### H1 — <Titel>  [bestätigt | verworfen | offen]
- **Vorhersage:** <was zu beobachten wäre, wenn sie stimmt>
- **Prüfung:** <Befehl, Grep, Test oder gelesene Stelle Datei:Zeile>
- **Ergebnis:** <was sich gezeigt hat — bei `verworfen` der Gegenbeleg>

#### H2 — <Titel>  [bestätigt | verworfen | offen]
(gleiche Struktur)

### Grundursache
- **Fundstelle:** <Datei:Zeile, an der das falsche Verhalten beginnt — nicht, wo es sichtbar wird>
- **Kausalkette:** <Ursache → Zwischenzustand → … → Symptom, je Glied mit Datei:Zeile>
- **Herkunft:** <Commit aus `git log -S`, `git log -L` oder `git blame`, oder „nicht ermittelt">
- **Fehlerklasse:** <aus `fault-categories.md`, Teil A>
- **Einstufung:** belegt | Verdacht — <bei `Verdacht`: was zur Bestätigung fehlt>

### Gleichartige Stellen
- <Datei:Zeile — dasselbe Muster, gefunden mit `<Grep-Muster>` — Schritt <Nr>, Nicht-Ziel — <Grund>, oder bei `Verdacht` Skizze unter `## Lösung`>

## Randfälle

| Nr | Eingabe | Ist | Soll | Abdeckung |
|----|---------|-----|------|-----------|
| R1 | <…> | <…> | <…> | Schritt 3 |
| R2 | <…> | <…> | <…> | Nicht-Ziel — <Grund> |

## Lösung

- **Strategie:** <Fix-Strategie aus `fault-categories.md`, Teil B, angesetzt an der Fundstelle der Grundursache>
- **Verworfene Alternativen:** <insbesondere das Symptom-Pflaster, das sich anbietet, und warum nicht>
- **Grenzvalidierung:** <Validierung oder Normalisierung der Eingabe an einer Systemgrenze, zusätzlich zum Fix und nur wo begründet — oder „keine"; die Regressionstests stehen in den Schritten, nicht hier>
- **Datenreparatur:** <bereits falsche Daten in Produktion: welche, wie erkennbar, wie zu reparieren — oder „nicht nötig", mit Grund>
- **Nach der Diagnose:** <nur bei `Verdacht`: Skizze des Fixes je Hypothese, der Hinweis „nach Diagnose: `/plan-debugging-create "<Plan-Datei> — Diagnose: <Ergebnis>"`" und zurückzubauendes Diagnose-Logging mit Datei:Zeile — oder „dauerhaft, siehe Rollout">

## Schritte

### Schritt 1 — <Titel>
- **Ergebnis:** <was danach nachweisbar gilt>
- **Betroffen:** <Dateien, Module>
- **Abhängig von:** <Schrittnummern oder —>
- **Vorgehen:** <beim Fix-Schritt zuerst: Test schreiben, gegen den Ist-Stand ausführen, Rot im Bericht festhalten; dann der Fix an der Fundstelle — zwei bis fünf Sätze, konkret genug zum Umsetzen, ohne fertigen Code>
- **Regressionstest:** <Testname und Befehl — bei einem Diagnose- oder Rückbau-Schritt „keiner — <Grund>">
- **Akzeptanz:** <beim Fix-Schritt: Testausgabe rot vor dem Fix (Ausgabe-Auszug), grün danach, übrige Tests im Stand der Baseline; beim Diagnose-Schritt: welches Ergebnis welche Hypothese bestätigt oder verwirft>

### Schritt 2 — <Titel>
(gleiche Struktur)

## Teststrategie

<Welche Ebene deckt was ab; was sich nur manuell prüfen lässt und wie. Die Regressionstests stehen in den Schritten, die Abdeckung der Randfälle in `## Randfälle`.>

## Rollout

<Auslieferung, woran in Produktion sichtbar wird, dass der Fehler weg ist — Log, Metrik, Fehlerrate —, Datenreparatur in ihrer Reihenfolge zum Fix, Rückweg. Dauerhaftes Diagnose-Logging mit Begründung; es protokolliert nur Form, Typ, Länge oder einen Hash mit Schlüssel (HMAC, Schlüssel nicht im Repository), nie personenbezogene Werte, Tokens oder Zugangsdaten und nie deren einfachen Hash.>

## Risiken

- <Risiko> → <Gegenmaßnahme oder bewusste Inkaufnahme>

## Annahmen

- <Annahme, die mangels Antwort, Beleg oder Reproduktion getroffen wurde> → <was sie umwirft>

## Offene Fragen

1. <Frage, die nur der Mensch beantworten kann> → betrifft Schritt <Nr>
````

---

## Notes on the template

- **The step overview and the parallel overview come before everything else.** They are the reason you open the plan; the root-cause analysis is what you read the second time.
- **The three dependency statements must agree** — the column "Abhängig von" in the overview, the wave in both tables, `Abhängig von` in the detailed step. Contradictions here are the most common lint finding.
- **The `Welle` column and the header block's `Status` record progress later.** `/plan-coding` appends `✓ erledigt` to the wave code in the step overview (`W1 ✓ erledigt`), puts a first bullet `**Status:** erledigt <Datum>` into the detailed step, and counts in the header block from `Entwurf` via `In Umsetzung — n von N Schritten erledigt` to `Umgesetzt`. That is why the table gets no status column of its own, and `Status: Entwurf` is the starting value, not a placeholder.
- **`Regressionstest` is a mandatory bullet in every step.** `/plan-review` treats a step without it as a blocker.
- **Red before the fix is part of the step, not of the tool.** `/plan-coding` only ever commits a green state and knows no test-first of its own, so test and fix share one step whose `Vorgehen` and `Akzeptanz` carry the red run. A test written after the fix is green from the start and proves nothing.
- **A suspected cause gets a diagnosis, not a fix.** If the root cause is `Verdacht`, the step sequence ends with a diagnosis step — a test with production-shaped data, logging or a metric, committable because it records the symptom instead of asserting it. Diagnostic logging records only the shape, type or length of a value, or a keyed hash (HMAC with a secret that is not committed), never personal values, tokens or credentials and never a plain hash of them; the step's acceptance names what it excludes. Fixes exist only as a sketch under `## Lösung`, and the plan is created anew with the diagnosis result: the follow-up plan reads this one, takes over the fix sketched for the hypothesis the diagnosis confirmed and, if noted for removal, the removal of diagnostic logging as steps, and names this plan as its predecessor, so the diagnosis stays the last step here.
- **Production data appears only as a minimal, redacted excerpt**, because the plan lives in the repository; if the error report consists of such content, the plan names only the shape and type of the values.
- **Rejected hypotheses stay**, with the counter-evidence. They are the reason the next reader does not search there again.
- **Every `R` is covered by a step or marked as a non-goal with a reason** in its table row; an edge case without either is a gap.
- **The plan fixes the one faulty behavior and nothing beside it.**
- **Assumptions and open questions stay in the document**, even once they are answered — then with the answer next to them. Cleaning up is done by `/plan-lint`, not the author.
- **A different output language translates the labels, not the structure.** Headings, column titles, header block labels and prose appear in the chosen language; order, tables, step numbers, the wave codes `W1, W2 …`, the identifiers `H1, H2 …` and `R1, R2 …` and the key `Claude-Session` stay unchanged so that the downstream skills find them again.
- **No code blocks with finished implementation** in the steps. Signatures, call chains and example inputs yes, whole functions no: they go stale between planning and implementation and are then read as truth.
