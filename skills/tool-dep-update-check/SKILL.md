---
name: tool-dep-update-check
description: Prüft, ob eine Dependency des Projekts von einer Version auf eine Zielversion aktualisiert werden kann — findet sie im Manifest, vermisst den Sprung über die Registry, liest Changelog und Release Notes für den gesamten Bereich, gleicht Bruchstellen mit der tatsächlichen Verwendung im Code ab, prüft Plattform- und Abhängigkeitskonflikte, macht einen Probelauf in einem Wegwerf-Worktree und übergibt ein Urteil mit Belegen, ohne das Projekt zu verändern. Nimmt Dependency, Ausgangsversion und Zielversion als Argumente, optional gefolgt vom Branch, auf dem geprüft wird — ohne Angabe der ausgecheckte Branch.
argument-hint: "[dependency] [sourceVersion] [targetVersion] [branch]"
arguments: dependency sourceVersion targetVersion branch
disable-model-invocation: true
allowed-tools: Bash Read Grep Glob Edit WebFetch WebSearch
---

**Ausgabesprache: Deutsch.** Bericht, Rückfragen und Abbruchmeldungen schreibst du auf Deutsch, auch wenn Projekt, Changelog und Release Notes englisch sind. Zitate aus Changelogs, Bezeichner, Befehle und Fehlermeldungen bleiben, wie sie sind.

Prüfe, ob die Dependency `$dependency` in diesem Projekt von Version `$sourceVersion` auf Version `$targetVersion` aktualisiert werden kann — auf dem Branch `$branch`, oder ohne Angabe auf dem ausgecheckten. Arbeite wie jemand, der vor dem Update-PR die Frage beantwortet, die der PR sonst erst in der Review aufwirft: Was bricht, was muss angepasst werden, was steht im Weg? Du führst das Update nicht durch; du beantwortest, ob und wie es geht.

Drei Regeln tragen den ganzen Ablauf: Du veränderst nichts, was bleibt — kein Commit, keine geänderte Manifest-Datei in der Arbeitskopie, kein Push; der Probelauf geschieht in einem Wegwerf-Worktree, der am Ende verschwindet. Jede Aussage über einen Bruch hat eine Quelle, und jede Aussage „nicht betroffen" hat den Grep, der sie belegt. Und das Urteil folgt den Belegen, nicht der Versionsnummer: Ein Major-Sprung ist nicht automatisch blockiert, ein Patch nicht automatisch harmlos.

## 1. Argumente auflösen

**Dependency, Ausgangs- und Zielversion.** Alle drei sind Pflicht. Fehlt eine, nenne den Aufruf — `/tool-dep-update-check <dependency> <sourceVersion> <targetVersion> [branch]` — und brich ab. Rate keine Version aus der Lock-Datei nach: Wer explizit zwei Versionen nennt, will genau dieses Paar geprüft haben, und wer sie nicht nennt, soll sie nennen. Führende `v`, `^` oder `~` streifst du ab; ein Bereich statt einer Version (`^5.0`, `5.*`) ist kein Ziel — nenne die höchste passende Version aus der Registry als Kandidaten und frag.

Vergleiche die beiden Versionen, bevor du irgendetwas prüfst: Sind sie gleich, gibt es nichts zu tun, sag das. Liegt das Ziel unter dem Ausgang, ist das ein Downgrade — eine andere Aufgabe, bei der Changelogs rückwärts gelesen werden; sag das und frag, ob es so gemeint ist. Wo die Versionen keinem Semver folgen (`1.2.3.RELEASE`, `2.0.0rc1`, Kalenderversionen), vergleichst du mit dem Werkzeug des Ökosystems statt nach Gefühl; die Befehle stehen in `${CLAUDE_SKILL_DIR}/references/oekosysteme.md`.

**Branch.** Der Branch ist der Stand, auf dem geprüft wird — nicht ein Branch, auf den gewechselt würde.

1. Ist `$branch` leer → der ausgecheckte Stand (`HEAD`). Ein losgelöster HEAD ist hier kein Hindernis, weil du nur liest und einen Worktree davon abzweigst; der Bericht nennt dann den Commit statt eines Branchnamens.
2. Existiert `$branch` als lokaler Branch (`git rev-parse --verify refs/heads/<name>`) → nimm ihn. Hat er einen Upstream und liegt dahinter, kommt das als Zeile in den Bericht — geprüft wird der Stand, den der Mensch hat.
3. Existiert er nur auf einem Remote (`origin/<name>`) → `git fetch <remote>` zuerst, dann der Remote-Zweig; notiere, ob er sich beim Holen bewegt hat. Anders als beim Rebase braucht es keinen lokalen Branch: Lesen und einen losgelösten Worktree abzweigen geht von jedem Ref.
4. Existiert er gar nicht → nenne die vorhandenen Branches und brich ab.

Mit `$branch` liest du alles über `git show <ref>:<pfad>` und suchst mit `git grep <muster> <ref> -- <pfade>`, ohne zu wechseln — die Arbeitskopie des Menschen fasst du nicht an. Ohne `$branch` liest du die Arbeitskopie, wie sie ist, samt uncommitteter Änderungen, weil das der Stand ist, der aktualisiert würde. Weichen Manifest oder Lock dabei vom Commit ab (`git status --porcelain -- <manifest> <lock>`), kommt das in den Bericht: Der Probelauf in Abschnitt 6 zweigt vom Commit ab, nicht von der Arbeitskopie.

## 2. Dependency im Projekt finden

Finde die Manifeste: `package.json`, `composer.json`, `pom.xml`, `build.gradle(.kts)` samt `gradle/libs.versions.toml`, `pyproject.toml` und `requirements*.txt`, `Cargo.toml`, `go.mod`, `Gemfile`, `*.csproj` samt `Directory.Packages.props`. Suche per Glob im ganzen Projekt, aber nicht in `node_modules`, `vendor`, `target`, `build`, `dist`, `.venv` — dort liegen die Manifeste fremder Pakete. Das Ökosystem bestimmt Lock-Datei, Registry-Abfragen und Befehle; lies `${CLAUDE_SKILL_DIR}/references/oekosysteme.md`, sobald es feststeht, und nicht erst, wenn ein Befehl fehlt.

**Name auflösen.** Erst der exakte Name (`lodash`, `symfony/console`, `org.springframework.boot:spring-boot-starter-web`, `requests`), dann Teilstring ohne Groß-/Kleinschreibung. Mehrere Treffer → liste sie und brich ab, statt zu raten; `guzzle` trifft `guzzlehttp/guzzle` und `guzzlehttp/psr7`, und eine Prüfung des falschen Pakets ist eine überzeugend aussehende falsche Antwort. Kein Treffer in einem Manifest → sieh in der Lock-Datei nach. Steht sie dort, ist sie eine transitive Dependency: Ihre Version wählt, wer sie verlangt. Die Prüfung fragt dann, welche direkten Dependencies sie hereinziehen und ob jede davon das Ziel zulässt, und endet mit den zwei Wegen — die Eltern anheben oder das Paket selbst als Root-Anforderung festnageln (npm `overrides`, Composer-Root-`require`, Maven `dependencyManagement`, bei Cargo als direkte Abhängigkeit mit festem Constraint); beides tust du nicht, du benennst es. Steht sie auch im Lock nicht → sag das, nenne die durchsuchten Manifeste, brich ab.

Taucht sie in mehreren Manifesten auf (Monorepo, Workspaces, Multi-Module) → prüfe alle. Möglich ist das Update erst, wenn es überall möglich ist; der Bericht trennt nach Manifest. Tragen sie verschiedene Versionen, gilt `$sourceVersion` für die passenden, die anderen listest du mit ihrem Stand.

**Constraint gegen Lock.** Lies den Constraint im Manifest (`^4.17.20`) und die aufgelöste Version im Lock (`4.17.21`). Was tatsächlich läuft, ist die Lock-Version, und sie ist der Ausgangspunkt für Changelogs und Probelauf; `$sourceVersion` bleibt im Berichtskopf stehen. Sind beide gleich, gut. Liegt der Lock zwischen Ausgang und Ziel, ist das Projekt schon ein Stück weit — sag es und miss den Sprung von der Lock-Version. Liegt der Lock unter dem Ausgang oder über dem Ziel, stimmt etwas nicht — falsches Paket, falscher Branch, falsches Manifest —, und du fragst, statt weiterzumachen. Ohne Lock (Maven mit festen Versionen, Projekte ohne Lock-Datei) ist die Manifest-Version der Ausgangspunkt; steht dort ein Bereich, nimm die installierte Version, falls vorhanden, sonst die aktuelle Auflösung des Bereichs, und sag, welche du genommen hast.

Notiere außerdem, ob das Ziel im bestehenden Constraint liegt: Dann ist das Update ein reines Lock-Update ohne Manifest-Änderung, sonst ändert sich das Manifest. Das gehört in die nächsten Schritte des Berichts, weil es den Befehl bestimmt.

## 3. Sprung vermessen

Frag die Registry nach dem Ziel, mit den Abfragen aus der Ökosystem-Referenz. Ein privates Registry (`.npmrc`, Composer-`repositories`, `settings.xml`, `pip.conf`) nimmt das Projekt vor, du nicht.

- **Gibt es das Ziel?** Nein → brich ab und nenne die nächsten existierenden Versionen darunter und darüber; vielleicht ist es ein Tippfehler. Ist es zurückgezogen (yanked, retracted), als deprecated markiert oder eine Vorabversion → das ist ein Befund im Berichtskopf, kein Abbruch.
- **Wie weit ist es?** Veröffentlichungsdaten von Ausgang und Ziel und die Zahl der Versionen dazwischen, getrennt nach Major, Minor, Patch. Drei Jahre zwischen zwei Versionen sagen mehr als die Nummern. Ob es Neueres als das Ziel gibt, kommt als eine Zeile dazu — nicht als Empfehlung, als Information.
- **Was für ein Sprung?** Patch, Minor, Major nach Semver — als Hinweis, nicht als Urteil. Bei `0.x` ist Minor der Bruch; Spring Boot, Django und viele Frameworks brechen in Minors; Kalenderversionen sagen gar nichts. Der Changelog entscheidet.
- **Was verlangt das Ziel?** Laufzeit und Plattform — Node-`engines`, PHP-Version und Extensions, Java-Version, `requires-python`, MSRV, Go-Direktive, Ruby-Version, Zielframework — gegen drei Stände: was das Projekt selbst deklariert (`engines`, `require.php` und `config.platform`, `maven.compiler.release`, `requires-python`, `rust-version`, `go`-Direktive, `.ruby-version`, `TargetFramework`), was die CI und ein Dockerfile tatsächlich verwenden, und was lokal installiert ist. Der Probelauf läuft mit der lokalen Laufzeit; weicht sie von der CI ab, ist ein grüner Probelauf nur für die lokale gültig, und das steht im Bericht.
- **Was zieht es mit?** Die eigenen Abhängigkeiten und Peer-Anforderungen des Ziels gegen die Root-Dependencies des Projekts: Ein Ziel, das `react >= 19` verlangt, während das Projekt auf 18 steht, ist ein Konfliktkandidat, bevor ein Befehl gelaufen ist.
- **Wer hängt noch daran?** Umgekehrte Abhängigkeiten im Projekt (`npm explain`, `composer why` und vor allem `composer why-not <paket> <ziel>`, `mvn dependency:tree -Dincludes=`, `gradle dependencyInsight`, `cargo tree -i`, `go mod why`, `dotnet nuget why`): Ein Plugin, das `^4` verlangt, blockiert den Sprung auf 5, bis es selbst aktualisiert ist. Das ist ein Blocker mit Vorbedingung, kein Nein.

Zwei Sonderfälle: Bei Go ist ein Major ab 2 ein anderer Modulpfad (`…/v2`) — das Update ist dann eine Änderung jedes Importpfads, und Abschnitt 5 listet sie alle. Und bei Cargo können zwei Versionen desselben Crates nebeneinander existieren; der Sprung erzeugt dann keinen Konflikt, sondern ein Duplikat, das erst bricht, wenn Typen der beiden Fassungen aufeinandertreffen.

## 4. Änderungen lesen

Quellen in dieser Reihenfolge: der Migrations- oder Upgrade-Leitfaden (`UPGRADE.md`, `UPGRADE-<major>.md`, ein Kapitel „Migration" oder „Upgrading" in der Dokumentation) → `CHANGELOG.md`, `CHANGES`, `HISTORY` → die Release-Seiten des Repositories (`/releases/tag/<tag>`; das Tag-Schema ist `v1.2.3`, `1.2.3` oder in Monorepos `<paket>@1.2.3` — `git ls-remote --tags` oder die Releases-API zeigen es) → als letztes der Diff zwischen den Tags (`/compare/<a>...<b>`) oder, nach dem Probelauf, der Diff zwischen installiertem Ausgang und installiertem Ziel, beschränkt auf die öffentliche Oberfläche: Typdefinitionen, Exporte, README. Die Repository-URL kommt aus den Registry-Metadaten; WebSearch dient dazu, den Changelog zu finden, nicht als Quelle für seinen Inhalt.

**Lies den ganzen Bereich.** Jede Version nach dem Ausgangspunkt bis einschließlich des Ziels. Ein Sprung von 4.9 auf 5.2 hat seine Brüche in 5.0.0, und genau die übersieht, wer nur die Notizen zu 5.2 aufschlägt. Bei langen Bereichen: Majors und Minors vollständig, Patches nach `breaking`, `security`, `revert`, `deprecat`, `drop` und `remove` überfliegen. Vorabversionen dazwischen tragen oft die Details, die die finale Version nur zusammenfasst.

Sortiere, was du findest, in drei Töpfe — die Muster und Verstecke stehen in `${CLAUDE_SKILL_DIR}/references/bruchstellen.md`, und du liest sie, bevor du den ersten Changelog öffnest, damit du weißt, wonach du suchst:

- **Brüche.** Entfernte oder umbenannte APIs, geänderte Signaturen, geänderte Defaults mit sichtbarer Wirkung, eingestellte Plattformunterstützung, geändertes Konfigurationsformat, entfernte Subpath-Exporte, geänderte Feature-Flags.
- **Deprecations.** Funktioniert noch, warnt, fällt später weg. Relevant, weil sie die Arbeit des nächsten Sprungs sind — und weil ein Projekt, das Warnungen zu Fehlern macht, sie schon jetzt als Bruch erlebt.
- **Verhaltensänderungen ohne API-Bruch.** Nicht als „breaking" markiert, aber Ausgabe, Reihenfolge, Genauigkeit, Fehlerklasse oder Zeitverhalten ändern sich. Der Grep findet sie nicht; nur Tests oder das Lesen der Aufrufstellen tun es.

Jeder Eintrag bekommt seine Quelle: Version, URL oder Datei, eine zitierte Zeile. Ein Bruch ohne Zitat ist eine Vermutung und wird als solche gekennzeichnet.

## 5. Verwendung abgleichen

Bestimme die Oberfläche, die das Projekt von der Dependency tatsächlich benutzt. Der Paketname ist selten das, was im Code steht: `Pillow` importiert sich als `PIL`, `symfony/console` als `Symfony\Component\Console\`, ein Maven-Artefakt als die Java-Pakete im Jar, ein Crate mit Bindestrich als Unterstrich. Die Zuordnung je Ökosystem steht in der Referenz. Dann suchst du — mit `$branch` per `git grep … <ref>`, sonst in der Arbeitskopie, immer ohne die Vendor-Verzeichnisse, damit du nicht den Code der Dependency selbst zählst:

- Import-, Require- und Use-Anweisungen, einschließlich Subpath-Importen (`<paket>/…`) und reinen Typ-Importen.
- Mittelbare Verwendung: Konfiguration in YAML, XML, JSON oder Umgebungsvariablen, die Klassen, Optionen oder Plugins der Dependency nennt; Framework-Verdrahtung über Service-Definitionen, Attribute, Annotationen, Dekoratoren; Build-Werkzeuge und Testaufbau, die sie als Plugin oder Preset laden; Klassennamen in Zeichenketten; ihre Kommandozeile in Skripten, Makefile und CI.
- Re-Exporte durch eigene Module: Ein Wrapper im Projekt, der die Dependency einmal importiert, hat viele Aufrufer, die sie nie nennen — folge dem Wrapper.

Dann jeder Bruch aus Abschnitt 4 gegen diese Oberfläche: **betroffen** mit `Datei:Zeile`, und zwar alle Stellen, nicht die erste — oder **nicht betroffen** mit dem Grep-Muster, das leer geblieben ist. Das Muster gehört in den Bericht, weil ein „nicht verwendet" ohne Muster nicht nachprüfbar ist. Bei Verhaltensänderungen liest du die Aufrufstellen und sagst, ob die Änderung dort ankommt und ob ein Test es bemerken würde.

Ist die Dependency das Framework der Anwendung — Symfony, Spring Boot, Django, Rails, Angular —, ist die Oberfläche die ganze Anwendung, und der Abgleich per Grep trägt nicht. Dann ist der Weg über Deprecations der wichtigste: Lass die Tests auf dem Ausgangsstand mit sichtbaren Deprecations laufen (die Schalter je Ökosystem stehen in der Referenz); was der Ausgang schon anmahnt, entfernt das Ziel. Gibt es für genau diesen Sprung ein Migrationswerkzeug — Rector-Sets, OpenRewrite-Rezepte, `ng update`, Framework-Codemods —, lass es im Worktree aus Abschnitt 6 laufen; sein Diff ist die Liste der nötigen Anpassungen. Es ist eine Befundquelle, nicht das Urteil.

## 6. Probelauf

Die statische Prüfung sagt, was brechen müsste; der Probelauf sagt, was bricht. Beides braucht es: Der Changelog verschweigt Undokumentiertes, und der Probelauf übersieht, was die Tests nicht abdecken.

**Worktree anlegen.** `git worktree add --detach <tmp>/dep-check-<paket> <ref>` — `<tmp>` aus `mktemp -d`, `<ref>` ist `HEAD` oder der aufgelöste `$branch`. `--detach`, weil Git einen Branch nicht zweimal auschecken lässt und weil du keinen Branch bewegen willst. Hat das Projekt Submodule, `git -C <worktree> submodule update --init --recursive`. Ab hier läuft jeder verändernde Befehl im Worktree — mit `-C <worktree>` oder von dort aus —, und vor jedem prüfst du den Pfad; die Arbeitskopie des Menschen ist Lesezone. Unversionierte Dateien wie `.env` oder lokale Konfiguration sind im Worktree nicht da, und du kopierst sie nicht: Sie können auf echte Systeme zeigen. Scheitert der Ausgangsstand daran, ist das ein Umgebungsbefund, kein Urteil über das Update.

**Reihenfolge:**

1. **Ausgangsstand installieren**, mit dem Lock wie es ist (`npm ci`, `composer install`, `mvn dependency:resolve`, `cargo fetch`, `go mod download`, `bundle install`, `dotnet restore`, `uv sync` oder `poetry install`). Schlägt das fehl — Toolchain fehlt, kein Netz, privates Registry ohne Zugang —, bleibt die Prüfung statisch: Melde es unter „Nicht geprüft" und mach mit Abschnitt 7 weiter. Was lokal fehlt, sagt `command -v`.
2. **Auflösung im Trockenlauf**, wo das Werkzeug einen hat (`npm install <paket>@<ziel> --dry-run`, `composer update <paket> --with <paket>:<ziel> --dry-run`, `pip install --dry-run`, `cargo update -p <paket> --precise <ziel> --dry-run`, …): Er zeigt Konflikte, bevor ein langer Install läuft. Erst die strenge Variante, die nur das Paket und seine eigenen Abhängigkeiten bewegt; dann die, die auch andere Root-Dependencies mitziehen darf (Composer `-w`, dann `-W`; bei Cargo und npm passiert das von selbst). Jedes Paket, das mitwandern muss, kommt in den Bericht — ob das in Ordnung ist, entscheidet der Mensch. `--legacy-peer-deps`, `--force`, `--ignore-platform-reqs` und Verwandte sind keine Auflösung, sondern verstecken den Konflikt; ein Probelauf, der nur damit durchgeht, meldet den Konflikt als Blocker.
3. **Update anwenden**, exakt auf das Ziel und nicht als Bereich: `^5.2.0` könnte 5.2.3 auflösen, und dann prüfst du eine andere Version als die gefragte. Die Befehle je Ökosystem stehen in der Referenz; bei Manifesten ohne Befehl (`pom.xml`, `build.gradle`, `requirements.txt`) änderst du die Versionsangabe in der Worktree-Kopie und lässt den Install laufen. Prüfe danach, dass Lock und Installation wirklich das Ziel halten, und lies den Lock-Diff (`git -C <worktree> diff --stat`): Welche anderen Pakete haben sich bewegt?
4. **Build, Typprüfung, Linter**, wie das Projekt sie vorsieht — `CLAUDE.md`, `README`, `package.json`-Skripte, `Makefile`, CI-Konfiguration. Compiler- und Typfehler sind die schärfsten Befunde: `Datei:Zeile`, welche API. Dann die **Tests**, die ganze Suite; ist sie sehr lang, zuerst die Module der betroffenen Oberfläche, dann der Rest, und der Bericht sagt, was gelaufen ist. Deprecation-Ausgaben während der Tests sammelst du: Das ist, was das Ziel jetzt anmahnt, also die Arbeit für den übernächsten Sprung.
5. **Rot?** Erst klären, ob es am Update liegt: Stell im Worktree Manifest und Lock zurück (`git -C <worktree> checkout -- <manifest> <lock>`, erneut installieren) und lass nur die roten Tests noch einmal laufen. Auch auf dem Ausgangsstand rot → nicht die Schuld des Updates, als Vorbestand melden. Nur mit dem Update rot → Befund, mit der Fehlermeldung in einer Zeile und der Bruchstelle aus Abschnitt 4, zu der er gehört — oder mit „undokumentiert", wenn kein Changelog-Eintrag passt; das ist ein eigener Befund und einen Satz mehr wert.
6. **Nicht reparieren.** Du passt im Worktree keinen Code an, damit der Probelauf grün wird; das ist der Update-PR, nicht die Prüfung. Eine Ausnahme: Blockiert eine einzelne offensichtliche Anpassung — ein umbenannter Import — alles Weitere, sodass kein anderer Befund erreichbar ist, darfst du sie im Worktree machen, um den Rest zu sehen, und sagst genau, was du geändert hast; im Bericht steht sie als nötige Anpassung. Mehr als das: aufhören, der Mensch migriert mit dem Bericht in der Hand.
7. **Aufräumen.** `git worktree remove --force <worktree>`, dann `git worktree prune`, und `git worktree list` muss wieder zeigen, was vorher da war. Ein vergessener Worktree fällt erst auf, wenn `git switch` den Branch als anderswo ausgecheckt meldet. Die Caches der Paketmanager liegen außerhalb des Repos und bleiben — dafür sind sie da.

**Terminierung:** Kommt der Probelauf in drei Anläufen aus Umgebungsgründen nicht durch Install und Build — nicht wegen des Updates —, hörst du auf, räumst den Worktree ab und legst dar, was du versucht hast und was dadurch ungeprüft bleibt. Das statische Urteil steht dann mit geringerer Belastbarkeit. Ein Probelaufergebnis wird nie erfunden, und ein Testlauf, der nicht zu Ende kam, wird als nicht zu Ende gekommen gemeldet.

## 7. Urteil und Übergabe

Das Urteil hat drei Stufen, und es wird aus den Belegen abgeleitet, nicht aus dem Bauchgefühl:

- **Möglich** — kein Bruch trifft die verwendete Oberfläche, kein Konflikt, Probelauf grün. Ohne Probelauf steht „nur statisch" daneben.
- **Möglich mit Anpassungen** — das Update geht, aber Code oder Konfiguration müssen sich ändern; jede Anpassung mit `Datei:Zeile` und dem, was zu tun ist.
- **Blockiert** — etwas steht im Weg, das dieses Update allein nicht löst: eine Plattformanforderung, eine andere Root-Dependency, die das Paket festhält, ein Ziel, das es nicht gibt, ein Go-Major mit neuem Modulpfad. Immer mit dem, was vorher passieren müsste — ein Blocker mit Vorbedingung ist eine Reihenfolge, kein Nein.

Getrennt davon die **Belastbarkeit:** Probelauf grün, Probelauf rot oder nur statisch geprüft — und ob die Tests die betroffene Oberfläche überhaupt berühren. Ein grüner Probelauf auf ungetestetem Code beweist nur, dass es kompiliert.

```
# Dependency-Update: <dependency> <sourceVersion> → <targetVersion> — Möglich | Möglich mit Anpassungen | Blockiert

**Geprüft auf:** <branch> @ <sha> — ausgecheckt | ohne Wechsel, per `git show`, `git grep` und Worktree
**Belastbarkeit:** Probelauf grün (<Befehle>) | Probelauf rot: <n> Tests | nur statisch — <warum>; Tests decken die betroffene Oberfläche ab | nicht ab
**Fundstelle:** <manifest>:<Zeile> `<constraint>` — Lock hält <version> (= sourceVersion | abweichend: <was>)
**Sprung:** <patch | minor | major> — <n> Versionen dazwischen (<Majors/Minors>), <source> vom <Datum>, <target> vom <Datum>; neuer als das Ziel: <version> | Ziel ist aktuell
**Voraussetzungen:** Ziel verlangt <Laufzeit> — Projekt <Stand>, CI <Stand>, lokal <Stand> → erfüllt | nicht erfüllt: <was>
**Auflösung:** ohne weitere Änderungen | zieht mit: <paket a→b, …> | Konflikt mit <paket> (verlangt <constraint>)

## Bruchstellen
### <API oder Verhalten> — seit <version>
- **Änderung:** <ein Satz, mit Quelle: URL oder Datei, zitierte Zeile>
- **Betroffen:** <Datei:Zeile, …> | nicht verwendet — geprüft mit `<grep-muster>`
- **Anpassung:** <was zu tun ist> | keine

## Deprecations
- <was ab dem Ziel angemahnt wird und wo das Projekt es benutzt — die Arbeit für den nächsten Sprung>

## Verhaltensänderungen ohne API-Bruch
- <Default, Reihenfolge, Fehlerklasse … — Aufrufstelle, und ob ein Test es bemerken würde>

## Probelauf
- Install: <Befehl> → <Ergebnis>
- Build/Typen/Lint: <Befehl> → <Ergebnis>
- Tests: <Befehl> → <n bestanden, m rot>; rot: <Test> — <Fehler in einer Zeile> → <Bruchstelle X | auch auf Ausgangsstand rot | undokumentiert>
- Mitgezogen im Lock: <paket a→b, …>
- Hinweise: <Advisories gegen das Ziel, Deprecation-Ausgabe, Lock-Format …>

## Nicht geprüft
- <was nicht geprüft werden konnte, warum — und was das für das Urteil heißt>

## Nächste Schritte
- <der Befehl für das echte Update in der Arbeitskopie — den führst nicht du aus>
- <Dateien, die anzupassen sind, in der Reihenfolge, in der es Sinn ergibt>
- <bei Blockiert: was vorher passieren muss — `<paket>` auf <version>, Laufzeit auf <version>>
```

Leere Abschnitte bekommen eine Zeile „keine", damit sichtbar bleibt, dass sie geprüft wurden. Der Bericht ist die Ausgabe; ins Projekt schreibst du nichts, es sei denn, der Mensch bittet darum.

## 8. Haltung

Die erste Versuchung ist, aus der Versionsnummer zu urteilen. „Patch, passt schon" und „Major, geht nicht" sind beides Vermutungen; die Prüfung ist so viel wert, wie du gelesen hast.

Die zweite ist, nur die Release Notes des Ziels zu lesen. Der Bruch steht in der ersten Version nach dem Ausgangspunkt, und genau die überspringt, wer nur ans Ende schaut.

Die dritte ist, nach einem Grep auf den Import „nicht verwendet" zu schreiben. Konfiguration, Zeichenketten, Framework-Verdrahtung, Subpath-Importe und eigene Wrapper sind die Orte, an denen Updates in Produktion scheitern. Nenne das Muster — kannst du es nicht nennen, hast du nicht geprüft.

Die vierte ist, das Update durchzuführen, weil es „fast dieselbe Arbeit" ist. Es ist eine Änderung am Projekt des Menschen. Die Prüfung endet mit einem Bericht und einer unveränderten Arbeitskopie; der PR ist seine Sache.

Die fünfte ist, den Worktree grün zu reparieren. Jede Korrektur, die du dort machst, ist ein Befund, den du versteckst. Berichte die nötigen Anpassungen, statt sie zu machen — abgesehen von der einen Ausnahme in Abschnitt 6, und die steht dann im Bericht.

Die sechste ist, einem grünen Probelauf zu trauen. Grün heißt: Was die Tests abdecken, funktioniert. Sag, was sie nicht abdecken, besonders wenn die betroffene Oberfläche in dieser Lücke liegt.

Die siebte ist, dem Update anzulasten, was vorher schon rot war. Vergleiche mit dem Ausgangsstand, bevor du „bricht" schreibst.

Die achte ist, den Worktree stehen zu lassen. Er liegt unter `/tmp`, niemand sieht ihn — bis `git switch` sagt, der Branch sei anderswo ausgecheckt. Entfernen, prunen, nachsehen.

Und die Antwort auf „geht das?" darf „ja, aber nicht allein" lauten. Wenn der Sprung verlangt, dass erst andere Pakete oder die Laufzeit nachziehen, ist das nicht blockiert, sondern eine Reihenfolge — und die schreibst du als Reihenfolge auf.
