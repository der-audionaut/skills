# Optimierungskategorien und Strategiekatalog

Teil A steuert das Sammeln der Problemfelder, Teil B die Wahl der Strategien. Beide Listen sind ein Sieb, keine Pflichtübung: Was die Messung und die Codebase nicht zeigen, bleibt weg. Aufbau und Einleitungssätze stammen aus `skills/plan-refactoring-create/references/problemkategorien.md`; wo beide Kataloge zu Performance etwas sagen, gilt dieser.

---

## Teil A — Problemkategorien

Der Beleg eines `P` ist gemessen oder ein Verdacht. Gemessen: Der Wert steht in der Baseline oder einer im Repository abgelegten Messung, und der Befund verweist auf die Zeile der Zielwerte. Gemessen ist nur, was die Baseline ausgibt — ein eigener Lauf, um einen Verdacht zu belegen, ist der Schritt „Messen zuerst", kein Beleg. Verdacht: Fundstelle `Datei:Zeile`, das Muster und die Datenmenge, ab der es trägt. Ein Muster auf einer Menge, die laut Fixtures, Konfiguration oder Fachlichkeit klein bleibt, ist kein Befund; Stil ohne Laufzeitwirkung auch nicht — Lesbarkeit ist Sache von `/plan-refactoring-create`. Was sich nicht belegen lässt, ist eine offene Frage, kein Befund. Vermutete Fehler bekommen kein `P`; sie stehen im eigenen Abschnitt der Vorlage.

### Engpässe

**Erkennungsmerkmale**
- Abfragen oder externe Aufrufe in Schleifen (N+1).
- Blockierende Arbeit im Request-Pfad, die in einen Job gehört.
- Sperren und serielle Wartezeiten, die unabhängige Arbeit hintereinander zwingen.
- Synchrone Ein- und Ausgabe in Hot Paths.

**Beleg**
- Gemessen: Anteil des Pfads an der Laufzeit. Verdacht: die Schleife oder der Aufruf und die Zahl der Durchläufe.

**Kein Finding**
- Blockierende Arbeit, deren Ergebnis die Antwort braucht — sie kann nicht in einen Job.

### Ineffiziente Logik

**Erkennungsmerkmale**
- Quadratische oder schlechtere Suche in Listen, wo eine Menge oder Abbildung dient.
- Wiederholte Berechnung desselben Werts je Aufruf oder je Durchlauf.
- Unnötige Kopien und Umwandlungen zwischen Darstellungen.
- Eine Datenstruktur, die zum Zugriffsmuster nicht passt.

**Beleg**
- Gemessen: Laufzeit der Operation über die Datenmenge. Verdacht: die Komplexität, die der Code zeigt.

**Kein Finding**
- Eine Berechnung, die der Code ohnehin nur einmal ausführt.

### Unnötige Arbeit

**Erkennungsmerkmale**
- Rendern unveränderter Teile.
- Laden nicht benötigter Daten oder ganzer Listen, wo Ausschnitt oder Virtualisierung reicht.
- Arbeit vor dem ersten Bedarf, deren Ergebnis oft ungenutzt bleibt.

**Beleg**
- Gemessen: Aufrufe, Renderings oder geladene Datensätze je Vorgang gegen die genutzten. Verdacht: die Stelle, die Arbeit ohne Bedarf auslöst.

**Kein Finding**
- Vorbereitende Arbeit, deren Ergebnis belegt immer gebraucht wird.
- Rendering-Muster in einem Projekt ohne Oberfläche.

### Speicher

**Erkennungsmerkmale**
- Unbegrenzt wachsende Caches und Sammlungen.
- Materialisieren ganzer Ergebnisse, wo Streamen oder Iterieren reicht.
- Halten großer Objekte über ihre Nutzung hinaus.
- Lecks über Registrierungen, Abonnements und Schließungen, die nie gelöst werden.

**Beleg**
- Gemessen: Spitzenverbrauch oder Verlauf. Verdacht: die Stelle, die hält oder materialisiert.

**Kein Finding**
- Speicher, der mit dem Vorgang endet.

### Skalierung

**Erkennungsmerkmale**
- Zustand im Prozess, der mehrere Instanzen verhindert.
- Fehlende Paginierung oder Begrenzung an Schnittstellen.
- Arbeit, die mit der Gesamtmenge statt mit dem Ergebnis wächst.
- Fehlende Indizes für Abfragen, die der Code beschreibt.

**Beleg**
- Gemessen: Laufzeit oder Durchsatz bei wachsender Datenmenge. Verdacht: die Arbeit, die mit der Gesamtmenge wächst.

**Kein Finding**
- Eine Schnittstelle ohne Paginierung, deren Menge die Fachlichkeit deckelt — Länder, Statuswerte.

### Schwere

- **Hoch** — liegt auf dem Hot Path und ist gemessen, oder bestimmt die Komplexität der Hauptoperation. Braucht eine Strategie.
- **Mittel** — liegt auf dem Hot Path als Verdacht. Bekommt eine Strategie, wenn sie auf demselben Weg liegt wie ein Hoch-Befund oder der geplante Messschritt ihre Kennzahl abdeckt.
- **Niedrig** — liegt auf einem kalten Pfad. Steht mit Grund unter Nicht-Ziele.

---

## Teil B — Strategiekatalog

Eine Strategie löst benannte `P`-Kennungen; eine Strategie ohne `P` ist Vorwegnahme.

Drei Regeln gelten für jede Technik. Jede Optimierung beginnt mit der Messung ihrer Kennzahl und endet mit derselben Messung, mit demselben Befehl. Struktur und Laufzeit ändern sich nie im selben Schritt wie Verhalten — Ergebnis, Reihenfolge, Fließkomma-Genauigkeit und Fehlerverhalten bleiben. Ein kalter Pfad wird nicht optimiert.

### Messen zuerst
- **Wann:** die Baseline fehlt oder deckt die Kennzahl nicht ab.
- **Kennzahl:** die, die der Plan als Erstes bewegen will; der Schritt liefert ihren Baseline-Wert.
- **Verhaltensschutz:** keiner nötig — der Schritt ändert keinen Produktionscode.
- **Falle:** einen Mikro-Benchmark bauen, der nicht den Hot Path misst, und dann dessen Zahl optimieren.

### Algorithmus und Datenstruktur wechseln
- **Wann:** quadratische Suche oder falsche Struktur fürs Zugriffsmuster.
- **Kennzahl:** Laufzeit der Operation über die realistische Datenmenge.
- **Verhaltensschutz:** Ergebnisgleichheit einschließlich Reihenfolge und Randfällen — leere Menge, Duplikate, Grenzwerte.
- **Falle:** eine Menge einführen und damit die Reihenfolge verlieren, auf die ein Aufrufer baut.

### Memoisierung und Caching
- **Wann:** dieselbe Eingabe wird wiederholt berechnet oder geladen.
- **Kennzahl:** Zahl der Berechnungen oder Zugriffe je Vorgang, dann Laufzeit.
- **Verhaltensschutz:** ein Test je Schreibpfad, der die Invalidierung belegt; ein Test für die Größenbegrenzung.
- **Falle:** ohne Invalidierungspfad oder ohne Größenbegrenzung — der erste Fall liefert veraltete Daten, der zweite wird zum Speicherbefund.

### Bündeln und Vorladen
- **Wann:** Abfragen oder Aufrufe je Schleifendurchlauf (N+1).
- **Kennzahl:** Zahl der Abfragen oder Aufrufe je Vorgang.
- **Verhaltensschutz:** Fehlerverhalten, wenn ein Element fehlt oder scheitert.
- **Falle:** alles vorladen, auch was der Vorgang nie anfasst — dann wandert der Befund zu Speicher.

### Verzögern, Paginieren, Streamen
- **Wann:** ganze Ergebnisse werden materialisiert oder geladen, obwohl nur ein Ausschnitt gebraucht wird.
- **Kennzahl:** Spitzenverbrauch und Zeit bis zum ersten Ergebnis.
- **Verhaltensschutz:** Vollständigkeit und Reihenfolge über alle Seiten; Verhalten bei Änderungen während des Durchlaufs.
- **Falle:** Paginierung ohne stabile Sortierung — Elemente erscheinen doppelt oder gar nicht.

### Indizes für beschriebene Abfragen
- **Wann:** eine Abfrage filtert oder sortiert über ein Feld ohne Index, und die Menge wächst.
- **Kennzahl:** Laufzeit der Abfrage bei realistischer Menge.
- **Verhaltensschutz:** der Abfragetest und die Messung eines Schreibpfads.
- **Falle:** einen Index für eine seltene Abfrage — er verteuert jedes Schreiben.

### Asynchron oder parallel
- **Wann:** die Messung zeigt Wartezeit auf unabhängige Ein- und Ausgabe oder externe Aufrufe.
- **Kennzahl:** Wartezeit je Vorgang gegen Gesamtlaufzeit.
- **Verhaltensschutz:** Ergebnisgleichheit bei beliebiger Reihenfolge der Antworten; Fehlerverhalten, wenn ein Teil scheitert.
- **Falle:** CPU-gebundene Arbeit parallelisieren, die sich den Interpreter oder eine Sperre teilt — Mehraufwand ohne Gewinn.

### Rendering: Memoisierung, stabile Schlüssel, Virtualisierung
- **Wann:** unveränderte Teile werden neu gerendert oder lange Listen vollständig gezeichnet — nur in Projekten mit Oberfläche.
- **Kennzahl:** Zahl der Renderings je Interaktion, Zeit bis zur Darstellung.
- **Verhaltensschutz:** sichtbares Ergebnis und Zustand der Eingabefelder bleiben; Snapshot- oder Komponententests.
- **Falle:** Zustand nach oben ziehen oder Schlüssel aus dem Index bilden — dann rendert mehr, nicht weniger.

### Allokationen senken, Puffer wiederverwenden
- **Wann:** ein Hot Path erzeugt je Durchlauf viele kurzlebige Objekte.
- **Kennzahl:** Allokationen je Vorgang, dann Laufzeit.
- **Verhaltensschutz:** ein Test mit zwei gleichzeitigen Vorgängen.
- **Falle:** einen Puffer teilen, der zwei Vorgänge zugleich sieht.

### Unnötige Arbeit entfernen
- **Wann:** eine Arbeit wird ausgelöst, deren Ergebnis belegt ungenutzt bleibt.
- **Kennzahl:** Zahl der Ausführungen je Vorgang, dann Laufzeit.
- **Verhaltensschutz:** der Test, der zeigt, dass das Ergebnis nirgends gebraucht wird — Grep auf die Verwender und ein Lauf ohne die Arbeit.
- **Falle:** eine Arbeit entfernen, die über Nebenwirkung gebraucht wird — Protokoll, Zähler, Cache-Vorwärmung.
