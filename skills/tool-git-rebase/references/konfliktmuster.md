# Konfliktmuster

Wiederkehrende Konflikte beim Rebase und ihre Behandlung. Für jedes gilt dieselbe Reihenfolge: erst mit `git show REBASE_HEAD` und `git log -p <merge-base>..<basis> -- <datei>` beide Absichten lesen, dann handeln. Und die Zuordnung nicht vergessen: `ours` ist beim Rebase die Basis, `theirs` der Commit des Branchs.

---

## A — Mechanische Konflikte

Beide Seiten haben recht, nur der Text passt nicht zusammen. Das ist die Mehrheit aller Konflikte, und hier geht am meisten durch Nachlässigkeit verloren.

- **Benachbarte Zeilen.** Beide Seiten fügen an derselben Stelle etwas ein — Importe, Listeneinträge, Enum-Werte, Routen, Abhängigkeiten. Beides behalten, in der Reihenfolge, die die Umgebung vorgibt (alphabetisch, nach Gruppe). Danach auf Dubletten prüfen: Oft haben beide Seiten denselben Import hinzugefügt.
- **Umbenennung auf der Basis, Verwendung im Commit.** Die Basis hat eine Funktion, ein Feld, eine Klasse umbenannt; der Commit benutzt den alten Namen. Den neuen Namen nehmen — und dann den gesamten Diff des Commits nach weiteren Verwendungen des alten Namens absuchen. An Stellen, die die Basis nicht angefasst hat, gibt es keinen textuellen Konflikt, aber denselben Fehler. Das ist der klassische semantische Konflikt, den erst der Compiler oder der Test findet.
- **Verschobener Code.** Die Basis hat einen Block in eine andere Datei oder an eine andere Stelle verschoben; der Commit ändert den Block am alten Ort. Die Änderung des Commits am neuen Ort anbringen; der alte Ort bleibt so, wie die Basis ihn hinterlassen hat.
- **Formatierungslauf auf der Basis.** Die Basis hat die ganze Datei neu formatiert (Prettier, Black, gofmt, PHP-CS-Fixer); der Commit hat ein paar Zeilen geändert. Die Fassung der Basis nehmen, die inhaltliche Änderung des Commits erneut anbringen, den Formatierer über diese Datei laufen lassen. Nie Zeile für Zeile von Hand angleichen — der Formatierer entscheidet, und er ist reproduzierbar.
- **Leerraum und Zeilenenden.** `git diff --ignore-all-space` zeigt, ob überhaupt eine inhaltliche Differenz besteht. Auflösen zur Konvention der Basis.

---

## B — Generierte und abgeleitete Dateien

Werden nicht von Hand zusammengeführt. Die Quelle wird aufgelöst, das Abgeleitete neu erzeugt.

- **Lock-Dateien** — `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `composer.lock`, `Cargo.lock`, `poetry.lock`, `Gemfile.lock`, `go.sum`. Erst die Fassung der Basis nehmen (`git checkout --ours -- <lock>`; ours ist hier die Basis), dann das Manifest (`package.json`, `composer.json`, …) von Hand auflösen, dann die Lock-Datei mit dem Werkzeug des Projekts neu erzeugen — `npm install --package-lock-only`, `composer update --lock`, `cargo generate-lockfile`, je nach Ökosystem — und stagen. Kontrolle: `git diff --cached -- <lock>` darf nur die Abhängigkeiten berühren, die der Commit im Manifest geändert hat. Fällt die Erzeugung aus (kein Netz, kein Werkzeug), sag das und lass den Rebase stehen.
- **Generierter Code** — API-Clients, Protobuf, GraphQL-Typen, ORM-Schnappschüsse, kompilierte Assets im Repo. Quelle auflösen, Generator laufen lassen, Ergebnis stagen. Ohne Generator keine Auflösung — melden, nicht raten.
- **Test-Snapshots.** Zuerst den Code auflösen, dann die Snapshots nur der betroffenen Tests aktualisieren lassen. Danach den Snapshot-Diff lesen: Er darf nur die Änderung zeigen, die der Commit beabsichtigt hat. Ein Snapshot, der nebenbei etwas anderes festschreibt, ist ein neuer Fehler mit grünem Häkchen.

---

## C — Strukturelle Konflikte

Eine Seite hat die Datei umgebaut, nicht nur geändert.

- **Gelöscht gegen geändert.** Basis hat gelöscht (`deleted by us`), Commit hat geändert: Wohin ist der Inhalt gewandert? Ist er verschoben, kommt die Änderung des Commits an den neuen Ort und der alte Pfad wird mit `git rm` bestätigt. Ist er ersatzlos weg, ist die Änderung des Commits vermutlich erledigt — aber erst prüfen, was sie tat: Ein Bugfix in einer gelöschten Datei wird oft an anderer Stelle weiter gebraucht. Umgekehrt (`deleted by them`, der Commit hat gelöscht, die Basis geändert): Die Löschung bleibt in der Regel, aber die Änderung der Basis muss dort landen, wohin der Commit den Inhalt verlagert hat.
- **Umbenannt.** Git erkennt Umbenennungen meist, aber nicht immer; dann erscheint ein Paar aus Löschung und Neuanlage mit Konflikt. `git log --follow` und `git diff -M` helfen bei der Zuordnung. Die Änderungen kommen an den umbenannten Pfad.
- **Beide neu angelegt** (`both added`). Zwei verschiedene Dateien unter demselben Pfad — fast immer zwei verschiedene Absichten. Tut die Datei der Basis, was der Commit wollte, entfällt die des Commits. Ist es etwas anderes, braucht sie einen anderen Namen, und das ist eine Entscheidung: fragen.
- **Datenbank-Migrationen.** Beide Seiten haben eine Migration angelegt; Nummern oder Zeitstempel kollidieren, oder die Schema-Stände laufen auseinander. Die Migration der Basis bleibt unangetastet — sie ist womöglich schon irgendwo gelaufen. Die Migration des Commits bekommt eine Nummer oder einen Zeitstempel nach ihr und muss auf dem Schema-Stand der Basis aufsetzen. Frameworks mit Zeiger auf die letzte Migration (Django `dependencies`, Rails `schema.rb`, Prisma-Migrationssperre) brauchen den Zeiger mit nachgezogen.
- **Registrierungs- und Sammeldateien** — Routen, Container-Konfiguration, Feature-Flags, CHANGELOG. Beide Seiten hängen an; beides behalten, in der Ordnung der Datei. Beim CHANGELOG kommt der Eintrag des Commits unter „Unreleased", auch wenn die Basis inzwischen eine Version geschnitten hat.

---

## D — Was keine Konfliktauflösung ist

Hier entscheidet der Mensch. Beide Seiten zitieren, sagen, was jede will, den eigenen Vorschlag mit Begründung nennen, fragen, den Rebase stehen lassen.

- Beide Seiten ändern dasselbe Verhalten unterschiedlich, und beides ist für sich gültig — etwa zwei verschiedene Validierungsregeln für dasselbe Feld.
- Die Basis hat eine Funktion entfernt, die der Commit erweitert.
- Die Tests des Commits behaupten ein Verhalten, das die Basis absichtlich geändert hat.
- Binärdateien — Bilder, PDFs, Kompilate — sind auf beiden Seiten geändert. Es gibt keine Mitte; eine Seite muss gewählt werden, und das steht dir nicht zu.
