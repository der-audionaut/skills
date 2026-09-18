# Bruchstellen

Wo Brüche stehen, woran man sie erkennt, wie schwer sie wiegen — und was keiner ist. Für alles gilt: erst die Quelle, dann der Abgleich mit der Verwendung im Projekt, dann die Einstufung. Ein Bruch ohne Zitat ist eine Vermutung; ein „nicht betroffen" ohne Grep-Muster ist keine Prüfung.

---

## A — Wo sie stehen

- **Migrations- und Upgrade-Leitfäden zuerst.** `UPGRADE.md`, `UPGRADE-<major>.md`, `MIGRATION.md`, ein Kapitel „Upgrading" oder „Migration guide" in der Dokumentation. Sie sind für genau diese Frage geschrieben und nennen die Reihenfolge der Anpassungen. Fehlen sie für einen Major-Sprung, ist das selbst ein Hinweis auf den Reifegrad des Pakets.
- **Changelog des Repositories.** `CHANGELOG.md`, `CHANGES`, `HISTORY.rst`, `NEWS`. In Monorepos liegt er im Unterordner des Pakets (`packages/<paket>/CHANGELOG.md`), und der Tag heißt `<paket>@<version>`.
- **Release-Seiten.** `/releases/tag/<tag>` und die Releases-API. Manche Projekte pflegen nur diese, manche nur den Changelog, manche generieren die Notizen aus Commit-Betreffs — dann sind `feat!:` und `BREAKING CHANGE:` die Marker.
- **Der Diff zwischen den Tags.** `/compare/<tag-ausgang>...<tag-ziel>`, beschränkt auf öffentliche Oberfläche: Typdefinitionen (`*.d.ts`, `py.typed`-Stubs, Header), Exportlisten, README, Beispiele. Nach dem Probelauf liegt das Ziel installiert im Worktree, der Ausgang in der Arbeitskopie: `diff -r` der beiden Paketverzeichnisse ist dann der Diff ohne Netz.
- **Die Vorabversionen dazwischen.** `5.0.0-rc.1` trägt oft die Einzelheiten, die `5.0.0` zu „siehe RC-Notizen" zusammenfasst.
- **Die Deprecations des Ausgangs.** Was die Ausgangsversion schon als deprecated markiert — in Typen, Docstrings, Laufzeitwarnungen —, ist die Liste dessen, was der nächste Major entfernt. Das steht oft nirgends als „removed", sondern nur als „deprecated" zwei Versionen früher.

## B — Woran man sie erkennt

**Ausdrücklich markiert.** `BREAKING`, `⚠️`, `!:` in Betreffzeilen, „removed", „dropped", „no longer", „renamed", „now requires", „minimum … version", „is now the default", „throws … instead of", Abschnitte „Breaking changes", „Migration", „Upgrading". Das ist der leichte Teil.

**Nicht markiert, aber Bruch.** Die Verstecke, in denen Updates in Produktion scheitern:

- **Geänderte Defaults.** „Strict mode is now enabled by default", „connection pooling on by default", „trailing slashes are no longer normalized". Der Code kompiliert, die API ist dieselbe, das Verhalten nicht. Aufrufstellen lesen: Wird die Option explizit gesetzt? Wenn nicht, gilt jetzt der neue Default.
- **Strengere Validierung.** „Now rejects invalid …", „now throws on unknown keys". Tolerante Eingaben, die vorher durchgingen, werden zu Fehlern — meist erst in Produktion, wo die Eingaben unordentlicher sind als in den Tests.
- **Geänderte Rückgabe- und Fehlerarten.** `null` wird zu Exception, Array zu Iterator, synchron zu asynchron (Promise, Future), eine Fehlerklasse zu einer anderen. Tests, die auf Fehlermeldungen oder Klassennamen prüfen, sind die ersten Opfer, und `catch`-Blöcke auf den alten Typ fangen still nichts mehr.
- **Reihenfolge, Genauigkeit, Format.** Sortierung nicht mehr stabil, Gleitkommarundung anders, Zeitstempel mit statt ohne Zeitzone, JSON-Schlüsselreihenfolge, Datumsformat, Log-Format. Bricht Snapshots, Vergleiche, nachgelagerte Parser.
- **Modulformat und Auslieferung.** ESM-only ohne CommonJS, entfernte `exports`-Subpaths (`<paket>/lib/…` gibt es nicht mehr), umbenannte Einstiegspunkte, kein Typ-Bundle mehr (Typen ausgelagert in `@types/…` oder umgekehrt), entfernte Browser-Builds.
- **Eingestellte Plattformen — mit Nebenwirkung.** „Dropped Node 16 / PHP 8.0 / Python 3.8" heißt oft: Das Paket benutzt jetzt Syntax, die die alte Laufzeit nicht parst, und scheitert beim Laden, nicht beim Aufruf. Bei TypeScript-Paketen kann eine neuere `typescript`-Version nötig sein, um die Typen zu lesen.
- **Nur Typen geändert.** Strengere Generics, `readonly`, entfernte Überladungen. Laufzeit unverändert, `tsc`, `mypy`, `phpstan` rot. Ein Bruch für den Build, nicht für die Anwendung — trotzdem eine Anpassung.
- **Peer- und Mitanforderungen angehoben.** Das Ziel verlangt `react >= 19`, `symfony/http-kernel ^7`, `tokio 1.40`. Der Bruch liegt nicht im Paket, sondern in dem, was es mitzieht.
- **Konfigurationsschema.** Umbenannte Optionen, geänderte Plugin-API, neues Format der Konfigurationsdatei, umbenannte Property-Namen (Spring Boot), umbenannte Umgebungsvariablen, neue Pflichtfelder. Grep auf die alten Namen in YAML, JSON, XML, `.env*`, Property-Dateien.
- **Kommandozeile.** Flags umbenannt oder entfernt, geänderter Exit-Code, geänderte Ausgabe, die Skripte parsen. Grep in `package.json`-Skripten, `Makefile`, CI, Shell-Skripten.
- **Mitgelieferte Migrationen und Dateilayout.** ORMs und Frameworks liefern Datenbankmigrationen; Asset-, Template- oder Fixture-Pfade ändern sich. Ein Update, das eine Migration verlangt, ist möglich, aber nicht ohne Deployment-Schritt — der gehört in die nächsten Schritte.
- **Feature-Flags und Compile-Optionen.** Cargo-Features umbenannt, Build-Tags geändert, optionale Extras (`paket[extra]`) umgestellt.
- **Sicherheitsfixes, die Verhalten verschärfen.** Ein CVE-Fix, der Parsing strenger macht, ist für tolerante Eingaben ein Bruch. Er ist trotzdem richtig — der Bericht nennt ihn als Anpassung, nicht als Grund zu bleiben.
- **Lizenzwechsel.** Nicht technisch, aber ein Blocker für viele Projekte: Wechsel zu AGPL, BUSL, SSPL oder einer kommerziellen Lizenz. Steht im Changelog, in `LICENSE`, in den Registry-Metadaten.
- **Reihenfolge der Entfernung.** Was das Ziel entfernt, war vorher deprecated. Wenn der Ausgang die Deprecation noch nicht hatte — weil sie erst in einer Zwischenversion kam —, hat das Projekt sie nie gesehen. Deshalb den ganzen Bereich lesen.

## C — Wie schwer sie wiegen

- **Blocker.** Trifft die verwendete Oberfläche und lässt sich innerhalb dieses Updates nicht lösen: Laufzeit oder Plattform nicht erfüllt, eine andere Root-Dependency hält das Paket fest, das Ziel existiert nicht oder ist zurückgezogen, ein anderer Modulpfad (Go-Major), ein Lizenzwechsel, den das Projekt nicht tragen kann. Immer mit der Vorbedingung, die den Blocker auflösen würde.
- **Anpassung.** Trifft die verwendete Oberfläche, und der Weg ist bekannt: Umbenennung nachziehen, Option explizit setzen, Fehlerklasse anpassen, Konfiguration umstellen, Migration einplanen. Jede mit `Datei:Zeile` und dem, was dort zu tun ist. Zehn Stellen derselben Umbenennung sind eine Anpassung mit zehn Fundstellen, nicht zehn Anpassungen.
- **Hinweis.** Trifft die Oberfläche nicht (mit Grep-Muster belegt), oder ist eine Deprecation, die noch funktioniert, oder eine Verhaltensänderung, die das Projekt nach Lesen der Aufrufstellen nicht erreicht. Steht im Bericht, damit es beim nächsten Sprung nicht neu recherchiert wird.

Bei transitiven Dependencies verschiebt sich die Frage: Nicht „verträgt das Projekt das Ziel", sondern „verträgt jedes Paket, das es verlangt, das Ziel" — und das steht in deren Constraints und Changelogs, nicht im Code des Projekts.

Bei Frameworks kehrt sich das Verhältnis um: Nicht die Bruchliste gegen den Code halten, sondern den Code gegen die Deprecation-Ausgabe des Ausgangsstands und den Diff des Migrationswerkzeugs. Die Bruchliste eines Framework-Majors ist zu lang, um sie Punkt für Punkt zu greppen, und das Werkzeug kennt sie besser.

## D — Was kein Bruch ist

Nicht aufblasen. Ein Bericht mit zwanzig Punkten, von denen siebzehn das Projekt nicht betreffen, wird nicht gelesen.

- Interne Umbauten, Performance-Verbesserungen, neue Features, Dokumentation, Änderungen am Build des Pakets selbst.
- Entfernte oder geänderte APIs, die das Projekt nicht benutzt — belegt mit dem Muster. Als Hinweis in einer Zeile, nicht als Bruchstelle mit drei Unterpunkten.
- Deprecations, solange das Projekt Warnungen nicht zu Fehlern macht. Sie sind der Abschnitt „Deprecations", nicht „Bruchstellen".
- Angehobene Peer- oder Mitanforderungen, die das Projekt bereits erfüllt.
- Ein rotes Ergebnis, das auf dem Ausgangsstand ebenso rot ist.
- Änderungen zwischen Ausgang und Lock-Version, wenn der Lock schon weiter ist als `$sourceVersion`: Die hat das Projekt bereits.
