# Ökosysteme

Je Ökosystem dieselben Fragen: Wo steht die Dependency, was läuft tatsächlich, was weiß die Registry über das Ziel, wer hängt noch daran, was verlangt es an Laufzeit, wie wird exakt auf das Ziel gesetzt, und wie heißt es im Code. Die Befehle hier sind Startpunkte, keine Garantien — Flags wandern zwischen Versionen. Prüfe ein Flag, das du nicht sicher kennst, mit `--help`, bevor du es einsetzt; ein unbekanntes Flag ist ein Fehler deines Aufrufs, kein Befund gegen das Update. Alles, was schreibt, läuft im Worktree.

Versionen vergleichen, wo kein Semver gilt: npm `npx semver -r '<bereich>' <version>`, Composer `composer show -a <paket>` listet sortiert, Python `python3 -c "from packaging.version import Version as V; print(V('a') < V('b'))"`, Maven sortiert `search.maven.org` chronologisch, Go `go list -m -versions` liefert aufsteigend.

---

## JavaScript und TypeScript — npm, pnpm, yarn

- **Manifest / Lock:** `package.json` / `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`. Workspaces (`workspaces` in `package.json`, `pnpm-workspace.yaml`) bedeuten mehrere Manifeste. `dependencies`, `devDependencies`, `peerDependencies`, `overrides` bzw. `resolutions` unterscheiden.
- **Läuft tatsächlich:** `npm ls <paket>` / `pnpm ls <paket>` / `yarn why <paket>`; `node_modules/<paket>/package.json`. Mehrere Versionen nebeneinander sind hier normal — der Bericht sagt, welche das Projekt direkt sieht.
- **Registry:** `npm view <paket> versions --json`; `npm view <paket>@<ziel> engines peerDependencies dependencies deprecated repository.url`; Daten mit `npm view <paket> time --json`. Funktioniert auch in pnpm- und yarn-Projekten; private Registries stehen in `.npmrc`.
- **Wer hängt daran:** `npm explain <paket>` (`npm why`), `pnpm why <paket>`, `yarn why <paket>`. Peer-Anforderungen anderer Pakete an dieses: `npm ls <paket>` zeigt die Kanten; ein `ERESOLVE` im Trockenlauf nennt den Verursacher.
- **Laufzeit:** `engines.node` im Projekt, `.nvmrc`, `.node-version`, `volta`, `node-version` in den Workflows; lokal `node -v`. Ein Ziel, das nur noch ESM ausliefert (`"type": "module"`, kein `main`/`require`-Export), bricht `require()` und ungewöhnlich konfigurierte Jest-Läufe — das steht im Changelog meist als „ESM only".
- **Trockenlauf:** `npm install <paket>@<ziel> --dry-run`. pnpm: `pnpm add <paket>@<ziel> --lockfile-only`; yarn Berry: `yarn add <paket>@<ziel> --mode=update-lockfile` schreibt nur den Lock — im Worktree ist das der Trockenlauf.
- **Exakt setzen:** `npm install <paket>@<ziel> --save-exact`, `pnpm add <paket>@<ziel> --save-exact`, `yarn add <paket>@<ziel> --exact`; `--save-dev` bzw. `-D`, wenn es eine Dev-Dependency ist, sonst wandert sie in `dependencies`. Liegt das Ziel im Constraint, reicht `npm update <paket>` — nur Lock, kein Manifest.
- **Ausgangsstand:** `npm ci`, `pnpm install --frozen-lockfile`, `yarn install --frozen-lockfile` (yarn 1) bzw. `yarn install --immutable` (Berry).
- **Im Code:** `import … from '<paket>'`, `from '<paket>/…'` (Subpath — die `exports`-Karte des Ziels entscheidet, ob er noch existiert), `require('<paket>')`, `import type`, `jest.mock('<paket>')`, `vi.mock`; Konfigurationsdateien, die Plugins und Presets nennen (`eslint`, `babel`, `vite`, `next`, `tailwind`, `postcss`); `package.json`-Skripte mit `npx <paket>`.
- **Deprecations:** meist `console.warn` zur Laufzeit; `@deprecated` in Typen meldet `tsc` nicht von selbst — eine ESLint-Regel für Deprecations, falls konfiguriert. Typprüfung nach dem Update: `npx tsc --noEmit`. Advisories: `npm audit` nach dem Install. Framework-Codemods (Next.js, React, Angular `ng update`) im Worktree laufen lassen; ihr Diff ist die Anpassungsliste.

## PHP — Composer

- **Manifest / Lock:** `composer.json` / `composer.lock`; `require` gegen `require-dev`.
- **Läuft tatsächlich:** `composer show <vendor/paket>`; `vendor/<vendor>/<paket>/composer.json`.
- **Registry:** `composer show -a <vendor/paket>` (verfügbare Versionen, achtet auf die `repositories` des Projekts), `composer show -a <vendor/paket> <ziel>` (Anforderungen genau dieser Version). Packagist direkt: `https://repo.packagist.org/p2/<vendor>/<paket>.json` — alle Versionen mit `require`, `time`, `source.url`.
- **Wer hängt daran:** `composer why <vendor/paket>`. Und die Frage der Prüfung in einem Befehl: `composer why-not <vendor/paket> <ziel>` nennt jedes Paket und jede Plattformanforderung, die das Ziel verhindert.
- **Laufzeit:** `require.php` und `require.ext-*` des Projekts; `config.platform.php` überschreibt für die Auflösung die lokale PHP-Version — mit gesetztem `platform` ist die lokale Version für die Auflösung egal, für den Probelauf nicht. Lokal `php -v`, `php -m`. Ein Ziel, das eine höhere PHP-Version verlangt, zeigt sich in `why-not`, nicht erst zur Laufzeit.
- **Trockenlauf:** `composer update <vendor/paket> --with <vendor/paket>:<ziel> --dry-run`; erst so (nur das Paket), dann `-w` (auch seine Abhängigkeiten, die keine Root-Anforderungen sind), dann `-W` (auch Root-Anforderungen). Die Ausgabe listet „Upgrading x (a => b)" — das sind die mitgezogenen Pakete.
- **Exakt setzen:** `composer require <vendor/paket>:<ziel>` (mit `-w`/`-W`, wenn nötig; `--dev`, wenn es eine Dev-Dependency ist — sonst wandert sie nach `require`). Liegt das Ziel im Constraint: `composer update <vendor/paket>`.
- **Ausgangsstand:** `composer install`. Danach `composer validate` und `composer show <vendor/paket>` als Kontrolle.
- **Im Code:** Namensräume aus `vendor/<vendor>/<paket>/composer.json` → `autoload.psr-4`, dann `use <Namespace>\…`, FQCN in Zeichenketten und Konfiguration (`config/services.yaml`, `config/packages/*.yaml`, `config/bundles.php`, XML), Attribute und Annotationen, Twig- oder Blade-Erweiterungen, `bin/`-Befehle in `composer.json`-Skripten.
- **Deprecations:** Symfony — `SYMFONY_DEPRECATIONS_HELPER` und der Bericht der `phpunit-bridge` am Ende des Testlaufs; PHPUnit 10+ `--display-deprecations`; allgemein `E_USER_DEPRECATED`. Statische Analyse nach dem Update fängt Signaturänderungen ohne Laufzeit: `vendor/bin/phpstan analyse`, `vendor/bin/psalm`. Rector im Worktree mit dem Set des Sprungs (`vendor/bin/rector process --dry-run`) liefert die Anpassungsliste. Advisories: `composer audit`.

## Java und Kotlin — Maven

- **Manifest = Lock:** `pom.xml`; Versionen oft in `<properties>` oder über eine Parent-BOM (`spring-boot-starter-parent`, `dependencyManagement`), Multi-Module mit Versionen im Root. `mvn help:effective-pom` zeigt, was wirklich gilt.
- **Läuft tatsächlich:** `mvn dependency:tree -Dincludes=<gruppe>:<artefakt>`; `mvn dependency:list`.
- **Registry:** `https://search.maven.org/solrsearch/select?q=g:"<gruppe>"+AND+a:"<artefakt>"&core=gav&rows=200&wt=json` (Versionen mit Zeitstempel); das POM des Ziels unter `https://repo1.maven.org/maven2/<gruppe/mit/schrägstrichen>/<artefakt>/<ziel>/<artefakt>-<ziel>.pom` (seine Abhängigkeiten); `mvn versions:display-dependency-updates -Dincludes=<gruppe>:<artefakt>`. Privates Nexus oder Artifactory über `settings.xml`; Existenz dann mit `mvn dependency:get -Dartifact=<gruppe>:<artefakt>:<ziel>`.
- **Wer hängt daran:** `mvn dependency:tree -Dverbose -Dincludes=<gruppe>:<artefakt>` zeigt, wer es hereinzieht und was „omitted for conflict" ist. Maven scheitert nicht an Versionskonflikten, es nimmt die nächstgelegene Version — das Risiko ist der stille Versionsmix. `maven-enforcer-plugin` mit `dependencyConvergence`, falls konfiguriert, macht ihn laut.
- **Laufzeit:** `maven.compiler.release` / `source` / `target`, `<java.version>`, Toolchains; lokal `java -version`, `mvn -v`. Die Java-Anforderung des Ziels steht in den Release Notes — oder im Bytecode: `javap -verbose -cp <jar> <eine.Klasse> | grep 'major version'` (52 = Java 8, 55 = 11, 61 = 17, 65 = 21).
- **Exakt setzen:** `mvn versions:use-dep-version -Dincludes=<gruppe>:<artefakt> -DdepVersion=<ziel> -DforceVersion=true`, oder `mvn versions:set-property -Dproperty=<property> -DnewVersion=<ziel>`, wenn die Version in einer Property steht — oder die `pom.xml` im Worktree editieren. Wird die Version von einer BOM gesetzt, überschreibt man die Property der BOM (`<jackson-bom.version>` etc.) und prüft mit `dependency:tree`, ob die Überschreibung greift.
- **Ausgangsstand / Bauen:** `./mvnw` oder `mvn -q dependency:resolve`; dann `mvn verify`. Deprecations beim Kompilieren: `-Dmaven.compiler.showDeprecation=true`.
- **Im Code:** Java-Pakete des Artefakts aus `unzip -l <jar> | grep '\.class$'` → `import <paket>.…`; Spring: Auto-Konfiguration, Property-Namen in `application.yml` (umbenannte Properties sind der klassische Spring-Boot-Bruch — `spring-boot-properties-migrator` meldet sie), Annotationen, `META-INF/spring.factories` und `AutoConfiguration.imports`.
- **Migration:** OpenRewrite mit `mvn org.openrewrite.maven:rewrite-maven-plugin:dryRun` und dem Rezept des Sprungs (Spring Boot, JUnit, Jakarta) — der Diff ist die Anpassungsliste.

## Java und Kotlin — Gradle

- **Manifest:** `build.gradle(.kts)`, `settings.gradle(.kts)`, Versionskatalog `gradle/libs.versions.toml`, `buildSrc`; Lock nur mit aktiviertem Dependency Locking (`gradle.lockfile`).
- **Läuft tatsächlich / wer hängt daran:** `./gradlew dependencyInsight --dependency <artefakt> --configuration runtimeClasspath` — gewählte Version, Grund, Constraints und wer sie verlangt, in einem Befehl. `./gradlew dependencies --configuration runtimeClasspath` für das ganze Bild.
- **Registry:** wie Maven.
- **Laufzeit:** `java.toolchain.languageVersion`, `sourceCompatibility`, `kotlin { jvmToolchain(…) }`; die Gradle-Version des Wrappers gegen das, was das Ziel verlangt — Gradle-*Plugins* haben eigene Gradle-Mindestversionen (`https://plugins.gradle.org/plugin/<id>/<version>`).
- **Exakt setzen:** Version im Katalog oder Build-Skript des Worktrees ändern; `./gradlew --refresh-dependencies dependencies`; mit Locking `./gradlew dependencies --write-locks`.
- **Bauen:** `./gradlew build`; `--warning-mode all` zeigt Gradle-eigene Deprecations, relevant, wenn die Dependency ein Gradle-Plugin ist.

## Python — pip, pip-tools, Poetry, uv, Pipenv

- **Manifest / Lock:** `pyproject.toml` (`[project] dependencies`, `[dependency-groups]`, `[tool.poetry.dependencies]`), `requirements.in` → `requirements.txt` (pip-tools: `.in` ist Manifest, `.txt` ist Lock), `poetry.lock`, `uv.lock`, `Pipfile` / `Pipfile.lock`, `setup.py` / `setup.cfg`, `constraints.txt`.
- **Läuft tatsächlich:** `pip show <paket>`, `uv pip show <paket>`, `poetry show <paket>`.
- **Registry:** `https://pypi.org/pypi/<paket>/json` (alle Releases mit `upload_time`, `yanked`), `https://pypi.org/pypi/<paket>/<ziel>/json` → `info.requires_python`, `info.requires_dist`, `info.project_urls` (Changelog, Source), `info.yanked`; `pip index versions <paket>`. Privater Index über `PIP_INDEX_URL` oder `pip.conf`. Ob das Ziel ein Wheel für Plattform und Python-Version mitbringt, zeigen die `urls[]` — ohne Wheel baut pip aus dem Quelltext, und dann braucht der Probelauf Compiler.
- **Wer hängt daran:** `pip show <paket>` → `Required-by`; `pipdeptree -r -p <paket>` (im Worktree-venv nachinstallieren ist in Ordnung); `poetry show --tree`; `uv tree --invert --package <paket>`.
- **Laufzeit:** `requires-python` des Projekts, `.python-version`, `tox.ini`, CI-Matrix; lokal `python3 --version`.
- **Trockenlauf:** `pip install '<paket>==<ziel>' --dry-run` (`--report -` für JSON); `poetry add '<paket>==<ziel>' --dry-run`; pip-tools: `.in` editieren, `pip-compile --dry-run`. uv schreibt bei `uv add` sofort — im Worktree ist das gleichbedeutend mit dem Trockenlauf; die Ausgabe listet, was sich bewegt.
- **Exakt setzen:** frisches venv im Worktree (`python3 -m venv .venv`), dann `pip install '<paket>==<ziel>'` und den Pin in `requirements.txt` ändern; `poetry add '<paket>==<ziel>'`; `uv add '<paket>==<ziel>'`; `pipenv install '<paket>==<ziel>'`.
- **Ausgangsstand:** `pip install -r requirements.txt`, `poetry install`, `uv sync`, `pipenv sync`.
- **Im Code:** Importname ist nicht der Distributionsname — `Pillow` → `PIL`, `beautifulsoup4` → `bs4`, `PyYAML` → `yaml`, `scikit-learn` → `sklearn`. Zuordnung über `top_level.txt` oder `RECORD` im `<dist>.dist-info/` des site-packages, oder `python3 -c "import importlib.metadata as m; print(m.packages_distributions())"`. Dann `import <modul>`, `from <modul> import`, dazu Zeichenketten: Django `INSTALLED_APPS` und `settings.py`, pytest-Plugins (`-p`, `conftest.py`), Entry Points, Alembic- und Celery-Konfiguration.
- **Deprecations:** `python3 -W error::DeprecationWarning -m pytest` macht sie zu Fehlern — das schärfste Werkzeug für Framework-Sprünge; `-W default::DeprecationWarning` zeigt sie nur. Typprüfung nach dem Update mit `mypy` oder `pyright`, wenn das Ziel Typen mitbringt. Advisories: `pip-audit`.

## Rust — Cargo

- **Manifest / Lock:** `Cargo.toml` (Workspace: `[workspace.dependencies]`), `Cargo.lock`. Feature-Flags gehören zum Manifest: `default-features = false` und die Feature-Liste.
- **Läuft tatsächlich:** `cargo pkgid <crate>`; `cargo tree -d` zeigt Duplikate — zwei Versionen desselben Crates sind erlaubt und brechen erst, wenn Typen beider Fassungen aufeinandertreffen (`expected Foo, found Foo`). Ein Major-Sprung erzeugt oft genau das, wenn ein anderes Crate die alte Version festhält.
- **Registry:** `cargo info <crate>@<ziel>` (Cargo ≥ 1.82: `rust-version`, Abhängigkeiten, Repository); `https://crates.io/api/v1/crates/<crate>` (Versionen mit `rust_version`, `yanked`, `created_at` — die API verlangt einen `User-Agent`-Header), `…/<crate>/<ziel>/dependencies`. Quelldiff zweier Versionen: `https://diff.rs/<crate>/<ausgang>/<ziel>`.
- **Wer hängt daran:** `cargo tree -i <crate>`; mit Features: `cargo tree -e features -i <crate>`.
- **Laufzeit:** `rust-version` (MSRV) des Ziels gegen `rust-toolchain.toml` und `rustc --version`; `edition`. Umbenannte oder entfernte Features sind ein Cargo-eigener Bruch — der Build meldet sie als unbekanntes Feature.
- **Trockenlauf:** `cargo update -p <crate> --precise <ziel> --dry-run` — nur, wenn das Ziel im Constraint liegt; sonst zuerst `Cargo.toml` im Worktree ändern.
- **Exakt setzen:** in `Cargo.toml` `<crate> = "=<ziel>"` (Features beibehalten), dann `cargo update -p <crate> --precise <ziel>`. Hat das Crate in `Cargo.toml` einen anderen Namen (`package = "…"`), zählt der.
- **Ausgangsstand / Bauen:** `cargo fetch`, dann `cargo build --all-targets`, `cargo test`, `cargo clippy`. Deprecations sind Compiler-Warnungen (`#[deprecated]`); der Build des Ausgangsstands listet, was jetzt schon angemahnt wird.
- **Im Code:** `use <crate_name>::…` mit Unterstrichen statt Bindestrichen; Makros und `#[derive(…)]` aus dem Crate; Feature-Verwendung in `Cargo.toml`. Advisories: `cargo audit` oder `cargo deny`, falls installiert.

## Go — Modules

- **Manifest / Lock:** `go.mod` / `go.sum`; `go.work` für Workspaces; `// indirect` markiert transitive Module.
- **Läuft tatsächlich:** `go list -m <modul>`, `go list -m all`.
- **Registry:** `go list -m -versions <modul>`; `go list -m -json <modul>@<ziel>` (`Time`, `GoVersion`); Proxy direkt: `https://proxy.golang.org/<modul>/@v/list`, `…/@v/<ziel>.info`, `…/@v/<ziel>.mod` (die `go`-Direktive und `require` des Ziels). Zurückgezogene Versionen: `go list -m -retracted <modul>`; Deprecation des Moduls: `go list -m -u -json <modul>` → `Deprecated`.
- **Wer hängt daran:** `go mod why -m <modul>`; `go mod graph | grep ' <modul>@'`.
- **Laufzeit:** `go`-Direktive des Ziels gegen die des Projekts und `go version`. Seit Go 1.21 lädt `toolchain` eine passende Toolchain selbst nach — der Probelauf kann so stillschweigend mit einer anderen Go-Version laufen als die CI; im Bericht nennen.
- **Major ab 2:** anderer Modulpfad (`…/v2`), also ein anderes Modul. Das „Update" ist die Änderung jedes Importpfads (`grep -rn '"<modul>' --include=*.go`), und die Prüfung listet sie alle.
- **Exakt setzen / Trockenlauf:** einen echten Trockenlauf gibt es nicht; im Worktree `go get <modul>@<ziel>`, `go mod tidy`, und `git -C <worktree> diff go.mod go.sum` ist die Antwort auf „was bewegt sich mit".
- **Bauen:** `go build ./...`, `go vet ./...`, `go test ./...`. Deprecations: `staticcheck` (SA1019), falls installiert. Advisories: `govulncheck ./...`, falls installiert.
- **Im Code:** der Importpfad ist der Modulpfad plus Unterpaket; dazu `//go:generate`-Direktiven, Build-Tags, `_test.go`.

## Ruby — Bundler

- **Manifest / Lock:** `Gemfile` / `Gemfile.lock`; `.gemspec` bei Bibliotheken; `.ruby-version`, `ruby '3.x'` im Gemfile.
- **Läuft tatsächlich:** `bundle info <gem>`; der Abschnitt in `Gemfile.lock`.
- **Registry:** `gem list -r -a <gem>` (alle Versionen); `gem specification -r <gem> -v <ziel>` (`required_ruby_version`, `dependencies`); API: `https://rubygems.org/api/v1/versions/<gem>.json` (Versionen mit `created_at`, `ruby_version`, `yanked`), `https://rubygems.org/api/v2/rubygems/<gem>/versions/<ziel>.json` (Abhängigkeiten).
- **Wer hängt daran:** in `Gemfile.lock` die eingerückten Zeilen unter anderen Gems lesen; `gem dependency <gem> --reverse-dependencies` für installierte Gems.
- **Laufzeit:** `required_ruby_version` gegen `.ruby-version` und `ruby -v`; native Erweiterungen brauchen Build-Werkzeuge, ein Ziel mit neuer nativer Abhängigkeit scheitert am Install, nicht am Code.
- **Trockenlauf:** `bundle lock --update <gem>` schreibt nur `Gemfile.lock` — im Worktree der Trockenlauf; `bundle update <gem> --conservative` bewegt nur das Gem, ohne `--conservative` auch seine Abhängigkeiten. `bundle outdated <gem>` als Übersicht.
- **Exakt setzen:** im Gemfile `gem '<gem>', '= <ziel>'`, dann `bundle install` (oder `bundle lock --update <gem>`).
- **Ausgangsstand / Tests:** `bundle install`, `bundle exec rake` oder `bundle exec rspec`. Deprecations: Rails `config.active_support.deprecation = :raise` in der Testumgebung, `RUBYOPT=-W:deprecated`; für Rails-Sprünge `bin/rails app:update` im Worktree — der Diff der Konfigurationsdateien ist Teil der Anpassungsliste. Advisories: `bundle-audit`, falls installiert.
- **Im Code:** `require '<gem>'` — der Pfad kann vom Gem-Namen abweichen (`rack-test` → `require 'rack/test'`), Bundler lädt automatisch nach Gem-Name mit `/` für `-`, oder nach der `require:`-Option im Gemfile; Konstanten und Namensräume aus `lib/` des Gems; Rails-Railties verdrahten sich über `config/initializers/*.rb` und Generatoren.

## .NET — NuGet

- **Manifest:** `*.csproj` / `*.fsproj` mit `<PackageReference Include="…" Version="…" />`, zentral `Directory.Packages.props` (`<PackageVersion>`), Legacy `packages.config`; Lock nur mit `RestorePackagesWithLockFile` (`packages.lock.json`).
- **Läuft tatsächlich / wer hängt daran:** `dotnet list package`, `--include-transitive`, `--outdated`, `--deprecated`, `--vulnerable`; `dotnet nuget why <projekt> <paket>` (SDK ≥ 8.0.4xx) zeigt die Abhängigkeitspfade.
- **Registry:** `https://api.nuget.org/v3-flatcontainer/<id-kleingeschrieben>/index.json` (Versionen), `…/<id>/<ziel>/<id>.nuspec` (Abhängigkeiten je Zielframework, `minClientVersion`); `dotnet package search <id> --exact-match`.
- **Laufzeit:** `<TargetFramework>` des Projekts gegen die Frameworks im nuspec des Ziels (`net8.0`, `netstandard2.0`); SDK aus `global.json` gegen `dotnet --version`.
- **Exakt setzen:** `dotnet add <projekt> package <id> --version <ziel>` (mit zentraler Versionsverwaltung stattdessen `Directory.Packages.props` im Worktree editieren), dann `dotnet restore` — `NU1605` (Downgrade) und `NU1107` (Versionskonflikt) sind die Konflikte.
- **Bauen:** `dotnet build`, `dotnet test`. `CS0618` = veraltete API benutzt, `CS0619` = entfernte API benutzt; mit `-warnaserror` werden Deprecations zu Fehlern.
- **Im Code:** `using <Namespace>` (Namensräume aus der Paketdokumentation oder dem Assembly), `global using`, DI-Registrierungen (`services.Add…`), Abschnitte in `appsettings.json`, Analyzer und Source-Generatoren, deren Versionen am SDK hängen.

## Andere Ökosysteme

Swift PM, CocoaPods, Dart pub, Hex, Terraform-Provider, Docker-Basisimages, GitHub Actions — dieselben Fragen, andere Befehle: Manifest, Lock, tatsächliche Version, Registry-Metadaten des Ziels, umgekehrte Abhängigkeiten, Laufzeitanforderung, exaktes Setzen, Kontrolle. Die Befehle findest du über `--help` und die Dokumentation des Werkzeugs; welcher Schritt ohne Befehl blieb, steht im Bericht unter „Nicht geprüft".
