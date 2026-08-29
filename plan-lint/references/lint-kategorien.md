# Lint-Kategorien

Erkennungsmerkmale für die drei Finding-Arten. Teil D am Ende betrifft die Architekturebene und ist die ergiebigste Quelle für Widersprüche, weil dieselbe Entscheidung dort an mehreren Stellen beschrieben wird.

---

## T — Totes Wissen

Kennzeichen ist immer: die Aussage beschreibt einen Zustand, den es nicht mehr gibt oder nie gab.

- **Erledigte Arbeit im Futur.** Ein umgesetzter Schritt, der weiterhin beschreibt, was gebaut *werden soll*. Abgleich über `## Umsetzungs-Historie` und den Code.
- **Abgehakte Fragen.** TODOs, "muss noch geklärt werden", "abhängig von X" — wo die Antwort inzwischen woanders im Plan oder in der Historie steht.
- **Verweise ins Leere.** Genannte Dateien, Funktionen, Felder, Endpunkte, Tabellen, Tickets, die es nicht (mehr) gibt. Per Grep prüfen, nicht schätzen.
- **Platzhalter.** "TBD", "hier später ergänzen", leere Abschnitte, Beispielwerte aus einer Vorlage.
- **Dubletten.** Derselbe Sachverhalt an zwei Stellen ausformuliert. Die schwächere Fassung geht, die stärkere bleibt — nicht umgekehrt, nur weil sie weiter oben steht.
- **Verwaister Kontext.** Ein Abschnitt, auf den kein Schritt und keine Entscheidung mehr zugreift, weil der Teil des Features gestrichen wurde.
- **Überholte Zustandsbeschreibungen.** "Aktuell macht das der Legacy-Service" — wo der Legacy-Service in Schritt 1 abgelöst wurde.

**Kein totes Wissen:** Begründungen, verworfene Alternativen mit Begründung, Nicht-Ziele, Risiken, offene Fragen an den Menschen, Anforderungen ohne zugehörigen Schritt (das ist eine Lücke und wird gemeldet, nicht gelöscht).

---

## I — Inkonsistenzen

Kennzeichen: dieselbe Sache, unterschiedlich dargestellt. Beide Stellen können richtig gemeint sein.

- **Benennung.** `invoice_status` / `billingState` / "Rechnungsstatus" für ein Feld. Abgleich mit Code und `CONTEXT.md`.
- **Typen und Formate.** Feld im Schema-Abschnitt als Integer, in der API als String. Zeitstempel mal mit, mal ohne Zeitzone.
- **Nummerierung und Reihenfolge.** Schritte, die nach einer Einfügung springen; Verweise auf "Schritt 4", der inzwischen Schritt 5 ist.
- **Granularität.** Ein Schritt in fünf Unterpunkten, der nächste als ein Satz — ein Hinweis darauf, dass der zweite nicht zu Ende gedacht ist.
- **Sprachebene.** Fachbegriff aus der Domäne hier, technisches Synonym dort, für dasselbe Konzept.
- **Struktur.** Schritte mit Akzeptanzkriterium und Schritte ohne, im selben Plan.

---

## W — Widersprüche

Kennzeichen: die beiden Aussagen können nicht gleichzeitig wahr sein. Immer beide Fundstellen zitieren.

- **Ausführungsmodell.** Synchron im Request an einer Stelle, asynchron im Job an der anderen.
- **Datenmodell.** Feld optional gegen Feld verpflichtend; ein Wert gegen eine Liste; eindeutig gegen mehrfach.
- **Quelle der Wahrheit.** Zwei Abschnitte, die verschiedene Systeme als führend beschreiben.
- **Zahlenwerte.** Gültigkeitsdauern, Grenzwerte, Seitengrößen, Zeitfenster, die an zwei Stellen verschieden angegeben sind.
- **Reihenfolge.** Schritt A setzt Schritt B voraus, während B laut Plan nach A kommt.
- **Scope.** Etwas steht in den Nicht-Zielen und taucht in einem Schritt als Aufgabe wieder auf.
- **Gegen dokumentierte Entscheidungen.** Der Plan widerspricht einem ADR oder einer Festlegung aus der `## Review-Historie`, ohne den Bruch zu benennen.

---

## D — Architekturebene

Dieselbe Entscheidung steht in einem Plan meist vier- oder fünfmal, in verschiedenen Kapiteln. Genau dort laufen die Fassungen auseinander:

- **Architektur gegen Datenfluss.** Beschreibt der Datenfluss Wege, die der Komponentenschnitt nicht zulässt?
- **Datenfluss gegen API.** Liefern die beschriebenen Endpunkte die Daten, die der Fluss an dieser Stelle voraussetzt?
- **API gegen Schema.** Jedes Feld der API im Datenmodell vorhanden, mit passendem Typ und passender Nullbarkeit? Und umgekehrt: Felder im Schema, die nie jemand liest?
- **Schema gegen Zugriffsmuster.** Setzen beschriebene Abfragen Indizes oder Beziehungen voraus, die das Schema nicht hat?
- **Caching gegen Datenfluss.** Wer schreibt die Daten, und kommt die Invalidierung in der beschriebenen Reihenfolge überhaupt vor dem nächsten Lesen an? Ein Cache ohne benannten Invalidierungspfad ist ein Widerspruch, kein Detail.
- **Caching gegen Konsistenzzusage.** Der Plan verspricht an einer Stelle sofortige Sichtbarkeit und legt an anderer eine Gültigkeitsdauer fest.
- **Entwurf gegen Schrittfolge.** Beschreibt der Entwurf einen Endzustand, den die Schritte in Summe gar nicht erreichen?
