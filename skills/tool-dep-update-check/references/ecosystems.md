# Ecosystems

The same questions per ecosystem: where is the dependency declared, what actually runs, what does the registry know about the target, who else depends on it, what runtime does it require, how is it set exactly to the target, and what is it called in the code. The commands here are starting points, not guarantees — flags move between versions. Check a flag you are not sure about with `--help` before you use it; an unknown flag is an error in your invocation, not a finding against the update. Everything that writes runs in the worktree.

Comparing versions where semver does not apply: npm `npx semver -r '<bereich>' <version>`, Composer `composer show -a <paket>` lists sorted, Python `python3 -c "from packaging.version import Version as V; print(V('a') < V('b'))"`, Maven: `search.maven.org` sorts chronologically, Go `go list -m -versions` returns ascending.

---

## JavaScript and TypeScript — npm, pnpm, yarn

- **Manifest / lock:** `package.json` / `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`. Workspaces (`workspaces` in `package.json`, `pnpm-workspace.yaml`) mean several manifests. Distinguish `dependencies`, `devDependencies`, `peerDependencies`, `overrides` or `resolutions`.
- **Actually runs:** `npm ls <paket>` / `pnpm ls <paket>` / `yarn why <paket>`; `node_modules/<paket>/package.json`. Several versions side by side are normal here — the report says which one the project sees directly.
- **Registry:** `npm view <paket> versions --json`; `npm view <paket>@<ziel> engines peerDependencies dependencies deprecated repository.url`; dates with `npm view <paket> time --json`. Also works in pnpm and yarn projects; private registries are in `.npmrc`.
- **Who depends on it:** `npm explain <paket>` (`npm why`), `pnpm why <paket>`, `yarn why <paket>`. Other packages' peer requirements on this one: `npm ls <paket>` shows the edges; an `ERESOLVE` in the dry run names the culprit.
- **Runtime:** `engines.node` in the project, `.nvmrc`, `.node-version`, `volta`, `node-version` in the workflows; locally `node -v`. A target that ships only ESM (`"type": "module"`, no `main`/`require` export) breaks `require()` and unusually configured Jest runs — the changelog usually says "ESM only".
- **Dry run:** `npm install <paket>@<ziel> --dry-run`. pnpm: `pnpm add <paket>@<ziel> --lockfile-only`; yarn Berry: `yarn add <paket>@<ziel> --mode=update-lockfile` writes only the lock — in the worktree that is the dry run.
- **Set exactly:** `npm install <paket>@<ziel> --save-exact`, `pnpm add <paket>@<ziel> --save-exact`, `yarn add <paket>@<ziel> --exact`; `--save-dev` or `-D` if it is a dev dependency, otherwise it moves into `dependencies`. If the target lies within the constraint, `npm update <paket>` suffices — lock only, no manifest.
- **Source state:** `npm ci`, `pnpm install --frozen-lockfile`, `yarn install --frozen-lockfile` (yarn 1) or `yarn install --immutable` (Berry).
- **In the code:** `import … from '<paket>'`, `from '<paket>/…'` (subpath — the target's `exports` map decides whether it still exists), `require('<paket>')`, `import type`, `jest.mock('<paket>')`, `vi.mock`; configuration files that name plugins and presets (`eslint`, `babel`, `vite`, `next`, `tailwind`, `postcss`); `package.json` scripts with `npx <paket>`.
- **Deprecations:** usually `console.warn` at runtime; `tsc` does not report `@deprecated` in types on its own — an ESLint rule for deprecations, if configured. Type check after the update: `npx tsc --noEmit`. Advisories: `npm audit` after the install. Run framework codemods (Next.js, React, Angular `ng update`) in the worktree; their diff is the list of adaptations.

## PHP — Composer

- **Manifest / lock:** `composer.json` / `composer.lock`; `require` versus `require-dev`.
- **Actually runs:** `composer show <vendor/paket>`; `vendor/<vendor>/<paket>/composer.json`.
- **Registry:** `composer show -a <vendor/paket>` (available versions, respects the project's `repositories`), `composer show -a <vendor/paket> <ziel>` (requirements of exactly this version). Packagist directly: `https://repo.packagist.org/p2/<vendor>/<paket>.json` — all versions with `require`, `time`, `source.url`.
- **Who depends on it:** `composer why <vendor/paket>`. And the check's question in one command: `composer why-not <vendor/paket> <ziel>` names every package and every platform requirement that prevents the target.
- **Runtime:** the project's `require.php` and `require.ext-*`; `config.platform.php` overrides the local PHP version for resolution — with `platform` set, the local version does not matter for resolution, but it does for the trial run. Locally `php -v`, `php -m`. A target that requires a higher PHP version shows up in `why-not`, not only at runtime.
- **Dry run:** `composer update <vendor/paket> --with <vendor/paket>:<ziel> --dry-run`; first like this (the package only), then `-w` (also its dependencies that are not root requirements), then `-W` (also root requirements). The output lists "Upgrading x (a => b)" — those are the packages pulled along.
- **Set exactly:** `composer require <vendor/paket>:<ziel>` (with `-w`/`-W` if needed; `--dev` if it is a dev dependency — otherwise it moves into `require`). If the target lies within the constraint: `composer update <vendor/paket>`.
- **Source state:** `composer install`. Afterwards `composer validate` and `composer show <vendor/paket>` as a check.
- **In the code:** namespaces from `vendor/<vendor>/<paket>/composer.json` → `autoload.psr-4`, then `use <Namespace>\…`, FQCNs in strings and configuration (`config/services.yaml`, `config/packages/*.yaml`, `config/bundles.php`, XML), attributes and annotations, Twig or Blade extensions, `bin/` commands in `composer.json` scripts.
- **Deprecations:** Symfony — `SYMFONY_DEPRECATIONS_HELPER` and the `phpunit-bridge` report at the end of the test run; PHPUnit 10+ `--display-deprecations`; generally `E_USER_DEPRECATED`. Static analysis after the update catches signature changes without runtime: `vendor/bin/phpstan analyse`, `vendor/bin/psalm`. Rector in the worktree with the set for the jump (`vendor/bin/rector process --dry-run`) delivers the list of adaptations. Advisories: `composer audit`.

## Java and Kotlin — Maven

- **Manifest = lock:** `pom.xml`; versions often in `<properties>` or via a parent BOM (`spring-boot-starter-parent`, `dependencyManagement`), multi-module with versions in the root. `mvn help:effective-pom` shows what really applies.
- **Actually runs:** `mvn dependency:tree -Dincludes=<gruppe>:<artefakt>`; `mvn dependency:list`.
- **Registry:** `https://search.maven.org/solrsearch/select?q=g:"<gruppe>"+AND+a:"<artefakt>"&core=gav&rows=200&wt=json` (versions with timestamp); the target's POM at `https://repo1.maven.org/maven2/<gruppe/mit/schrägstrichen>/<artefakt>/<ziel>/<artefakt>-<ziel>.pom` (its dependencies); `mvn versions:display-dependency-updates -Dincludes=<gruppe>:<artefakt>`. Private Nexus or Artifactory via `settings.xml`; existence then with `mvn dependency:get -Dartifact=<gruppe>:<artefakt>:<ziel>`.
- **Who depends on it:** `mvn dependency:tree -Dverbose -Dincludes=<gruppe>:<artefakt>` shows who pulls it in and what is "omitted for conflict". Maven does not fail on version conflicts, it takes the nearest version — the risk is the silent version mix. `maven-enforcer-plugin` with `dependencyConvergence`, if configured, makes it loud.
- **Runtime:** `maven.compiler.release` / `source` / `target`, `<java.version>`, toolchains; locally `java -version`, `mvn -v`. The target's Java requirement is in the release notes — or in the bytecode: `javap -verbose -cp <jar> <eine.Klasse> | grep 'major version'` (52 = Java 8, 55 = 11, 61 = 17, 65 = 21).
- **Set exactly:** `mvn versions:use-dep-version -Dincludes=<gruppe>:<artefakt> -DdepVersion=<ziel> -DforceVersion=true`, or `mvn versions:set-property -Dproperty=<property> -DnewVersion=<ziel>` if the version is in a property — or edit the `pom.xml` in the worktree. If the version is set by a BOM, override the BOM's property (`<jackson-bom.version>` etc.) and check with `dependency:tree` whether the override takes effect.
- **Source state / build:** `./mvnw` or `mvn -q dependency:resolve`; then `mvn verify`. Deprecations during compilation: `-Dmaven.compiler.showDeprecation=true`.
- **In the code:** the artifact's Java packages from `unzip -l <jar> | grep '\.class$'` → `import <paket>.…`; Spring: auto-configuration, property names in `application.yml` (renamed properties are the classic Spring Boot break — `spring-boot-properties-migrator` reports them), annotations, `META-INF/spring.factories` and `AutoConfiguration.imports`.
- **Migration:** OpenRewrite with `mvn org.openrewrite.maven:rewrite-maven-plugin:dryRun` and the recipe for the jump (Spring Boot, JUnit, Jakarta) — the diff is the list of adaptations.

## Java and Kotlin — Gradle

- **Manifest:** `build.gradle(.kts)`, `settings.gradle(.kts)`, version catalog `gradle/libs.versions.toml`, `buildSrc`; lock only with dependency locking enabled (`gradle.lockfile`).
- **Actually runs / who depends on it:** `./gradlew dependencyInsight --dependency <artefakt> --configuration runtimeClasspath` — selected version, reason, constraints and who requires it, in one command. `./gradlew dependencies --configuration runtimeClasspath` for the full picture.
- **Registry:** as for Maven.
- **Runtime:** `java.toolchain.languageVersion`, `sourceCompatibility`, `kotlin { jvmToolchain(…) }`; the wrapper's Gradle version against what the target requires — Gradle *plugins* have their own minimum Gradle versions (`https://plugins.gradle.org/plugin/<id>/<version>`).
- **Set exactly:** change the version in the worktree's catalog or build script; `./gradlew --refresh-dependencies dependencies`; with locking `./gradlew dependencies --write-locks`.
- **Build:** `./gradlew build`; `--warning-mode all` shows Gradle's own deprecations, relevant if the dependency is a Gradle plugin.

## Python — pip, pip-tools, Poetry, uv, Pipenv

- **Manifest / lock:** `pyproject.toml` (`[project] dependencies`, `[dependency-groups]`, `[tool.poetry.dependencies]`), `requirements.in` → `requirements.txt` (pip-tools: `.in` is the manifest, `.txt` is the lock), `poetry.lock`, `uv.lock`, `Pipfile` / `Pipfile.lock`, `setup.py` / `setup.cfg`, `constraints.txt`.
- **Actually runs:** `pip show <paket>`, `uv pip show <paket>`, `poetry show <paket>`.
- **Registry:** `https://pypi.org/pypi/<paket>/json` (all releases with `upload_time`, `yanked`), `https://pypi.org/pypi/<paket>/<ziel>/json` → `info.requires_python`, `info.requires_dist`, `info.project_urls` (changelog, source), `info.yanked`; `pip index versions <paket>`. Private index via `PIP_INDEX_URL` or `pip.conf`. Whether the target ships a wheel for the platform and Python version is shown by `urls[]` — without a wheel pip builds from source, and then the trial run needs compilers.
- **Who depends on it:** `pip show <paket>` → `Required-by`; `pipdeptree -r -p <paket>` (installing it additionally into the worktree venv is fine); `poetry show --tree`; `uv tree --invert --package <paket>`.
- **Runtime:** the project's `requires-python`, `.python-version`, `tox.ini`, CI matrix; locally `python3 --version`.
- **Dry run:** `pip install '<paket>==<ziel>' --dry-run` (`--report -` for JSON); `poetry add '<paket>==<ziel>' --dry-run`; pip-tools: edit `.in`, `pip-compile --dry-run`. uv writes immediately on `uv add` — in the worktree that is equivalent to the dry run; the output lists what moves.
- **Set exactly:** fresh venv in the worktree (`python3 -m venv .venv`), then `pip install '<paket>==<ziel>'` and change the pin in `requirements.txt`; `poetry add '<paket>==<ziel>'`; `uv add '<paket>==<ziel>'`; `pipenv install '<paket>==<ziel>'`.
- **Source state:** `pip install -r requirements.txt`, `poetry install`, `uv sync`, `pipenv sync`.
- **In the code:** the import name is not the distribution name — `Pillow` → `PIL`, `beautifulsoup4` → `bs4`, `PyYAML` → `yaml`, `scikit-learn` → `sklearn`. Mapping via `top_level.txt` or `RECORD` in the `<dist>.dist-info/` in site-packages, or `python3 -c "import importlib.metadata as m; print(m.packages_distributions())"`. Then `import <modul>`, `from <modul> import`, plus strings: Django `INSTALLED_APPS` and `settings.py`, pytest plugins (`-p`, `conftest.py`), entry points, Alembic and Celery configuration.
- **Deprecations:** `python3 -W error::DeprecationWarning -m pytest` turns them into errors — the sharpest tool for framework jumps; `-W default::DeprecationWarning` only shows them. Type check after the update with `mypy` or `pyright` if the target ships types. Advisories: `pip-audit`.

## Rust — Cargo

- **Manifest / lock:** `Cargo.toml` (workspace: `[workspace.dependencies]`), `Cargo.lock`. Feature flags belong to the manifest: `default-features = false` and the feature list.
- **Actually runs:** `cargo pkgid <crate>`; `cargo tree -d` shows duplicates — two versions of the same crate are allowed and only break when types of both versions meet (`expected Foo, found Foo`). A major jump often produces exactly that when another crate holds on to the old version.
- **Registry:** `cargo info <crate>@<ziel>` (Cargo ≥ 1.82: `rust-version`, dependencies, repository); `https://crates.io/api/v1/crates/<crate>` (versions with `rust_version`, `yanked`, `created_at` — the API requires a `User-Agent` header), `…/<crate>/<ziel>/dependencies`. Source diff of two versions: `https://diff.rs/<crate>/<ausgang>/<ziel>`.
- **Who depends on it:** `cargo tree -i <crate>`; with features: `cargo tree -e features -i <crate>`.
- **Runtime:** the target's `rust-version` (MSRV) against `rust-toolchain.toml` and `rustc --version`; `edition`. Renamed or removed features are a Cargo-specific break — the build reports them as an unknown feature.
- **Dry run:** `cargo update -p <crate> --precise <ziel> --dry-run` — only if the target lies within the constraint; otherwise change `Cargo.toml` in the worktree first.
- **Set exactly:** in `Cargo.toml` `<crate> = "=<ziel>"` (keep the features), then `cargo update -p <crate> --precise <ziel>`. If the crate has a different name in `Cargo.toml` (`package = "…"`), that one counts.
- **Source state / build:** `cargo fetch`, then `cargo build --all-targets`, `cargo test`, `cargo clippy`. Deprecations are compiler warnings (`#[deprecated]`); the build of the source state lists what is already being warned about now.
- **In the code:** `use <crate_name>::…` with underscores instead of hyphens; macros and `#[derive(…)]` from the crate; feature usage in `Cargo.toml`. Advisories: `cargo audit` or `cargo deny`, if installed.

## Go — Modules

- **Manifest / lock:** `go.mod` / `go.sum`; `go.work` for workspaces; `// indirect` marks transitive modules.
- **Actually runs:** `go list -m <modul>`, `go list -m all`.
- **Registry:** `go list -m -versions <modul>`; `go list -m -json <modul>@<ziel>` (`Time`, `GoVersion`); proxy directly: `https://proxy.golang.org/<modul>/@v/list`, `…/@v/<ziel>.info`, `…/@v/<ziel>.mod` (the target's `go` directive and `require`). Retracted versions: `go list -m -retracted <modul>`; module deprecation: `go list -m -u -json <modul>` → `Deprecated`.
- **Who depends on it:** `go mod why -m <modul>`; `go mod graph | grep ' <modul>@'`.
- **Runtime:** the target's `go` directive against the project's and `go version`. Since Go 1.21, `toolchain` downloads a matching toolchain by itself — the trial run can thus silently run with a different Go version than CI; name it in the report.
- **Major from 2 upwards:** a different module path (`…/v2`), i.e. a different module. The "update" is the change of every import path (`grep -rn '"<modul>' --include=*.go`), and the check lists them all.
- **Set exactly / dry run:** there is no real dry run; in the worktree `go get <modul>@<ziel>`, `go mod tidy`, and `git -C <worktree> diff go.mod go.sum` is the answer to "what moves along".
- **Build:** `go build ./...`, `go vet ./...`, `go test ./...`. Deprecations: `staticcheck` (SA1019), if installed. Advisories: `govulncheck ./...`, if installed.
- **In the code:** the import path is the module path plus subpackage; plus `//go:generate` directives, build tags, `_test.go`.

## Ruby — Bundler

- **Manifest / lock:** `Gemfile` / `Gemfile.lock`; `.gemspec` for libraries; `.ruby-version`, `ruby '3.x'` in the Gemfile.
- **Actually runs:** `bundle info <gem>`; the section in `Gemfile.lock`.
- **Registry:** `gem list -r -a <gem>` (all versions); `gem specification -r <gem> -v <ziel>` (`required_ruby_version`, `dependencies`); API: `https://rubygems.org/api/v1/versions/<gem>.json` (versions with `created_at`, `ruby_version`, `yanked`), `https://rubygems.org/api/v2/rubygems/<gem>/versions/<ziel>.json` (dependencies).
- **Who depends on it:** read the indented lines under other gems in `Gemfile.lock`; `gem dependency <gem> --reverse-dependencies` for installed gems.
- **Runtime:** `required_ruby_version` against `.ruby-version` and `ruby -v`; native extensions need build tools, a target with a new native dependency fails on install, not in the code.
- **Dry run:** `bundle lock --update <gem>` writes only `Gemfile.lock` — in the worktree that is the dry run; `bundle update <gem> --conservative` moves only the gem, without `--conservative` also its dependencies. `bundle outdated <gem>` as an overview.
- **Set exactly:** in the Gemfile `gem '<gem>', '= <ziel>'`, then `bundle install` (or `bundle lock --update <gem>`).
- **Source state / tests:** `bundle install`, `bundle exec rake` or `bundle exec rspec`. Deprecations: Rails `config.active_support.deprecation = :raise` in the test environment, `RUBYOPT=-W:deprecated`; for Rails jumps `bin/rails app:update` in the worktree — the diff of the configuration files is part of the list of adaptations. Advisories: `bundle-audit`, if installed.
- **In the code:** `require '<gem>'` — the path can differ from the gem name (`rack-test` → `require 'rack/test'`), Bundler auto-requires by gem name with `/` for `-`, or by the `require:` option in the Gemfile; constants and namespaces from the gem's `lib/`; Rails railties wire themselves in via `config/initializers/*.rb` and generators.

## .NET — NuGet

- **Manifest:** `*.csproj` / `*.fsproj` with `<PackageReference Include="…" Version="…" />`, centrally `Directory.Packages.props` (`<PackageVersion>`), legacy `packages.config`; lock only with `RestorePackagesWithLockFile` (`packages.lock.json`).
- **Actually runs / who depends on it:** `dotnet list package`, `--include-transitive`, `--outdated`, `--deprecated`, `--vulnerable`; `dotnet nuget why <projekt> <paket>` (SDK ≥ 8.0.4xx) shows the dependency paths.
- **Registry:** `https://api.nuget.org/v3-flatcontainer/<id-kleingeschrieben>/index.json` (versions), `…/<id>/<ziel>/<id>.nuspec` (dependencies per target framework, `minClientVersion`); `dotnet package search <id> --exact-match`.
- **Runtime:** the project's `<TargetFramework>` against the frameworks in the target's nuspec (`net8.0`, `netstandard2.0`); SDK from `global.json` against `dotnet --version`.
- **Set exactly:** `dotnet add <projekt> package <id> --version <ziel>` (with central package management, edit `Directory.Packages.props` in the worktree instead), then `dotnet restore` — `NU1605` (downgrade) and `NU1107` (version conflict) are the conflicts.
- **Build:** `dotnet build`, `dotnet test`. `CS0618` = obsolete API used, `CS0619` = removed API used; with `-warnaserror` deprecations become errors.
- **In the code:** `using <Namespace>` (namespaces from the package documentation or the assembly), `global using`, DI registrations (`services.Add…`), sections in `appsettings.json`, analyzers and source generators whose versions depend on the SDK.

## Other ecosystems

Swift PM, CocoaPods, Dart pub, Hex, Terraform providers, Docker base images, GitHub Actions — the same questions, different commands: manifest, lock, actual version, the target's registry metadata, reverse dependencies, runtime requirement, setting exactly, verification. You find the commands via `--help` and the tool's documentation; which step remained without a command goes into the report under „Nicht geprüft".
