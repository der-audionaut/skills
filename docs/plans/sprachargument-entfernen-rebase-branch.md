---
Feature: Sprachargument entfernen und Zielbranch für tool-git-rebase
Erstellt: 2026-09-15
Claude-Session: be31b1ec-ca9f-4819-b74d-919a419486a0
Status: Umgesetzt — alle Schritte erledigt (2026-09-15)
---

# Sprachargument entfernen und Zielbranch für tool-git-rebase

Zwei Änderungen an den Skills dieses Repos. Erstens verlieren `plan-review`, `plan-lint`, `plan-coding` und `tool-git-rebase` ihr letztes, optionales Argument `sprache` samt der Auflösungslogik dafür; sie antworten künftig immer auf Deutsch. Zweitens bekommt `tool-git-rebase` an dieser frei gewordenen Stelle ein optionales Argument `branch`: Ist es gesetzt, wird dieser Branch auf den Basis-Branch neu aufgesetzt und danach der vorher ausgecheckte Branch wieder ausgecheckt; sonst wie bisher der ausgecheckte. Die README und die Konventionen werden nachgezogen.

## Arbeitsschritte

| Nr | Schritt | Ergebnis | Abhängig von | Welle |
|----|---------|----------|--------------|-------|
| 1 | `plan-review`: Sprachargument entfernen | Skill nimmt nur noch `plan`, antwortet seit Schritt 7 in der Plansprache, erkennt Historie in jeder Sprache | — | W1 ✓ erledigt |
| 2 | `plan-lint`: Sprachargument entfernen | dito für `plan-lint` | — | W1 ✓ erledigt |
| 3 | `plan-coding`: Sprachargument entfernen | Skill nimmt nur noch `plan schritt`; Sprach-Sonderfall bei `$schritt` entfällt | — | W1 ✓ erledigt |
| 4 | `tool-git-rebase`: Sprachargument entfernen | Skill nimmt nur noch `basebranch`; Regel „einziges Argument ist eine Sprache" entfällt | — | W1 ✓ erledigt |
| 5 | `tool-git-rebase`: optionales Argument `branch` | `/tool-git-rebase <basebranch> [branch]` wechselt auf den Zielbranch, setzt ihn neu auf, verifiziert und kehrt auf den vorher ausgecheckten Branch zurück | 4 | W2 ✓ erledigt |
| 6 | README und Abschlussprüfung | Tabelle, Sprachabsätze, Tool-Abschnitt und Konventionen beschreiben den neuen Stand; repo-weiter Grep findet kein totes Sprachargument mehr | 1, 2, 3, 4, 5 | W3 ✓ erledigt |
| 7 | Plan-Skills auf Plansprache umstellen | `plan-review`, `plan-lint`, `plan-coding` schreiben Bericht, Rückfragen und Historie in der Sprache des Plans (Revision von A2) | 1, 2, 3 | W4 ✓ erledigt |

## Parallel abarbeitbar

| Welle | Schritte | Voraussetzung | Berührungspunkte |
|-------|----------|---------------|------------------|
| W1 | 1, 2, 3, 4 | — | keine gemeinsamen Dateien; jeder Schritt ändert genau eine `SKILL.md`. Damit die drei Plan-Skills nachher gleich klingen, nutzen alle die Formulierungen aus dem Entwurf (Abschnitt „Schnittstellen") |
| W2 | 5 | Schritt 4 abgeschlossen | dieselbe Datei wie Schritt 4 (`skills/tool-git-rebase/SKILL.md`) |
| W3 | 6 | W2 abgeschlossen | `README.md` beschreibt beide Features, deshalb erst nach allen Skill-Änderungen |
| W4 | 7 | Schritte 1–3 abgeschlossen; Revision von A2 am 2026-09-15 | dieselben drei `SKILL.md` wie W1, deshalb nach W1 |

Die vier Skill-Bearbeitungen in W1 sind echt unabhängig und verkürzen den Durchlauf; W2 und W3 sind eine Kette, weil Schritt 5 dieselbe Datei wie Schritt 4 anfasst und die README den Endzustand dokumentiert.

## Ziel

- Die Frontmatter der vier Skills nennt kein Argument `sprache` mehr (`argument-hint`, `arguments`, `description`), und im Anweisungstext kommt weder `$sprache` noch die Auflösungslogik dafür vor.
- Die drei Plan-Skills antworten in der Sprache des Plans — Kopfblock und Überschriften entscheiden —, erkennen einen vorhandenen Historie-Abschnitt in jeder Sprache und legen nie einen zweiten an, weil `/plan-create` Pläne weiterhin in anderen Sprachen erzeugen kann. `tool-git-rebase` antwortet fest auf Deutsch.
- `/tool-git-rebase main` verhält sich wie heute. `/tool-git-rebase main feature/x` prüft die Ausgangslage, wechselt erst unmittelbar vor dem Rebase auf `feature/x`, setzt ihn auf `main` neu auf, verifiziert und kehrt auf den vorher ausgecheckten Branch zurück; der Bericht nennt Wechsel, Rückkehr und den Rückweg für `feature/x`. Nur ein Rebase, der auf eine Entscheidung wartet, bleibt auf `feature/x` stehen — den kann man nicht wegwechseln.
- Die README beschreibt in Übersichtstabelle, Sprachabsätzen, Tool-Abschnitt und Konventionen genau diesen Stand; `claude plugin validate .` ist grün.

## Nicht-Ziele

- `plan-create` und seine Vorlage behalten das Argument `sprache` unverändert — der Aufruf nennt es nicht (Annahme A1).
- Die News-Skills und ihr Zusatz `sprache <Sprache>` bleiben unberührt.
- Kein Anlegen lokaler Branches aus Remote-Branches durch `tool-git-rebase` (Annahme A4).
- Kein temporärer Worktree für den Zielbranch (verworfene Alternative, siehe Datenfluss).
- `references/konfliktmuster.md` und die Manifeste unter `.claude-plugin/` ändern sich nicht; sie erwähnen kein Sprachargument.
- Nebenbei aufgefallen, aber nicht Teil dieses Plans: `skills/plan-coding/SKILL.md:2` heißt in der Frontmatter noch `plan-implement`, und `skills/plan-create/references/plan-vorlage.md:3` verweist auf `/plan-implement`. Beides ist ein eigener, kleiner Aufräumschritt.

## Ist-Zustand

Zeilenangaben beziehen sich auf den Stand vor der Umsetzung (Commit `f5d295c`). Alle vier Skills folgen demselben Muster, das die README unter „Conventions" beschreibt (`README.md:117`): Sprachdirektive als erste Zeile, Auflösung der Sprache in Abschnitt 1, Kontrollfrage am Ende.

**`skills/plan-review/SKILL.md`** — `:3` Beschreibung endet mit „optional gefolgt von der Ausgabesprache (Standard Deutsch)"; `:4` `argument-hint: "[plan-datei] [sprache]"`; `:5` `arguments: plan sprache`; `:10` Direktive `**Ausgabesprache: `$sprache`**`; `:14` Überschrift „## 1. Sprache und Plan auflösen"; `:16` Absatz „**Sprache.**"; `:18` Absatz „Die Sprache ändert den Text, nicht die Struktur" — enthält als zweiten Teil die Regel, `## Review-Historie` in jeder Sprache zu erkennen und nie doppelt anzulegen; `:130` Schlusskontrolle „Steht die Antwort in der Ausgabesprache?".

**`skills/plan-lint/SKILL.md`** — gleiches Muster: `:3`, `:4`, `:5`, `:10`, `:16`, `:18`, `:20` (Struktur-Absatz mit Historie-Erkennung für drei Abschnitte), `:122`. `:22` („Den Plan selbst lintest du auf Inhalt, nicht auf Sprache") ist keine Regel zum Argument, sondern zum Lint-Inhalt, und bleibt.

**`skills/plan-coding/SKILL.md`** — `:3`, `:4` `"[plan-datei] [schritt] [sprache]"`, `:5` `arguments: plan schritt sprache`, `:10` (mit dem Zusatz „Code folgt dem Bestand"), `:16` „## 1. Sprache, Plan und Schritt auflösen", `:18`, `:20` (Struktur-Absatz; enthält „Code, Bezeichner, Kommentare und Tests folgen den Konventionen des Bestands" und die Historie-Erkennung), `:28` letzter Satz des Absatzes „**Schritt.**": ein als Schritt übergebenes Sprachwort wird als Sprache gedeutet; `:120` Schlusskontrolle.

**`skills/tool-git-rebase/SKILL.md`** — `:3`, `:4` `"[basebranch] [sprache]"`, `:5` `arguments: basebranch sprache`, `:10` Direktive mit Sonderfall „Sprache als einziges Argument in `$basebranch`", `:12` „Setze den aktuellen Branch per Rebase auf `$basebranch` neu auf", `:16` „## 1. Sprache und Basis-Branch auflösen", `:18` Absatz „**Sprache.**", `:20`–`:25` Absatz „**Basis-Branch.**" mit vier Regeln, davon `:23` die Sprachdeutung eines unbekannten Branchnamens; `:33` Abbruchgrund „Kein Branch ausgecheckt (detached HEAD), oder der aktuelle Branch ist der Basis-Branch"; `:35`–`:42` alle Vorprüfungen und die Sicherung des Ausgangspunkts arbeiten mit `HEAD`, also mit dem ausgecheckten Branch; `:88` Berichtskopf `# Rebase: <branch> auf <basis>`; `:126` Schlusskontrolle. Ein Wechsel des Branchs ist nirgends vorgesehen.

**`README.md`** — `:14`–`:17` Aufrufspalte der vier Skills mit `[sprache]`; `:19` „All seven skills answer in German by default. The plan and tool skills take a different output language …"; `:45` Absatz „**Choosing a language:**" für die drei nachgelagerten Skills; `:47` Absatz zur Sprache in `/plan-create` (bleibt inhaltlich richtig); `:61` Tool-Abschnitt „rebases the branch you are on"; `:65` Schlusssatz des Tool-Abschnitts mit `git reset --hard ORIG_HEAD` als Rückweg; `:111` Anatomie: Beispielvariablen `$plan`, `$schritt`, `$sprache`; `:117` Konvention „German by default" begründet die Direktive in der ersten Zeile mit der Umschaltbarkeit; `:118` Konvention „The language changes the text, not the structure" spricht von Berichten, Historien, Code und Commit-Botschaften.

**Testbarkeit.** Das Repo hat keine automatisierten Tests. Prüfbar sind Frontmatter und Manifeste per `claude plugin validate .` (README `:94`) und das Verhalten nur manuell. Die Marketplace-Installation ist auf dieser Maschine aktiv; Änderungen an der Arbeitskopie sind deshalb nicht direkt live (README `:92`).

## Entwurf

### Architektur und Einordnung

Beides sind Änderungen an Markdown-Anweisungen, kein Laufzeitcode. Der Aufwand liegt in der Konsistenz: Vier Skills und die README müssen nachher dieselbe Geschichte erzählen. Die Skills bleiben unabhängig voneinander; die einzige Kopplung ist fachlich — `/plan-create` erzeugt weiterhin Pläne in fremder Sprache, und die drei nachgelagerten Skills müssen damit umgehen können.

### Komponenten

- **Sprachdirektive** (erste Zeile jedes Skills): verliert den Parameter. In `tool-git-rebase` wird sie fest „Deutsch" (A5); in den drei Plan-Skills lautet sie „die Sprache des Plans" (Revision von A2, Schritt 7). In den Plan-Skills bleibt darum auch die Schlusskontrolle, weil die Ausgabe vom Deutschen abweichen kann; in `tool-git-rebase` entfällt sie.
- **Argumentauflösung** (Abschnitt 1 jedes Skills): verliert den Absatz „Sprache" und die Sonderfälle, in denen ein Argument als Sprache gedeutet wird. Bei `tool-git-rebase` kommt an derselben Stelle die Auflösung des Zielbranchs hinzu.
- **Historie-Erkennung** (nur Plan-Skills): bleibt — neue Abschnitte entstehen in der Plansprache (A2 revidiert).
- **Zielbranch** (nur `tool-git-rebase`): neuer Begriff für „der Branch, der neu aufgesetzt wird". Abschnitt 1 löst ihn auf, Abschnitt 2 prüft und sichert ihn ohne Wechsel über `<ziel>`, Abschnitt 3 wechselt unmittelbar vor `git rebase` dorthin, ab dort ist er `HEAD` und die Abschnitte 3 bis 5 laufen unverändert; nach der Verifikation geht es zurück auf den vorher ausgecheckten Branch (Entscheidung zu A3, Review B1).

### Datenfluss

`tool-git-rebase` mit `$branch`: Argumente → Basis auflösen → Zielbranch auflösen → Abbruchgründe (sauberes Arbeitsverzeichnis, kein laufender Rebase, kein detached HEAD) → Fetch, Upstream-Vergleich der Basis, Merge-Base, Vorfahr- und Upstream-Prüfung des Zielbranchs, Sicherung — alles über `<ziel>`, ohne Wechsel → `git switch <ziel>` nur wenn Zielbranch ≠ ausgecheckter Branch, vorherigen Branch notieren, unmittelbar vor `git rebase` → Rebase, Konflikte, Verifikation → `git switch <vorheriger Branch>`, sobald kein Rebase mehr läuft, nicht aber, wenn er auf eine Entscheidung wartet → Bericht mit Wechsel- und Rückwegzeile.

Der Wechsel steht so spät wie möglich: nach allen Prüfungen und der Sicherung, unmittelbar vor `git rebase`. So hat der Skill nach dem Wechsel nur noch Rebase-Ausgänge; die frühen Ausgänge — Basis bereits Vorfahr, also nichts zu tun (`skills/tool-git-rebase/SKILL.md:39`), Upstream des Zielbranchs hat Commits, die lokal fehlen (`:40`), Rückfrage bei unveröffentlichten Commits auf der Basis (`:35`) — enden auf dem Branch des Menschen, ohne dass etwas zurückgewechselt werden müsste (Review B1). Preis: Die Befehle in Abschnitt 2 nehmen `<ziel>` statt `HEAD` — `git merge-base <basis> <ziel>`, `git merge-base --is-ancestor <basis> <ziel>`, `git log --oneline <basis>..<ziel>`, `git log --oneline <ziel>..<ziel>@{u}`, `git rev-parse <ziel>`, `git diff $(git merge-base <basis> <ziel>) <ziel>`. Ohne `$branch` ist `<ziel>` der ausgecheckte Branch, und die Befehle liefern dasselbe wie heute mit `HEAD`. Die Prüfung auf sauberes Arbeitsverzeichnis bleibt trotzdem der erste Abbruchgrund: Wechsel und Rebase brauchen es beide, und ein `git switch` mit ungesicherten Änderungen trägt sie mit oder scheitert. Der Rückwechsel steht **nach** der Verifikation, weil Build, Tests und Diff-Vergleich auf dem neu aufgesetzten Stand laufen müssen; ein Rebase, der auf eine Entscheidung wartet, lässt sich nicht verlassen, ohne ihn abzubrechen — dort bleibt der Zielbranch ausgecheckt, und der Bericht sagt das.

Verworfene Alternative: `git rebase <basis> <ziel>` wechselt implizit. Nach der Umstellung von Abschnitt 2 auf `<ziel>` wäre das gleichwertig; der explizite `git switch` bleibt trotzdem, weil er den vorherigen Branch sichtbar notiert, bei Fehlschlag eine eigene Meldung hat und der Rückwechsel sein erkennbares Gegenstück ist.

Ebenfalls verworfen: den Zielbranch in einem temporären Worktree (`git worktree add`) neu aufzusetzen, damit das Arbeitsverzeichnis nie angefasst wird. Das klingt nach dem sauberen Weg zu „nichts damit zu tun haben", scheitert aber an der Verifikation: Ein frischer Worktree hat keine installierten Abhängigkeiten und keine Build-Artefakte, Build und Tests liefen also nicht oder erst nach einer Installation, und ein Konflikt läge in einem Verzeichnis, das der Mensch erst suchen muss. Hin- und Rückwechsel im echten Arbeitsverzeichnis sind zwei Zeilen Git und lassen die Verifikation, wie sie ist.

### Schnittstellen

**Frontmatter nach der Änderung:**

| Skill | `argument-hint` | `arguments` |
|-------|-----------------|-------------|
| plan-review | `"[plan-datei]"` | `plan` |
| plan-lint | `"[plan-datei]"` | `plan` |
| plan-coding | `"[plan-datei] [schritt]"` | `plan schritt` |
| tool-git-rebase | `"[basebranch] [branch]"` | `basebranch branch` |

In allen vier `description`-Feldern entfällt der Halbsatz „optional gefolgt von der Ausgabesprache (Standard Deutsch)"; bei `tool-git-rebase` tritt an seine Stelle „optional gefolgt vom Branch, der neu aufgesetzt wird — ohne Angabe der ausgecheckte Branch".

**Direktive** (erste Zeile, ersetzt die parametrisierte). `tool-git-rebase` (A5): *Ausgabesprache Deutsch — Bericht, Rückfragen und Abbruchmeldungen auf Deutsch, auch wenn Repository und Commit-Botschaften englisch sind; Commit-Botschaften und Code folgen dem Bestand.* Die drei Plan-Skills (Schritt 7, Revision von A2): *Ausgabesprache: die Sprache des Plans — Bericht, Rückfragen, Abbruchmeldungen und Historie in der Sprache, in der der Plan verfasst ist; bis der Plan gelesen ist, Deutsch.* `plan-coding` hängt an: *Code, Bezeichner, Kommentare und Tests folgen dem Bestand.* Dazu in Abschnitt 1 nach der Planauflösung ein Absatz „**Sprache.**": Kopfblock und Überschriften entscheiden, nicht Zitate oder Bezeichner; die Sprache ändert den Text, nicht die Struktur. Die Schlusskontrolle „Steht die Antwort in der Sprache des Plans?" bleibt in den Plan-Skills und entfällt nur in `tool-git-rebase`.

**Historie-Erkennung** (Plan-Skills, in Abschnitt 1 nach der Planauflösung; A2), sinngemäß: *Wo diese Anweisung `## Review-Historie` sagt, ist der Abschnitt dieser Bedeutung gemeint, gleich in welcher Sprache seine Überschrift steht — ein Plan kann aus `/plan-create` in einer anderen Sprache stammen. Einen vorhandenen erkennst du auch in Übersetzung und schreibst ihn fort, statt einen zweiten anzulegen; einen neuen legst du in der Plansprache an.* `plan-lint` nennt dabei alle drei Historie-Abschnitte, `plan-coding` Umsetzungs- und Review-Historie, `plan-review` nur die Review-Historie — so wie heute.

**Zielbranch-Auflösung** (`tool-git-rebase`, Abschnitt 1, neuer Absatz „**Zielbranch.**" nach „**Basis-Branch.**"):

1. `$branch` leer → Zielbranch ist der ausgecheckte Branch (`git symbolic-ref --quiet --short HEAD`).
2. `$branch` existiert als lokaler Branch (`git rev-parse --verify refs/heads/$branch`) → das ist der Zielbranch.
3. `$branch` existiert nur auf einem Remote → abbrechen und den Einzeiler nennen, der den lokalen Branch anlegt (`git switch $branch`); der Skill legt keine Branches an (A4).
4. `$branch` existiert gar nicht → vorhandene Branches nennen und abbrechen. Das fängt nebenbei den alten Aufruf `/tool-git-rebase main english` sichtbar ab.
5. Zielbranch und Basis-Branch sind derselbe Ref → abbrechen; ersetzt den heutigen Abbruchgrund „der aktuelle Branch ist der Basis-Branch".

Der Basis-Branch-Absatz behält seine Regeln „leer → Kandidaten fragen", „nur remote → Remote-Zweig", „nicht vorhanden → abbrechen"; die Kandidatenliste nennt statt „Upstream des aktuellen Branchs" den Upstream des Zielbranchs.

**Vorprüfungen** (Abschnitt 2): laufen ohne Wechsel. Fetch und Upstream-Vergleich der Basis wie heute; Merge-Base, Commit-Liste, Vorfahr-Prüfung, Upstream-Prüfung des Zielbranchs und die Sicherung des Ausgangspunkts mit `<ziel>` statt `HEAD` und `<ziel>@{u}` statt `@{u}` (Befehle im Datenfluss). Die Abbruchgründe sauberes Arbeitsverzeichnis, laufender Rebase und detached HEAD bleiben unverändert am Anfang.

**Wechsel** (Abschnitt 3, unmittelbar vor `git rebase`): Ist der Zielbranch nicht ausgecheckt → vorherigen Branch notieren, `git switch <ziel>`. Schlägt der Wechsel fehl, abbrechen und melden — bis hierhin ist nichts verändert. Detached HEAD bleibt in jedem Fall ein Abbruchgrund, auch mit `$branch`: Von einem losgelösten Commit wegzuwechseln kann Commits unerreichbar machen, und ob die jemand braucht, weiß der Skill nicht — die eine Zeile `git switch` vorher macht der Mensch.

**Rückwechsel** (Ende von Abschnitt 5, nach Verifikation, Diff-Vergleich und Commit-Zählung): Wurde in Abschnitt 3 gewechselt und läuft kein Rebase mehr — abgeschlossen, gegebenenfalls samt Reparatur-Commit —, dann `git switch <vorheriger Branch>`. Schlägt der Rückwechsel fehl (etwa weil unversionierte Verifikationsartefakte mit Dateien des vorherigen Branchs kollidieren), bleibt der Skill stehen, meldet es und erzwingt nichts. Wartet der Rebase auf eine Entscheidung, bleibt der Zielbranch ausgecheckt: Wer den Zielbranch angibt, will nichts mit ihm zu tun haben (Entscheidung zu A3), aber ein stehender Rebase lässt sich nicht verlassen.

**Bericht** (Abschnitt 6), nur wenn `$branch` gesetzt war: Zeile `**Ausgecheckt:** <vorheriger Branch> — für den Rebase nach <ziel> gewechselt und zurück` beziehungsweise `— steht auf <ziel>, Rebase wartet`. Der Rückweg nach dem Rückwechsel heißt `git branch -f <ziel> <alter HEAD des Zielbranchs>` statt `git reset --hard <alter HEAD>`; solange der Rebase steht, bleibt er `git rebase --abort`, wie `skills/tool-git-rebase/SKILL.md:124` „während" und „danach" unterscheidet — `git branch -f` verweigert auf einem ausgecheckten Branch ohnehin (Review S1). Die Rückweg-Zeile warnt ausdrücklich: `ORIG_HEAD` zeigt nach dem Rückwechsel weiter auf den alten Stand des Zielbranchs, ein `git reset --hard ORIG_HEAD` auf dem ausgecheckten Branch würde diesen zerstören. Der Push-Hinweis nennt den Branch explizit — `git push --force-with-lease <remote> <ziel>` —, weil ein Push ohne Refspec den ausgecheckten Branch nähme. Ohne `$branch` bleibt der Bericht, wie er ist.

Im übrigen Text ersetzt „Zielbranch" die Wendung „der aktuelle Branch", wo sie den Branch meint, der neu aufgesetzt wird (`:12`, `:22`, `:33`, `:40`).

### Datenmodell und Migration

Kein Datenmodell. Die einzige „Migration" ist der geänderte Aufruf: `/plan-review <plan> english` liefert künftig einen Bericht in der Plansprache und verwirft `english` stillschweigend, weil nur die in `arguments` benannten Argumente positional zugeordnet werden und die Skills `$ARGUMENTS` nicht auswerten (README `:111`). Bei `/tool-git-rebase main english` landet `english` in `$branch` und führt zum Abbruch mit Branchliste — sichtbar und harmlos. Beides wird in Kauf genommen; siehe Risiken.

### Caching

Kein Caching — es gibt nichts, was zwischengespeichert würde.

## Schritte

### Schritt 1 — `plan-review`: Sprachargument entfernen
- **Status:** erledigt 2026-09-15 (W1)
- **Revision:** Direktive und Historie-Sprache am 2026-09-15 in Schritt 7 auf die Plansprache umgestellt (A2 revidiert).
- **Ergebnis:** Der Skill nimmt nur noch `plan`, antwortet fest auf Deutsch (seit Schritt 7: in der Plansprache) und erkennt eine vorhandene Review-Historie weiter in jeder Sprache.
- **Betroffen:** `skills/plan-review/SKILL.md`
- **Abhängig von:** —
- **Vorgehen:** Frontmatter nach Tabelle im Entwurf anpassen, den Halbsatz zur Ausgabesprache aus `description` streichen. Zeile 10 durch die feste Direktive ersetzen. Überschrift von Abschnitt 1 zu „## 1. Plan auflösen" kürzen, den Absatz „**Sprache.**" streichen und den Struktur-Absatz auf die Historie-Erkennung nach Entwurf reduzieren — die Aussage über die Kennungen B, S, O kann als ein Satz bleiben („bleiben unübersetzt", weil die Historie sie referenziert) oder entfallen, sie ist ohne Sprachwahl trivial. Schlusskontrolle in Abschnitt 8 streichen.
- **Akzeptanz:** `grep -n 'sprache\|Sprache' skills/plan-review/SKILL.md` trifft nur noch die feste Direktive und die Historie-Erkennung; `claude plugin validate .` grün; der Text liest sich ohne Bruch von Direktive zu Abschnitt 1.

### Schritt 2 — `plan-lint`: Sprachargument entfernen
- **Status:** erledigt 2026-09-15 (W1)
- **Revision:** Direktive und Historie-Sprache am 2026-09-15 in Schritt 7 auf die Plansprache umgestellt (A2 revidiert).
- **Ergebnis:** Wie Schritt 1 für `plan-lint`; die Historie-Erkennung nennt weiterhin alle drei Historie-Abschnitte.
- **Betroffen:** `skills/plan-lint/SKILL.md`
- **Abhängig von:** —
- **Vorgehen:** Gleiches Schema wie Schritt 1 (Frontmatter, Direktive, Überschrift „## 1. Plan auflösen", Absätze in Abschnitt 1, Schlusskontrolle in Abschnitt 7). Zeile 22 („Den Plan selbst lintest du auf Inhalt, nicht auf Sprache …") bleibt stehen: Sie regelt, was ein I-Finding ist, nicht die Ausgabesprache — und sie wird wichtiger, wenn deutsche Historie in einem englischen Plan steht.
- **Akzeptanz:** Grep wie in Schritt 1 trifft nur Direktive, Historie-Erkennung und den Satz zum Lint auf Inhalt statt Sprache; `claude plugin validate .` grün.

### Schritt 3 — `plan-coding`: Sprachargument entfernen
- **Status:** erledigt 2026-09-15 (W1)
- **Revision:** Direktive und Historie-Sprache am 2026-09-15 in Schritt 7 auf die Plansprache umgestellt (A2 revidiert).
- **Ergebnis:** Der Skill nimmt `plan schritt`; ein Sprachwort an der Stelle von `$schritt` wird nicht mehr als Sprache gedeutet; Code folgt weiterhin ausdrücklich dem Bestand.
- **Betroffen:** `skills/plan-coding/SKILL.md`
- **Abhängig von:** —
- **Vorgehen:** Frontmatter nach Tabelle. Direktive mit dem Zusatz zu Code und Bezeichnern ersetzen. Überschrift „## 1. Plan und Schritt auflösen"; Absatz „**Sprache.**" streichen; aus dem Struktur-Absatz den Satz zu Code, Bezeichnern, Kommentaren und Tests in die feste Direktive übernehmen (steht dann nicht doppelt) und die Historie-Erkennung nach Entwurf behalten. Im Absatz „**Schritt.**" den letzten Satz (Sprachwort statt Schritt) streichen. Schlusskontrolle in Abschnitt 6 streichen.
- **Akzeptanz:** Grep trifft nur Direktive und Historie-Erkennung; `$sprache` kommt nicht mehr vor; `claude plugin validate .` grün.

### Schritt 4 — `tool-git-rebase`: Sprachargument entfernen
- **Status:** erledigt 2026-09-15 (W1)
- **Ergebnis:** Der Skill nimmt nur noch `basebranch`; die Deutung eines unbekannten Branchnamens als Sprache ist weg; die Basis-Branch-Regeln sind durchnummeriert und vollständig.
- **Betroffen:** `skills/tool-git-rebase/SKILL.md`
- **Abhängig von:** —
- **Vorgehen:** Frontmatter: `argument-hint: "[basebranch]"`, `arguments: basebranch`, Halbsatz aus `description` streichen (Schritt 5 erweitert beides wieder). Direktive in Zeile 10 durch die feste Fassung für `tool-git-rebase` ersetzen, der Sonderfall „Sprache als einziges Argument" entfällt. Überschrift „## 1. Basis-Branch auflösen"; Absatz „**Sprache.**" streichen; Regel 2 (Sprachdeutung) streichen und die verbleibenden drei Regeln neu nummerieren. Schlusskontrolle in Abschnitt 7 streichen.
- **Akzeptanz:** `grep -n 'sprache\|Sprache' skills/tool-git-rebase/SKILL.md` trifft nur die feste Direktive; `claude plugin validate .` grün; Nummerierung der Regeln lückenlos.

### Schritt 5 — `tool-git-rebase`: optionales Argument `branch`
- **Status:** erledigt 2026-09-15 (W2)
- **Ergebnis:** `/tool-git-rebase <basebranch> [branch]` setzt den Zielbranch neu auf: ohne `branch` den ausgecheckten, mit `branch` den genannten lokalen Branch nach einem Wechsel, verifiziert dort und kehrt auf den vorher ausgecheckten Branch zurück; nur ein auf Entscheidung wartender Rebase bleibt auf dem Zielbranch stehen. Remote-only, unbekannter Branch, Zielbranch gleich Basis und detached HEAD brechen mit klarer Meldung ab.
- **Betroffen:** `skills/tool-git-rebase/SKILL.md`
- **Abhängig von:** 4
- **Hinweis nach W1:** Die Zeilenangaben zu `skills/tool-git-rebase/SKILL.md` liegen ab Abschnitt 1 um drei niedriger als im Ist-Zustand (z. B. `:33` → `:30`, `:35` → `:32`, `:39` → `:36`, `:40` → `:37`, `:46` → `:43`, `:88` → `:85`, `:124` → `:121`); Abschnitt 1 heißt jetzt „## 1. Basis-Branch auflösen" mit drei Regeln.
- **Vorgehen:** Frontmatter nach Tabelle im Entwurf, `description` um den Zielbranch ergänzen. Einleitung (`:12`) auf den Zielbranch umformulieren. In Abschnitt 1 die Überschrift zu „## 1. Basis-Branch und Zielbranch auflösen" erweitern und den Absatz „**Zielbranch.**" mit den fünf Regeln aus dem Entwurf einfügen; im Basis-Branch-Absatz „Upstream des aktuellen Branchs" durch den Upstream des Zielbranchs ersetzen. In Abschnitt 2 den Abbruchgrund „der aktuelle Branch ist der Basis-Branch" streichen (jetzt Regel 5 in Abschnitt 1), detached HEAD als Abbruchgrund belassen und die Befehle auf `<ziel>` umstellen — `merge-base`, `--is-ancestor`, `log <basis>..<ziel>`, `log <ziel>..<ziel>@{u}`, `rev-parse <ziel>`, Diff-Sicherung —; „der aktuelle Branch" durch „der Zielbranch" ersetzen, wo der neu aufzusetzende Branch gemeint ist. In Abschnitt 3 unmittelbar vor `git rebase` den bedingten Wechsel mit Notiz des vorherigen Branchs einfügen, mit der Begründung aus dem Entwurf, warum er so spät steht. Am Ende von Abschnitt 5 den Rückwechsel nach Entwurf einfügen: Bedingung (gewechselt und kein Rebase mehr im Gang), Befehl, Verhalten bei Fehlschlag, Ausnahme für den wartenden Rebase. In Abschnitt 6 die Berichtszeile `**Ausgecheckt:**` ergänzen und für den Fall mit `$branch` die Rückweg- und Push-Zeilen nach Entwurf umstellen (`git branch -f <ziel> <sha>` als Rückweg nach dem Rückwechsel mit Warnung vor `ORIG_HEAD`, `git rebase --abort` als Rückweg, solange der Rebase steht, `git push --force-with-lease <remote> <ziel>`); im Fall „wartet auf Entscheidung" unter „Nächste Schritte" den Rückwechsel `git switch <vorheriger Branch>` nach `git rebase --continue` oder `--abort` nennen. In Abschnitt 7 (`:124`) das „danach" um den Fall mit `$branch` ergänzen: dort `git branch -f <ziel> <sha>` statt `git reset --hard ORIG_HEAD`.
- **Akzeptanz:** `claude plugin validate .` grün. Manueller Test im Wegwerf-Repo (Teststrategie): die acht Fälle dort verhalten sich wie beschrieben, insbesondere bleibt bei schmutzigem Arbeitsverzeichnis und bei „nichts zu tun" `HEAD` unverändert, nach erfolgreichem Rebase ist wieder `feature-a` ausgecheckt, der Bericht nennt `git branch -f` statt `git reset --hard ORIG_HEAD`, und im Konfliktfall bleibt der Zielbranch mit stehendem Rebase ausgecheckt.

### Schritt 6 — README und Abschlussprüfung
- **Status:** erledigt 2026-09-15 (W3)
- **Ergebnis:** Die README beschreibt beide Features; im Repo gibt es außer `plan-create` und den News-Skills keine Erwähnung eines Spracharguments mehr.
- **Betroffen:** `README.md`
- **Abhängig von:** 1, 2, 3, 4, 5
- **Vorgehen:** Tabelle `:14`–`:17`: `[sprache]` entfernen, bei `tool-git-rebase` `[branch]` ergänzen und die Zweckspalte um „or the branch you name" erweitern. `:19` neu fassen: Nur `/plan-create` nimmt eine Ausgabesprache als letztes Argument, die News-Skills den Zusatz `sprache <language>`, die drei nachgelagerten Plan-Skills folgen der Sprache des Plans, `tool-git-rebase` antwortet auf Deutsch. `:45` („Choosing a language") auf den neuen Kern kürzen: die drei nachgelagerten Skills schreiben in der Plansprache (Kopfblock und Überschriften entscheiden), erkennen Historie-Abschnitte in jeder Sprache und legen sie nie doppelt an; `:47` bleibt, verliert aber die Abgrenzung „For `/plan-create`" gegen die anderen, wenn sie nun der einzige Fall ist. `:61` im Tool-Abschnitt um den zweiten Aufruf `/tool-git-rebase main feature/x` ergänzen: Wechsel erst unmittelbar vor dem Rebase, Rückkehr auf den vorherigen Branch nach der Verifikation, Ausnahme für den wartenden Rebase, kein Anlegen aus Remote. `:65` (Schlusssatz „the report ends with the `git push --force-with-lease` line … and with `git reset --hard ORIG_HEAD` for when you don't") um den Fall mit Branch ergänzen: Push mit Branchnamen, Rückweg per `git branch -f` (Review S2). `:111` Beispielvariablen zu `$plan`, `$schritt`, `$basebranch`, `$branch` — der Satz zu optionalen Endargumenten bleibt wahr. `:117` und `:118` so umformulieren, dass die Sprache über `plan-create`, die News-Skills und die Plansprache der nachgelagerten Skills wechselt und nur `tool-git-rebase` fest „Deutsch" sagt. Abschließend repo-weit greppen.
- **Akzeptanz:** `grep -rn -i '\$sprache\|\[sprache\]' skills/plan-review skills/plan-lint skills/plan-coding skills/tool-git-rebase` ist leer; `grep -n '\[sprache\]' README.md` trifft nur die Zeile zu `plan-create`; `grep -n -i 'sprache\|language' README.md` trifft nur Stellen zu `plan-create`, zu den News-Skills und die neu formulierten Konventionen; `claude plugin validate .` grün; Tabelle und Tool-Abschnitt nennen `[branch]`, und der Schlusssatz `:65` unterscheidet den Fall mit Branch.

### Schritt 7 — Plan-Skills auf Plansprache umstellen
- **Status:** erledigt 2026-09-15 (W4)
- **Ergebnis:** `plan-review`, `plan-lint` und `plan-coding` schreiben Bericht, Rückfragen und Historie in der Sprache des Plans, erkannt aus Kopfblock und Überschriften; bis der Plan gelesen ist, Deutsch. `tool-git-rebase` bleibt unverändert deutsch.
- **Betroffen:** `skills/plan-review/SKILL.md`, `skills/plan-lint/SKILL.md`, `skills/plan-coding/SKILL.md`
- **Abhängig von:** 1, 2, 3
- **Vorgehen:** Direktive in Zeile 10 auf „die Sprache des Plans" umstellen (Text im Entwurf). In Abschnitt 1 vor dem Absatz „**Historie.**" einen Absatz „**Sprache.**" einfügen: Kopfblock und Überschriften entscheiden, Text statt Struktur. Im Absatz „**Historie.**" „auf Deutsch" durch „in der Plansprache" ersetzen. Schlusskontrolle „Steht die Antwort in der Sprache des Plans?" am Ende wieder einfügen. Die README-Sprachabsätze (`:19`, `:45`, `:117`, `:118`) sagen dasselbe — erledigt in Schritt 6.
- **Akzeptanz:** `claude plugin validate .` grün; `grep -n 'Sprache des Plans'` trifft in allen drei Skills Direktive, Absatz „**Sprache.**" und Schlusskontrolle, in `tool-git-rebase` nichts; Smoke-Test `plan-lint` (Wip-Kopie) gegen einen englischen Wegwerf-Plan → englischer Bericht und `## Lint History` statt `## Lint-Historie`.

## Teststrategie

Es gibt keine Testsuite; geprüft wird auf drei Ebenen.

- **Statisch:** `claude plugin validate .` nach jedem Schritt (Frontmatter, Manifeste). Die Grep-Befehle aus den Akzeptanzkriterien als Regressionsschutz gegen übersehene Stellen.
- **Manuell, Plan-Skills:** Je einen Aufruf `/plan-review <plan>` und `/plan-lint <plan>` gegen diesen Plan, `/plan-coding <plan> <schritt>` gegen einen Wegwerf-Plan mit einem trivialen Schritt, jeweils einmal zusätzlich mit angehängtem `english`: Bericht auf Deutsch, `english` ohne Wirkung. Dazu ein Wegwerf-Plan mit englischer Überschrift `## Review History`: `/plan-review` schreibt ihn fort, statt `## Review-Historie` daneben anzulegen.
- **Manuell, Plansprache (Schritt 7):** `plan-lint` als Wip-Kopie gegen einen englischen Wegwerf-Plan → Bericht auf Englisch, Abschnitt `## Lint History` statt `## Lint-Historie`; derselbe Plan auf Deutsch → deutscher Bericht.
- **Manuell, `tool-git-rebase`:** Wegwerf-Repo im Scratchpad mit Bare-Remote; Branches `main` (mit Commits nach der Abzweigung), `feature-a` (ausgecheckt), `feature-b` (lokal, konfliktfrei) und `feature-c` (lokal, ändert dieselbe Zeile wie `main` anders), alle drei von einem älteren `main` abgezweigt und mit eigenen Commits, `feature-d` (lokal, auf dem aktuellen `main` aufgesetzt, `main` ist also bereits Vorfahr), dazu `remote-only` (nur im Remote). Fälle: (1) `main` ohne `branch` → wie bisher; (2) `main feature-b` → Wechsel, Rebase, Verifikation, zurück auf `feature-a`; Bericht mit Wechselzeile, `git branch -f feature-b <sha>` als Rückweg und `git push --force-with-lease origin feature-b`; (3) `main gibt-es-nicht` → Abbruch mit Branchliste; (4) `main remote-only` → Abbruch mit `git switch`-Hinweis, kein neuer lokaler Branch; (5) `main main` → Abbruch; (6) schmutziges Arbeitsverzeichnis plus `feature-b` → Abbruch, `HEAD` weiter auf `feature-a`; (7) `main feature-c` → Rebase steht auf dem Entscheidungskonflikt, `feature-c` bleibt ausgecheckt, Bericht sagt das und nennt `git switch feature-a` als Schritt nach `--continue` oder `--abort`; (8) `main feature-d` → „nichts zu tun", kein Wechsel, `HEAD` bleibt auf `feature-a`.
- **Testumgebung:** Weil die Marketplace-Installation aktiv ist, läuft `/skills:<name>` hier mit dem gepushten Stand, nicht mit der Arbeitskopie. Für den Test wird das Skill-Verzeichnis nach `.claude/skills/<name>-wip/` kopiert, dort das Frontmatter-`name` auf `<name>-wip` gesetzt und der Skill mit `claude -p "/<name>-wip <args>"` aufgerufen; danach Kopie und leeres `.claude/` wieder entfernen, damit das Repo sauber bleibt. Der Rebase-Test läuft ausschließlich im Wegwerf-Repo, nie in diesem Checkout.

## Rollout

Direkt, ohne Flag: Merge nach `main`, dann `claude plugin update skills@der-audionaut` auf den Maschinen (oder Auto-Update). Rückweg ist `git revert` des Merge-Commits. Ein Fehlschlag wird sichtbar, wenn `claude plugin validate .` rot ist oder `/tool-git-rebase main <branch>` nach dem Update den zweiten Wert nicht als Zielbranch behandelt.

## Risiken

- Vier Bearbeitungen parallel in W1 driften in der Formulierung auseinander → die Direktive und die Historie-Erkennung stehen als Referenztext im Entwurf; Schritt 6 liest die vier ersten Zeilen und Abschnitte 1 nebeneinander gegen.
- Gewohnheitsaufruf `/plan-review <plan> english` verwirft `english` stillschweigend → in Kauf genommen: Der deutsche Bericht macht es sofort sichtbar, und ein Skill ohne `$ARGUMENTS`-Auswertung kann es nicht besser melden.
- `git switch` ist eine Zustandsänderung, die der Skill bisher nie vorgenommen hat → erst unmittelbar vor `git rebase`, nach allen Prüfungen, immer im Bericht, und nur, wenn `$branch` gesetzt ist. Ohne `$branch` ändert sich nichts am Verhalten.
- Nach dem Rückwechsel zeigt `ORIG_HEAD` auf den alten Stand des Zielbranchs; ein gewohnheitsmäßiges `git reset --hard ORIG_HEAD` würde den ausgecheckten Branch zerstören → der Bericht nennt im `$branch`-Fall ausschließlich `git branch -f <ziel> <sha>` als Rückweg und warnt ausdrücklich vor `ORIG_HEAD`.
- Der Rückwechsel kann scheitern, wenn unversionierte Artefakte der Verifikation mit Dateien des vorherigen Branchs kollidieren → Skill bleibt stehen und meldet es, statt zu erzwingen.
- Ein auf Entscheidung wartender Rebase lässt den Menschen doch auf dem Zielbranch zurück → unvermeidbar, weil ein stehender Rebase nicht verlassen werden kann; der Bericht sagt es und nennt den Rückwechsel als nächsten Schritt.
- Die README hat viele verstreute Sprachstellen; eine übersehene widerspricht den Skills → grep-getriebene Checkliste in Schritt 6, Zeilennummern im Ist-Zustand.
- Die Spracherkennung aus dem Plan kann bei gemischten Plänen kippen → Kopfblock und Überschriften sind maßgeblich, nicht Zitate oder Bezeichner; `plan-lint` bewertet Sprachmischung nicht als Finding (`skills/plan-lint/SKILL.md:22`).

## Annahmen

- **A1** (bestätigt 2026-09-15) — `plan-create` und `plan-vorlage.md` behalten das Sprachargument, weil der Aufruf genau vier Skills nennt → umgeworfen, wenn auch `plan-create` gemeint war: dann kommt ein Schritt für `skills/plan-create/SKILL.md` und die Vorlage hinzu, und Schritt 6 wird größer.
- **A2** (bestätigt 2026-09-15, revidiert am selben Tag nach W1) — ursprünglich: Was die drei Plan-Skills in einen fremdsprachigen Plan schreiben, ist Deutsch. Neue Entscheidung: Die drei Plan-Skills schreiben Bericht, Rückfragen und Historie in der Sprache des Plans; die Erkennung kommt ohne Argument aus Kopfblock und Überschriften. Umgesetzt in Schritt 7.
- **A3** (verworfen 2026-09-15) — ursprünglich: `tool-git-rebase` bleibt nach dem Rebase auf dem Zielbranch. Antwort: nein — wer den Zielbranch angibt, will nichts mit ihm zu tun haben. Der Skill kehrt nach der Verifikation auf den vorher ausgecheckten Branch zurück; Rückweg- und Push-Zeilen nennen den Zielbranch explizit, der wartende Konfliktfall ist die benannte Ausnahme. Eingearbeitet in Entwurf, Schritt 5, Schritt 6 und Teststrategie.
- **A4** (bestätigt 2026-09-15) — Ein nur auf dem Remote vorhandener `$branch` führt zum Abbruch mit Hinweis, der Skill legt keine Branches an → umgeworfen, wenn er den Tracking-Branch anlegen soll: dann kommt `git switch <branch>` als weitere Zustandsänderung in Abschnitt 2 und in den Bericht.
- **A5** (bestätigt 2026-09-15) — Die erste Zeile bleibt eine feste Sprachdirektive „Deutsch", statt ganz zu entfallen — seit der Revision von A2 heißt sie in den Plan-Skills „die Sprache des Plans" → umgeworfen, wenn gar keine Sprachzeile mehr gewünscht ist: dann entfällt sie in Schritt 1 bis 4 ersatzlos, und `plan-coding` behält den Satz zu Code und Bezeichnern an anderer Stelle.

## Offene Fragen

1. Bleibt das Sprachargument in `/plan-create` (A1)? Vorschlag: ja — der Aufruf nennt es nicht, und ein Plan in Projektsprache ist ein anderer Fall als ein Bericht. → betrifft Schritte 6 und ggf. einen neuen Schritt — **Antwort 2026-09-15:** Vorschlag angenommen.
2. Historie-Abschnitte in einem englischen Plan: Deutsch oder Plansprache (A2)? Vorschlag: Deutsch — sonst braucht jeder Skill wieder Spracherkennung. → betrifft Schritte 1, 2, 3 — **Antwort 2026-09-15:** Vorschlag angenommen. **Revidiert 2026-09-15 nach W1:** Plansprache — die drei Plan-Skills folgen der Sprache des Plans (Schritt 7).
3. Nach `/tool-git-rebase main feature/x`: auf `feature/x` bleiben oder zurückwechseln (A3)? Vorschlag: bleiben — ein Endzustand, und `push`/`reset` im Bericht wirken auf den sichtbaren Branch. → betrifft Schritt 5 — **Antwort 2026-09-15:** nein, zurückwechseln — bei Angabe des Branchs will der Mensch mit ihm nichts zu tun haben. Eingearbeitet.
4. `branch` existiert nur auf dem Remote: abbrechen oder Tracking-Branch anlegen (A4)? Vorschlag: abbrechen mit `git switch`-Hinweis — der Skill pflegt keine fremden Branches. → betrifft Schritt 5 — **Antwort 2026-09-15:** Vorschlag angenommen.
5. Erste Zeile: feste Direktive „Deutsch" behalten oder ersatzlos streichen (A5)? Vorschlag: behalten — eine Zeile, die den englischen Plan nicht ins Englische ziehen lässt, und die README-Konvention bleibt wahr. → betrifft Schritte 1 bis 4 und 6 — **Antwort 2026-09-15:** Vorschlag angenommen.

## Review-Historie
### Runde 1 — 2026-09-15 — Urteil: Nachschärfen
- B1 Rückwechsel deckt die frühen Ausgänge nach dem Wechsel nicht ab (`SKILL.md:35`, `:39`, `:40`) — behoben: Weg (b), Wechsel unmittelbar vor `git rebase`, Abschnitt 2 auf `<ziel>` umgestellt (Entwurf „Datenfluss", „Vorprüfungen", „Wechsel"; Schritt 5; Teststrategie Fall 8)
- S1 Rückweg im wartenden Fall bleibt `git rebase --abort`; `git branch -f` gilt nur nach dem Rückwechsel — behoben in Entwurf „Bericht" und Schritt 5 (einschließlich `:124`)
- S2 `README.md:65` (Rückweg per `git reset --hard ORIG_HEAD`) fehlt in Schritt 6 — behoben in Schritt 6
- O1 Testaufruf in der Teststrategie ohne `env -u CLAUDECODE` und `--allowedTools` — offen, nicht übernommen
- O2 `git switch <branch>` als Einzeiler setzt genau ein Remote mit diesem Namen voraus — offen, nicht übernommen
### Runde 2 — 2026-09-15 — Urteil: Umsetzbar
- B1 — behoben, geprüft: Datenfluss, „Vorprüfungen", „Wechsel", „Rückwechsel", Schritt 5 und Teststrategie (Fall 8) stimmen überein; Tabelle und Wellen unverändert, keine Regression
- S1 — behoben, geprüft: „Bericht" und Schritt 5 unterscheiden „während" (`git rebase --abort`) und „danach" (`git branch -f <ziel> <sha>`), einschließlich `:124`
- S2 — behoben, geprüft: `:65` in Ist-Zustand, Schritt 6 Vorgehen und Akzeptanz
- O1, O2 — offen, nicht übernommen
- Kein Blocker, kein offener Sollte-Punkt — Review beendet


## Umsetzungs-Historie
### Schritte 1–4 (W1) — 2026-09-15 — abgeschlossen in 1 Runde
- Geändert: skills/plan-review/SKILL.md, skills/plan-lint/SKILL.md, skills/plan-coding/SKILL.md, skills/tool-git-rebase/SKILL.md
- Verifikation: `claude plugin validate .` grün (bekannte Warnung: keine `version`); Greps der Akzeptanzkriterien treffen nur Direktive, Historie-Erkennung und den Satz „Inhalt, nicht Sprache"; Smoke-Test `/plan-lint-wip <wegwerf-plan> english` (Haiku, temporäre Wip-Kopie) → Bericht auf Deutsch, `english` ohne Wirkung, keine englische Historie
- Keine Findings, keine Abweichung vom Plan
- Erkenntnis: Zeilenangaben zu `skills/tool-git-rebase/SKILL.md` verschieben sich ab Abschnitt 1 um −3 — als Hinweis in Schritt 5 eingetragen
- Erkenntnis: Smoke-Tests für `plan-review` und `plan-coding` (Teststrategie) noch nicht gelaufen — gehören zur Abschlussprüfung in Schritt 6

### Schritt 5 (W2) — 2026-09-15 — abgeschlossen in 2 Runden
- Geändert: skills/tool-git-rebase/SKILL.md
- Verifikation: `claude plugin validate .` grün; die acht Fälle der Teststrategie in Wegwerf-Repos mit Wip-Kopie (Sonnet): (1) ohne `branch` wie bisher; (2) `main feature-b` → Wechsel, Rebase, `make test` → TESTS-OK, zurück auf `feature-a`, Bericht mit Wechselzeile, `git branch -f feature-b <sha>` und `git push --force-with-lease origin feature-b`; (3) unbekannter Branch → Abbruch mit Branchliste; (4) `remote-only` → Abbruch mit `git switch remote-only`, kein lokaler Branch; (5) `main main` → Abbruch; (6) schmutziges Arbeitsverzeichnis → Abbruch, HEAD auf `feature-a`; (7) Entscheidungskonflikt → Rebase steht auf `feature-c`, beide Seiten zitiert, Rückfrage; (8) `main` bereits Vorfahr → „nichts zu tun", kein Wechsel
- F1 Verschachtelter Platzhalter `<alter Stand von <ziel>>` in der Berichtsvorlage — behoben (`<sha, alter Stand des Zielbranchs>`)
- F2 Entscheidungs- und Terminierungsabsatz in Abschnitt 4 nannten nur `git rebase --abort`, nicht den Rückwechsel — behoben; Fall 7 erneut gelaufen, Bericht nennt jetzt `git rebase --abort`, danach `git switch feature-a`
- Keine Abweichung vom Plan
- Erkenntnis: Während eines stehenden Rebase ist HEAD losgelöst, `.git/rebase-merge/head-name` nennt den Zielbranch — wer den Zustand prüft, darf „detached" dort nicht als Fehler lesen

### Schritt 6 (W3) — 2026-09-15 — abgeschlossen in 1 Runde
- Geändert: README.md
- Verifikation: `claude plugin validate .` grün; `[sprache]` nur noch in der `plan-create`-Zeile, `[branch]` in Tabelle (`:17`) und Tool-Abschnitt (`:61`), Schlusssatz `:65` unterscheidet den Branch-Fall; `sprache|language` trifft nur `:13`, `:19`, `:45`, `:47`, `:57`, `:117`, `:118`; Smoke-Tests aus der Teststrategie (Sonnet, Wip-Kopien): `/plan-review-wip <wegwerf-plan> english` → deutscher Bericht, `english` ohne Wirkung, vorhandene Review-Historie als Runde 3 fortgeschrieben statt doppelt angelegt; `/plan-coding-wip hello.md 1 english` im Wegwerf-Repo → deutsche Ausgabe, `hello.txt` angelegt, `make test` → TESTS-OK, `## Umsetzungs-Historie` auf Deutsch
- Keine Findings
- Abweichung: Die Sprachabsätze `:19`, `:45`, `:117`, `:118` wurden nach der Revision von A2 noch in derselben Welle auf die Plansprache der nachgelagerten Skills umgeschrieben — Schritt 7 dokumentiert das als „erledigt in Schritt 6"
- Erkenntnis: `plan-coding` markiert einen erledigten Schritt von sich aus mit ✅ in der Überschrift — passt zur Tabellenmarkierung „✓ erledigt"

### Schritt 7 (W4) — 2026-09-15 — abgeschlossen in 2 Runden
- Geändert: skills/plan-review/SKILL.md, skills/plan-lint/SKILL.md, skills/plan-coding/SKILL.md
- Verifikation: `claude plugin validate .` grün; `grep -n 'Sprache des Plans'` trifft je Skill drei Stellen (Direktive, Absatz „**Sprache.**", Schlusskontrolle), in `tool-git-rebase` keine; Smoke-Test `plan-lint` (Wip-Kopie, Sonnet) gegen deutschen Wegwerf-Plan → deutscher Bericht und `## Lint-Historie`; gegen englischen Wegwerf-Plan → englischer Bericht, `## Lint History` und englische Historienzeile
- F1 Historie-Vorlagen zeigten die deutsche Überschrift wörtlich, der erste englische Lauf schrieb `## Lint-Historie` mit englischer Zeile — behoben: Satz zur übersetzten Überschrift in `plan-lint` (Abschnitt 6), `plan-review` (Abschnitt 7) und `plan-coding` (Abschnitt 5); englischer Lauf wiederholt, Überschrift jetzt `## Lint History`
- Keine Abweichung vom Plan
