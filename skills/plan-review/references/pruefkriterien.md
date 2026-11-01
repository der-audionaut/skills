# Prüfdimensionen

Je Dimension: trifft sie zu? Wenn ja, welcher Beleg im Code oder im Plan stützt deine Bewertung? Überschriften und Punktnamen, die eine Dimension nennt, meinen das Element dieser Bedeutung — in einem fremdsprachigen Plan suchst und grepst du ihre Übersetzung.

## Scope und Vollständigkeit
- Beschreibt der Plan, was **nicht** dazugehört? Ein Plan ohne Nicht-Ziele wächst während der Umsetzung.
- Deckt er das Feature vollständig ab oder endet er bei der Happy Path?
- Gibt es Anforderungen aus dem Ticket oder der Diskussion, die im Plan fehlen?

## Faktentreue gegenüber der Codebase
- Existieren alle genannten Dateien, Klassen, Funktionen, Tabellen, Endpunkte?
- Stimmen Signaturen, Rückgabetypen und Parameter mit dem Plan überein?
- Baut der Plan etwas nach, das es schon gibt (Helper, Service, Utility)?
- Passt der vorgeschlagene Ansatz zu den Mustern im Bestand, oder führt er ein zweites, konkurrierendes Muster ein?

## Offene Entscheidungen
- Formulierungen wie "vermutlich", "sollte man klären", "TODO", "je nachdem" markieren ungeklärte Punkte.
- Wo trifft der Plan implizit eine Entscheidung, ohne die Alternative zu nennen?
- Welche Entscheidung kann nur der Mensch treffen (Produkt, Priorität, Risiko)?

## Reihenfolge und Abhängigkeiten
- Ist jeder Schritt mit dem Zustand nach dem vorherigen Schritt ausführbar?
- Bleibt das System nach jedem Schritt lauffähig, oder gibt es einen kaputten Zwischenzustand?
- Migration vor Code-Nutzung, Feature-Flag vor Rollout, Verbraucher vor Entfernung der alten Schnittstelle?
- Externe Abhängigkeiten (fremdes Team, Drittsystem, Freigabe) benannt und terminiert?

## Datenmodell und Migration
- Migration rückwärtskompatibel? Was passiert mit laufenden Instanzen der alten Version?
- Sind bestehende Daten abgedeckt (Backfill, Defaults, Nullable)?
- Gibt es einen Rückweg, wenn die Migration schiefgeht?
- Indizes, Constraints, Fremdschlüssel mitgedacht?

## Schnittstellen und Verträge
- Wer konsumiert die geänderte API, das Event, die Bibliothek? Sind alle Aufrufer erfasst?
- Breaking Change erkannt und versioniert oder vermieden?
- Verträge nach außen (Kunden, Partner, andere Teams) berührt?

## Fehlerfälle und Randbedingungen
- Was passiert bei Timeout, Teilausfall, doppelter Ausführung, leerer Menge, sehr großer Menge?
- Idempotenz nötig? Nebenläufigkeit, Race Conditions, Sperren?
- Wie verhält sich das Feature bei ungültiger Eingabe?

## Sicherheit und Berechtigungen
- Wer darf das? Wird Autorisierung geprüft, nicht nur Authentifizierung?
- Personenbezogene Daten berührt? Löschkonzept, Protokollierung, Zweckbindung?
- Neue Angriffsfläche: Eingaben, Uploads, externe Aufrufe, Secrets im Code?

## Performance
- Zusätzliche Abfragen pro Request, N+1, unbegrenzte Ergebnismengen?
- Skaliert der Ansatz mit der realistischen Datenmenge, nicht nur mit der im Test?
- Blockierende Arbeit im Request-Pfad, die in einen Job gehört?

## Tests und Verifikation
- Woran wird "fertig" gemessen? Steht das je Schritt im Plan?
- Welche Testebene je Änderung (Unit, Integration, End-to-End)?
- Sind die Fehlerfälle aus den Randbedingungen testbar beschrieben?
- Gibt es einen manuellen Prüfpfad für das, was Tests nicht abdecken?

## Verhaltenserhalt (Pläne mit Problemfeldern)
Trifft zu, wenn der Plan einen Abschnitt `## Problemfelder` hat; sonst überspringen. Eine Frage mit dem Vorsatz „Bei `## …`:" gilt nur, wenn der Plan diesen Abschnitt hat — was der Plan nicht hat, prüfst du nicht, auch wenn ein Schritt inhaltlich danach aussieht.
- Hat jeder Schritt den Punkt `**Verhaltensschutz:**`? Ein Grep mit Zeilennummern auf `^## |^### Schritt |^- \*\*(Verhaltensschutz|Messung):\*\*` zeigt es: Unter `## Schritte` folgt auf jede Schritt-Überschrift genau ein Verhaltensschutz-Treffer und, bei `## Zielwerte`, genau ein Messung-Treffer; Strategien und Historie zählen nicht. Ein Schritt ohne Verhaltensschutz ist ein Blocker.
- Existieren die genannten Tests (Beleg aus Faktentreue), oder geht der Bewegung ein Charakterisierungstest-Schritt in einer früheren Welle voraus?
- Ändert eine Strategie oder ein Schritt beobachtbares Verhalten — API, Ausgabe, Persistenz, Reihenfolge?
- Hat jeder Befund der Schwere Hoch eine Strategie, oder trägt der Grund, mit dem er unter Nicht-Ziele steht?
- Bei `## Zielarchitektur`: Zeigen ihre Abhängigkeiten nach innen, oder ist jede Ausnahme benannt und begründet (Framework-Basisklassen, Active Record), und erreicht die Schrittfolge die Zielarchitektur?
- Bei `## Zielwerte`: Verweist der Punkt `**Messung:**` bei jedem Schritt, dessen `**Betroffen:**` eine Datei aus `### Hot Paths` nennt, auf eine Zeile der Zielwerte, sonst auf „keine" mit Grund? Trägt jede Zeile der Zielwerte einen Messbefehl oder „nicht gemessen", und nennt ein Schritt mit eigenem Vorher-Wert den Schritt, der ihn erzeugt hat? Ein Hot-Path-Schritt ohne Messung ist ein Blocker.

## Betrieb
- Feature-Flag oder direkter Rollout? Wer schaltet, nach welchem Kriterium?
- Rollback möglich, und zwar ohne Datenverlust?
- Logging, Metriken, Alarme: woran merkt jemand, dass es kaputt ist?
- Konfiguration, Secrets, Umgebungsvariablen für alle Umgebungen ergänzt?

## Schnitt der Schritte
- Ist ein Schritt so groß, dass er selbst wieder einen Plan bräuchte?
- Hat jeder Schritt ein überprüfbares Ergebnis, oder endet er bei "implementieren"?
- Steht der Aufwand im Verhältnis zum Nutzen, oder ließe sich das Ziel deutlich kleiner erreichen?

## Sprache und Konsistenz
- Nutzt der Plan die Begriffe der Domäne so, wie das Projekt sie nutzt (`CONTEXT.md`, ADRs, bestehender Code)?
- Widerspricht der Plan einer dokumentierten Entscheidung? Falls ja: begründet er den Bruch?
- Werden dieselben Dinge im Plan durchgängig gleich benannt?
