---
name: plan-refactoring-create
description: Erstellt aus einer Codebase-Analyse ein Refactoring-Plandokument im Markdown-Format und legt es im Projekt ab — belegte Problemfelder mit Datei:Zeile, Zielarchitektur nach Clean-Architecture-Maßstäben mit neuer Ordnerstruktur und Schritte, die das Verhalten unverändert lassen; im Aufbau kompatibel zu plan-review, plan-lint und plan-coding. Nimmt optional einen Bereich (Pfad, Modul oder Fokus in Anführungszeichen) und die Ausgabesprache als Argumente; ohne Angabe die ganze Codebase und Deutsch.
argument-hint: "[bereich] [sprache]"
arguments: bereich sprache
disable-model-invocation: true
allowed-tools: Read Grep Glob Bash Write Edit
---

**Ausgabesprache: `$sprache`** — steht dort nichts oder ein Wort, das erkennbar keine Sprache benennt, Deutsch. Alles, was du an den Menschen richtest oder in die Plandatei schreibst, schreibst du in dieser Sprache: Plandokument, Rückfragen, Abbruchmeldungen, Übergabe. Dass diese Anweisung deutsch ist, ändert daran nichts.

Plane das Refactoring für den Bereich `$bereich`. Denke wie ein erfahrener Entwickler, der eine große, fremde Codebase übernimmt und in eine saubere Architektur überführt: Belange trennen, Modularität erhöhen, Kopplung senken. Das Verhalten bleibt, die Struktur wird besser. Erst verstehen — Architektur und Datenfluss —, dann belegen, dann planen. Das Ergebnis ist ein Dokument, kein Code: Der verbesserte Code entsteht Schritt für Schritt über `/plan-coding`.

## 1. Argumente auflösen

**Bereich.** `$bereich` ist ein Pfad (`src/billing`), ein Modulname oder ein Fokus in Anführungszeichen (`"Duplikate im Import"`). Leer heißt: die ganze Codebase. Sieht der Bereich wie ein Pfad aus, prüfst du ihn per Glob; gibt es ihn nicht, fragst du nach, statt stillschweigend alles zu analysieren.

**Sprache.** Die Ausgabesprache ist das letzte, optionale Argument — ein Sprachname oder Kürzel in beliebiger Schreibweise (`englisch`, `english`, `en`). Der Aufruf wird wie eine Shell-Kommandozeile zerlegt, deshalb gilt der vollständige Aufruf **$ARGUMENTS**: Benennt sein letztes Wort erkennbar eine Sprache, ist das die Ausgabesprache und alles davor der Bereich — auch wenn davor nichts steht. Sonst ist der ganze Aufruf der Bereich, und die Sprache bleibt Deutsch. Könnte das letzte Wort auch zum Bereich gehören („auf englisch"), rate nicht: nimm die Frage in die Rückfragen aus Abschnitt 7 auf und nimm bis zur Antwort Deutsch an.

Die Sprache ändert den Text, nicht die Struktur: Überschriften, Spaltentitel, Kopfblock-Beschriftungen und Prosa werden übersetzt; Schrittnummern, die Wellen-Kürzel `W1, W2 …`, die Kennungen `P1, P2 …` und der Schlüssel `Claude-Session` bleiben, weil `/plan-review`, `/plan-lint` und `/plan-coding` sie später referenzieren.

## 2. Orientieren

Eine große Codebase lässt sich nicht vollständig lesen, und ein Plan, der so tut, erfindet. Lies deshalb in dieser Reihenfolge und halte an, sobald ein Bild steht: (1) Verzeichnisbaum bis Tiefe zwei, Manifeste, Build- und Testbefehle, `CLAUDE.md`, `CONTEXT.md`, ADRs — Terminologie und getroffene Entscheidungen sind bindend; (2) Einstiegspunkte: main, Routen, CLI, Handler, Jobs; (3) Hotspots: die dreißig am häufigsten geänderten Dateien aus `git log --format= --name-only -n 500 | sort | uniq -c | sort -rn | head -30` und die dreißig größten aus `git ls-files -- <bereich> | xargs wc -l | sort -rn | head -30`; (4) die Datenflüsse aus Abschnitt 3; (5) die Greps aus Abschnitt 4. Bash nutzt du nur lesend; die einzige Ausnahme ist der Baseline-Lauf unten. Ist das Verzeichnis kein Git-Repository, entfallen die Hotspots, und die Wartbarkeitsrisiken stützen sich auf Größe und Tests.

Die Baseline ist der Stand, gegen den jeder Schritt später gemessen wird. Führe den dokumentierten Test- oder Buildbefehl — aus `CLAUDE.md`, README, Manifest-Skripten oder Makefile — genau einmal aus, mit einem Zeitlimit von fünf Minuten und ohne Abhängigkeiten zu installieren, und notiere Befehl und Ergebnis unter „Build, Tests und Baseline": grün, rot mit den roten Tests, damit kein Schritt sie erbt, oder „unbekannt", wenn kein Befehl dokumentiert ist. Läuft er über das Limit, brich ab, notiere das als Ergebnis und starte keinen zweiten Versuch.

Was du nicht gelesen hast, steht in der Architektur-Zusammenfassung unter „Nicht gelesen".

## 3. Datenfluss verfolgen

Wähle zwei bis drei zentrale Flüsse und verfolge jeden vom Einstieg bis zur Persistenz, als Kette von `Datei:Zeile`. Daran zeigen sich Schichtverletzungen und vermischte Belange zuverlässiger als an jeder Verzeichnisstruktur.

## 4. Problemfelder sammeln

Lies jetzt `${CLAUDE_SKILL_DIR}/references/problemkategorien.md` ganz, bevor du den ersten Befund notierst — Teil A für die Befunde, Teil B für Abschnitt 5. Die Einleitung von Teil A regelt, was als Beleg zählt, woher die Kennzahlen kommen und was kein `P` wird. Jeder Befund wird ein `P<n>` mit Fundstelle, Befund, Beleg und Schwere.

## 5. Strategien wählen

Nach Teil B derselben Referenz: die kleinste Menge an Strategien, die jeden Befund löst, dem die Schwere-Skala aus Teil A eine Strategie zuweist.

## 6. Zielarchitektur ableiten

Aus Architektur-Zusammenfassung und Problemfeldern leitest du die vier Teile der Zielarchitektur aus der Vorlage ab: die Schichten mit Abhängigkeitsrichtung nach innen zur Domäne und benannten, begründeten Ausnahmen, wo das Framework sie erzwingt; die Zuständigkeit je Modul; die Ordnerstruktur vorher und nachher; die Umzugstabelle mit dem Schritt je Zeile. Die Importzahlen für „nachher" zählst du am geplanten Graphen, ohne zweiten Grep. Die Struktur folgt den Konventionen des Ökosystems und des Frameworks: Wo das Framework die Ablage vorgibt, bleibt sie, und die Trennung geschieht innerhalb seiner Vorgaben.

## 7. Rückfragen stellen

Stell die Fragen, die den Plan verändern würden — gebündelt in einer Nachricht, höchstens fünf, jede mit deinem Vorschlag und einer kurzen Begründung, damit ein „passt" als Antwort reicht: die Zielarchitektur (Schichtenschnitt und Ordnerstruktur bestätigen), die Priorität (Struktur, Performance oder Wartbarkeit zuerst), die Grenzen (öffentliche API, Datenbankschema, Dateiformate, die unverändert bleiben müssen), Bereiche, die nicht angefasst werden dürfen, und das Budget an Schritten. Frag **nicht** nach dem, was im Code steht — das liest du selbst.

Bleibt eine Frage offen, plane mit deiner Annahme weiter, markiere sie im Plan als Annahme und trag die Frage in `## Offene Fragen` ein.

## 8. Ablageort bestimmen

1. Liegen schon Pläne im Projekt (`docs/plans/`, `.claude/plans/`, `plans/`)? Dann dorthin, im dort üblichen Namensschema.
2. Sonst `docs/plans/` anlegen.
3. Dateiname `refactoring-<bereich>.md` in Kebab-Case; ohne Bereich `refactoring-<projektname>.md` mit dem Namen aus Manifest oder Verzeichnis; bei `english` entsprechend `refactoring-<scope>.md`.
4. Existiert die Datei bereits → nicht überschreiben. Nenne sie und frag, ob ergänzen oder neuer Name.

## 9. Plan schreiben

Die vollständige Struktur steht in `${CLAUDE_SKILL_DIR}/references/refactoring-vorlage.md`; Struktur und Reihenfolge sind verbindlich. Der Kopfblock trägt `Refactoring`, `Bereich`, `Erstellt`, `Claude-Session: ${CLAUDE_SESSION_ID}` und `Status: Entwurf`; Schrittübersicht und Parallel-Übersicht stehen ganz oben. Die Datei schreibst du einmal, nachdem Schritte, Wellen und Gegenlesen aus den Abschnitten 10 bis 12 stehen. Die Vorlage ist deutsch; bei einer anderen Ausgabesprache übersetzt du ihre Beschriftungen beim Übernehmen, nicht erst beim Gegenlesen.

## 10. Schritte schneiden

Ein Schritt hat ein Anliegen und ein überprüfbares Ergebnis und nennt Ergebnis, betroffene Dateien, Abhängigkeiten, Verhaltensschutz und Akzeptanz. Für ein Refactoring kommt dazu: Ein Schritt ist eine Refactoring-Bewegung, und Struktur ändert sich nie im selben Schritt wie Verhalten — ein Umzug ändert deshalb nur Pfade, Namespaces beziehungsweise Packages und Importe, damit Git die Umbenennung erkennt und der Diff lesbar bleibt. Die Reihenfolge ist Charakterisierungstest vor Naht, Naht vor Verschiebung, Verschiebung vor Löschung. Nach jedem Schritt laufen Build und Tests im Stand der Baseline, also ohne neue rote Tests. Zu groß ist ein Schritt, wenn er selbst wieder einen Plan bräuchte; zu klein, wenn sein Ergebnis für sich genommen nichts bedeutet.

## 11. Parallelität bestimmen

Zwei Schritte dürfen nur dann in dieselbe Welle, wenn beides gilt: keine fachliche Abhängigkeit und keine gemeinsam geschriebenen Dateien. Schritte am selben Modul laufen nie parallel; Charakterisierungstests verschiedener Bereiche dürfen gemeinsam in W1. Bilde Wellen: W1 sind alle Schritte ohne Vorbedingung, W2 alles, was nur von W1 abhängt, und so weiter; nenne je Welle die Berührungspunkte. Ergibt sich keine echte Parallelität, schreib genau das hin.

## 12. Gegenlesen

Prüfe den verfassten Text, als hättest du ihn nicht selbst geschrieben — am Text, nicht durch erneutes Lesen der Datei:

- Deckt die Summe der Schritte das Ziel ab, und nichts darüber hinaus?
- Stimmen die Abhängigkeiten in Übersicht, Wellen und Schritten überein?
- Ändert ein Schritt beobachtbares Verhalten?
- Zeigen die Abhängigkeiten der Zielarchitektur nach innen, oder ist jede Ausnahme dort benannt und begründet?
- Hat jede Zeile der Umzugstabelle einen Schritt? Hat jedes `P` eine Strategie oder ein Nicht-Ziel, und hat jede Strategie mindestens einen Schritt?
- Steht unter „Nicht gelesen", was ungelesen blieb? Ist jede erfundene Angabe raus, jede Annahme als Annahme markiert, und steht alles in der Ausgabesprache?

Korrigiere, was du findest, und prüfe erneut, bis ein Durchgang nichts Neues ergibt. Für die gründliche Prüfung verweise danach auf `/plan-lint` und `/plan-review`.

## 13. Übergeben

Nenne den Pfad der Datei, die Zahl der `P` je Kategorie und Schwere, die Zielarchitektur in einem Satz (Schichten, Zahl der Umzüge), die Zahl der Schritte, die Wellen in einer Zeile, die offenen Fragen, was ungelesen blieb und was der Baseline-Lauf im Arbeitsverzeichnis hinterlassen hat (`git status --porcelain`). Kein Wiederkäuen des Inhalts — der Plan steht in der Datei.

## 14. Haltung

Die Versuchungen dieses Skills: neu schreiben statt refaktorieren; eine Schablonen-Architektur über ein Framework stülpen, das seine eigene Ablage hat; alles auf einmal umziehen; Fehler nebenbei beheben; Performance aus dem Bauch beurteilen; zufällige Ähnlichkeit als Duplikat zusammenführen; Geschmack als Problem ausgeben; einen Schritt planen, nach dem die Tests nicht laufen; einen Befund ohne `Datei:Zeile` notieren; Vollständigkeit behaupten, wo Stichproben gelesen wurden. Ein Plan mit drei belegten Problemfeldern und einer offenen Frage ist besser als einer mit zwölf vermuteten und keiner.

Und bevor du absendest: Stehen Antwort und Plandatei in der Ausgabesprache? Die deutsche Anweisung und die deutsche Vorlage ziehen sonst ins Deutsche, auch wenn oben etwas anderes verlangt ist.
