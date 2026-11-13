# Template for the optimization plan document

The header block keys `Claude-Session` and `Status`, the table headers of `## Arbeitsschritte` and `## Parallel abarbeitbar`, `## Schritte` with `### Schritt N — <Titel>` and the note on progress marking are taken verbatim from `skills/plan-create/references/plan-template.md`; the `## Problemfelder` section, the `P` identifiers and the `Verhaltensschutz` point from `skills/plan-refactoring-create/references/refactoring-template.md`. Both are updated here whenever the marking logic of `/plan-coding` or the review dimension "Behavior preservation" in plan-review changes — only then do `/plan-review`, `/plan-lint` and `/plan-coding` find the same anchors in an optimization plan as in a feature or refactoring plan. Deliberately different: the classification of a finding here is „gemessen | Verdacht" (measured | suspected) instead of „Verdacht | belegt", because the evidence is a measurement.

Structure and order are binding. Sections the optimization does not touch are left out rather than left empty — an empty section is dead knowledge before the plan has even been touched. Do **not** create history sections; they come into being when `/plan-review`, `/plan-coding` or `/plan-lint` run for the first time.

Replace placeholders in angle brackets; do not carry over the comments in parentheses.

---

````markdown
---
Optimierung: <kurzer Titel>
Bereich: <Pfad, Modul oder Fokus — oder „gesamte Codebase">
Erstellt: <YYYY-MM-DD>
Claude-Session: <Session-ID>
Status: Entwurf
---

# <Optimierungs-Titel>

<Zwei, drei Sätze: welcher Bereich, welche Kennzahlen, welche Engpässe. Das Verhalten bleibt unverändert — das steht hier einmal und gilt für jeden Schritt.>

## Arbeitsschritte

| Nr | Schritt | Ergebnis | Abhängig von | Welle |
|----|---------|----------|--------------|-------|
| 1 | Messung einrichten für <Kennzahl> (nur ohne Baseline) | Baseline für <Kennzahl> steht | — | W1 |
| 2 | Charakterisierungstests für <Modul> | Verhalten von <Modul> ist durch Tests festgehalten | — | W1 |
| 3 | <Suche in Modul A auf Index umstellen> | <Kennzahl> erreicht <Ziel> | 1, 2 | W2 |
| 4 | <Titel> | <…> | 3 | W3 |

## Parallel abarbeitbar

| Welle | Schritte | Voraussetzung | Berührungspunkte |
|-------|----------|---------------|------------------|
| W1 | 1, 2 | — | keine gemeinsamen Dateien |
| W2 | 3 | W1 abgeschlossen | — |
| W3 | 4 | W2 abgeschlossen | Schritt 4 fasst `<datei>` an, die auch <…> berührt |

<Ein Satz zur Einordnung: was sich dadurch tatsächlich verkürzt — oder dass keine echte Parallelität besteht.>

## Ziel

<Was gilt als erreicht? Prüfbar formuliert: welche Zeilen der Zielwerte erreicht und welche Problemfelder damit gelöst sind.>

## Nicht-Ziele

- <was ausdrücklich nicht dazugehört — darunter jedes Problemfeld, das diese Optimierung nicht angeht, als `P<n> <Titel> — <Grund>`>

## Architektur-Zusammenfassung

### Schichten und Module
<Welche Schichten oder Module es gibt, wer wen kennt — mit Datei:Zeile für die Grenzen, soweit die Hot Paths sie berühren.>

### Einstiegspunkte
<main, Routen, CLI, Handler, Jobs — mit Datei:Zeile.>

### Hot Paths
<Zwei bis drei Pfade vom Einstieg bis zum Ergebnis, je als Kette von Datei:Zeile, mit der Datenmenge je Station.>

### Build und Tests
<Die dokumentierten Befehle und das Ergebnis des Laufs zum Zeitpunkt der Analyse: grün, rot mit den roten Tests, oder „unbekannt", wenn kein Befehl dokumentiert ist.>

### Baseline-Messung
<Der dokumentierte Messbefehl, die Zahl der Läufe, die Umgebung (lokaler Rechner, nicht CI), bei nur einem Lauf der Vermerk „Einzelmessung, Streuung unbekannt"; die Werte je Kennzahl stehen in den Zielwerten. Ohne Befehl: „nicht gemessen".>

### Nicht gelesen
<Was ungelesen blieb und warum.>

## Problemfelder

<Jeder Befund hat einen Beleg und die Einstufung gemessen oder Verdacht; was sich nicht belegen lässt, ist eine offene Frage, kein Befund. Merkmale, Belegregeln und Schwere je Kategorie stehen in `optimization-categories.md`, Teil A.>

### Engpässe

#### P1 — <Titel>  [Hoch | Mittel | Niedrig] — gemessen | Verdacht
- **Fundstelle:** <Datei:Zeile>
- **Befund:** <was nicht stimmt>
- **Beleg:** <gemessen: Zeile der Zielwerte oder Profilanteil mit Befehl; Verdacht: Muster und Datenmenge, ab der es trägt>
- **Strategie:** <Strategie-Nr, oder „Nicht-Ziel">

### Ineffiziente Logik

#### P2 — <Titel>  [Hoch | Mittel | Niedrig] — gemessen | Verdacht
(gleiche Struktur)

### Unnötige Arbeit

#### P3 — <Titel>  [Hoch | Mittel | Niedrig] — gemessen | Verdacht
(gleiche Struktur)

### Speicher

#### P4 — <Titel>  [Hoch | Mittel | Niedrig] — gemessen | Verdacht
(gleiche Struktur)

### Skalierung

#### P5 — <Titel>  [Hoch | Mittel | Niedrig] — gemessen | Verdacht
(gleiche Struktur)

### Vermutete Fehler — kein Optimierungs-Gegenstand

- <Datei:Zeile — was vermutlich falsch ist>

## Zielwerte

| Kennzahl | Baseline | Ziel | Messbefehl |
|----------|----------|------|------------|
| <Laufzeit Suche, p50> | <Wert ± Streuung> | <Wert> | `<Befehl>` |

<Ein Satz zur Herkunft der Ziele: vom Menschen bestätigt, oder Verweis auf die Annahme oder offene Frage.>

## Optimierungsstrategien

### Strategie 1 — <Titel>
- **Löst:** P<n>, P<m>
- **Kennzahl:** <welche Zeile der Zielwerte sich bewegen muss>
- **Technik:** <aus `optimization-categories.md`, Teil B>
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
- **Messung:** <Zeile der Zielwerte, die sich bewegen muss; Messbefehl und Vorher-Wert nur, wenn sie von der Zeile abweichen — oder „keine — <Grund>">
- **Akzeptanz:** <strukturelle Prüfung: Grep auf das alte Muster leer, gezählte Abfragen oder Allokationen>

### Schritt 2 — <Titel>
(gleiche Struktur)

## Teststrategie

<Welche Ebene deckt was ab; was sich nur manuell prüfen lässt und wie. Baseline, Charakterisierungstests und Messungen stehen in der Architektur-Zusammenfassung, den Zielwerten und den Schritten.>

## Rollout

<Zusammenspiel mit laufenden Branches, woran ein Verhaltensbruch oder eine Verschlechterung sichtbar würde, Rückweg.>

## Risiken

- <Risiko> → <Gegenmaßnahme oder bewusste Inkaufnahme>

## Annahmen

- <Annahme, die mangels Antwort, Lesen oder Messung getroffen wurde> → <was sie umwirft>

## Offene Fragen

1. <Frage, die nur der Mensch beantworten kann> → betrifft Schritt <Nr>
````

---

## Notes on the template

- **Step overview and parallel overview come before everything else.** They are the reason one opens the plan; goal and target values are read the second time.
- **The three dependency entries must agree** — the "Abhängig von" column in the overview, the wave in both tables, `Abhängig von` in the detailed step. Contradictions here are the most common lint finding.
- **The `Welle` column and the header block `Status` later record progress.** `/plan-coding` appends `✓ erledigt` to the wave code in the step overview (`W1 ✓ erledigt`), sets a first point `**Status:** erledigt <Datum>` in the detailed step, and counts in the header block from `Entwurf` via `In Umsetzung — n von N Schritten erledigt` to `Umgesetzt`. That is why the table gets no status column of its own, and `Status: Entwurf` is the starting value, not a placeholder.
- **Measure before optimizing.** Without a baseline, step 1 sets up the measurement, and every finding is `Verdacht`. Every step carries the `Messung` point — on a hot path as a reference to the row of the target values, for measurement and test steps as „keine — <Grund>". Before-values are measured, not estimated.
- **Numbers appear once.** Baseline and final target per metric are in the `## Zielwerte` table; a step refers to the row and names measurement command or before-value only if they differ — the before-value together with the step that produced it.
- **A cache only goes into the plan with a named invalidation path** — otherwise it is a contradiction for `/plan-lint` (caching versus data flow), not a detail.
- **Every `P` has a strategy or is listed with a reason under `## Nicht-Ziele`**, and a strategy without a `P` is anticipation — it solves no evidenced problem.
- **`Verhaltensschutz` is a mandatory point, not decoration.** A step without a test command or characterization tests is not fully thought through; if the path lacks a test, a characterization-test step goes before it in an earlier wave.
- **Suspected bugs are listed, not fixed.** A correction changes behavior and belongs in a plan of its own.
- **Assumptions and open questions stay in the document**, even once answered — then with the answer next to them. Cleaning up is done by `/plan-lint`, not by the author.
- **A different output language translates the labels, not the structure.** Headings, column titles, header block labels and prose appear in the chosen language; order, tables, step numbers, the wave codes `W1, W2 …`, the `P` identifiers and the key `Claude-Session` stay unchanged so that the downstream skills can find them.
- **No code blocks with finished implementation** in the steps. Signatures, measurement commands and example payloads yes, whole functions no: they go stale between planning and implementation and are then read as truth.
