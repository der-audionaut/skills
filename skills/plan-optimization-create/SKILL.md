---
name: plan-optimization-create
description: Erstellt aus Messung und Codebase-Analyse ein Optimierungs-Plandokument im Markdown-Format und legt es im Projekt ab — Baseline aus den dokumentierten Messbefehlen, belegte Problemfelder mit Datei:Zeile und der Einstufung gemessen oder Verdacht, Zielwerte je Kennzahl, Optimierungsstrategien und Schritte, die das Verhalten unverändert lassen und je eine Messung vorher und nachher tragen; im Aufbau kompatibel zu plan-review, plan-lint und plan-coding. Nimmt optional einen Bereich (Pfad, Modul oder Fokus in Anführungszeichen) und die Ausgabesprache als Argumente; ohne Angabe die ganze Codebase und Deutsch.
argument-hint: "[bereich] [sprache]"
arguments: bereich sprache
disable-model-invocation: true
allowed-tools: Read Grep Glob Bash Write Edit
---

**Ausgabesprache: `$sprache`** — steht dort nichts oder ein Wort, das erkennbar keine Sprache benennt, Deutsch. Alles, was du an den Menschen richtest oder in die Plandatei schreibst, schreibst du in dieser Sprache: Plandokument, Rückfragen, Abbruchmeldungen, Übergabe. Dass diese Anweisung deutsch ist, ändert daran nichts.

Plane die Optimierung für den Bereich `$bereich`. Denke wie ein Performance-Ingenieur: erst messen, dann suchen, dann planen. Die Ziele sind Laufzeit, Speicher und Skalierung. Das Verhalten bleibt, die Zahlen bewegen sich. Das Ergebnis ist ein Dokument, kein Code: Der schnellere Code entsteht Schritt für Schritt über `/plan-coding`, und jeder Schritt trägt seine Messung vorher und nachher.

## 1. Argumente auflösen

**Bereich.** `$bereich` ist ein Pfad (`src/search`), ein Modulname oder ein Fokus in Anführungszeichen (`"Import-Durchsatz"`). Leer heißt: die ganze Codebase. Sieht der Bereich wie ein Pfad aus, prüfst du ihn per Glob; gibt es ihn nicht, fragst du nach, statt stillschweigend alles zu analysieren. Bestimme jetzt auch Ablageort und Dateinamen nach Abschnitt 8 und prüfe per Glob, ob die Datei schon existiert — dann fragst du vor Test- und Messlauf, statt fünf Minuten zu messen und das Ergebnis zu verwerfen.

**Sprache.** Die Ausgabesprache ist das letzte, optionale Argument — ein Sprachname oder Kürzel in beliebiger Schreibweise (`englisch`, `english`, `en`). Der Aufruf wird wie eine Shell-Kommandozeile zerlegt, deshalb gilt der vollständige Aufruf **$ARGUMENTS**: Benennt sein letztes Wort erkennbar eine Sprache, ist das die Ausgabesprache und alles davor der Bereich — auch wenn davor nichts steht. Sonst ist der ganze Aufruf der Bereich, und die Sprache bleibt Deutsch. Könnte das letzte Wort auch zum Bereich gehören („auf englisch"), rate nicht: nimm die Frage in die Rückfragen aus Abschnitt 7 auf und nimm bis zur Antwort Deutsch an.

Die Sprache ändert den Text, nicht die Struktur: Überschriften, Spaltentitel, Kopfblock-Beschriftungen und Prosa werden übersetzt; Schrittnummern, die Wellen-Kürzel `W1, W2 …`, die Kennungen `P1, P2 …` und der Schlüssel `Claude-Session` bleiben, weil `/plan-review`, `/plan-lint` und `/plan-coding` sie später referenzieren.

## 2. Orientieren

Eine große Codebase lässt sich nicht vollständig lesen, und ein Plan, der so tut, erfindet. Lies in dieser Reihenfolge und halte an, sobald ein Bild steht: (1) Verzeichnisbaum bis Tiefe zwei, Manifeste, Build-, Test- und Messbefehle, `CLAUDE.md`, `CONTEXT.md`, ADRs — Terminologie und getroffene Entscheidungen sind bindend; (2) Messwerkzeuge: Benchmark-, Profil- und Performance-Testdateien, im Repository abgelegte Messergebnisse, Metriken und Logs; (3) Einstiegspunkte: main, Routen, CLI, Handler, Jobs, Render-Komponenten; (4) die Hot Paths aus Abschnitt 3; (5) die Datenmengen dieser Pfade aus Fixtures, Migrationen und Konfiguration, weil ein Muster erst ab einer Menge trägt; (6) gezielte Greps entlang der Hot Paths für Abschnitt 4. Bash nutzt du nur lesend; die Ausnahmen sind Testlauf und Messlauf unten.

Zahlen im Plan haben genau zwei Quellen: Messwerte aus dem Messlauf oder aus abgelegten Messergebnissen, Datenmengen aus Fixtures, Migrationen und Konfiguration. Was die Baseline nicht ausgibt, ist Verdacht (Teil A des Katalogs).

**Testlauf.** Führe den dokumentierten Test- oder Buildbefehl — aus `CLAUDE.md`, README, Manifest-Skripten oder Makefile — genau einmal aus, mit einem Zeitlimit von fünf Minuten und ohne Abhängigkeiten zu installieren, und notiere Befehl und Ergebnis unter „Build und Tests": grün, rot mit den roten Tests, damit kein Schritt sie erbt, oder „unbekannt", wenn kein Befehl dokumentiert ist. Läuft er über das Limit, brich ab, notiere das und starte keinen zweiten Versuch.

**Messlauf.** Führe den dokumentierten Mess- oder Benchmark-Befehl aus denselben Quellen aus, mit einem Zeitlimit von fünf Minuten für die gesamte Messung (`timeout 300 <Befehl>`, der erste Lauf unter `time`) und ohne Installation. Lag der erste Lauf unter einer Minute, wiederhole ihn zweimal und nimm je Kennzahl den Median aus drei Läufen mit der Spanne als Streuung; sonst bleibt es bei einem Lauf, und die Baseline trägt den Vermerk „Einzelmessung, Streuung unbekannt". Lauf und Umgebung stehen unter „Baseline-Messung", wie die Vorlage es vorgibt; die Werte je Kennzahl nur in den Zielwerten. Gibt es keinen Befehl, steht die Baseline als „nicht gemessen", und jeder Befund ist Verdacht; läuft der Befehl über das Limit oder rot, gilt dasselbe mit dem Grund daneben. Prüfe nach dem Lauf `git status --porcelain`; was er hinterlassen hat, räumst du nicht auf.

Was du nicht gelesen hast, steht in der Architektur-Zusammenfassung unter „Nicht gelesen".

## 3. Hot Paths verfolgen

Wähle zwei bis drei Pfade — zuerst die, die der Messbefehl misst, sonst die Einstiege mit der größten Datenmenge oder Häufigkeit — und verfolge jeden vom Einstieg bis zum Ergebnis als Kette von `Datei:Zeile`, mit der Datenmenge je Station. Daran zeigt sich, welche Schleife über welche Menge läuft: Eine quadratische Suche über zehn Elemente ist kein Befund, über zehntausend die Hauptoperation. Was auf keinem dieser Pfade liegt, ist ein kalter Pfad.

## 4. Problemfelder sammeln

Lies jetzt `${CLAUDE_SKILL_DIR}/references/optimierungskategorien.md` ganz, bevor du den ersten Befund notierst — Teil A für die Befunde, Teil B für Abschnitt 6. Die Einleitung von Teil A regelt, was als gemessen und was als Verdacht gilt und was kein `P` wird. Jeder Befund wird ein `P<n>` mit Fundstelle, Befund, Beleg, Schwere und der Einstufung gemessen oder Verdacht. Gesucht wird in den Dateien der Hot Paths und ihrer direkten Aufrufer; kalte Pfade durchsuchst du nicht — was dir dort beim Lesen begegnet, wird ein Niedrig-`P` unter Nicht-Ziele.

## 5. Zielwerte setzen

Je Kennzahl, die ein Hot Path aus Abschnitt 3 bewegt, eine Zeile der Zielwerte: Baseline-Wert mit Streuung, Ziel, Messbefehl; übrige Ausgaben des Messbefehls übernimmst du nicht. Das Ziel ist eine Rückfrage aus Abschnitt 7 mit deinem Vorschlag relativ zur Baseline — „halbieren", „unter 200 ms" —, weil lokale Zahlen nicht die Produktion sind; ohne Antwort steht „messbare Verbesserung, Zielwert offen" als Annahme. Ohne Baseline trägt die Zeile „nicht gemessen" und den Messbefehl, den der Messschritt einrichtet. Zahlen stehen nur hier; Problemfelder und Schritte verweisen auf die Zeile.

## 6. Strategien wählen

Nach Teil B derselben Referenz: die kleinste Menge an Strategien, die jeden Befund löst, dem die Schwere-Skala aus Teil A eine Strategie zuweist.

## 7. Rückfragen stellen

Stell die Fragen, die den Plan verändern würden — gebündelt in einer Nachricht, höchstens fünf, jede mit deinem Vorschlag und einer kurzen Begründung, damit ein „passt" als Antwort reicht: die Zielwerte je Kennzahl, die Priorität (Laufzeit, Speicher oder Skalierung zuerst), die Grenzen (Ergebnisformat, Reihenfolge, Fließkomma-Genauigkeit, Schnittstellen, die unverändert bleiben müssen), Bereiche, die nicht angefasst werden dürfen, und das Budget an Schritten. Frag **nicht** nach dem, was im Code steht oder die Messung zeigt — das liest und misst du selbst.

Bleibt eine Frage offen, plane mit deiner Annahme weiter, markiere sie im Plan als Annahme und trag die Frage in `## Offene Fragen` ein.

## 8. Ablageort bestimmen

1. Liegen schon Pläne im Projekt (`docs/plans/`, `.claude/plans/`, `plans/`)? Dann dorthin, im dort üblichen Namensschema.
2. Sonst `docs/plans/` anlegen.
3. Dateiname `optimierung-<bereich>.md` in Kebab-Case; ohne Bereich `optimierung-<projektname>.md` mit dem Namen aus Manifest oder Verzeichnis; bei `english` entsprechend `optimization-<scope>.md`.
4. Existiert die Datei bereits → nicht überschreiben. Nenne sie und frag, ob ergänzen oder neuer Name.

## 9. Plan schreiben

Die vollständige Struktur steht in `${CLAUDE_SKILL_DIR}/references/optimierungs-vorlage.md`; Struktur und Reihenfolge sind verbindlich. Der Kopfblock trägt `Optimierung`, `Bereich`, `Erstellt`, `Claude-Session: ${CLAUDE_SESSION_ID}` und `Status: Entwurf`; Schrittübersicht und Parallel-Übersicht stehen ganz oben. Die Datei schreibst du einmal, nachdem Schritte, Wellen und Gegenlesen aus den Abschnitten 10 bis 12 stehen. Die Vorlage ist deutsch; bei einer anderen Ausgabesprache übersetzt du ihre Beschriftungen beim Übernehmen, nicht erst beim Gegenlesen.

## 10. Schritte schneiden

Ein Schritt hat ein Anliegen und ein überprüfbares Ergebnis und nennt Ergebnis, betroffene Dateien, Abhängigkeiten, Verhaltensschutz, Messung und Akzeptanz. Für eine Optimierung kommt dazu: Ohne Baseline richtet Schritt 1 die Messung ein, vor allem anderen; fehlt einem Hot Path ein Test, geht ihm ein Charakterisierungstest voraus; ein Schritt ist eine Optimierung auf einem Pfad, und die drei Regeln aus der Einleitung von Teil B gelten je Schritt — Struktur nie mit Verhalten, Nachher-Messung mit demselben Befehl wie die Baseline. Der Punkt `Messung` verweist auf die Zeile der Zielwerte, statt Zahlen zu wiederholen; bei Mess- und Testschritten lautet er „keine — <Grund>". Nach jedem Schritt laufen Build und Tests im Stand der Baseline, also ohne neue rote Tests. Zu groß ist ein Schritt, wenn er selbst wieder einen Plan bräuchte; zu klein, wenn sein Ergebnis für sich genommen keine Kennzahl bewegt.

## 11. Parallelität bestimmen

Zwei Schritte dürfen nur dann in dieselbe Welle, wenn beides gilt: keine fachliche Abhängigkeit und keine gemeinsam geschriebenen Dateien. Schritte auf demselben Hot Path laufen nie parallel, weil sich sonst keine Messung einem Schritt zuordnen lässt; Mess- und Charakterisierungstest-Schritte verschiedener Pfade dürfen gemeinsam in W1. Bilde Wellen: W1 sind alle Schritte ohne Vorbedingung, W2 alles, was nur von W1 abhängt, und so weiter; nenne je Welle die Berührungspunkte. Ergibt sich keine echte Parallelität, schreib genau das hin.

## 12. Gegenlesen

Prüfe den verfassten Text, als hättest du ihn nicht selbst geschrieben — am Text, nicht durch erneutes Lesen der Datei:

- Deckt die Summe der Schritte das Ziel ab, und nichts darüber hinaus?
- Stimmen die Abhängigkeiten in Übersicht, Wellen und Schritten überein?
- Ändert ein Schritt beobachtbares Verhalten?
- Hat jeder Schritt den Punkt `Messung`, und verweist er auf einem Hot Path auf eine Zeile der Zielwerte? Ist jeder Vorher-Wert gemessen, nicht geschätzt?
- Hat jeder Cache Invalidierungspfad und Größenbegrenzung? Hat jeder Hoch-Befund eine Strategie, und liegt keine Strategie auf einem kalten Pfad?
- Hat jedes `P` eine Strategie oder ein Nicht-Ziel, und hat jede Strategie mindestens einen Schritt? Stehen Zahlen nur in den Zielwerten?
- Steht unter „Nicht gelesen", was ungelesen blieb? Ist jede erfundene Angabe raus, jede Annahme als Annahme markiert, und steht alles in der Ausgabesprache?

Korrigiere, was du findest, und prüfe erneut, bis ein Durchgang nichts Neues ergibt. Für die gründliche Prüfung verweise danach auf `/plan-lint` und `/plan-review`.

## 13. Übergeben

Nenne den Pfad der Datei, die Zahl der `P` je Kategorie mit der Einstufung gemessen oder Verdacht, die Zielwerte in einer Zeile (Kennzahl: Baseline → Ziel), die Zahl der Schritte, die Wellen in einer Zeile, die offenen Fragen, was ungelesen blieb und was Test- und Messlauf im Arbeitsverzeichnis hinterlassen haben (aus dem `git status --porcelain` nach den Läufen). Kein Wiederkäuen des Inhalts — der Plan steht in der Datei.

## 14. Haltung

Die Versuchungen dieses Skills: ohne Messung optimieren; einem einzelnen Lauf trauen; kalte Pfade polieren; einen Cache ohne Invalidierung einplanen; Reihenfolge oder Fließkomma-Ergebnis nebenbei ändern; Mikro-Benchmarks für Makro-Aussagen nehmen; Zielwerte erfinden, statt zu fragen; Fehler nebenbei beheben; einen Befund ohne `Datei:Zeile` notieren; Vollständigkeit behaupten, wo Stichproben gelesen wurden. Ein Plan mit zwei gemessenen Problemfeldern und einer offenen Frage ist besser als einer mit zwölf Verdachtsfällen und keiner.

Und bevor du absendest: Stehen Antwort und Plandatei in der Ausgabesprache? Die deutsche Anweisung und die deutsche Vorlage ziehen sonst ins Deutsche, auch wenn oben etwas anderes verlangt ist.
