# Template for the frontend plan document

The header block keys `Claude-Session` and `Status`, the table headers of `## Arbeitsschritte` and `## Parallel abarbeitbar`, `## Schritte` with `### Schritt N — <Titel>` and the note on progress marking are taken verbatim from `skills/plan-create/references/plan-template.md` and are brought in line here whenever the marking logic of `/plan-coding` changes there — only then do `/plan-review`, `/plan-lint` and `/plan-coding` find the same anchors in a frontend plan as in a feature plan. The frontend plan's own anchors — `## Komponentenarchitektur`, `## Komponentenkatalog`, `#### K<n> — ` and the step bullet `**Zustände:**` — originate here; the section "Frontend plans" in `skills/plan-review/references/review-criteria.md` and the entry "Components (frontend plans)" in `skills/plan-lint/references/lint-categories.md` quote them and are brought in line whenever they change here.

Structure and order are binding. Sections the frontend does not touch you leave out instead of leaving them empty — an empty section is dead knowledge before the plan has even been touched. Do **not** create history sections; they come into being when `/plan-review`, `/plan-coding` or `/plan-lint` run for the first time.

Replace placeholders in angle brackets; do not carry over the comments in parentheses.

---

````markdown
---
Frontend: <kurzer Titel>
Erstellt: <YYYY-MM-DD>
Claude-Session: <Session-ID>
Status: Entwurf
---

# <Titel der Oberfläche>

<Zwei, drei Sätze: welche Oberfläche entsteht, für wen und warum. Kein Marketing, keine Wiederholung des Titels.>

## Arbeitsschritte

| Nr | Schritt | Ergebnis | Abhängig von | Welle |
|----|---------|----------|--------------|-------|
| 1 | <UI-Baustein K<n>> | <was danach funktioniert> | — | W1 |
| 2 | <Datenanbindung> | <…> | — | W1 |
| 3 | <Feature-Komponente K<n>> | <…> | 1, 2 | W2 |
| 4 | <Seite oder Route K<n>> | <…> | 3 | W3 |

## Parallel abarbeitbar

| Welle | Schritte | Voraussetzung | Berührungspunkte |
|-------|----------|---------------|------------------|
| W1 | 1, 2 | — | keine gemeinsamen Dateien |
| W2 | 3 | W1 abgeschlossen | — |
| W3 | 4 | W2 abgeschlossen | Schritt 4 fasst `<Routen-Datei>` an, die auch <…> berührt |

<Ein Satz zur Einordnung: was sich dadurch tatsächlich verkürzt — oder dass keine echte Parallelität besteht.>

## Ziel

<Was gilt als erreicht? Prüfbar formuliert: welche Oberfläche mit welchen Zuständen auf welchen Breiten bedienbar ist und nach dem Maßstab aus `### Maßstab Barrierefreiheit` barrierefrei.>

## Nicht-Ziele

- <was ausdrücklich nicht dazugehört, inklusive der Dinge, die der Entwurf vorsieht, diese Oberfläche aber noch nicht braucht>

## Ist-Zustand

### Stack
<Framework, Sprache, Styling, Komponentenbibliothek, Zustandsverwaltung, Routing — mit Datei:Zeile aus Manifest und Konfiguration.>

### Vorhandene Bausteine
<Was diese Oberfläche wiederverwenden kann und nicht als `K` im Katalog steht — Tokens, Hooks, Layouts, Hilfsfunktionen —, mit Datei:Zeile. Was fehlt, was im Weg steht.>

### Build und Tests
<Die dokumentierten Befehle für Build, Tests, Typprüfung, Linter und A11y-Checks; was davon im Projekt fehlt.>

### Maßstab Barrierefreiheit
<WCAG 2.2 AA, oder die Vorgabe des Projekts mit Fundstelle.>

## Komponentenarchitektur

### Komponentenbaum

```
<Seite oder Route>                K4 neu
├── <Feature-Komponente>          K2 neu
│   ├── <UI-Baustein>             K1 wiederverwendet
│   └── <UI-Baustein>             K3 erweitert
└── <…>
```

### Zustandshaltung und Datenfluss
<Woher die Daten kommen, wo der Zustand lebt — Server-Zustand, URL, lokaler Zustand, globaler Store —, wer ihn ändert, wie Lade- und Fehlerzustand nach unten gereicht werden.>

### Responsive-Konzept
<Breakpoints des Projekts, was sich je Breite ändert — Spalten, Navigation, verborgene Inhalte.>

## Komponentenkatalog

<Je Komponente aus dem Baum ein Eintrag. Neue und erweiterte Komponenten tragen alle Punkte; wiederverwendete nur Verantwortung und Fundstelle. Regeln für Schnitt, Props, Zustände und Barrierefreiheit stehen in `frontend-criteria.md`.>

#### K1 — <Name>  [wiederverwendet] — <Ebene>
- **Verantwortung:** <ein Satz>
- **Fundstelle:** <Datei:Zeile>

#### K2 — <Name>  [neu | erweitert] — <Ebene>
- **Verantwortung:** <ein Satz: was sie tut und was ausdrücklich nicht>
- **Props:**

  | Prop | Typ | Pflicht | Default | Bedeutung |
  |------|-----|---------|---------|-----------|
  | `<name>` | `<Typ>` | ja | — | <…> |

- **Ereignisse und Slots:** <Callbacks mit Signatur, Slots oder Children>
- **Zustände:**
  - <Laden> → <Darstellung>
  - <Leer> → <Darstellung>
  - <Fehler> → <Darstellung und Weg zur Wiederholung>
- **Barrierefreiheit:** <Rolle oder semantisches Element, Tastaturbedienung, Fokusführung, Beschriftung — mit Kriterium des Maßstabs>
- **Responsive:** <Abweichungen vom Responsive-Konzept, oder „wie Konzept“>
- **Verwendung:**

  ```<Code-Sprache, z. B. tsx>
  <kurzer Aufruf mit Props, kein Komponenten-Body>
  ```

## Schnittstellen

<Nur bei Datenanbindung: Endpunkte oder Signaturen, die die Oberfläche nutzt, Fehlerformat, Ladeverhalten, Pagination.>

## Schritte

### Schritt 1 — <Titel>
- **Ergebnis:** <was danach nachweisbar funktioniert>
- **Betroffen:** <Dateien, Module, K<n>>
- **Abhängig von:** <Schrittnummern oder —>
- **Vorgehen:** <zwei bis fünf Sätze, konkret genug zum Umsetzen, ohne fertigen Code>
- **Zustände:** <K<n>: abgedeckte Zustände aus dem Katalog, oder „alle“; A11y-Prüfung mit Kriterium des Maßstabs — oder „keine — <Grund>“ für Schritte ohne UI>
- **Akzeptanz:** <woran geprüft wird: Testfall, Befehl, Tastaturdurchlauf, Viewport-Breite>

### Schritt 2 — <Titel>
(gleiche Struktur)

## Teststrategie

<Welche Ebene deckt was ab: Komponententests der Zustände, Integrations- oder End-to-End-Tests der Abläufe, automatischer A11y-Check (siehe Build und Tests). Was sich nur manuell prüfen lässt — Tastaturdurchlauf, Screenreader-Stichprobe, Viewport-Breiten — und wie.>

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

- **The step overview and the parallel overview come before everything else.** They are the reason you open the plan; goal and component architecture are what you read the second time.
- **The three dependency statements must agree** — the column "Abhängig von" in the overview, the wave in both tables, `Abhängig von` in the detailed step. Contradictions here are the most common lint finding.
- **The `Welle` column and the header block `Status` record progress later.** `/plan-coding` appends `✓ erledigt` to the wave code in the step overview (`W1 ✓ erledigt`), sets a first bullet `**Status:** erledigt <Datum>` in the detailed step and counts in the header block from `Entwurf` via `In Umsetzung — n von N Schritten erledigt` to `Umgesetzt`. That is why the table gets no status column of its own, and `Status: Entwurf` is the starting value, not a placeholder.
- **Every new or extended `K` has exactly one step that builds it**, and that step names it in its title or under `Betroffen`. A `K` without a step is a gap; a step that builds a component the catalogue does not know is a contradiction.
- **`Zustände` is a mandatory bullet in every step, not decoration.** A component step names the catalogue states it delivers and how its accessibility is checked; a step without that bullet is not thought through.
- **`Verwendung` shows a call, not an implementation.** A few lines with the props, the way a consumer writes it; the component body is produced by `/plan-coding`.
- **Assumptions and open questions stay in the document**, even once they are answered — then with the answer next to them. Cleaning up is done by `/plan-lint`, not the author.
- **A different output language translates the labels, not the structure.** Headings, column titles, header block labels and prose appear in the chosen language; order, tables, step numbers, the wave codes `W1, W2 …`, the `K` identifiers and the key `Claude-Session` stay unchanged so that the downstream skills find them again.
- **No code blocks with finished implementation** in the steps. Signatures, props tables, component trees and usage calls yes, whole components no: they go stale between planning and implementation and are then read as truth.
