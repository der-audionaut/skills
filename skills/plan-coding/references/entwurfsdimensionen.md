# Entwurfsdimensionen und Finding-Kategorien

Teil A steuert den Entwurf vor dem Code, Teil B die Findings-Runden danach. Beide Listen sind ein Sieb, keine Pflichtübung: was der Schritt nicht berührt, lässt du weg.

---

## Teil A — Entwurf

### Architektur und Einordnung
- In welche Schicht gehört die Änderung, und welche Schichten darf sie kennen?
- Läuft sie synchron im Request oder asynchron in einem Job? Was spricht dagegen?
- Welche Grenze wird hier gezogen — und wer darf sie überschreiten?

### Komponentenschnitt
- Welche Komponente bekommt die Verantwortung, und warum nicht die naheliegende andere?
- Was ist öffentlich, was bleibt Implementierungsdetail?
- Gibt es die Zuständigkeit schon irgendwo? Erweitern schlägt Neubauen.

### Datenfluss
- Woher kommen die Daten, wer transformiert sie, wo landen sie?
- Wo ist die Quelle der Wahrheit, und wo entstehen Kopien, die veralten können?
- Was passiert, wenn ein Teil des Flusses ausfällt oder zweimal läuft?

### Schnittstellen und API
- Signatur, Fehlerformat, Statuscodes: passen sie zum Bestand?
- Breaking Change für bestehende Aufrufer? Wenn ja: Übergang oder Version?
- Verträge stabil formuliert, nicht an interne Struktur gekoppelt?
- Paginierung, Filter, Sortierung — nur wenn dieser Schritt sie braucht, aber im Entwurf mitgedacht.

### Datenmodell und Migration
- Feldtypen, Nullbarkeit, Defaults, Constraints — passend statt bequem?
- Indizes für die Abfragen, die dieser Schritt einführt.
- Migration rückwärtskompatibel, damit alte und neue Version kurz nebeneinander laufen können?
- Backfill nötig? Rückweg vorhanden?

### Caching und Invalidierung
- Braucht dieser Schritt überhaupt einen Cache, oder ist das Vorwegnahme? Ohne belegten Bedarf: nein.
- Wenn ja: Ebene (Request, Anwendung, verteilt, HTTP), Schlüssel, Gültigkeitsdauer.
- Invalidierung ist der schwierige Teil — wer schreibt, und woher weiß der Cache davon?
- Was passiert bei Cache-Miss unter Last, und was bei kaltem Start?

### Verifikationsweg
- Woran wird sichtbar, dass dieser Schritt funktioniert — welcher Befehl, welcher Testfall, welcher manuelle Pfad?
- Was lässt sich nicht automatisiert prüfen, und wie wird es stattdessen abgesichert?

---

## Teil B — Findings

### Korrektheit gegenüber dem Plan
- Tut der Code, was der Schritt zugesagt hat — vollständig, nicht nur im Kern?
- Sind Abweichungen bewusst und begründet, oder unbemerkt entstanden?

### Fehlerfälle und Randbedingungen
- Leere Menge, sehr große Menge, ungültige Eingabe, Timeout, Teilausfall.
- Doppelte Ausführung: idempotent oder abgesichert?
- Nebenläufigkeit: gemeinsamer Zustand, Race Conditions, Transaktionsgrenzen.

### Sicherheit
- Autorisierung geprüft, nicht nur Authentifizierung.
- Eingaben validiert, Ausgaben kontextgerecht kodiert, Abfragen parametrisiert.
- Keine Secrets im Code, keine personenbezogenen Daten im Log.

### Performance
- Abfragen in Schleifen, N+1, fehlende Indizes, unbegrenzte Ergebnismengen.
- Blockierende Arbeit im Request-Pfad, die in einen Job gehört.
- Verhält sich das auch bei realistischer Datenmenge so, oder nur im Test?

### Konsistenz mit dem Bestand
- Benennung, Fehlerbehandlung, Struktur wie im umliegenden Code?
- Zweites Muster für ein bereits gelöstes Problem eingeführt?
- Begriffe wie im Domänenmodell des Projekts verwendet?

### Tests
- Decken sie die Zusage des Schritts ab und nicht nur den Happy Path?
- Prüfen sie Verhalten oder nur Implementierung?
- Laufen sie tatsächlich grün, und zwar auch isoliert?

### Reste
- Auskommentierter Code, Debug-Ausgaben, TODOs ohne Adressat, toter Code.
- Halbfertige Abstraktionen mit genau einem Aufrufer.
- Aktualisierte Konfiguration, Migrationen, Dokumentation vergessen?
