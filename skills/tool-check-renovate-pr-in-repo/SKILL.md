---
name: tool-check-renovate-pr-in-repo
description: Inventarisiert die offenen Renovate-PRs eines GitHub-Repositories in einem frischen Klon, prüft jedes Dependency-Paar mit tool-dep-update-check in einem eigenen Subagent und berichtet je PR Urteil, Belastbarkeit und empfohlene Aktion — ohne etwas zu ändern, zu pushen oder zu approven. Nimmt die Repo-URL als Argument, optional gefolgt von PR-Nummern in Anführungszeichen, die den Lauf auf diese PRs beschränken.
argument-hint: "[repoUrl] [prs]"
arguments: repoUrl prs
disable-model-invocation: true
allowed-tools: Bash Read Grep Glob Edit Write Agent Skill
---

**Ausgabesprache: Deutsch.** Bericht, Rückfragen und Abbruchmeldungen schreibst du auf Deutsch, auch wenn Repository, PR-Titel und Changelogs englisch sind. Zitate aus PR-Bodies, Bezeichner, Befehle und Fehlermeldungen bleiben, wie sie sind.

Prüfe die offenen Renovate-PRs des GitHub-Repositories `$repoUrl` — alle, oder mit `$prs` nur die genannten. Arbeite wie jemand, der vor dem Review-Klick die Frage beantwortet, die ein Renovate-PR sonst erst nach dem Merge beantwortet: Kann dieses Projekt das Update tragen, und was wäre dafür zu tun? Jedes Dependency-Paar — ein Paket mit Ausgangs- und Zielversion — prüft `tool-dep-update-check`; du klonst, inventarisierst, klassifizierst, delegierst und berichtest. Du veränderst nichts: kein Commit, kein Push, keine Approval, keine Datei in der Arbeitskopie des Menschen.

Drei Regeln tragen den ganzen Ablauf: Geschrieben wird nur unter dem Temp-Verzeichnis dieses Laufs — im Klon und in den Berichtsdateien daneben —, nie im Verzeichnis, aus dem der Skill gestartet wurde. Jede Aussage über einen PR hat einen Prüfbericht hinter sich, und wo keiner entstehen konnte, steht der Grund statt eines Urteils. Und nichts verlässt den Rechner: Dieser Skill liest GitHub und schreibt nicht dorthin.

## 1. Argumente und Voraussetzungen

**Repository.** `$repoUrl` ist Pflicht. Akzeptiert sind `https://github.com/<owner>/<repo>`, mit oder ohne `.git`, mit Pfadanhang wie `/pulls`, die SSH-Form `git@github.com:<owner>/<repo>.git` und die Kurzform `<owner>/<repo>`. Alles davon wird auf `<owner>/<repo>` abgebildet; was sich nicht abbilden lässt — leer, kein GitHub-Host, nur ein Owner — nennt den Aufruf `/tool-check-renovate-pr-in-repo <repoUrl> [prs]` und bricht ab. Rate kein Repository aus der Arbeitskopie: Wer den Skill startet, nennt das Repo.

**PR-Auswahl.** `$prs` ist optional: PR-Nummern als ein Argument in Anführungszeichen (`"12 15"`), getrennt durch Leerzeichen oder Kommas, mit oder ohne `#`. Ohne Angabe gilt jeder offene Renovate-PR. Eine genannte Nummer, zu der kein offener Renovate-PR gehört, ist kein Abbruchgrund — sie steht im Bericht unter „Übersprungen" mit dem Grund „kein offener Renovate-PR".

**Werkzeuge und Rechte.** Drei Prüfungen, bevor du klonst, jede mit der Zeile, die der Mensch ausführen würde:

1. `command -v gh` — fehlt es, brich ab und nenne die Installation laut https://cli.github.com. Einen Ersatz über `curl` und Token gibt es nicht.
2. `gh auth status` — endet es mit Exit-Code 1, brich ab und nenne `gh auth login`.
3. `gh repo view <owner>/<repo> --json viewerPermission,defaultBranchRef` — `viewerPermission` muss `WRITE`, `MAINTAIN` oder `ADMIN` sein; `READ` oder `TRIAGE` heißt, dass die Folgeschritte dieses Skills — Anpassungs-Commit und Approval — unmöglich wären, und du brichst ab und sagst das. Ein Fehler hier heißt meist: Repo existiert nicht, Tippfehler in der URL oder kein Zugang; nenne die Antwort von `gh`. Den Namen des Default-Branchs aus `defaultBranchRef.name` merkst du dir.

Alles Weitere — Node, PHP, Java, Paketmanager — braucht nicht dieser Skill, sondern der Probelauf von `tool-dep-update-check`, und der meldet selbst, was ihm fehlt.

## 2. Klon

`TMP=$(mktemp -d)`, dann `mkdir -p "$TMP/reports" "$TMP/prs"`, dann `gh repo clone <owner>/<repo> "$TMP/renovate-check-<repo>"`. Der Klon über `gh` nimmt Auth und Protokollwahl aus der `gh`-Konfiguration, und die Verzeichnisse legst du an, bevor irgendein Subagent läuft — `cat > <datei>` legt anders als das Write-Tool keine Verzeichnisse an, und ein Subagent, der an einem fehlenden Verzeichnis scheitert, sieht aus wie einer ohne Urteil.

Der Klon ist der einzige Ort, an dem Git-Befehle etwas verändern dürfen, und du gibst seinen absoluten Pfad jedem Befehl mit: `git -C "$KLON" …`, oder für Paketmanager und Skripte eine Subshell `(cd "$KLON" && …)`. Ein nacktes `cd` gibt es nicht — das Verzeichnis, aus dem der Skill gestartet wurde, ist Lesezone und bleibt unberührt; der Bericht am Ende nennt `git status --porcelain` dieses Verzeichnisses als Beleg, wenn es ein Repository ist. Scheitert der Klon — kein Netz, kein Zugang, voller Datenträger —, brich ab mit der Ausgabe von `gh`; bis hierhin ist nichts entstanden außer dem leeren Temp-Verzeichnis, und das nennst du mit `rm -rf "$TMP"`.

Nach dem Klon sind alle Branches als `origin/<name>` da; die PR-Branches brauchen keinen lokalen Checkout, weil Diff und Prüfung über Refs laufen. Fehlt ein `origin/<head>` trotzdem — der Branch wurde zwischen `gh pr list` und Klon neu geschrieben —, holt `git -C "$KLON" fetch origin <head>` ihn nach; bleibt er weg, ist der PR „nicht prüfbar: Branch nicht erreichbar".

## 3. Inventar und Klassifikation

Lies `${CLAUDE_SKILL_DIR}/references/renovate-pr.md`, bevor du den ersten PR anfasst: Dort stehen, mit Quellen, die Erkennung von Renovate-PRs, die Titel- und Body-Vorlagen, die PR-Arten mit ihrer Prüfbarkeit und die `gh`-Befehle. Dieser Abschnitt setzt sie voraus und wiederholt sie nicht.

**Inventar.** Zwei Listen, deren Vereinigung die Arbeitsliste ist — und du holst beide, auch wenn die zweite alle offenen PRs enthält: Die Autor-Suche findet PRs des Renovate-App-Kontos auch dann, wenn ihr Branchpräfix von dem abweicht, das du aus der Konfiguration gelesen hast, und der Präfix-Filter findet ein selbst gehostetes Renovate, das nicht als App auftritt. Eine Liste allein hat eine blinde Stelle.

```
gh pr list --repo <owner>/<repo> --state open --limit 500 --search 'author:app/renovate' \
  --json number,title,headRefName,baseRefName,isDraft,mergeable,reviewDecision,statusCheckRollup,author,url,updatedAt
gh pr list --repo <owner>/<repo> --state open --limit 500 \
  --json number,title,headRefName,baseRefName,isDraft,mergeable,reviewDecision,statusCheckRollup,author,url,updatedAt
```

Die zweite Liste filterst du auf Branches mit dem Präfix aus der Renovate-Konfiguration des Klons — `branchPrefix`, davor gegebenenfalls `additionalBranchPrefix`, Standard `renovate/`; die Dateinamen stehen in der Referenz, Abschnitt A, und `grep -o '"branchPrefix"[[:space:]]*:[[:space:]]*"[^"]*"'` reicht für JSON wie JSON5. `--limit 500` ist Pflicht, weil `gh pr list` sonst stillschweigend bei 30 kappt; die Gesamtzahl offener PRs (`gh pr list --repo <owner>/<repo> --state open --limit 500 --json number --jq length`) kommt neben die Inventarzahl in den Bericht, damit sichtbar ist, was kein Renovate-PR war. `body` steht bewusst nicht in der Feldliste: Renovate-Bodies tragen eingeklappte Release Notes von oft zehn bis fünfzig Kilobyte, und zwanzig davon im Kontext sind das Ende des Laufs. Mit `$prs` behältst du nur die genannten Nummern.

Das Inventar ist leer? Dann sag das — Gesamtzahl, Autorfilter, Präfix — und beende den Lauf mit dem Bericht; der Klon wird wie sonst entfernt.

**Body und Diff je PR.** Beides holst du dir in Dateien, nicht in den Kontext:

- `gh pr view <n> --repo <owner>/<repo> --json body --jq .body > "$TMP/prs/<n>.md"`, und daraus nur die Tabellenzeilen: `grep -E '^\|.*(→|->)' "$TMP/prs/<n>.md"`. Die `Change`-Spalte liest sich `` `a` → `b` `` im aktuellen Renovate und `` `a` -> `b` `` in älteren PRs; ein Body ohne solche Zeile ist bei Lockfile-Maintenance normal (`All locks refreshed`) und sonst ein Hinweis auf eine Art, die du in der Referenz nachschlägst.
- `git -C "$KLON" diff --name-only origin/<basis>...origin/<head>` mit `<basis>` aus `baseRefName`, und für die Paare `git -C "$KLON" diff origin/<basis>...origin/<head> -- <manifeste>`; welche Dateien Manifeste und Locks sind, steht in `tool-dep-update-check`, Abschnitt 2 — `package.json`, `composer.json`, `pom.xml`, `build.gradle(.kts)` samt `gradle/libs.versions.toml`, `pyproject.toml` und `requirements*.txt`, `Cargo.toml`, `go.mod`, `Gemfile`, `*.csproj` samt `Directory.Packages.props` — und die Locks daneben.

**Klassifikation**, in dieser Reihenfolge, der erste Treffer entscheidet; jeder Grund kommt wörtlich in den Bericht:

1. `isDraft` → übersprungen: Draft.
2. `mergeable` ist `CONFLICTING` → übersprungen: Konflikt mit `<basis>`. Neu aufsetzen ist Renovates Rebase-Checkbox, die steht im Bericht unter „Nächste Schritte" — du hakst sie nicht an.
3. Eigene Approval auf dem aktuellen Head → übersprungen. Belegt, nicht geraten: `gh api repos/<owner>/<repo>/pulls/<n>/reviews --paginate --jq '.[] | select(.state=="APPROVED") | {user: .user.login, commit: .commit_id}'` gegen `gh pr view <n> --repo <owner>/<repo> --json headRefOid --jq .headRefOid` und den eigenen Login aus `gh api user --jq .login`. Eine eigene Approval auf einem älteren Commit zählt nicht als Grund — der PR wurde seitdem neu geschrieben und wird neu geprüft.
4. Nur Lock-Dateien geändert und Branch oder Titel nennen Lockfile-Maintenance → übersprungen: Lockfile-Maintenance, kein Paar.
5. Nur Dateien unter `.github/workflows/`, `Dockerfile*`, `docker-compose*`, `action.y(a)ml` oder nur Digests im Diff → übersprungen: Docker, Actions oder Digest — kein Ökosystem der Prüfung.
6. Titel beginnt mit `Pin`, `Roll back` oder `Replace` → übersprungen: Pin-, Rollback- oder Ersetzungs-PR, mit dem Grund aus der Referenz, Abschnitt C.
7. Manifest-Diff liefert ein oder mehrere Paare → prüfbar. Je geändertem Paket: Name, alte und neue Version aus dem Diff, führende `^`, `~`, `=` und `v` gestreift; dasselbe Paket in mehreren Manifesten eines Monorepos ist ein Paar, weil der Prüf-Skill alle Manifeste selbst findet. Ist nur der Lock geändert, aber der Branch trägt `-lockfile` oder die Tabelle nennt ein Paket, kommt das Paar aus dem Lock-Diff (`version`-Zeilen des genannten Pakets). Dann die Gegenprobe: Jede Tabellenzeile des Bodys muss zu einem Paar passen und jedes Paar zu einer Zeile; passt es nicht — anderes Paket, andere Version, eine Zeile mehr —, ist der PR „nicht prüfbar: Manifest und Body widersprechen sich", mit beiden Fassungen im Bericht. Mehrere Paare sind ein Gruppen-PR.
8. Nichts davon — kein Manifest, kein Lock, keine Tabellenzeile → „nicht prüfbar: keine Dependency-Änderung erkannt", mit den geänderten Pfaden.

Den CI-Stand liest du aus `statusCheckRollup`: ein `FAILURE` oder `ERROR` darin heißt rot, nur `SUCCESS`, `NEUTRAL` oder `SKIPPED` heißt grün, leer oder ausstehend heißt unbekannt. Er entscheidet nichts an der Prüfung, nur an der Empfehlung.

Die Klassifikation steht als Tabelle im Kontext, eine Zeile je PR — Nummer, Titel, Branch, Basis, Art, Paare, CI —, und du schreibst sie nach jedem PR fort. Sie ist dein Zustand; der Klon ist der andere.

## 4. Prüfung je Paar

Jedes Paar prüft `tool-dep-update-check` — und zwar in einem Subagent mit frischem Kontext, weil ein einzelner Prüfbericht samt Changelogs und Probelauf Zehntausende Tokens kostet und ein Repo leicht zehn Renovate-PRs hat. Die Subagents laufen nacheinander, PRs nach Nummer aufsteigend, Paare innerhalb eines PRs in der Reihenfolge der Manifeste: Zwei Probeläufe im selben Klon stritten sich um Worktrees, Caches und CPU.

Die Prüfung läuft gegen `<basis>` — den `baseRefName` des PRs —, nicht gegen den PR-Branch. Dort hält der Lock schon das Ziel, und der Prüf-Skill würde „Lock über dem Ziel" melden und fragen, statt zu urteilen.

**Der Vertrag.** Starte je Paar genau einen Subagent über das Agent-Tool, Typ `general-purpose`, mit diesem Prompt und ausgefüllten Platzhaltern:

```
Arbeitsverzeichnis: <absoluter Klonpfad>. Jeder Bash-Befehl läuft dort (`cd <pfad> && …`
oder `git -C <pfad>`), Glob und Grep nur mit absoluten Pfaden unter diesem Verzeichnis.
Rufe über das Skill-Tool den Skill `tool-dep-update-check` (mit Plugin-Präfix `skills:`
oder ohne) mit den Argumenten `<paket> <ausgang> <ziel> <basis>` auf und folge ihm
vollständig. Schreibe seinen kompletten Bericht per Bash (`cat > <datei> <<'EOF'`) nach
<tmp>/reports/pr-<n>-<paket>.md — nicht mit dem Write-Tool, das nicht vorgenehmigt ist.
Gib nur zurück: die Titelzeile mit dem Urteil, die Zeilen Belastbarkeit, Auflösung und
Voraussetzungen, jede Anpassung aus „Bruchstellen" mit Datei:Zeile, die Befehle aus
„Probelauf" (Install, Build/Typen/Lint, Tests) und den Abschnitt „Nicht geprüft".
Endet der Skill mit einer Rückfrage oder einem Abbruch statt eines Urteils, gib
„nicht prüfbar: <Grund in einem Satz>" zurück. Verändere nichts im Klon außerhalb des
Worktrees, den der Skill selbst anlegt und entfernt.
```

**Die Antwort** ist höchstens etwa dreißig Zeilen: Die erste beginnt mit `# Dependency-Update:` und endet mit `Möglich`, `Möglich mit Anpassungen` oder `Blockiert` — oder sie beginnt mit `nicht prüfbar:`. Alles andere ist ein Fehlschlag des Subagents, kein Urteil: Starte ihn genau einmal erneut, mit demselben Prompt; bleibt das Urteil aus, trägt das Paar „nicht prüfbar: Subagent ohne Urteil", und der Lauf geht mit dem nächsten Paar weiter. Du liest nie die ganze Berichtsdatei in den Kontext; was du brauchst, steht in der Antwort, und der Pfad zur Datei kommt in den Bericht.

**Von Urteilen zu Empfehlungen.** Je PR fasst du die Urteile seiner Paare nach dieser Tabelle zusammen; die erste zutreffende Zeile gilt:

| Urteile der Paare | Belastbarkeit | CI auf dem Head | Empfehlung |
|---|---|---|---|
| mindestens ein Paar „nur statisch" (Probelauf lief nicht) | — | beliebig | nicht freigeben: unverifiziert — mit dem Grund aus „Nicht geprüft" |
| mindestens eins `Blockiert` oder `nicht prüfbar` | beliebig | beliebig | nicht freigeben — mit Vorbedingung oder Grund |
| mindestens eins `Möglich mit Anpassungen`, alle übrigen `Möglich` oder ebenfalls `Möglich mit Anpassungen` | Probelauf gelaufen | beliebig | anpassen, dann approven — mit der Liste der Anpassungen aller Paare; das gilt auch für Gruppen-PRs, Anpassungsbedarf ist kein Blocker |
| alle `Möglich` | Probelauf grün | rot | nicht freigeben: CI rot — Mensch entscheidet |
| alle `Möglich` | Probelauf grün | grün oder unbekannt | approven |

Ein Gruppen-PR mit einem blockierten, nicht prüfbaren oder nur statisch geprüften Paar ist als Ganzes nicht freizugeben, auch wenn die anderen Paare möglich wären; die Gruppe zu teilen ist Renovate-Konfiguration, nicht deine Aufgabe. Ein Gruppen-PR, dessen Paare alle möglich sind und von denen eines oder mehrere Anpassungen brauchen, ist dagegen „anpassen, dann approven" — genau wie ein Einzel-PR mit Anpassungen.

## 5. Bericht und Aufräumen

Vor dem Bericht räumst du den Klon ab — `rm -rf "$KLON"` —, denn nichts darin ist entstanden, das jemand braucht; `"$TMP/reports/"` und `"$TMP/prs/"` bleiben, weil der Bericht dorthin verlinkt, und `rm -rf "$TMP"` ist die Aufräumzeile für den Menschen. Ist das Startverzeichnis ein Git-Repository, zeigt `git status --porcelain` dort denselben Stand wie vor dem Lauf, und das steht als Zeile im Bericht.

```
# Renovate-PRs: <owner/repo> — <n> offen, davon <r> Renovate-PRs: <a> approven, <b> anpassen und approven, <c> nicht freigeben, <d> übersprungen

**Klon:** <pfad> — entfernt; Berichte unter <tmp>/reports/ → `rm -rf <tmp>` räumt alles
**Startverzeichnis:** unverändert (`git status --porcelain` leer) | kein Git-Repository
**Freigabe:** entfällt — dieser Skill ändert nichts, pusht nichts, approved nichts; Commit, Push, Approve und reviewDecision bleiben „—"

| PR | Paket(e) | Sprung | Urteil | Belastbarkeit | CI | Empfehlung | Commit | Push | Approve | reviewDecision |
|----|----------|--------|--------|---------------|----|------------|--------|------|---------|----------------|
| #<n> | <paket a→b> | patch \| minor \| major | <Urteil> | <Belastbarkeit> | grün \| rot \| unbekannt | <Empfehlung> | — | — | — | — |

## PR #<n> — <Titel>
- **Branch:** <head> → <basis>, Autor <login>, gefunden über Autor | Branchpräfix
- **Prüfung:** <Kopfblock je Paar — Urteil, Belastbarkeit, Auflösung, Voraussetzungen>; voller Bericht: <tmp>/reports/pr-<n>-<paket>.md
- **Anpassungen:** <Datei:Zeile — was zu tun ist, je Paar> | keine nötig
- **Nicht geprüft:** <aus dem Prüfbericht> | nichts

## Übersprungen
- #<n> <Titel> — Draft | Konflikt mit <basis> | eigene Approval auf dem aktuellen Head | Lockfile-Maintenance | Docker/Actions/Digest | Pin | Rollback | Ersetzung | kein offener Renovate-PR (aus `prs`)

## Nicht prüfbar
- #<n> <Titel> — Manifest und Body widersprechen sich: <beide Fassungen> | keine Dependency-Änderung erkannt: <Pfade> | Subagent ohne Urteil | Rückfrage des Prüf-Skills: <Satz> | Branch nicht erreichbar

## Nächste Schritte
- <je „approven": `gh pr review <n> --repo <owner/repo> --approve` — die Approval erteilst nicht du>
- <je „anpassen": Branch <head> auschecken, die Anpassungen aus dem Bericht umsetzen, verifizieren mit <Befehle aus „Probelauf">, dann approven — den Commit machst nicht du>
- <je „nicht freigeben": die Vorbedingung aus dem Prüfbericht; bei Konflikt: Renovates Rebase-Checkbox; bei CI rot: der Lauf, den der Mensch ansehen sollte>
- `rm -rf <tmp>` — Berichte und Bodies, wenn sie nicht mehr gebraucht werden
```

Leere Abschnitte bekommen eine Zeile „keine", damit sichtbar bleibt, dass sie geprüft wurden. Der Bericht ist die Ausgabe; ins Repository und auf GitHub schreibst du nichts.

**Terminierung:** Ein PR, der sich nicht klassifizieren oder prüfen lässt, hält den Lauf nicht an — er bekommt seinen Grund, und der nächste PR kommt dran. Abgebrochen wird nur in Abschnitt 1 und 2, wo ohne Werkzeug, Zugang oder Klon kein einziger PR geprüft werden könnte. Ein Subagent bekommt genau einen zweiten Anlauf; ein Ergebnis wird nie erfunden.

## 6. Haltung

Die erste Versuchung ist, den Titel zu glauben. `Update dependency lodash to v4.17.21` ist eine Vorlage, kein Beleg; das Paar steht im Diff, und die Tabelle ist die Gegenprobe. Wo beide auseinanderlaufen, ist das ein Befund, keine Auslegungssache.

Die zweite ist, Bodies und Berichte in den Kontext zu ziehen, weil es bequem ist. Ein Body sind zehn Kilobyte Release Notes, ein Prüfbericht Tausende Tokens — nach dem dritten PR ist der Lauf dann nicht mehr zu retten. Dateien und Grep, Kopfblöcke und Pfade.

Die dritte ist, die Prüfung auf dem PR-Branch laufen zu lassen, weil „da das Update ja schon drin ist". Dort hält der Lock schon das Ziel, und der Prüf-Skill fragt statt zu urteilen. Die Basis ist der Stand, der aktualisiert würde.

Die vierte ist, einen Subagent im Sitzungsverzeichnis arbeiten zu lassen. Ohne `cd <klon> &&` in jedem Befehl prüft er das falsche Projekt — und das Startverzeichnis des Menschen ist Lesezone. Der Pfad steht im Vertrag, und der Bericht belegt mit `git status --porcelain`, dass dort nichts passiert ist.

Die fünfte ist, aus „nur statisch" ein „passt schon" zu machen. Ein Probelauf, der nicht lief, hat nichts bewiesen; ein solches Paar ist nicht freizugeben, und der Grund aus „Nicht geprüft" gehört in den Bericht, damit die Umgebung repariert wird statt die Prüfung aufgeweicht.

Die sechste ist, Subagents parallel zu starten, weil es schneller ginge. Sie teilen sich den Klon, und zwei Probeläufe gleichzeitig sind zwei Worktrees, zwei Installs und ein Ergebnis, dem niemand trauen kann.

Die siebte ist, mehr zu tun als zu berichten — eine Approval, „weil ohnehin alles grün ist", oder ein Commit auf dem PR-Branch, „weil die Anpassung klein ist". Beides sind Entscheidungen des Menschen; der Bericht nennt die Befehle dafür.

Und die achte ist, den Klon stehen zu lassen oder das Temp-Verzeichnis zu löschen, in dem die Berichte liegen. Der Klon geht, die Berichte bleiben, und die Aufräumzeile steht im Bericht.
