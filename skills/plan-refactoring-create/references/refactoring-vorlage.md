# Vorlage für das Refactoring-Plandokument

Die Kopfblock-Schlüssel `Claude-Session` und `Status`, die Tabellenköpfe von `## Arbeitsschritte` und `## Parallel abarbeitbar`, `## Schritte` mit `### Schritt N — <Titel>` und der Hinweis zur Fortschrittsmarkierung stammen wörtlich aus `skills/plan-create/references/plan-vorlage.md` und werden hier nachgezogen, wenn sich dort die Markierungslogik von `/plan-coding` ändert — nur so finden `/plan-review`, `/plan-lint` und `/plan-coding` in einem Refactoring-Plan dieselben Anker wie in einem Feature-Plan.

Struktur und Reihenfolge sind verbindlich. Abschnitte, die das Refactoring nicht berührt, lässt du weg statt sie leer zu lassen — ein leerer Abschnitt ist totes Wissen, bevor der Plan überhaupt angefasst wurde. Historie-Abschnitte legst du **nicht** an; die entstehen, wenn `/plan-review`, `/plan-coding` oder `/plan-lint` das erste Mal laufen.

Platzhalter in spitzen Klammern ersetzen, die Kommentare in Klammern nicht übernehmen.

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

<Jeder Befund hat einen Beleg; was sich nicht belegen lässt, ist eine offene Frage, kein Befund. Merkmale, Belegregeln und Schwere je Kategorie stehen in `problemkategorien.md`, Teil A.>

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
- **Technik:** <aus `problemkategorien.md`, Teil B>
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

## Hinweise zur Vorlage

- **Schrittübersicht und Parallel-Übersicht stehen vor allem anderen.** Sie sind der Grund, warum man den Plan öffnet; Ziel und Zielarchitektur liest man beim zweiten Mal.
- **Die drei Abhängigkeitsangaben müssen übereinstimmen** — Spalte "Abhängig von" in der Übersicht, Welle in beiden Tabellen, `Abhängig von` im Detailschritt. Widersprüche hier sind der häufigste Lint-Fund.
- **Die Spalte `Welle` und der Kopfblock-`Status` nehmen später den Fortschritt auf.** `/plan-coding` hängt in der Schrittübersicht `✓ erledigt` an das Wellen-Kürzel (`W1 ✓ erledigt`), setzt im Detailschritt einen ersten Punkt `**Status:** erledigt <Datum>` und zählt im Kopfblock von `Entwurf` über `In Umsetzung — n von N Schritten erledigt` bis `Umgesetzt`. Deshalb bekommt die Tabelle keine eigene Statusspalte, und `Status: Entwurf` ist der Startwert, kein Platzhalter.
- **Jedes `P` hat eine Strategie oder steht mit Grund unter Nicht-Ziele**, und eine Strategie ohne `P` ist Vorwegnahme — sie löst kein belegtes Problem.
- **`Verhaltensschutz` ist Pflichtpunkt, kein Schmuck.** Ein Schritt ohne Testbefehl oder Charakterisierungstests ist nicht fertig gedacht; fehlt dem Pfad ein Test, kommt ein Charakterisierungstest-Schritt in einer früheren Welle davor.
- **Jede Zeile der Umzugstabelle nennt den Schritt, der sie ausführt**, und ein Umzug ändert nur Pfade, Namespaces und Importe, keine Logik. Eine Zeile ohne Schritt ist eine Lücke in der Schrittfolge; ein Schritt, der an einen Ort verschiebt, den der Ordnerbaum nicht kennt, ist ein Widerspruch.
- **Vermutete Fehler werden gelistet, nicht behoben.** Eine Korrektur ändert Verhalten und gehört in einen eigenen Plan.
- **Annahmen und offene Fragen bleiben im Dokument**, auch wenn sie beantwortet wurden — dann mit der Antwort daneben. Das Aufräumen erledigt `/plan-lint`, nicht der Autor.
- **Eine andere Ausgabesprache übersetzt die Beschriftungen, nicht den Aufbau.** Überschriften, Spaltentitel, Kopfblock-Beschriftungen und Prosa erscheinen in der gewählten Sprache; Reihenfolge, Tabellen, Schrittnummern, die Wellen-Kürzel `W1, W2 …`, die `P`-Kennungen und der Schlüssel `Claude-Session` bleiben unverändert, damit die Folge-Skills sie wiederfinden.
- **Keine Code-Blöcke mit fertiger Implementierung** in den Schritten. Signaturen, Ordnerbäume und Beispiel-Payloads ja, ganze Funktionen nein: sie veralten zwischen Planung und Umsetzung und werden dann als Wahrheit gelesen.
