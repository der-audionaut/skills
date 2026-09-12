# Vorlage für das Plandokument

Struktur und Reihenfolge sind verbindlich. Abschnitte, die das Feature nicht berührt, lässt du weg statt sie leer zu lassen — ein leerer Abschnitt ist totes Wissen, bevor der Plan überhaupt angefasst wurde. Historie-Abschnitte legst du **nicht** an; die entstehen, wenn `/plan-review`, `/plan-implement` oder `/plan-lint` das erste Mal laufen.

Platzhalter in spitzen Klammern ersetzen, die Kommentare in Klammern nicht übernehmen.

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

## Hinweise zur Vorlage

- **Schrittübersicht und Parallel-Übersicht stehen vor allem anderen.** Sie sind der Grund, warum man den Plan öffnet; Ziel und Entwurf liest man beim zweiten Mal.
- **Die drei Abhängigkeitsangaben müssen übereinstimmen** — Spalte "Abhängig von" in der Übersicht, Welle in beiden Tabellen, `Abhängig von` im Detailschritt. Widersprüche hier sind der häufigste Lint-Fund.
- **Annahmen und offene Fragen bleiben im Dokument**, auch wenn sie beantwortet wurden — dann mit der Antwort daneben. Das Aufräumen erledigt `/plan-lint`, nicht der Autor.
- **Eine andere Ausgabesprache übersetzt die Beschriftungen, nicht den Aufbau.** Überschriften, Spaltentitel, Kopfblock-Beschriftungen und Prosa erscheinen in der gewählten Sprache; Reihenfolge, Tabellen, Schrittnummern, die Wellen-Kürzel `W1, W2 …` und der Schlüssel `Claude-Session` bleiben unverändert, damit die Folge-Skills sie wiederfinden.
- **Keine Code-Blöcke mit fertiger Implementierung** in den Schritten. Signaturen, Schemata und Beispiel-Payloads ja, ganze Funktionen nein: sie veralten zwischen Planung und Umsetzung und werden dann als Wahrheit gelesen.
