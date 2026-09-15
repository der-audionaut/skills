---
name: tool-git-rebase
description: Setzt den ausgecheckten oder einen benannten Branch per Rebase auf einen Basis-Branch neu auf — holt dessen aktuellen Stand, prüft die Ausgangslage, löst Konflikte Commit für Commit so auf, dass die Absicht beider Seiten erhalten bleibt, verifiziert das Ergebnis mit den Befehlen des Projekts und übergibt mit einem Bericht, ohne zu pushen. Nimmt den Basis-Branch als Argument, optional gefolgt vom Branch, der neu aufgesetzt wird — ohne Angabe der ausgecheckte Branch.
argument-hint: "[basebranch] [branch]"
arguments: basebranch branch
disable-model-invocation: true
allowed-tools: Bash Read Grep Glob Edit
---

**Ausgabesprache: Deutsch.** Bericht, Rückfragen und Abbruchmeldungen schreibst du auf Deutsch, auch wenn Repository und Commit-Botschaften englisch sind. Commit-Botschaften und Code folgen dem Bestand, nicht dieser Vorgabe.

Setze den Zielbranch — `$branch`, oder ohne Angabe den ausgecheckten Branch — per Rebase auf `$basebranch` neu auf. Arbeite wie jemand, der fremde Commits durch fremde Änderungen trägt: Jeder Commit soll hinterher genau das tun, was sein Autor wollte — nur eben auf dem neuen Stand der Basis. Du baust hier nichts Neues und räumst nichts nebenbei auf.

Drei Regeln tragen den ganzen Ablauf: Du pushst nicht. Du änderst weder Reihenfolge noch Botschaft noch Zuschnitt der Commits. Und jeder Konflikt wird gelesen, bevor er aufgelöst wird — eine Seite pauschal zu übernehmen ist keine Auflösung, sondern ein verstecktes Zurücksetzen.

## 1. Basis-Branch und Zielbranch auflösen

**Basis-Branch.** `$basebranch` ist der Branch, auf den der Zielbranch aufgesetzt wird — als lokaler Name (`main`) oder mit Remote (`origin/main`). Prüfe mit `git rev-parse --verify`, was es davon gibt, und dann:

1. Ist `$basebranch` leer → nenne die Kandidaten, die es gibt (der Upstream des Zielbranchs, `main`, `master`, `develop`), und frag, welcher gemeint ist. Rate nicht: Ein Rebase auf den falschen Branch ist rückholbar, aber der Weg zurück kostet Vertrauen.
2. Existiert der Name nur auf einem Remote (`origin/<name>`) → nimm den Remote-Zweig.
3. Existiert er gar nicht → nenne die vorhandenen Branches und brich ab.

**Zielbranch.** Der Zielbranch ist der Branch, der neu aufgesetzt wird. Er muss lokal existieren, weil der Rebase ihn auscheckt — ein Remote-Zweig kann nur Basis sein. Prüfe mit `git rev-parse --verify refs/heads/<name>`, und dann:

1. Ist `$branch` leer → der ausgecheckte Branch ist der Zielbranch (`git symbolic-ref --quiet --short HEAD`; liefert das nichts, ist kein Branch ausgecheckt, und Abschnitt 2 bricht ab).
2. Existiert `$branch` als lokaler Branch → das ist der Zielbranch.
3. Existiert der Name nur auf einem Remote (`origin/<name>`) → brich ab und nenne den Einzeiler, der ihn lokal anlegt: `git switch <name>`. Branches anzulegen ist nicht deine Aufgabe, der Rebase ist es.
4. Existiert er gar nicht → nenne die vorhandenen Branches und brich ab.
5. Zeigen Zielbranch und Basis-Branch auf denselben Ref → brich ab: Ein Branch lässt sich nicht auf sich selbst aufsetzen.

## 2. Ausgangslage prüfen

Bevor `git rebase` läuft, muss die Lage eindeutig sein. Jeder der folgenden Punkte ist ein Abbruchgrund — melde ihn und hör auf, statt zu reparieren:

- **Arbeitsverzeichnis nicht sauber.** `git status --porcelain` zeigt geänderte oder gestagte Dateien. Nenne sie und schlag Commit oder Stash vor; du stashst nicht selbst, weil ein vergessener Stash schlimmer ist als ein abgebrochener Rebase. Unversionierte Dateien stören nur, wenn die Basis eine Datei desselben Pfads mitbringt — dann ebenso.
- **Ein Rebase, Merge oder Cherry-Pick läuft bereits.** `git status` sagt es, `.git/rebase-merge`, `.git/rebase-apply` oder `.git/MERGE_HEAD` bestätigen es. Fremde Zwischenstände aufzuräumen steht dir nicht zu.
- **Kein Branch ausgecheckt** (detached HEAD). Das gilt auch mit `$branch`: Von einem losgelösten Commit wegzuwechseln kann Commits unerreichbar machen, und ob die jemand braucht, weißt du nicht — die eine Zeile `git switch` vorher macht der Mensch.

Dann hol den Stand — immer, nicht nur wenn `main` alt aussieht, denn ob er alt ist, weiß man erst danach: `git fetch <remote>` für das Remote, auf dem der Basis-Branch liegt, und notiere, ob sich `<remote>/<basis>` dabei bewegt hat; das kommt als Zeile in den Bericht. Ein Rebase auf einen veralteten lokalen `main` löst die Konflikte von gestern und bringt morgen dieselben noch einmal. Hat der lokale Basis-Branch einen Upstream, vergleiche beide Stände: gleich → nimm den lokalen Namen; lokal hinterher → nimm den Upstream (`origin/main`) und sag das im Bericht; lokal voraus, also unveröffentlichte Commits auf der Basis → frag, welcher Stand gemeint ist. Den lokalen Basis-Branch bewegst du dabei nicht: Du setzt auf `origin/main` auf, statt erst `main` nachzuziehen — der Rebase ist deine Aufgabe, die Pflege fremder Branches nicht.

Nun der Blick auf die Arbeit, die neu aufgesetzt wird — alles auf `<ziel>` und ohne Branchwechsel; ohne `$branch` ist `<ziel>` der ausgecheckte Branch, und gegenüber dem gewohnten Ablauf ändert sich nichts:

- `git merge-base <basis> <ziel>` und `git log --oneline <basis>..<ziel>` — das sind die Commits, die gleich neu geschrieben werden. Ist die Basis bereits Vorfahr von `<ziel>` (`git merge-base --is-ancestor <basis> <ziel>`), gibt es nichts zu tun: sag das und hör auf.
- Hat der Zielbranch einen Upstream, wird der Rebase veröffentlichte Historie umschreiben; der Push braucht danach `--force-with-lease`. Das ist kein Abbruchgrund, gehört aber in den Bericht. Ein Abbruchgrund ist es, wenn der Upstream Commits hat, die lokal fehlen (`git log --oneline <ziel>..<ziel>@{u}` ist nicht leer): Erst die holen, sonst überschreibt der spätere Push fremde Arbeit.

Sichere den Ausgangspunkt, bevor du etwas veränderst: `git rev-parse <ziel>` für den Bericht, und den Diff des Branchs gegen die alte Merge-Base in eine Datei außerhalb des Repos, etwa `git diff $(git merge-base <basis> <ziel>) <ziel> > /tmp/rebase-<ziel>-vorher.diff`. Git setzt beim Rebase außerdem `ORIG_HEAD`; das ist der Weg zurück, solange der Zielbranch ausgecheckt ist — nach einem Rückwechsel nicht mehr, siehe Abschnitt 5.

## 3. Rebase ausführen

Ist der Zielbranch nicht ausgecheckt, wechsle jetzt — und erst jetzt: Notiere den ausgecheckten Branch für den Rückweg, dann `git switch <ziel>`. Schlägt der Wechsel fehl, brich ab und melde es; bis hierhin hast du nichts verändert. Der Wechsel steht so spät, damit alle Prüfungen und Rückfragen davor auf dem Branch des Menschen enden, ohne dass etwas zurückgewechselt werden müsste.

`GIT_EDITOR=true git rebase <basis>` — schlicht, ohne `-i`, ohne Autosquash, ohne Umsortieren. Das vorangestellte `GIT_EDITOR=true` verhindert, dass ein Editor aufgeht und der Lauf hängt. Läuft der Rebase ohne Halt durch, weiter mit Abschnitt 5.

Hält er an, ist ein Konflikt da. `git status` zeigt die betroffenen Dateien, `git show --stat REBASE_HEAD` den Commit, der gerade angewendet wird.

## 4. Konflikte auflösen

Erst verstehen, dann ändern. Für jeden Halt:

**Die Seiten richtig zuordnen.** Beim Rebase ist es umgekehrt zu dem, was man vom Merge kennt: `ours` — `HEAD`, der obere Block ab `<<<<<<<` — ist die **Basis** samt allem, was bereits neu aufgesetzt wurde; `theirs` — `REBASE_HEAD`, der untere Block bis `>>>>>>>` — ist der **Commit des Branchs**, der gerade angewendet wird. Wer das verwechselt, wirft mit `--ours` die eigene Arbeit weg und glaubt, sie behalten zu haben.

**Beide Absichten ermitteln.** `git show REBASE_HEAD` zeigt, was der Commit wollte; `git log -p <merge-base>..<basis> -- <datei>` zeigt, was die Basis an derselben Stelle wollte und warum. Lies die Commit-Botschaften — sie sind die einzige Begründung, die du bekommst.

**Auflösen heißt: beide Absichten überleben.** Die Änderung der Basis bleibt, und der Commit tut auf dem neuen Stand dasselbe wie vorher — angepasst an neue Namen, neue Signaturen, verschobenen Code. Hat die Basis das Anliegen des Commits inzwischen selbst erledigt, darf seine Seite entfallen; das belegst du mit der Stelle in der Basis, statt es zu vermuten. Wollen beide Seiten an derselben Stelle etwas anderes, und beides ist für sich gültig, dann ist das keine Konfliktauflösung, sondern eine Entscheidung: Leg beide Seiten mit Zitat vor, sag, was du vorschlagen würdest und warum, frag, und lass den Rebase bis zur Antwort stehen. Nenne dabei den Ausweg `git rebase --abort` — und, wenn du für den Rebase gewechselt hast, den Rückwechsel `git switch <vorheriger Branch>`, der nach `--continue` wie nach `--abort` folgt.

Wiederkehrende Muster — Lock-Dateien, generierter Code, Importblöcke, gelöscht gegen geändert, Umbenennungen, Formatierungsläufe, kollidierende Migrationen — stehen mit ihrer Behandlung in `${CLAUDE_SKILL_DIR}/references/konfliktmuster.md`. Lies die Datei beim ersten Konflikt, nicht erst beim dritten.

**Bevor du weitergehst:**

- Die Änderung der Basis ist noch da: `git diff <merge-base> <basis> -- <datei>` zeigt, was die Basis an dieser Datei geändert hat, und jede dieser Zeilen muss in deiner Auflösung stehen — neue Namen, neue Signaturen, neue Aufrufe. Fehlt eine, hast du die Basis zurückgesetzt; genau das passiert, wenn man den unteren Block einfach übernimmt, weil er „die eigene Arbeit" ist.
- Die Änderung des Commits ist noch da: `git show REBASE_HEAD -- <datei>` gegen deine Fassung halten. Seine Absicht muss erkennbar bleiben, auch wenn sie jetzt andere Namen trägt.
- Kein Marker mehr in den betroffenen Dateien: `grep -n '^<<<<<<<\|^=======\|^>>>>>>>' <dateien>`. Ein vergessener Marker ist der peinlichste Commit, den ein Rebase erzeugen kann.
- `git diff` gegen den Index anschauen: Ist wirklich nur das geändert, was der Konflikt verlangte?
- `git add <dateien>`, dann `GIT_EDITOR=true git rebase --continue`.
- Meldet Git, der Commit sei leer geworden, prüfe mit `git diff --cached`: Ist der Index leer, steckt die Änderung schon in der Basis, und `git rebase --skip` ist richtig — notiere den übersprungenen Commit für den Bericht. Ist der Index nicht leer, ist etwas anderes los, und du liest erneut.

Dann der nächste Halt, bis der Rebase durch ist. Derselbe Konflikt kommt in mehreren Commits wieder, wenn der Branch dieselbe Stelle mehrfach angefasst hat; löse ihn jedes Mal konsistent zur ersten Auflösung, denn Git merkt sich das nicht für dich.

**Terminierung:** Bleibt derselbe Commit nach drei Anläufen ungelöst — weil jede Auflösung neue Konflikte oder Fehler nach sich zieht — hörst du auf, lässt den Rebase stehen und legst den Stand vor: welcher Commit, welche Dateien, was du versucht hast, wie es weitergeht (`git rebase --continue` nach eigener Auflösung oder `git rebase --abort`, danach gegebenenfalls `git switch <vorheriger Branch>`). Ein halb fertiger Rebase mit ehrlicher Meldung ist besser als einer, der mit erfundenen Auflösungen durchläuft.

## 5. Verifizieren

Ein Rebase ohne Konfliktmarker ist noch nicht richtig. Compiler und Tests sehen Konflikte, die Git nicht sieht: Die Basis hat eine Funktion umbenannt, der Branch ruft sie an einer neuen Stelle unter dem alten Namen auf — kein textueller Konflikt, aber kaputt.

- **Ausführen, was das Projekt vorsieht.** Build, Tests, Linter, Typprüfung — wie in `CLAUDE.md`, `README`, `package.json`, `Makefile` oder der CI-Konfiguration beschrieben. Verifizieren heißt ausführen: Eine Verifikationszeile im Bericht, die keinen ausgeführten Befehl mit seinem Ergebnis nennt, ist keine. Läuft etwas nicht, sag das und benenne, was dadurch ungeprüft bleibt.
- **Den Diff vergleichen.** `git diff <basis> HEAD` gegen den gesicherten `vorher.diff`: Der Unterschied darf nur dort liegen, wo du Konflikte aufgelöst hast, und dort, wo die Basis dieselben Dateien geändert hat. Eine Datei, die vorher im Branch-Diff war und jetzt fehlt, ist eine verlorene Änderung — es sei denn, du hast den zugehörigen Commit belegt übersprungen.
- **Die Commits zählen.** `git log --oneline <basis>..HEAD` muss dieselben Betreffzeilen in derselben Reihenfolge zeigen wie vor dem Rebase, abzüglich der übersprungenen.

Schlägt eine Prüfung fehl, kläre zuerst, ob der Rebase die Ursache ist: Betrifft sie Dateien, die beide Seiten geändert haben? Schlug sie auf dem Ausgangspunkt auch schon fehl? Ist der Rebase die Ursache, behebe es in einem eigenen, klar benannten Commit obenauf — in den ursprünglichen Commit käme die Korrektur nur mit einem interaktiven Rebase, den du hier nicht hast, und das sagst du im Bericht. War die Prüfung vorher schon rot, melde es und fasse es nicht an: Das ist nicht deine Baustelle.

Hast du in Abschnitt 3 gewechselt und läuft kein Rebase mehr — er ist durch, gegebenenfalls samt Reparatur-Commit —, dann zurück: `git switch <vorheriger Branch>`. Wer den Zielbranch angibt, will mit ihm nichts zu tun haben; auf ihm stehen zu bleiben wäre eine Überraschung. Schlägt der Rückwechsel fehl, etwa weil unversionierte Artefakte der Verifikation mit Dateien des vorherigen Branchs kollidieren, bleib stehen, melde es und erzwinge nichts. Wartet der Rebase dagegen auf eine Entscheidung, bleibt der Zielbranch ausgecheckt: Einen stehenden Rebase kann man nicht verlassen, ohne ihn abzubrechen — der Bericht sagt das und nennt den Rückwechsel als nächsten Schritt.

## 6. Übergeben

```
# Rebase: <ziel> auf <basis> — abgeschlossen | wartet auf Entscheidung | abgebrochen

**Ausgecheckt:** <vorheriger Branch> — für den Rebase nach <ziel> gewechselt und zurück | steht auf <ziel>, Rebase wartet   (Zeile entfällt ohne `$branch`)
**Ausgangspunkt:** <sha, alter Stand des Zielbranchs> — zurück mit `git reset --hard <sha>` (auch als `ORIG_HEAD`) | nach dem Rückwechsel: `git branch -f <ziel> <sha>`, nicht `git reset --hard ORIG_HEAD`
**Geholt:** `git fetch <remote>` → <remote>/<basis> unverändert | von <sha> auf <sha> bewegt
**Basis:** <basis> @ <sha> — <n> Commits, die der Branch vorher nicht hatte
**Neu aufgesetzt:** <m> Commits, <k> übersprungen
**Verifikation:** <Befehl> → <Ergebnis>

## Konflikte
### <Datei> — in <sha> „<Betreff>"
- **Basis wollte:** <ein Satz>
- **Commit wollte:** <ein Satz>
- **Auflösung:** <was jetzt drinsteht und warum>

## Übersprungen
- <sha> „<Betreff>" — bereits in der Basis durch <sha der Basis>

## Offen
- <Entscheidung, die der Mensch treffen muss / Prüfung, die nicht laufen konnte>

## Nächste Schritte
- <`git push --force-with-lease`, sofern der Zielbranch einen Upstream hat — den Push machst nicht du; nach einem Rückwechsel mit Branchnamen: `git push --force-with-lease <remote> <ziel>`>
- <wartet der Rebase: `git rebase --continue` nach eigener Auflösung oder `git rebase --abort`, danach `git switch <vorheriger Branch>`>
```

Jeder Konflikt bekommt seinen Eintrag. Der Bericht ist die einzige Stelle, an der nachvollziehbar bleibt, warum eine Stelle jetzt so aussieht — die Commits selbst zeigen die Auflösung nicht mehr, weil sie aussehen, als hätte der Autor es gleich so geschrieben.

Mit `$branch` verdient der Rückweg besondere Sorgfalt. `ORIG_HEAD` zeigt nach dem Rückwechsel weiter auf den alten Stand des Zielbranchs, aber `git reset --hard ORIG_HEAD` trifft den ausgecheckten Branch — den des Menschen — und würde ihn zerstören. Darum nennt der Bericht `git branch -f <ziel> <sha>`; solange der Rebase steht, bleibt der Rückweg `git rebase --abort`. Und der Push braucht den Branchnamen, weil `git push` ohne Refspec den ausgecheckten Branch nähme.

## 7. Haltung

Die erste Versuchung ist `--ours` oder `--theirs`. Es geht schnell, die Marker sind weg, und eine Seite ist stillschweigend zurückgesetzt — beim Rebase bemerkt das erst, wer die Änderung vermisst.

Die zweite ist, im Konflikt nebenbei aufzuräumen: Formatierung, eine Umbenennung, ein Fehler, der einem auffällt. Jede Änderung, die nicht der Konflikt verlangt, macht den Rebase unlesbar; wer ihn prüft, kann Auflösung und Eigenmächtigkeit nicht mehr trennen. Beobachtungen gehören in den Bericht, nicht in den Commit.

Die dritte ist, einen nach dem Rebase roten Test zu „reparieren", indem man ihn anpasst. Der Test ist womöglich der Einzige, der den semantischen Konflikt bemerkt hat.

Die vierte ist, zu pushen, weil alles grün ist. Der Rebase schreibt veröffentlichte Historie um, und das ist die Entscheidung des Menschen — mit `--force-with-lease`, nie mit `--force`.

Und der Rebase ist jederzeit rückholbar: `git rebase --abort` während, `git reset --hard ORIG_HEAD` danach — oder `git branch -f <ziel> <sha>`, wenn du schon auf den vorherigen Branch zurückgewechselt bist —, `git reflog` immer. Schreib das in den Bericht, statt zu hoffen, dass es bekannt ist.
