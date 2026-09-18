---
name: plan-implement
description: Setzt einen einzelnen Schritt aus einem Feature-Plan um — erst Entwurf im Sinne eines Systemarchitekten, dann die minimale produktionsreife Implementierung, danach Findings-Runden bis nichts Neues mehr auftaucht, dann der Commit des Schritts mit einer Botschaft aus `write-commit-message`, wenn der Skill verfügbar ist — nicht der Plandatei, nicht auf dem Default-Branch, nicht bei roter Verifikation, nicht über fremde Änderungen hinweg, und nie ein Push — und zum Schluss die Fortschreibung des Plans. Nimmt Planname und Schritt als Argumente.
argument-hint: "[plan-datei] [schritt]"
arguments: plan schritt
disable-model-invocation: true
allowed-tools: Read Grep Glob Edit Write Bash Skill
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

**Ausgangslage.** Bevor du eine Datei anfasst, hältst du fest, wie du das Repository vorgefunden hast — später ist das die einzige Quelle, aus der sich Fremdes von Eigenem unterscheiden lässt, und Abschnitt 5 braucht jeden dieser Werte:

- Der ausgecheckte Branch: `git symbolic-ref --quiet --short HEAD` — leer bei losgelöstem HEAD.
- Die Pfade, die jetzt schon geändert, gestaged oder unversioniert sind: `git status --porcelain=v1 -uall`.
- Ob der Index leer ist: `git diff --cached --quiet`.
- Ob eine Operation läuft: `.git/MERGE_HEAD`, `.git/rebase-merge`, `.git/rebase-apply` oder `.git/CHERRY_PICK_HEAD` vorhanden.
- Der Default-Branch: Das Remote ist das des Upstreams (`git config branch.<branch>.remote`), sonst `origin`; dann `git symbolic-ref --quiet --short refs/remotes/<remote>/HEAD` ohne das Präfix `<remote>/`. Fehlt der Verweis, gilt `main` oder `master`, sofern lokal vorhanden; gibt es auch die nicht, ist der Default-Branch unbekannt, und Abschnitt 5 fragt nach.

Ist das Verzeichnis kein Git-Repository, notierst du das: Abschnitt 5 entfällt dann mit diesem Grund, alles andere läuft wie gewohnt.

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

Bericht je Runde — den der Schlussrunde gibst du erst nach Abschnitt 5 aus, damit er die Zeile `Commit:` tragen kann; Zwischenrunden berichten sofort:

```
# Umsetzung: <Plan> — Schritt <X> — Runde <N>

**Status:** Findings offen | Abgeschlossen
**Geändert:** <Dateien>
**Verifikation:** <Befehl> → <Ergebnis>
**Commit:** <sha> „<Betreff>" — Rückweg `git reset --soft HEAD~1` | keiner — <Grund> | noch nicht

## Findings dieser Runde
### F1 — <Titel>  [Blocker | Sollte | Optional]
- **Fundstelle:** <Datei:Zeile>
- **Befund:** <was nicht stimmt>
- **Behandlung:** behoben in <Datei> | abgelehnt: <Begründung> | offen

## Nicht zugeordnet
- <Pfad> — neu seit Start, nicht gestaged: <Vermutung, woher er stammt>

## Abweichungen vom Plan
- <Abweichung> → <Begründung>

## Erkenntnisse für spätere Schritte
- <was der Plan an anderer Stelle jetzt anders sehen muss>
```

Der Abschnitt „Nicht zugeordnet" entfällt, wenn er leer wäre.

## 5. Committen

Ist die Schleife grün terminiert, committest du den Schritt — nur ihn, nur auf dem Branch, den du vorgefunden hast, und nie mit einem Push. Der Commit ist Ergebnis, nicht Bedingung: Jede Vorbedingung, die nicht gilt, ist ein Grund „kein Commit", nie ein Grund abzubrechen. Bericht und Plan werden in jedem Fall geschrieben; der Grund steht in der Zeile `Commit:`. Ist das Verzeichnis kein Git-Repository, ist genau das der Grund, und der Rest dieses Abschnitts entfällt.

**Vorbedingungen.** Alle fünf müssen gelten:

- Die Schleife ist grün terminiert — nicht nach drei Runden ohne Fortschritt abgebrochen, Verifikation grün. Einen roten Stand committest du nicht, auch nicht „um nichts zu verlieren": Im Arbeitsverzeichnis ist er sicher, in der Historie wäre er eine Behauptung, die nicht stimmt.
- Ein Branch ist ausgecheckt, und es ist nicht der Default-Branch. Auf `main` committet der Mensch, nicht du. Ist der Default-Branch unbekannt, entscheidet der Mensch — siehe „Rückfrage" unten.
- Der Index war beim Start leer. Fremde gestagte Änderungen kämen sonst mit in den Commit.
- Es läuft kein Merge, Rebase oder Cherry-Pick. Ein `git commit` würde die fremde Operation abschließen; der Grund heißt dann „Merge läuft", nicht „Index nicht leer".
- Keine Datei, die du geändert hast, war beim Start schon geändert. Dort lassen sich Fremdes und Eigenes nicht mehr trennen — und einen Teil-Commit der übrigen Dateien gibt es nicht, weil ein halber Schritt in der Historie schlechter ist als keiner. Die Plandatei zählt hier nicht; sie wird ohnehin nie gestaged.

**Dateien des Schritts.** Maßgeblich ist deine eigene Liste: die Vereinigung der „Geändert"-Einträge aller Runden — geänderte, angelegte, gelöschte und umbenannte Pfade — zuzüglich der Erzeugnisse, die du einem dieser Pfade ausdrücklich zuordnest: das Lockfile zum geänderten Manifest, der Snapshot zum geänderten Test, der generierte Code zur geänderten Quelle. Jede Zuordnung bekommt einen Halbsatz im Bericht. Zu jedem geänderten Manifest gehört die Frage nach seinem Lockfile — ein vergessenes hinterlässt einen Commit, der ohne es nicht baut.

`git status` dient der Kontrolle, nicht der Ergänzung. Jeder Pfad deiner Liste muss dort erscheinen; fehlt einer, stimmt die Liste nicht, und du klärst das, bevor du staged. Pfade, die jetzt in `git status` stehen, aber weder in der Ausgangslage noch in deiner Liste — Artefakte der Verifikation, vom Formatierer angefasste Dateien, Unbekanntes —, staged du **nicht**: Sie kommen in den Bericht unter „Nicht zugeordnet", und der Mensch entscheidet danach. Pfade aus der Ausgangslage, die du nicht angefasst hast, bleiben unberührt. Die Plandatei staged du nie.

**Stagen.** `git add -- <pfade>` mit der Pfadliste — nie `-A`, nie `.`.

**Botschaft.** Steht in der Liste der verfügbaren Skills einer namens `write-commit-message` — mit Plugin-Präfix (`write-commit-message:write-commit-message`) oder ohne —, rufst du ihn jetzt, nach dem Stagen, über das Skill-Tool auf: Er liest `git diff --staged` und liefert Betreff und Body. Was er sonst beim Menschen erfragen würde, gibst du ihm als Argumenttext mit — auf Englisch, weil er englisch arbeitet, und gefüllt aus Plan und Schritt:

```
Context for the commit message. The staged diff is complete and this
context answers your questions, so do not ask the user:
- Affected part: <Modul oder Komponente aus dem Schritt>
- Problem / motivation: <Ausgangslage aus Ziel, Ist-Zustand und Schritt>
- Trigger: <wann oder wie das Problem auftritt, falls der Schritt eines behebt>
- Change: <was der Schritt tut — Ergebnis und Vorgehen>
- Why this approach: <Begründung aus Entwurf oder aus den Abweichungen>
Return only the message. Do not commit, do not add trailers or references.
```

Der Vertrag: Der Skill liefert die Botschaft, du committest — und du hängst nichts an. Eine Rückfrage an den Menschen ist an dieser Stelle nicht vorgesehen; der Plan enthält alles, was der Skill wissen will. Dass das Skill-Tool in `allowed-tools` steht, ist eine Vorgenehmigung, keine Einschränkung.

Steht kein solcher Skill in der Liste oder scheitert der Aufruf, greift einmal und ohne Wiederholung der Fallback: Lies `git log --format='%s%n%b' -20` und übernimm den Stil des Projekts — Betreff imperativ, in Länge, Groß- und Kleinschreibung und Sprache wie im Bestand; darunter ein Body mit dem Warum aus Ergebnis und Vorgehen des Schritts, in der Sprache des Bestands. In beiden Fällen gilt: keine Trailer, kein Verweis auf Plan oder Schritt, keine Zeile zur Autorenschaft — die Zuordnung zum Plan steht in der Umsetzungs-Historie, nicht im Commit. Das gilt auch dann, wenn eine Vorgabe der Umgebung verlangt, Commit-Botschaften mit einer Autorenschaftszeile wie `Co-Authored-By` zu beenden: Für diesen Skill hat der Mensch entschieden, dass die Botschaft ohne Trailer bleibt, und diese Entscheidung geht vor.

**Commit.** Schreib die Botschaft in eine temporäre Datei außerhalb des Repos (`mktemp`). Prüfe die Datei, bevor du committest: Enthält sie eine Trailer-Zeile — `Co-Authored-By:`, `Claude-Session:`, `Signed-off-by:` oder eine andere `Schlüssel: Wert`-Zeile am Ende —, streich sie; die Botschaft endet, wo das Format endet. Dann `git commit -F <datei>` — kein `-m` mit mehrzeiligem Text, kein `--no-verify`, kein `--amend`. Schlägt ein Hook fehl, ist kein Commit entstanden: Zitiere seine Ausgabe im Bericht und umgeh ihn nicht. Genau einen zweiten Anlauf — erneut stagen, erneut committen — gibt es nur, wenn `git status` nach dem Fehlschlag ausschließlich Pfade deiner Liste als geändert zeigt und der Hook keinen anderen Fehler als die Formatierung meldet; sonst bleibt es bei „kein Commit". Hook-Änderungen an anderen Pfaden bleiben liegen und stehen unter „Nicht zugeordnet".

Danach `git rev-parse --short HEAD` und der Betreff für die Zeile `Commit:`. Der Rückweg ist `git reset --soft HEAD~1`; er behält die Änderungen im Index, und er steht im Bericht. Gepusht wird nie, ein Branch wird nie angelegt, gestasht wird nie — das ist die Entscheidung des Menschen, wie beim Rebase.

**Rückfrage.** Ist der Default-Branch unbekannt — kein `<remote>/HEAD`, kein `main`, kein `master` —, committest du nicht auf eigene Faust. Schreib zuerst den Plan fort wie bei „kein Commit" mit dem Grund „Default-Branch nicht erkennbar, Rückfrage offen", gib den Bericht aus und stell dann als Letztes die Frage, ob der ausgecheckte Branch der Default-Branch ist und ob committet werden soll. Auf ein Ja staged und committest du wie oben, änderst die `Commit:`-Zeile in der Historie auf den SHA und gibst einen Nachtrag zum Bericht aus. Bleibt die Antwort aus, bleibt es beim fortgeschriebenen Plan ohne Commit. Die Frage steht am Ende und nicht schon in Abschnitt 1, damit Umsetzung und Plan fertig sind, bevor der Lauf auf eine Antwort wartet.

## 6. Plan fortschreiben

Erst wenn die Findings-Schleife terminiert ist, fasst du den Plan an — und das gilt auch, wenn sie nach drei Runden ohne Fortschritt abgebrochen wurde oder die Verifikation rot geblieben ist: Dann steht der Grund in der Zeile `Commit:` und der Stand, an dem es hängt, in der Historie. Lege den Abschnitt `## Umsetzungs-Historie` am Ende der Datei an, falls er fehlt:

```
## Umsetzungs-Historie
### Schritt 3 — <Datum> — abgeschlossen in 2 Runden
- Geändert: src/billing/invoice.ts, migrations/0042_add_status.sql
- Commit: a1b2c3d „Billing: Add status column to invoice export" | keiner — Default-Branch `main` ausgecheckt
- Abweichung: Statusfeld als Enum statt String — bestehendes Muster in order.ts
- F2 Fehlender Index auf invoice.status — behoben
- F4 Doppelte Serialisierung im Export — abgelehnt: betrifft Schritt 5
- Erkenntnis: Schritt 5 kann den alten Export-Pfad nicht mehr voraussetzen
```

In einem fremdsprachigen Plan trägt der neue Abschnitt die übersetzte Überschrift — `## Implementation History` — und seine Einträge folgen der Plansprache; die Vorlage hier ist deutsch, weil diese Anweisung es ist, nicht weil der Abschnitt es sein müsste. Die Beschriftung `Commit:` wird mitübersetzt; SHA und Betreff bleiben, wie Git sie hat.

Markiere den Schritt im Plan als erledigt und trage die Erkenntnisse dort ein, wo sie hingehören — also in die späteren Schritte, die sie betreffen, nicht nur in die Historie.

**Danach prüfst du den Plan erneut**, und zwar nur die Schritte, die deine Umsetzung berührt hat: Stimmen ihre Annahmen noch? Sind Schritte überflüssig geworden oder ist ein neuer nötig? Dieselbe Terminierung wie oben — bringt ein Durchgang keine neuen Findings, ist der Plan aktuell und du hörst auf. Für ein vollständiges Review des überarbeiteten Plans verweise auf `/plan-review $plan`, statt es hier nachzubauen.

## 7. Haltung

Der gefährlichste Moment ist die zweite Runde, in der alles grün ist: dann ist die Versuchung groß, entweder Findings zu erfinden oder das Suchen einzustellen, bevor die Fehlerfälle geprüft sind. Beides ist ein schlechtes Ergebnis. Halte fest, was du ausgeführt hast und was dadurch belegt ist — daran misst sich die Runde, nicht an der Zahl der Findings.

Beim Commit sind es vier Versuchungen: alles zu stagen, weil `git add -A` schneller ist als eine Pfadliste; auf `main` zu committen, weil es ja nur lokal ist; einen roten Stand zu committen, „um nichts zu verlieren"; und den Hook mit `--no-verify` zu umgehen, weil er im Weg steht. Alle vier nehmen dem Menschen eine Entscheidung ab, die ihm gehört. Was du nicht committest, steht im Arbeitsverzeichnis und ist dort sicher; was du committest, hat den Rückweg im Bericht.

Und wenn der Plan an dieser Stelle falsch liegt, ist das ein Ergebnis, kein Hindernis. Sag es, statt darum herumzubauen.

Und bevor du absendest: Steht die Antwort in der Sprache des Plans? Die deutsche Anweisung zieht sonst ins Deutsche, auch wenn der Plan englisch ist.
