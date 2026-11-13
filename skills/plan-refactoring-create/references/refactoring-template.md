# Template for the refactoring plan document

The header block keys `Claude-Session` and `Status`, the table headers of `## Arbeitsschritte` and `## Parallel abarbeitbar`, `## Schritte` with `### Schritt N — <Titel>` and the note on progress marking are taken verbatim from `skills/plan-create/references/plan-template.md` and are brought in line here whenever the marking logic of `/plan-coding` changes there — only then do `/plan-review`, `/plan-lint` and `/plan-coding` find the same anchors in a refactoring plan as in a feature plan.

Structure and order are binding. Sections the refactoring does not touch you leave out instead of leaving them empty — an empty section is dead knowledge before the plan has even been touched. Do **not** create history sections; they come into being when `/plan-review`, `/plan-coding` or `/plan-lint` run for the first time.

Replace placeholders in angle brackets; do not carry over the comments in parentheses.

---

````markdown
---
Refactoring: <kurzer Titel>
Bereich: <Pfad, Modul oder Fokus — oder „gesamte Codebase">
Erstellt: <YYYY-MM-DD>
Claude-Session: <Session-ID>
Status: Entwurf
---

# <Refactoring-Titel>

<Zwei, drei Sätze: welcher Bereich, welche Probleme, welche Zielstruktur. Das Verhalten bleibt unverändert — das steht hier einmal und gilt für jeden Schritt.>

## Arbeitsschritte

| Nr | Schritt | Ergebnis | Abhängig von | Welle |
|----|---------|----------|--------------|-------|
| 1 | Charakterisierungstests für <Modul A> | Verhalten von <Modul A> ist durch Tests festgehalten | — | W1 |
| 2 | Charakterisierungstests für <Modul B> | Verhalten von <Modul B> ist durch Tests festgehalten | — | W1 |
| 3 | <Naht einziehen in Modul A> | <was danach gilt> | 1 | W2 |
| 4 | <Modul B umziehen> | <…> | 2, 3 | W3 |

## Parallel abarbeitbar

| Welle | Schritte | Voraussetzung | Berührungspunkte |
|-------|----------|---------------|------------------|
| W1 | 1, 2 | — | keine gemeinsamen Dateien |
| W2 | 3 | W1 abgeschlossen | — |
| W3 | 4 | W2 abgeschlossen | Schritt 4 fasst `<datei>` an, die auch <…> berührt |

<Ein Satz zur Einordnung: was sich dadurch tatsächlich verkürzt — oder dass keine echte Parallelität besteht.>

## Ziel

<Was gilt als erreicht? Prüfbar formuliert: welche Problemfelder gelöst sind und welche Maßzahlen sich wie ändern — Importe je Modul, Zyklen, Dateien je Verzeichnis.>

## Nicht-Ziele

- <was ausdrücklich nicht dazugehört — darunter jedes Problemfeld, das dieses Refactoring nicht angeht, als `P<n> <Titel> — <Grund>`>

## Architektur-Zusammenfassung

### Schichten und Module
<Welche Schichten oder Module es gibt, wer wen kennt, in welche Richtung die Abhängigkeiten zeigen — mit Datei:Zeile für die Grenzen.>

### Einstiegspunkte
<main, Routen, CLI, Handler, Jobs — mit Datei:Zeile.>

### Zentrale Datenflüsse
<Zwei bis drei Flüsse vom Einstieg bis zur Persistenz, je als Kette von Datei:Zeile.>

### Build, Tests und Baseline
<Die dokumentierten Befehle und das Ergebnis des Laufs zum Zeitpunkt der Analyse: grün, rot mit den roten Tests, oder „unbekannt", wenn kein Befehl dokumentiert ist.>

### Nicht gelesen
<Was ungelesen blieb und warum.>

## Problemfelder

<Jeder Befund hat einen Beleg; was sich nicht belegen lässt, ist eine offene Frage, kein Befund. Merkmale, Belegregeln und Schwere je Kategorie stehen in `problem-categories.md`, Teil A.>

### Strukturprobleme

#### P1 — <Titel>  [Hoch | Mittel | Niedrig]
- **Fundstelle:** <Datei:Zeile; bei Kopplung beide Seiten>
- **Befund:** <was nicht stimmt>
- **Beleg:** <Grep-Muster oder Kennzahl>
- **Strategie:** <Strategie-Nr, oder „Nicht-Ziel">

### Duplikate

#### P2 — <Titel>  [Hoch | Mittel | Niedrig]
(gleiche Struktur; divergent kopierte Duplikate tragen den Vermerk „divergent")

### Performance-Engpässe

#### P3 — <Titel>  [Hoch | Mittel | Niedrig] — Verdacht | belegt
(gleiche Struktur)

### Wartbarkeitsrisiken

#### P4 — <Titel>  [Hoch | Mittel | Niedrig]
(gleiche Struktur)

### Vermutete Fehler — kein Refactoring-Gegenstand

- <Datei:Zeile — was vermutlich falsch ist>

## Zielarchitektur

### Schichten und Abhängigkeitsrichtung
<Welche Schichten es danach gibt und wer wen kennen darf; Abhängigkeiten zeigen nach innen zur Domäne. Benannte und begründete Ausnahmen, etwa Framework-Basisklassen oder Active Record.>

### Zuständigkeit je Modul
<Je Modul ein Satz: was es verantwortet und was seine öffentliche Oberfläche ist.>

### Ordnerstruktur

```
vorher                      nachher
src/                        src/
├── <…>                     ├── <…>
```

### Umzugstabelle

| Datei alt | Datei neu | Schritt |
|-----------|-----------|---------|
| `<alt>` | `<neu>` | <Nr> |

## Refactoring-Strategien

### Strategie 1 — <Titel>
- **Löst:** P<n>, P<m>
- **Zielzustand:** <wie der Bereich danach aussieht>
- **Technik:** <aus `problem-categories.md`, Teil B>
- **Reihenfolge:** <die Schritte dieser Strategie in ihrer Folge, etwa „Schritt 1 → 3">
- **Verhaltensschutz:** <welche Tests das Verhalten dieser Strategie festhalten>
- **Verworfene Alternative:** <was auch ginge und warum nicht>

### Strategie 2 — <Titel>
(gleiche Struktur)

## Schritte

### Schritt 1 — <Titel>
- **Ergebnis:** <was danach nachweisbar gilt>
- **Betroffen:** <Dateien, Module>
- **Abhängig von:** <Schrittnummern oder —>
- **Vorgehen:** <zwei bis fünf Sätze, konkret genug zum Umsetzen, ohne fertigen Code>
- **Verhaltensschutz:** <Testbefehl oder Charakterisierungstests, die nach dem Schritt im Stand der Baseline laufen>
- **Akzeptanz:** <strukturelle Prüfung: Grep auf alte Pfade leer, Importzahl je Modul, Zyklen>

### Schritt 2 — <Titel>
(gleiche Struktur)

## Teststrategie

<Welche Ebene deckt was ab; was sich nur manuell prüfen lässt und wie. Baseline und Charakterisierungstests stehen in der Architektur-Zusammenfassung und in den Schritten.>

## Rollout

<Zusammenspiel mit laufenden Branches, woran ein Verhaltensbruch sichtbar würde, Rückweg.>

## Risiken

- <Risiko> → <Gegenmaßnahme oder bewusste Inkaufnahme>

## Annahmen

- <Annahme, die mangels Antwort oder Lesen getroffen wurde> → <was sie umwirft>

## Offene Fragen

1. <Frage, die nur der Mensch beantworten kann> → betrifft Schritt <Nr>
````

---

## Notes on the template

- **The step overview and the parallel overview come before everything else.** They are the reason you open the plan; goal and target architecture are what you read the second time.
- **The three dependency statements must agree** — the column "Abhängig von" in the overview, the wave in both tables, `Abhängig von` in the detailed step. Contradictions here are the most common lint finding.
- **The `Welle` column and the header block `Status` record progress later.** `/plan-coding` appends `✓ erledigt` to the wave code in the step overview (`W1 ✓ erledigt`), sets a first bullet `**Status:** erledigt <Datum>` in the detailed step and counts in the header block from `Entwurf` via `In Umsetzung — n von N Schritten erledigt` to `Umgesetzt`. That is why the table gets no status column of its own, and `Status: Entwurf` is the starting value, not a placeholder.
- **Every `P` has a strategy or is listed under non-goals with a reason**, and a strategy without a `P` is anticipation — it solves no evidenced problem.
- **`Verhaltensschutz` is a mandatory bullet, not decoration.** A step without a test command or characterization tests is not thought through; if the path lacks a test, a characterization-test step in an earlier wave comes before it.
- **Every row of the relocation table names the step that carries it out**, and a relocation changes only paths, namespaces and imports, no logic. A row without a step is a gap in the step sequence; a step that moves something to a place the folder tree does not know is a contradiction.
- **Suspected bugs are listed, not fixed.** A fix changes behavior and belongs in a plan of its own.
- **Assumptions and open questions stay in the document**, even once they are answered — then with the answer next to them. Cleaning up is done by `/plan-lint`, not the author.
- **A different output language translates the labels, not the structure.** Headings, column titles, header block labels and prose appear in the chosen language; order, tables, step numbers, the wave codes `W1, W2 …`, the `P` identifiers and the key `Claude-Session` stay unchanged so that the downstream skills find them again.
- **No code blocks with finished implementation** in the steps. Signatures, folder trees and example payloads yes, whole functions no: they go stale between planning and implementation and are then read as truth.
