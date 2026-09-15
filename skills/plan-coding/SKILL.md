---
name: plan-implement
description: Setzt einen einzelnen Schritt aus einem Feature-Plan um — erst Entwurf im Sinne eines Systemarchitekten, dann die minimale produktionsreife Implementierung, danach Findings-Runden bis nichts Neues mehr auftaucht, und zum Schluss die Fortschreibung des Plans. Nimmt Planname und Schritt als Argumente.
argument-hint: "[plan-datei] [schritt]"
arguments: plan schritt
disable-model-invocation: true
allowed-tools: Read Grep Glob Edit Write Bash
---

**Ausgabesprache: die Sprache des Plans.** Entwurf, Berichte, Rückfragen, Abbruchmeldungen und Historie schreibst du in der Sprache, in der der Plan verfasst ist — bis der Plan gelesen ist, auf Deutsch. Code, Bezeichner, Kommentare und Tests folgen dem Bestand, nicht dieser Vorgabe. Dass diese Anweisung deutsch ist, ändert daran nichts.

Setze Schritt `$schritt` aus dem Plan `$plan` um. Denke wie ein erfahrener Systemarchitekt: entwirf tragfähig für Wachstum, baue dann die minimale Version, die produktiv laufen kann. Nicht die ganze Zukunft, aber auch keinen Entwurf, der beim ersten Lastanstieg umgeworfen werden muss.

Du setzt genau diesen einen Schritt um. Nicht den nächsten, nicht die Hälfte des übernächsten.

## 1. Plan und Schritt auflösen

**Plan.** `$plan` kann ein Pfad, ein Dateiname oder ein Namensfragment ohne Endung sein.

1. Existiert `$plan` als Pfad → nimm ihn.
2. Sonst per Glob `**/*$plan*.md`, bevorzugt in `docs/plans/`, `.claude/plans/`, `plans/`, `docs/`, Projektwurzel.
3. Mehrere Treffer → auflisten und abbrechen, statt zu raten.

**Schritt.** `$schritt` kann eine Nummer, eine Überschrift oder ein Fragment davon sein. Kein Argument übergeben → nenne die noch offenen Schritte und frage, welcher gemeint ist. Mehrdeutig → dasselbe.

**Sprache.** Die Ausgabesprache ist die Sprache des Plans. Kopfblock und Überschriften entscheiden — nicht Zitate, Bezeichner oder Codeblöcke, die in einem deutschen Plan oft englisch sind. Die Sprache ändert den Text, nicht die Struktur: Das Berichtsformat bleibt, nur seine Beschriftungen werden übersetzt.

**Historie.** Wo diese Anweisung `## Umsetzungs-Historie` oder `## Review-Historie` sagt, ist der Abschnitt dieser Bedeutung gemeint, gleich in welcher Sprache seine Überschrift steht — ein Plan kann aus `/plan-create` in einer anderen Sprache stammen. Einen vorhandenen erkennst du auch in Übersetzung und schreibst ihn fort, einen neuen legst du in der Plansprache an. Die Finding-Kennung F bleibt unübersetzt.

Lies den ganzen Plan, nicht nur den Schritt. Ein Schritt, der isoliert gelesen wird, wird isoliert falsch umgesetzt. Prüfe in der `## Umsetzungs-Historie` und in einer eventuellen `## Review-Historie`, was frühere Schritte bereits verändert oder als Erkenntnis hinterlassen haben.

## 2. Entwurf vor Code

Bevor du eine Zeile schreibst, halte den Entwurf für diesen Schritt fest — kurz, in Prosa, keine Tapete. Die Dimensionen stehen in `${CLAUDE_SKILL_DIR}/references/entwurfsdimensionen.md`; behandle davon nur, was dieser Schritt tatsächlich berührt:

Architektur und Einordnung · Komponentenschnitt · Datenfluss · Schnittstellen und API · Datenmodell und Migration · Caching und Invalidierung · Verifikationsweg

Zwei Leitplanken, die dabei gegeneinander arbeiten und beide gelten:

- **Tragfähig entwerfen.** Schnittstellen, Datenmodell und Zuständigkeiten so schneiden, dass die zehnfache Last oder der zweite Anwendungsfall keine Neuschreibung erzwingt.
- **Minimal bauen.** Keine Abstraktion ohne zweiten Aufrufer, kein Cache ohne belegten Bedarf, keine Konfigurierbarkeit ohne zweiten Wert. Was der Entwurf vorsieht, aber dieser Schritt nicht braucht, notierst du als Folgeschritt statt es zu bauen.

Weicht der Entwurf vom Plan ab, sag das jetzt und begründe es. Trägt der Plan an dieser Stelle gar nicht, brich ab und melde das, statt still etwas anderes zu bauen.

## 3. Umsetzen

- Folge den Mustern des Bestands. Ein zweites, konkurrierendes Muster ist teurer als ein unschönes, das zum Rest passt.
- Fehlerfälle gehören in denselben Durchgang wie der Happy Path, nicht in eine spätere Runde.
- Kein Scope-Zuwachs. Was dir unterwegs auffällt und nicht zu diesem Schritt gehört, wandert in "Erkenntnisse für spätere Schritte".
- Tests für das, was dieser Schritt behauptet zu können — auf der Ebene, die das Projekt üblicherweise nutzt.

## 4. Findings-Runden

Nach der Umsetzung prüfst du deine eigene Arbeit in Runden. Eine Runde besteht aus: verifizieren → Findings sammeln → beheben → erneut verifizieren.

**Verifizieren heißt ausführen.** Build, Tests, Linter, Typprüfung — die Befehle, die das Projekt vorsieht. Ein Finding-Durchgang, der nur liest, findet die Hälfte. Läuft etwas nicht ausführbar, sag das und benenne, was dadurch ungeprüft bleibt.

Die Finding-Kategorien stehen in `${CLAUDE_SKILL_DIR}/references/entwurfsdimensionen.md`. Jedes Finding bekommt eine stabile ID (F1, F2, …), eine Fundstelle `Datei:Zeile` und eine Einstufung:

- **Blocker** — falsches, unsicheres oder nicht lauffähiges Verhalten.
- **Sollte** — trägt, aber weicht vom Plan ab, lässt einen Fehlerfall offen oder bricht mit dem Bestand.
- **Optional** — Geschmack, kein Risiko.

Blocker und Sollte behebst du in derselben Runde. Optional-Findings sammelst du und legst sie am Ende vor, ohne sie eigenmächtig umzusetzen.

**Terminierung:** Die Schleife endet, wenn eine vollständige Runde keinen neuen Blocker und kein neues Sollte-Finding mehr hervorbringt und die Verifikation grün ist. Sag das dann klar und hör auf zu suchen.

Damit die Schleife auch wirklich konvergiert:

- Ab Runde 2 keine neuen Optional-Findings, außer sie sind durch eine Korrektur entstanden.
- Ein Finding, das du bewusst nicht behebst, wird mit Begründung abgelehnt und nicht in der nächsten Runde neu aufgemacht.
- Nach drei Runden ohne Fortschritt hörst du auf und benennst, woran es hängt: fehlende Information, ein Plan, der nicht trägt, oder ein Entwurf, der die falsche Grundannahme hat. Das entscheidet der Mensch.

Bericht je Runde:

```
# Umsetzung: <Plan> — Schritt <X> — Runde <N>

**Status:** Findings offen | Abgeschlossen
**Geändert:** <Dateien>
**Verifikation:** <Befehl> → <Ergebnis>

## Findings dieser Runde
### F1 — <Titel>  [Blocker | Sollte | Optional]
- **Fundstelle:** <Datei:Zeile>
- **Befund:** <was nicht stimmt>
- **Behandlung:** behoben in <Datei> | abgelehnt: <Begründung> | offen

## Abweichungen vom Plan
- <Abweichung> → <Begründung>

## Erkenntnisse für spätere Schritte
- <was der Plan an anderer Stelle jetzt anders sehen muss>
```

## 5. Plan fortschreiben

Erst wenn die Findings-Schleife terminiert ist, fasst du den Plan an. Lege den Abschnitt `## Umsetzungs-Historie` am Ende der Datei an, falls er fehlt:

```
## Umsetzungs-Historie
### Schritt 3 — <Datum> — abgeschlossen in 2 Runden
- Geändert: src/billing/invoice.ts, migrations/0042_add_status.sql
- Abweichung: Statusfeld als Enum statt String — bestehendes Muster in order.ts
- F2 Fehlender Index auf invoice.status — behoben
- F4 Doppelte Serialisierung im Export — abgelehnt: betrifft Schritt 5
- Erkenntnis: Schritt 5 kann den alten Export-Pfad nicht mehr voraussetzen
```

In einem fremdsprachigen Plan trägt der neue Abschnitt die übersetzte Überschrift — `## Implementation History` — und seine Einträge folgen der Plansprache; die Vorlage hier ist deutsch, weil diese Anweisung es ist, nicht weil der Abschnitt es sein müsste.

Markiere den Schritt im Plan als erledigt und trage die Erkenntnisse dort ein, wo sie hingehören — also in die späteren Schritte, die sie betreffen, nicht nur in die Historie.

**Danach prüfst du den Plan erneut**, und zwar nur die Schritte, die deine Umsetzung berührt hat: Stimmen ihre Annahmen noch? Sind Schritte überflüssig geworden oder ist ein neuer nötig? Dieselbe Terminierung wie oben — bringt ein Durchgang keine neuen Findings, ist der Plan aktuell und du hörst auf. Für ein vollständiges Review des überarbeiteten Plans verweise auf `/plan-review $plan`, statt es hier nachzubauen.

## 6. Haltung

Der gefährlichste Moment ist die zweite Runde, in der alles grün ist: dann ist die Versuchung groß, entweder Findings zu erfinden oder das Suchen einzustellen, bevor die Fehlerfälle geprüft sind. Beides ist ein schlechtes Ergebnis. Halte fest, was du ausgeführt hast und was dadurch belegt ist — daran misst sich die Runde, nicht an der Zahl der Findings.

Und wenn der Plan an dieser Stelle falsch liegt, ist das ein Ergebnis, kein Hindernis. Sag es, statt darum herumzubauen.

Und bevor du absendest: Steht die Antwort in der Sprache des Plans? Die deutsche Anweisung zieht sonst ins Deutsche, auch wenn der Plan englisch ist.
