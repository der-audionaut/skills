---
name: plan-create
description: Erstellt aus einer Feature-Beschreibung ein Plandokument im Markdown-Format und legt es im Projekt ab — mit Session-ID, Schrittübersicht, Parallelisierungsübersicht und Entwurf für Architektur, Datenfluss, API, Schema und Caching. Nimmt die Feature-Beschreibung als Argument, optional gefolgt von der Ausgabesprache (Standard Deutsch).
argument-hint: "[feature-beschreibung] [sprache]"
arguments: feature sprache
disable-model-invocation: true
allowed-tools: Read Grep Glob Write Edit
---

**Ausgabesprache: `$sprache`** — steht dort nichts oder ein Wort, das erkennbar keine Sprache benennt, Deutsch. Alles, was du an den Menschen richtest oder in die Plandatei schreibst, schreibst du in dieser Sprache: Plandokument, Rückfragen, Abbruchmeldungen, Übergabe. Dass diese Anweisung deutsch ist, ändert daran nichts.

Plane das Feature aus diesem Aufruf: **$ARGUMENTS**

Denke wie ein erfahrener Systemarchitekt: entwirf so, dass Wachstum den Entwurf nicht umwirft, und schneide dann die kleinste produktionsreife Fassung in Schritte. Das Ergebnis ist ein Dokument, kein Code — du implementierst hier nichts.

## 1. Sprache und Feature auflösen

**Sprache.** Die Ausgabesprache ist das letzte, optionale Argument — ein Sprachname oder Kürzel in beliebiger Schreibweise (`englisch`, `english`, `en`). Ohne Angabe bleibt es bei Deutsch, auch wenn die Feature-Beschreibung oder die Codebase englisch ist; wer den Plan in einer anderen Sprache will, gibt sie an.

Die Sprache ändert den Text, nicht die Struktur: Aufbau und Reihenfolge der Vorlage bleiben, ebenso die Tabellen; übersetzt werden Überschriften, Spaltentitel, Beschriftungen und Prosa. Die Schrittnummern, die Wellen-Kürzel `W1, W2 …` und der Schlüssel `Claude-Session` im Kopfblock bleiben in jeder Sprache gleich, weil `/plan-review`, `/plan-lint` und `/plan-coding` sie später referenzieren.

**Feature.** Die Feature-Beschreibung ist Fließtext, der Aufruf wird aber wie eine Shell-Kommandozeile in Wörter zerlegt. Steht die Beschreibung in Anführungszeichen, ist die Zuordnung eindeutig — `$feature` ist die Beschreibung, `$sprache` die Sprache — und so ist der Aufruf gemeint: `/plan-create "Rechnungsexport als CSV" english`. Ohne Anführungszeichen enthält `$sprache` nur das zweite Wort der Beschreibung; dann zählt der vollständige Aufruf oben:

1. Benennt sein letztes Wort erkennbar eine Sprache, ist das die Ausgabesprache und alles davor die Beschreibung.
2. Sonst ist der ganze Aufruf die Beschreibung, und die Sprache bleibt Deutsch.
3. Könnte das letzte Wort auch zur Beschreibung gehören („Login-Seite auf englisch"), rate nicht: nimm die Frage in die Rückfragen aus Abschnitt 3 auf und nimm bis zur Antwort Deutsch an.

## 2. Verstehen, bevor du planst

Erst die Codebase, dann der Plan. Ein Plan, der ohne Blick ins Projekt entsteht, erfindet Dateipfade und Muster.

- Wo im System sitzt das Feature? Welche Module, Endpunkte, Tabellen berührt es?
- Gibt es das schon halb? Suche per Grep nach benachbarter Funktionalität, bevor du Neubau planst.
- Welche Muster gelten hier — Schichtung, Fehlerbehandlung, Tests, Migrationen?
- Lies `CLAUDE.md`, `CONTEXT.md`, ADRs und vorhandene Pläne im Projekt. Terminologie und getroffene Entscheidungen sind bindend.

## 3. Rückfragen stellen

Stell die Fragen, die den Plan verändern würden — gebündelt in einer Nachricht, höchstens fünf, jede mit deinem Vorschlag und einer kurzen Begründung, damit ein "passt" als Antwort reicht.

Frag nach bei: Produktentscheidungen, Umfang und Abgrenzung, Verhalten in Sonderfällen, Kompatibilitätszusagen, Prioritäten. Frag **nicht** nach dem, was im Code steht — das liest du selbst.

Bleibt eine Frage offen, plane mit deiner Annahme weiter, markiere sie im Plan als Annahme und trag die Frage in `## Offene Fragen` ein. Ein Plan, der auf eine Antwort wartet, ist wertlos; eine unmarkierte Annahme ist gefährlich.

## 4. Ablageort bestimmen

1. Liegen schon Pläne im Projekt (`docs/plans/`, `.claude/plans/`, `plans/`)? Dann dorthin, im dort üblichen Namensschema.
2. Sonst `docs/plans/` anlegen.
3. Dateiname aus der Feature-Beschreibung als Kebab-Case, kurz und sprechend, in der Ausgabesprache: `rechnungsexport-csv.md`, bei `english` `invoice-export-csv.md` — es sei denn, das Projekt gibt ein anderes Namensschema vor.
4. Existiert die Datei bereits → nicht überschreiben. Nenne sie und frag, ob ergänzen oder neuer Name.

## 5. Plan schreiben

Die vollständige Struktur steht in `${CLAUDE_SKILL_DIR}/references/plan-vorlage.md`. Verbindlich sind:

- **Kopfblock** mit Feature, Datum, Status und `Claude-Session: ${CLAUDE_SESSION_ID}` — damit später nachvollziehbar ist, aus welcher Sitzung der Plan stammt.
- **Schrittübersicht ganz oben**, als Tabelle, vor allem Fließtext. Wer den Plan öffnet, sieht zuerst die Arbeitsschritte.
- **Parallel-Übersicht** direkt darunter, in Wellen.
- Danach Ziel, Nicht-Ziele, Ist-Zustand, Entwurf, die Schritte im Detail, Teststrategie, Rollout, Risiken, offene Fragen.

Zum Entwurf gehören Architektur und Einordnung, Komponentenschnitt, Datenfluss, Schnittstellen, Datenmodell und Migration sowie Caching — jeweils nur so weit, wie dieses Feature es berührt. Caching bekommt nur einen Abschnitt, wenn es einen belegten Bedarf gibt; sonst ein Satz, warum nicht. Was der Entwurf vorsieht, das Feature aber noch nicht braucht, gehört unter Nicht-Ziele, nicht in einen Schritt.

Die Vorlage ist deutsch. Bei einer anderen Ausgabesprache übersetzt du ihre Überschriften, Spaltentitel und Kopfblock-Beschriftungen beim Übernehmen — nicht erst beim Gegenlesen.

## 6. Schritte schneiden

- Ein Schritt hat **ein** Anliegen und ein überprüfbares Ergebnis. Endet die Beschreibung bei "implementieren", ist er nicht fertig gedacht.
- Nach jedem Schritt läuft das System. Kein kaputter Zwischenzustand.
- Jeder Schritt nennt: Ergebnis, betroffene Dateien, Abhängigkeiten, Akzeptanzkriterium.
- Migration vor Nutzung, Schnittstelle vor Verbraucher, Feature-Flag vor Rollout.
- Zu groß ist ein Schritt, wenn er selbst wieder einen Plan bräuchte. Zu klein, wenn sein Ergebnis für sich genommen nichts bedeutet.

## 7. Parallelität bestimmen

Zwei Schritte dürfen nur dann in dieselbe Welle, wenn **beides** gilt: keine fachliche Abhängigkeit, und keine gemeinsam geschriebenen Dateien. Gleichzeitig dieselbe Datei zu ändern ist kein Parallelisierungsgewinn, sondern ein Merge-Konflikt mit Umweg.

Typische Kollisionspunkte, die nie parallel laufen: Migrationen und Schemaänderungen, zentrale Konfiguration, Routen- und Container-Registrierung, gemeinsam genutzte Typen.

Bilde daraus Wellen: W1 sind alle Schritte ohne Vorbedingung, W2 alles, was nur von W1 abhängt, und so weiter. Nenne je Welle die Schritte und die Dateien, an denen sie sich berühren könnten. Ergibt sich keine echte Parallelität, schreib genau das hin — eine erfundene Welle kostet später mehr als eine ehrliche Kette.

## 8. Gegenlesen

Lies den geschriebenen Plan einmal vollständig, als hättest du ihn nicht selbst verfasst, und prüfe:

- Deckt die Summe der Schritte das Ziel ab, und nichts darüber hinaus?
- Stimmen die Abhängigkeiten in der Übersicht mit denen in den Schritten überein? Und mit den Wellen?
- Ist jede erfundene Angabe raus — Dateien, Felder, Funktionen, die du nicht belegt hast?
- Steht jede Annahme als Annahme da?
- Steht der Plan durchgängig in der Ausgabesprache — auch die Überschriften und Tabellenköpfe, die aus der deutschen Vorlage stammen?

Korrigiere, was du findest, und lies erneut. Schluss ist, wenn ein Durchgang nichts Neues ergibt. Für die gründliche Prüfung verweise danach auf `/plan-lint` und `/plan-review` — die baust du hier nicht nach.

## 9. Übergeben

Nenne den Pfad der Datei, die Anzahl der Schritte, die Wellen in einer Zeile und die offenen Fragen. Kein Wiederkäuen des Inhalts — der Plan steht ja in der Datei.

Ein Plan mit drei Schritten und einer offenen Frage ist ein besseres Ergebnis als einer mit zwölf Schritten und keiner. Wenn das Feature zu vage beschrieben ist, um Schritte zu schneiden, sag das, statt Struktur zu erfinden.

Und bevor du absendest: Stehen Antwort und Plandatei in der Ausgabesprache? Die deutsche Anweisung und die deutsche Vorlage ziehen sonst ins Deutsche, auch wenn oben etwas anderes verlangt ist.
