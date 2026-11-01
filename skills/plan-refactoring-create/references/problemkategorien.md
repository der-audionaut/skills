# Problemkategorien und Strategiekatalog

Teil A steuert das Sammeln der Problemfelder, Teil B die Wahl der Strategien. Beide Listen sind ein Sieb, keine Pflichtübung: Was die Codebase nicht zeigt, bleibt weg.

---

## Teil A — Problemkategorien

Der Beleg eines `P` ist die Fundstelle `Datei:Zeile` plus das Grep-Muster, das sie fand, oder die Kennzahl, die sie misst — nicht der Eindruck beim Lesen. Was nur beim Lesen auffällt, etwa strukturell gleiche Duplikate oder Kommentare gegen den Code, belegt die zitierte Zeile. Kennzahlen zu Größe und Änderungshäufigkeit kommen aus der Hotspot-Liste der Orientierung, nicht aus einem zweiten Lauf. Was sich nicht belegen lässt, ist eine offene Frage, kein Befund. Vermutete Fehler bekommen kein `P`; sie stehen im eigenen Abschnitt der Vorlage.

### Strukturprobleme

**Erkennungsmerkmale**
- Schichtverletzung: ein Aufruf gegen die Abhängigkeitsrichtung, etwa Persistenz direkt aus dem Handler oder Fachlogik im Controller.
- Zyklische Abhängigkeiten: Modul A importiert B und B importiert A, direkt oder über Dritte.
- Gott-Module: groß, oft geändert und von vielen abhängig — alle drei zugleich; Kandidaten sind die großen, oft geänderten Dateien der Hotspot-Liste.
- Falsche Zuständigkeit: ein Modul rechnet mit den Daten eines anderen, statt es zu fragen.
- Vermischte Belange: ein Modul trägt Ein- und Ausgabe, Fachregeln und Darstellung zugleich.
- Durchgriffe auf Interna: Zugriff auf private Felder, Unterpfade oder Hilfsfunktionen eines fremden Moduls statt auf dessen öffentliche Oberfläche.
- Implizite Kopplung: Verdrahtung über Zeichenketten, Reflection, globalen Zustand oder gemeinsam genutzte Konfiguration, die kein Import sichtbar macht.

**Beleg**
- Beide Seiten der Abhängigkeit.
- Kopplung als Zahl: ein Grep über alle Import-Anweisungen liefert die Kantenliste; daraus werden ein- und ausgehende Importe je Modul und die Zyklen abgelesen, ohne zweiten Lauf. Gezählt wird für die Module, die ein `P` oder die Umzugstabelle nennt.

**Kein Finding**
- Eine Schichtung, die das Projekt selbst nirgends einhält.
- Abhängigkeiten auf Framework-Basisklassen, die das Framework erzwingt — eine benannte Ausnahme der Zielarchitektur.

### Duplikate

**Erkennungsmerkmale**
- Identisch: derselbe Block an zwei Stellen, bis auf Leerraum.
- Strukturell gleich: dieselbe Abfolge mit anderen Bezeichnern oder Typen.
- Divergent kopiert: gemeinsame Herkunft, inzwischen verschieden — erkennbar an gleicher Form mit abweichenden Randfällen. Trägt den Vermerk „divergent", weil die Zusammenführung Verhalten ändern kann.

**Beleg**
- Beide Fundstellen und das Muster — ein markanter Ausdruck oder Funktionsname —, das beide fand; bei strukturell gleichen Blöcken die zitierten Zeilen.

**Kein Finding**
- Zufällige Ähnlichkeit ohne gemeinsamen Grund — eine Zusammenführung koppelt, was getrennt bleiben sollte.
- Tests, die bewusst wiederholen, um lesbar zu bleiben.

### Performance-Engpässe

**Erkennungsmerkmale**
- Abfragen oder externe Aufrufe in Schleifen (N+1).
- Unbegrenzte Ergebnismengen: Laden ohne Limit, Paginierung oder Streaming.
- Wiederholte Berechnung ohne Wiederverwendung innerhalb eines Durchlaufs.
- Blockierende Arbeit im Request-Pfad, die in einen Job gehört.
- Fehlende Indizes für Abfragen, die der Code beschreibt.

**Beleg**
- Statisch gefunden heißt „Verdacht": das Muster, das die Schleife oder den Aufruf zeigt. Ein Verdacht kann eine Strategie bekommen; die Akzeptanz des Schritts ist dann die gezählte Abfragezahl.
- „belegt" nur mit einer im Repository abgelegten Messung — Benchmark-Ergebnis, Profil-Ausgabe. Fehlt sie, bleibt es beim Verdacht; gesucht wird nicht weiter.

**Kein Finding**
- Ein Muster, das laut Datenmenge nicht trägt: Eine Schleife über drei Konfigurationseinträge ist kein N+1.

### Wartbarkeitsrisiken

**Erkennungsmerkmale**
- Hotspots ohne Tests: hohe Änderungshäufigkeit und kein Test auf dem Pfad.
- Magische Werte, lange Parameterlisten, boolesche Flags als Verhaltensschalter.
- Kommentare, die dem Code widersprechen; tote Pfade; TODOs ohne Adressat.
- Konfiguration an mehreren Stellen für dasselbe Ding.

**Beleg**
- Kennzahl aus der Hotspot-Liste (Änderungen, Zeilen je Datei), Parameter je Funktion — oder das Grep-Muster.

**Kein Finding**
- Stil, der den Konventionen des Projekts folgt, auch wenn er anderswo anders gelöst würde.

### Schwere

- **Hoch** — behindert jede Änderung im Bereich oder wächst mit jedem Feature. Braucht eine Strategie.
- **Mittel** — kostet bei jeder Berührung, bleibt sonst stabil. Bekommt eine Strategie, wenn sie auf demselben Weg liegt wie ein Hoch-Befund.
- **Niedrig** — lokal oder Geschmack. Steht mit Grund unter Nicht-Ziele.

---

## Teil B — Strategiekatalog

Eine Strategie löst benannte `P`-Kennungen; eine Strategie ohne `P` ist Vorwegnahme. Die Zielarchitektur misst sich an drei Maßstäben — Belange getrennt, Module mit klarer Oberfläche, Kopplung gesenkt — und belegt sie mit den Importzahlen aus Teil A vorher und nachher; „nachher" wird am geplanten Importgraphen gezählt.

Drei Regeln gelten für jede Technik. Jede Reihenfolge beginnt mit den Tests, die der Verhaltensschutz nennt — fehlen sie, geht ein Charakterisierungstest-Schritt voraus. Entfernt wird erst, wenn der Grep auf das Alte leer ist; Zeichenketten, Konfiguration und Reflection zählen als Aufrufer, und dieser Grep ist die Akzeptanz des Schritts. Struktur ändert sich nie im selben Schritt wie Logik.

### Extrahieren (Funktion, Modul, Klasse)
- **Wann:** ein Block tut Erkennbares, das einen Namen verdient, oder zwei Belange teilen sich eine Funktion.
- **Reihenfolge:** Block extrahieren → am alten Ort aufrufen.
- **Verhaltensschutz:** die Tests der umgebenden Funktion; die extrahierte Einheit bekommt eigene Tests erst mit eigener Oberfläche.
- **Falle:** den Block mit allen Variablen der Umgebung extrahieren — eine Funktion mit acht Parametern ist kein Gewinn; erst die Daten bündeln, dann extrahieren.

### Verschieben mit Umleitung der Aufrufer
- **Wann:** eine Funktion lebt im falschen Modul, oder ein Modul kennt eine Schicht, die es nicht kennen dürfte.
- **Reihenfolge:** am neuen Ort anlegen → alter Ort delegiert → Aufrufer einzeln umstellen → alten Ort entfernen.
- **Verhaltensschutz:** die Tests der Aufrufer.
- **Falle:** die Delegation am alten Ort stehen lassen — dann gibt es zwei Orte für eine Funktion, und der nächste Aufrufer nimmt den falschen.

### Zusammenführen von Duplikaten
- **Wann:** identische, strukturell gleiche oder divergent kopierte Blöcke mit gemeinsamem Grund.
- **Reihenfolge:** gemeinsame Funktion aus einer Seite extrahieren → zweite Seite umstellen.
- **Verhaltensschutz:** beide Testsätze, bei divergenten Duplikaten mit den Randfällen, in denen sich die Seiten unterscheiden.
- **Falle:** divergente Duplikate auf die „schönere" Seite vereinheitlichen und die Randfälle der anderen verlieren.

### Naht einziehen (Schnittstelle, Adapter)
- **Wann:** eine Abhängigkeit soll austauschbar oder testbar werden, bevor sie bewegt wird.
- **Reihenfolge:** Schnittstelle aus der bestehenden Nutzung ableiten → Bestand dahinter stellen, ohne ihn zu ändern → Aufrufer auf die Schnittstelle umstellen.
- **Verhaltensschutz:** die Tests der Aufrufer; die Schnittstelle bekommt einen Vertragstest, den alter und neuer Bestand bestehen müssen.
- **Falle:** die Schnittstelle nach dem Wunschbild statt nach der tatsächlichen Nutzung schneiden — dann passt der Bestand nicht dahinter.

### Parallel Change
- **Wann:** eine Signatur, ein Datenformat oder ein Modul soll ersetzt werden, während die Aufrufer weiterlaufen.
- **Reihenfolge:** Neues neben dem Alten anlegen → Aufrufer einzeln umstellen, jeder Schritt lauffähig → Altes entfernen.
- **Verhaltensschutz:** die Tests der Aufrufer nach jeder Umstellung.
- **Falle:** die Übergangsphase abkürzen und alt und neu in einem Schritt tauschen.

### Strangler für große Module
- **Wann:** ein Gott-Modul, das sich nicht in einem Zug ersetzen lässt.
- **Reihenfolge:** Fassade vor das Modul → eine Verantwortung nach der anderen hinter der Fassade in ein neues Modul ziehen, bis die Fassade nur noch auf Neues zeigt → Fassade entfernen.
- **Verhaltensschutz:** Charakterisierungstests gegen die Fassade, die über alle Umbauten gleich bleiben.
- **Falle:** zwei Verantwortungen gleichzeitig ziehen, weil sie „zusammengehören" — dann ist unklar, welcher Zug den Test brach.

### Abfragen bündeln (gegen N+1)
- **Wann:** eine Schleife löst je Durchlauf eine Abfrage oder einen externen Aufruf aus.
- **Reihenfolge:** Daten vor der Schleife in einem Zug laden → Schleife auf die geladene Menge umstellen.
- **Verhaltensschutz:** der Ergebnistest der Schleife; die gezählte Abfragezahl ist die Akzeptanz.
- **Falle:** Reihenfolge oder Fehlerverhalten ändern — eine gebündelte Abfrage scheitert anders als zehn einzelne.

### Entkoppeln über Port oder Ereignis
- **Wann:** zwei Module kennen sich gegenseitig, oder ein innerer Teil kennt einen äußeren.
- **Reihenfolge:** die Richtung bestimmen, die bleiben soll → für die andere einen Port (Schnittstelle im inneren Modul, Umsetzung außen) oder ein Ereignis einführen → den Gegenimport entfernen.
- **Verhaltensschutz:** die Tests beider Module; der Import-Grep belegt, dass der Zyklus weg ist.
- **Falle:** ein Ereignis einführen, wo ein Port reicht — asynchrone Kopplung ist schwerer zu testen als eine Schnittstelle.

### Schichten trennen (Clean Architecture)
- **Wann:** Fachlogik, Anwendungsfälle und Adapter liegen durcheinander, und die Zielarchitektur verlangt Abhängigkeiten nach innen.
- **Reihenfolge:** Domäne benennen und über Ports von Außenabhängigkeiten befreien → Anwendungsfälle darüber legen → Adapter außen anschließen.
- **Verhaltensschutz:** Tests der Anwendungsfälle gegen die Ports; Charakterisierungstests an den Adaptern.
- **Falle:** eine Schablone über ein Framework stülpen, das seine eigene Ablage und seine eigenen Basisklassen vorgibt — die Trennung geschieht innerhalb seiner Vorgaben.

### Modul umziehen
- **Wann:** die Zielarchitektur gibt dem Modul einen anderen Ort.
- **Reihenfolge:** Verzeichnis wechseln → Namespace oder Package, Pfade und alle Importe im selben Schritt umstellen.
- **Verhaltensschutz:** der Testlauf im Stand der Baseline.
- **Falle:** beim Umzug gleich aufräumen — dann erkennt Git die Umbenennung nicht mehr (`git diff -M` braucht Ähnlichkeit), und der Diff versteckt die Logikänderung hinter hundert verschobenen Zeilen.
