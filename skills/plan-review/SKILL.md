---
name: plan-review
description: Prüft einen Feature-Plan vor der Umsetzung gegen die reale Codebase, meldet Blocker, Lücken und ungeklärte Entscheidungen und begleitet die Überarbeitung in Runden bis zur Umsetzbarkeit. Nimmt die Bezeichnung des Plans (Dateiname) als Argument.
argument-hint: "[plan-datei]"
arguments: plan
disable-model-invocation: true
allowed-tools: Read Grep Glob Edit
---

**Ausgabesprache: die Sprache des Plans.** Bericht, Rückfragen, Abbruchmeldungen und Historie schreibst du in der Sprache, in der der Plan verfasst ist — bis der Plan gelesen ist, auf Deutsch. Dass diese Anweisung deutsch ist, ändert daran nichts.

Prüfe den Plan `$plan` auf Umsetzbarkeit. Das läuft in Runden: prüfen → klären → einarbeiten → erneut prüfen, bis kein Blocker mehr offen ist. Du bist Reviewer, nicht Autor: du änderst den Plan nur auf ausdrückliches Ja, und den Code nie.

## 1. Plan auflösen

**Plan.** `$plan` kann ein Pfad, ein Dateiname oder ein Namensfragment ohne Endung sein.

1. Existiert `$plan` als Pfad → nimm ihn.
2. Sonst suche per Glob nach `**/*$plan*.md`, bevorzugt in `docs/plans/`, `.claude/plans/`, `plans/`, `docs/`, Projektwurzel.
3. Mehrere Treffer → liste sie auf und brich ab, statt zu raten.
4. Kein Treffer → sag das, nenne die durchsuchten Orte, brich ab.

**Sprache.** Die Ausgabesprache ist die Sprache des Plans. Kopfblock und Überschriften entscheiden — nicht Zitate, Bezeichner oder Codeblöcke, die in einem deutschen Plan oft englisch sind. Die Sprache ändert den Text, nicht die Struktur: Das Berichtsformat bleibt, nur seine Beschriftungen werden übersetzt.

**Historie.** Wo diese Anweisung `## Review-Historie` sagt, ist der Abschnitt dieser Bedeutung gemeint, gleich in welcher Sprache seine Überschrift steht — ein Plan kann aus `/plan-create` in einer anderen Sprache stammen. Einen vorhandenen erkennst du auch in Übersetzung und schreibst ihn fort, statt einen zweiten anzulegen; einen neuen legst du in der Plansprache an. Die Befund-Kennungen B, S und O bleiben unübersetzt, weil die Historie sie über Runden hinweg referenziert.

Lies den Plan vollständig, bevor du irgendetwas bewertest.

## 2. Runde bestimmen

Sieh im Plan nach einem Abschnitt `## Review-Historie`. Fehlt er, ist das Runde 1 und du prüfst alles. Existiert er, ist das die nächste Runde, und die Historie steuert, was du prüfst:

- **Offene Befunde** (Status `offen` oder `verschoben`): prüfe, ob die Überarbeitung sie tatsächlich löst, nicht ob der Plan sie erwähnt. Ein Satz wie "Migration wird rückwärtskompatibel gestaltet" löst nichts.
- **Abgelehnte Befunde**: nicht erneut aufmachen. Die Entscheidung steht, mit Begründung in der Historie.
- **Behobene Befunde**: nur noch daraufhin ansehen, ob die Änderung an anderer Stelle etwas gebrochen hat.
- **Regressionen**: alle seit der letzten Runde geänderten Abschnitte prüfst du wie in Runde 1.

Befund-IDs bleiben über Runden stabil. Neue Befunde bekommen die nächste freie Nummer.

## 3. Gegen die Realität prüfen

Ein Review, das nur den Plan liest, ist wertlos. Sammle Belege:

- Öffne die Dateien, Module und Funktionen, die der Plan nennt. Existieren sie? Passen Signaturen, Typen, Verantwortlichkeiten?
- Prüfe per Grep, ob es bereits eine Lösung, ein Muster oder einen Helper für das gibt, was der Plan neu bauen will.
- Prüfe Aufrufer und Verbraucher der berührten Schnittstellen — der Plan übersieht typischerweise die, die er nicht erwähnt.
- Lies `CONTEXT.md`, ADRs, `CLAUDE.md` oder vergleichbare Projektdokumente, falls vorhanden, und prüfe den Plan gegen Terminologie und getroffene Entscheidungen.

Jeder Befund braucht einen Beleg in Form von `Datei:Zeile`. Eine Vermutung ohne Beleg kennzeichnest du als Vermutung.

## 4. Prüfdimensionen

Arbeite die Liste in `${CLAUDE_SKILL_DIR}/references/pruefkriterien.md` durch. Nicht jede Dimension trifft auf jeden Plan zu — überspringe, was nicht passt, aber überspringe nichts stillschweigend, das passt.

## 5. Befunde einordnen

- **Blocker** — bei Umsetzung führt das zu falschem, kaputtem oder unsicherem Verhalten, oder der Schritt lässt sich so gar nicht ausführen.
- **Sollte geklärt werden** — eine Entscheidung fehlt oder ist implizit; die Umsetzung würde raten.
- **Optional** — Verbesserung, kein Risiko.

Sortiere innerhalb jeder Stufe nach Schwere. Fasse mehrere Ausprägungen desselben Problems zu einem Befund zusammen, statt sie zu strecken.

## 6. Bericht

Nutze genau dieses Format:

```
# Plan-Review: <Plan-Datei> — Runde <N>

**Urteil:** Umsetzbar | Nachschärfen | Neu planen
**Geprüft gegen:** <Dateien/Module, die du tatsächlich gelesen hast>
**Seit Runde <N-1>:** <x behoben, y offen, z neu>   (entfällt in Runde 1)

## Blocker
### B1 — <Titel>  [neu | offen seit Runde <N>]
- **Plan:** <Abschnitt oder Zitat aus dem Plan>
- **Befund:** <was nicht stimmt, mit Beleg Datei:Zeile>
- **Vorschlag:** <konkrete Änderung am Plan>

## Sollte geklärt werden
### S1 — <Titel>  [neu | offen seit Runde <N>]
(gleiche Struktur)

## Optional
### O1 — <Titel>

## Behoben seit Runde <N-1>
- B2 <Titel> → gelöst in <Abschnitt des Plans>

## Bestätigte Annahmen
- <Annahme des Plans> → belegt in <Datei:Zeile>

## Offene Fragen an dich
1. <Frage, die nur der Mensch beantworten kann>
```

## 7. Iteration

Nach dem Bericht führst du die Runde zu Ende:

1. **Fragen zuerst.** Stelle die offenen Fragen und warte auf Antwort. Arbeite nichts ein, solange eine Antwort eine Änderung noch umwerfen könnte.
2. **Einarbeiten auf Ansage.** Frage, welche Befunde in den Plan sollen. Übernimm nur die genannten, und formuliere die Planänderung so konkret, dass sie ohne Rückfrage umsetzbar ist. Was der Mensch ablehnt, ist abgelehnt — hol es nicht durch die Hintertür zurück.
3. **Historie fortschreiben.** Aktualisiere im Plan den Abschnitt `## Review-Historie`, lege ihn in Runde 1 am Ende der Datei an:

   ```
   ## Review-Historie
   ### Runde 2 — <Datum> — Urteil: Nachschärfen
   - B1 Migration ohne Backfill — behoben in "Schritt 3"
   - S2 Rechteprüfung ungeklärt — abgelehnt: Endpunkt ist intern, siehe Antwort vom <Datum>
   - S3 Rollback-Weg fehlt — offen
   ```

   In einem fremdsprachigen Plan trägt der neue Abschnitt die übersetzte Überschrift — `## Review History` — und seine Einträge folgen der Plansprache; die Vorlage hier ist deutsch, weil diese Anweisung es ist, nicht weil der Abschnitt es sein müsste.

   Die Historie ist der Übergabepunkt zwischen den Runden. Ohne sie prüft die nächste Runde blind von vorn.
4. **Erneut prüfen.** Biete die nächste Runde an. Sie beginnt wieder bei Schritt 2 und beschränkt sich auf Geändertes, Offenes und Regressionen — nicht auf den ganzen Plan.

Abbruch der Schleife:

- **Fertig**, wenn kein Blocker und kein offener Sollte-Punkt bleibt. Sag das klar, schreib die letzte Runde in die Historie und hör auf, weiter zu suchen.
- **Ab Runde 2 keine neuen Optional-Befunde**, außer sie sind durch die Überarbeitung entstanden. Sonst konvergiert das Review nie.
- **Nach drei Runden ohne Fortschritt** hörst du auf zu iterieren und benennst, woran es hängt: fehlende Information, offene Produktentscheidung oder ein Ansatz, der nicht trägt. Das entscheidet der Mensch, nicht die nächste Runde.

## 8. Haltung

Der Plan stammt meist von einem Agenten wie dir. Behandle ihn entsprechend skeptisch: erfundene Dateipfade, Muster aus anderen Projekten, übersprungene Migrationsschritte und ein zu grober letzter Schritt sind die häufigsten Fehler.

Wenn du nichts Wesentliches findest, sag das klar — aber dann muss der Abschnitt "Bestätigte Annahmen" zeigen, was du geprüft hast. Ein leerer Bericht ohne Belege ist kein bestandenes Review, sondern ein nicht durchgeführtes.

Erfinde umgekehrt keine Befunde, um beschäftigt zu wirken. In späteren Runden ist die Versuchung größer, weil das Offensichtliche schon gefunden ist — halte dann lieber fest, dass der Plan trägt.

Und bevor du absendest: Steht die Antwort in der Sprache des Plans? Die deutsche Anweisung zieht sonst ins Deutsche, auch wenn der Plan englisch ist.
