# Template for the plan document

Structure and order are binding. Sections the feature does not touch you leave out rather than leaving them empty — an empty section is dead knowledge before the plan has even been touched. You do **not** create history sections; they arise when `/plan-review`, `/plan-implement` or `/plan-lint` run for the first time.

Replace placeholders in angle brackets; do not carry over the comments in parentheses.

---

````markdown
---
Feature: <kurzer Titel>
Erstellt: <YYYY-MM-DD>
Claude-Session: <Session-ID>
Status: Entwurf
---

# <Feature-Titel>

<Zwei, drei Sätze: was gebaut wird und warum. Kein Marketing, keine Wiederholung des Titels.>

## Arbeitsschritte

| Nr | Schritt | Ergebnis | Abhängig von | Welle |
|----|---------|----------|--------------|-------|
| 1 | <Titel> | <was danach funktioniert> | — | W1 |
| 2 | <Titel> | <…> | — | W1 |
| 3 | <Titel> | <…> | 1 | W2 |
| 4 | <Titel> | <…> | 2, 3 | W3 |

## Parallel abarbeitbar

| Welle | Schritte | Voraussetzung | Berührungspunkte |
|-------|----------|---------------|------------------|
| W1 | 1, 2 | — | keine gemeinsamen Dateien |
| W2 | 3 | W1 abgeschlossen | — |
| W3 | 4 | W2 abgeschlossen | Schritt 4 fasst `<datei>` an, die auch <…> berührt |

<Ein Satz zur Einordnung: was sich dadurch tatsächlich verkürzt — oder dass keine echte Parallelität besteht.>

## Ziel

<Was gilt als erreicht? Prüfbar formuliert, nicht als Absichtserklärung.>

## Nicht-Ziele

- <was ausdrücklich nicht dazugehört, inklusive der Dinge, die der Entwurf vorsieht, dieses Feature aber noch nicht braucht>

## Ist-Zustand

<Wie löst das System das heute, mit Verweisen auf Datei:Zeile. Was fehlt, was steht im Weg.>

## Entwurf

### Architektur und Einordnung
<Schicht, synchron oder asynchron, welche Grenze gezogen wird.>

### Komponenten
<Wer bekommt welche Verantwortung, was bleibt Implementierungsdetail.>

### Datenfluss
<Woher die Daten kommen, wer sie transformiert, wo sie landen, wo die Quelle der Wahrheit liegt.>

### Schnittstellen
<Endpunkte oder Signaturen, Fehlerformat, Kompatibilität für bestehende Aufrufer.>

### Datenmodell und Migration
<Felder, Typen, Nullbarkeit, Indizes. Migrationsweg, Rückwärtskompatibilität, Backfill.>

### Caching
<Nur bei belegtem Bedarf: Ebene, Schlüssel, Gültigkeit, Invalidierungspfad. Sonst ein Satz, warum keins.>

## Schritte

### Schritt 1 — <Titel>
- **Ergebnis:** <was danach nachweisbar funktioniert>
- **Betroffen:** <Dateien, Module, Migrationen>
- **Abhängig von:** <Schrittnummern oder —>
- **Vorgehen:** <zwei bis fünf Sätze, konkret genug zum Umsetzen, ohne fertigen Code>
- **Akzeptanz:** <woran geprüft wird: Testfall, Befehl, beobachtbares Verhalten>

### Schritt 2 — <Titel>
(gleiche Struktur)

## Teststrategie

<Welche Ebene deckt was ab. Welche Fehlerfälle getestet werden. Was sich nur manuell prüfen lässt und wie.>

## Rollout

<Feature-Flag oder direkt, Reihenfolge der Ausbringung, Rückweg, woran ein Fehlschlag sichtbar wird.>

## Risiken

- <Risiko> → <Gegenmaßnahme oder bewusste Inkaufnahme>

## Annahmen

- <Annahme, die mangels Antwort getroffen wurde> → <was sie umwirft>

## Offene Fragen

1. <Frage, die nur der Mensch beantworten kann> → betrifft Schritt <Nr>
````

---

## Notes on the template

- **The step overview and the parallel overview come before everything else.** They are the reason anyone opens the plan; goal and design get read the second time around.
- **The three dependency statements must agree** — the column "Abhängig von" in the overview, the wave in both tables, `Abhängig von` in the detailed step. Contradictions here are the most common lint finding.
- **The `Welle` column and the header block's `Status` record progress later.** `/plan-coding` appends `✓ erledigt` to the wave code in the step overview (`W1 ✓ erledigt`), puts a first bullet `**Status:** erledigt <Datum>` into the detailed step, and counts in the header block from `Entwurf` via `In Umsetzung — n von N Schritten erledigt` to `Umgesetzt`. That is why the table gets no status column of its own, and `Status: Entwurf` is the starting value, not a placeholder.
- **Assumptions and open questions stay in the document**, even once they are answered — then with the answer next to them. Cleaning up is the job of `/plan-lint`, not of the author.
- **A different output language translates the labels, not the structure.** Headings, column titles, header block labels and prose appear in the chosen language; order, tables, step numbers, the wave codes `W1, W2 …` and the key `Claude-Session` stay unchanged so the follow-up skills can find them.
- **No code blocks with finished implementation** in the steps. Signatures, schemas and example payloads yes, whole functions no: they go stale between planning and implementation and are then read as truth.
