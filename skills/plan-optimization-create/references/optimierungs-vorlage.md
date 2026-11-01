# Vorlage für das Optimierungs-Plandokument

Die Kopfblock-Schlüssel `Claude-Session` und `Status`, die Tabellenköpfe von `## Arbeitsschritte` und `## Parallel abarbeitbar`, `## Schritte` mit `### Schritt N — <Titel>` und der Hinweis zur Fortschrittsmarkierung stammen wörtlich aus `skills/plan-create/references/plan-vorlage.md`, die Abschnitte `## Problemfelder`, die `P`-Kennungen und der Punkt `Verhaltensschutz` aus `skills/plan-refactoring-create/references/refactoring-vorlage.md`; beides wird hier nachgezogen, wenn sich dort die Markierungslogik von `/plan-coding` oder die Review-Dimension „Verhaltenserhalt" ändert — nur so finden `/plan-review`, `/plan-lint` und `/plan-coding` in einem Optimierungsplan dieselben Anker wie in einem Feature- oder Refactoring-Plan. Bewusst anders: Die Einstufung eines Befunds heißt hier „gemessen | Verdacht" statt „Verdacht | belegt", weil der Beleg eine Messung ist.

Struktur und Reihenfolge sind verbindlich. Abschnitte, die die Optimierung nicht berührt, lässt du weg statt sie leer zu lassen — ein leerer Abschnitt ist totes Wissen, bevor der Plan überhaupt angefasst wurde. Historie-Abschnitte legst du **nicht** an; die entstehen, wenn `/plan-review`, `/plan-coding` oder `/plan-lint` das erste Mal laufen.

Platzhalter in spitzen Klammern ersetzen, die Kommentare in Klammern nicht übernehmen.

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

<Jeder Befund hat einen Beleg und die Einstufung gemessen oder Verdacht; was sich nicht belegen lässt, ist eine offene Frage, kein Befund. Merkmale, Belegregeln und Schwere je Kategorie stehen in `optimierungskategorien.md`, Teil A.>

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
- **Technik:** <aus `optimierungskategorien.md`, Teil B>
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

## Hinweise zur Vorlage

- **Schrittübersicht und Parallel-Übersicht stehen vor allem anderen.** Sie sind der Grund, warum man den Plan öffnet; Ziel und Zielwerte liest man beim zweiten Mal.
- **Die drei Abhängigkeitsangaben müssen übereinstimmen** — Spalte "Abhängig von" in der Übersicht, Welle in beiden Tabellen, `Abhängig von` im Detailschritt. Widersprüche hier sind der häufigste Lint-Fund.
- **Die Spalte `Welle` und der Kopfblock-`Status` nehmen später den Fortschritt auf.** `/plan-coding` hängt in der Schrittübersicht `✓ erledigt` an das Wellen-Kürzel (`W1 ✓ erledigt`), setzt im Detailschritt einen ersten Punkt `**Status:** erledigt <Datum>` und zählt im Kopfblock von `Entwurf` über `In Umsetzung — n von N Schritten erledigt` bis `Umgesetzt`. Deshalb bekommt die Tabelle keine eigene Statusspalte, und `Status: Entwurf` ist der Startwert, kein Platzhalter.
- **Messen vor Optimieren.** Ohne Baseline richtet Schritt 1 die Messung ein, und jeder Befund ist Verdacht. Jeder Schritt trägt den Punkt `Messung` — auf einem Hot Path als Verweis auf die Zeile der Zielwerte, bei Mess- und Testschritten als „keine — <Grund>". Vorher-Werte sind gemessen, nicht geschätzt.
- **Zahlen stehen einmal.** Baseline und Endziel je Kennzahl stehen in der Zielwerte-Tabelle; ein Schritt verweist auf die Zeile und nennt Messbefehl oder Vorher-Wert nur, wenn sie abweichen — den Vorher-Wert mit dem Schritt, der ihn erzeugt hat.
- **Ein Cache steht nur mit benanntem Invalidierungspfad im Plan** — sonst ist er für `/plan-lint` ein Widerspruch (Caching gegen Datenfluss), kein Detail.
- **Jedes `P` hat eine Strategie oder steht mit Grund unter Nicht-Ziele**, und eine Strategie ohne `P` ist Vorwegnahme — sie löst kein belegtes Problem.
- **`Verhaltensschutz` ist Pflichtpunkt, kein Schmuck.** Ein Schritt ohne Testbefehl oder Charakterisierungstests ist nicht fertig gedacht; fehlt dem Pfad ein Test, kommt ein Charakterisierungstest-Schritt in einer früheren Welle davor.
- **Vermutete Fehler werden gelistet, nicht behoben.** Eine Korrektur ändert Verhalten und gehört in einen eigenen Plan.
- **Annahmen und offene Fragen bleiben im Dokument**, auch wenn sie beantwortet wurden — dann mit der Antwort daneben. Das Aufräumen erledigt `/plan-lint`, nicht der Autor.
- **Eine andere Ausgabesprache übersetzt die Beschriftungen, nicht den Aufbau.** Überschriften, Spaltentitel, Kopfblock-Beschriftungen und Prosa erscheinen in der gewählten Sprache; Reihenfolge, Tabellen, Schrittnummern, die Wellen-Kürzel `W1, W2 …`, die `P`-Kennungen und der Schlüssel `Claude-Session` bleiben unverändert, damit die Folge-Skills sie wiederfinden.
- **Keine Code-Blöcke mit fertiger Implementierung** in den Schritten. Signaturen, Messbefehle und Beispiel-Payloads ja, ganze Funktionen nein: sie veralten zwischen Planung und Umsetzung und werden dann als Wahrheit gelesen.
