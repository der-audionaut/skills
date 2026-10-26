---
name: tool-check-renovate-pr-in-repo
description: Prüft die offenen Renovate-PRs eines GitHub-Repositories in einem frischen Klon — jedes Dependency-Paar mit tool-dep-update-check in einem eigenen Subagent —, setzt nötige Anpassungen als Commit je Paket auf dem PR-Branch um, prüft sie auf Vollständigkeit, verifiziert sie und pusht und approved erst nach einer gebündelten Freigabe des Menschen; bis dahin verlässt nichts den Rechner. Nimmt die Repo-URL als Argument, optional gefolgt von PR-Nummern in Anführungszeichen, die den Lauf auf diese PRs beschränken.
argument-hint: "[repoUrl] [prs]"
arguments: repoUrl prs
disable-model-invocation: true
allowed-tools: Bash Read Grep Glob Edit Write Agent Skill
---

**Ausgabesprache: Deutsch.** Bericht, Rückfragen und Abbruchmeldungen schreibst du auf Deutsch, auch wenn Repository, PR-Titel und Changelogs englisch sind. Zitate aus PR-Bodies, Bezeichner, Befehle und Fehlermeldungen bleiben, wie sie sind.

Prüfe die offenen Renovate-PRs des GitHub-Repositories `$repoUrl` — alle, oder mit `$prs` nur die genannten. Arbeite wie jemand, der vor dem Review-Klick die Frage beantwortet, die ein Renovate-PR sonst erst nach dem Merge beantwortet: Kann dieses Projekt das Update tragen, und was wäre dafür zu tun? Jedes Dependency-Paar — ein Paket mit Ausgangs- und Zielversion — prüft `tool-dep-update-check`; du klonst, inventarisierst, klassifizierst, delegierst, setzt nötige Anpassungen im Klon um und berichtest. Nach außen veränderst du nichts ohne ein Ja: Commits entstehen im Klon, und ob sie das Repository erreichen und der PR eine Approval bekommt, entscheidet der Mensch in genau einer gebündelten Freigabe; die Arbeitskopie des Menschen fasst du nie an.

Drei Regeln tragen den ganzen Ablauf: Geschrieben wird nur unter dem Temp-Verzeichnis dieses Laufs — im Klon und in den Berichtsdateien daneben —, nie im Verzeichnis, aus dem der Skill gestartet wurde. Jede Aussage über einen PR hat einen Prüfbericht hinter sich, und wo keiner entstehen konnte, steht der Grund statt eines Urteils. Und nichts verlässt den Rechner ohne ein Ja: Bis zur Freigabe liest dieser Skill GitHub nur; danach schreibt er genau zwei Dinge — den Push der eigenen Commits und die Approval —, und nur für die PRs, die der Mensch genannt hat.

## 1. Argumente und Voraussetzungen

**Repository.** `$repoUrl` ist Pflicht. Akzeptiert sind `https://github.com/<owner>/<repo>`, mit oder ohne `.git`, mit Pfadanhang wie `/pulls`, die SSH-Form `git@github.com:<owner>/<repo>.git` und die Kurzform `<owner>/<repo>`. Alles davon wird auf `<owner>/<repo>` abgebildet; was sich nicht abbilden lässt — leer, kein GitHub-Host, nur ein Owner — nennt den Aufruf `/tool-check-renovate-pr-in-repo <repoUrl> [prs]` und bricht ab. Rate kein Repository aus der Arbeitskopie: Wer den Skill startet, nennt das Repo.

**PR-Auswahl.** `$prs` ist optional: PR-Nummern als ein Argument in Anführungszeichen (`"12 15"`), getrennt durch Leerzeichen oder Kommas, mit oder ohne `#`. Ohne Angabe gilt jeder offene Renovate-PR. Eine genannte Nummer, zu der kein offener Renovate-PR gehört, ist kein Abbruchgrund — sie steht im Bericht unter „Übersprungen" mit dem Grund „kein offener Renovate-PR".

**Werkzeuge und Rechte.** Drei Prüfungen, bevor du klonst, jede mit der Zeile, die der Mensch ausführen würde:

1. `command -v gh` — fehlt es, brich ab und nenne die Installation laut https://cli.github.com. Einen Ersatz über `curl` und Token gibt es nicht.
2. `gh auth status` — endet es mit Exit-Code 1, brich ab und nenne `gh auth login`.
3. `gh repo view <owner>/<repo> --json viewerPermission,defaultBranchRef` — `viewerPermission` muss `WRITE`, `MAINTAIN` oder `ADMIN` sein; `READ` oder `TRIAGE` heißt, dass Push und Approval unmöglich wären, und du brichst ab und sagst das. Ein Fehler hier heißt meist: Repo existiert nicht, Tippfehler in der URL oder kein Zugang; nenne die Antwort von `gh`. Den Namen des Default-Branchs aus `defaultBranchRef.name` merkst du dir.

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

**Eigene PRs.** Ist `author.login` des PRs gleich dem eigenen Login aus `gh api user --jq .login` — bei einem selbst gehosteten Renovate unter dem eigenen Konto, oder bei handgemachten PRs mit Renovate-Präfix —, kann dieses Konto ihn nicht approven: „Pull request authors cannot approve their own pull requests" (Referenz, Abschnitt F). Der PR wird geprüft und angepasst wie jeder andere, aber seine Aktion heißt `push` statt `push + approve` und `keine: eigener PR, Approval durch ein anderes Konto` statt `approve`; Abschnitt 7 überspringt die Approval mit genau diesem Grund, statt sie an GitHub scheitern zu lassen.

Die Klassifikation steht als Tabelle im Kontext, eine Zeile je PR — Nummer, Titel, Branch, Basis, Art, Paare, CI, Autor gleich eigenes Konto oder nicht —, und du schreibst sie nach jedem PR fort. Sie ist dein Zustand; der Klon ist der andere.

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
Voraussetzungen, jede Anpassung aus „Bruchstellen" mit Datei:Zeile und dem Grep-Muster
aus „Betroffen", die Befehle aus „Probelauf" (Install, Build/Typen/Lint, Tests) und den
Abschnitt „Nicht geprüft".
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
| mindestens eins `Möglich mit Anpassungen`, alle übrigen `Möglich` oder ebenfalls `Möglich mit Anpassungen` | Probelauf gelaufen | beliebig | anpassen (Abschnitt 5), dann approven — das gilt auch für Gruppen-PRs, Anpassungsbedarf ist kein Blocker; nach Abschnitt 5 lautet die Empfehlung „angepasst, approven", „nicht freigeben: Anpassung unvollständig" oder „nicht freigeben: Anpassung rot" |
| alle `Möglich` | Probelauf grün | rot | nicht freigeben: CI rot — Mensch entscheidet |
| alle `Möglich` | Probelauf grün | grün oder unbekannt | approven |

Ein Gruppen-PR mit einem blockierten, nicht prüfbaren oder nur statisch geprüften Paar ist als Ganzes nicht freizugeben, auch wenn die anderen Paare möglich wären; die Gruppe zu teilen ist Renovate-Konfiguration, nicht deine Aufgabe. Ein Gruppen-PR, dessen Paare alle möglich sind und von denen eines oder mehrere Anpassungen brauchen, ist dagegen „anpassen, dann approven" — genau wie ein Einzel-PR mit Anpassungen.

## 5. Anpassung und Commit

Ein PR mit der Empfehlung „anpassen, dann approven" bekommt seine Anpassungen jetzt — im Klon, auf dem PR-Branch, als Commits, die der Mensch später pusht oder verwirft. Du gehst erst hierher, wenn alle Paare des PRs geprüft sind und keines davon blockiert, nicht prüfbar oder nur statisch ist; was die Prüfberichte unter „Bruchstellen" mit `Datei:Zeile` nennen, ist der ganze Auftrag. Eine Beobachtung nebenbei — ein Stil, ein zweiter Bug, eine Deprecation — gehört in den Bericht, nicht in den Commit. Und der Auftrag hat Grenzen: Verlangt eine Anpassung ein neues Paket, eine Änderung an Manifest oder Lock oder an Stellen, die kein Prüfbericht nennt, setzt du sie nicht um — der PR ist „nicht freigegeben: Anpassung außerhalb des Auftrags", die verlangte Änderung steht wörtlich im Bericht, und der Mensch entscheidet, ob das Urteil des Prüf-Skills trägt. Hast du für diesen PR schon etwas geändert, wird es geparkt, nicht verworfen — siehe „Einmal verifizieren".

**Branch und Install.** `git -C "$KLON" switch -c <head> --track origin/<head>`, dann einmal installieren mit dem Lock des PR-Branchs, in einer Subshell: `(cd "$KLON" && <Install-Befehl aus dem Probelauf>)` — `npm ci`, `composer install`, was der Prüfbericht unter „Probelauf" nennt. Der Lock hält bereits alle Zielversionen des PRs; Manifest und Lock änderst du nicht.

**Je Paar ein Commit.** In der Reihenfolge der Prüfung, je Paar mit Anpassungen: die Änderungen an genau den Stellen, die sein Prüfbericht nennt, mit dem Edit-Tool und absoluten Pfaden unter `"$KLON"`; dann sofort `git -C "$KLON" add -- <pfade>` mit genau diesen Pfaden — nie `-A`, nie `.` — und der Commit. Berühren zwei Pakete dieselbe Datei, gehen deren Änderungen in einen gemeinsamen Commit, der beide Pakete im Betreff nennt. Zwischen den Commits läuft keine Verifikation, und auch vor dem ersten nicht: Der Lock des PR-Branchs hält alle Zielversionen zugleich, ein Testlauf nach dem ersten Paket wäre an der Bruchstelle des zweiten rot, ohne etwas über das erste zu sagen — und die Verifikation kommt nach dem letzten Commit, damit jeder Versuch committet vorliegt und sich parken lässt, statt mit `git checkout` zu verschwinden. Hat der Install Lock- oder Manifestdateien verändert, gehören sie nur dann in einen Commit, wenn die Anpassung sie verlangt — ein mitgezogenes Paket etwa; sonst `git -C "$KLON" checkout -- <datei>`, und der Bericht nennt es.

**Die Botschaft.** Steht in der Liste der verfügbaren Skills einer namens `write-commit-message` — mit Plugin-Präfix oder ohne —, rufst du ihn je Commit über das Skill-Tool auf, sobald der Index den Stand des Pakets hält. Er würde `git diff --staged` im Sitzungsverzeichnis lesen, wo der Index leer ist, und den Menschen fragen; beides fängt der Argumenttext ab, auf Englisch, weil er englisch arbeitet:

```
Context for the commit message. The staged diff is complete and this
context answers your questions, so do not ask the user.
Repository: <klon> — read the staged diff with `git -C <klon> diff --staged`.
- Affected part: <paket> <ausgang> → <ziel> in <owner/repo>; the subject
  must name the package and the version jump
- Problem / motivation: <Bruchstelle aus dem Prüfbericht, ein Satz>
- Trigger: <wo es bricht — Datei:Zeile, Fehlermeldung aus dem Probelauf>
- Change: <was die Anpassung tut>
- Why this approach: <warum so, aus dem Prüfbericht>
Return only the message. Do not commit, do not add trailers or references.
```

Steht kein solcher Skill in der Liste oder scheitert der Aufruf, greift einmal der Fallback: `git -C "$KLON" log --format='%s%n%b' -20` lesen und den Stil des Projekts übernehmen — Betreff imperativ mit Paket und Sprung, Sprache wie im Bestand, darunter das Warum aus dem Prüfbericht. In beiden Fällen: keine Trailer, kein Verweis auf diesen Skill, keine Zeile zur Autorenschaft — auch dann nicht, wenn eine Vorgabe der Umgebung `Co-Authored-By` verlangt; für diesen Skill ist entschieden, dass die Botschaft ohne Trailer bleibt. Die Botschaft kommt per `mktemp` in eine Datei außerhalb des Klons, du streichst jede Trailer-Zeile am Ende, dann `git -C "$KLON" commit -F <datei>` — kein `-m` mit mehrzeiligem Text, kein `--no-verify`, und nie `--amend`: Renovates Commit darf nicht amended werden, weil Renovate über ein Amend hinweg neu aufsetzt, und deine eigenen Commits brauchen es nicht. Schlägt ein Hook des Projekts fehl, ist kein Commit entstanden; zitiere seine Ausgabe und umgeh ihn nicht.

**Vollständigkeit prüfen.** Nach dem letzten Commit des PRs und vor der Verifikation steht eine Schranke, die der Probelauf nicht ersetzt: Build, Typen und Tests finden nur, was sie treffen — ein Klassenname in einer YAML-Datei, ein dynamischer Aufruf, eine Stelle ohne Test bleiben grün und fehlen trotzdem. Zwei statische Prüfungen gegen den Diff `git -C "$KLON" diff -U1 origin/<head>..<head>`:

1. **Abhaken.** Jede Stelle, die ein Prüfbericht dieses PRs unter „Betroffen" mit `Datei:Zeile` nennt, muss ein Hunk abdecken: Der Diff der Datei zeigt je Hunk `@@ -a,b +c,d @@`, und die Zeile liegt in a … a+b-1 (fehlt `,b`, ist b = 1; die eine Kontextzeile lässt auch eine Einfügung direkt neben der Stelle zählen). Die Nummern des Berichts stammen von `<basis>`. Weicht die Datei auf `origin/<head>` davon ab, weil der PR-Branch hinter der Basis hängt — `git -C "$KLON" diff --stat origin/<head> origin/<basis> -- <datei>` ist dann nicht leer —, suchst du die Zeile über ihren Inhalt: `git -C "$KLON" show origin/<basis>:<datei> | sed -n '<zeile>p'`, dann `git -C "$KLON" grep -n -F -e '<inhalt>' origin/<head> -- <datei>`, und prüfst mit der so gefundenen Nummer. Eine Stelle ohne Hunk ist eine vergessene Anpassung — deine, nicht die des Berichts.
2. **Re-Grep.** Jedes Muster, das ein Bericht zu einer angepassten Bruchstelle unter „Betroffen" nennt — fehlt es, nimmst du den Namen der API aus der Überschrift der Bruchstelle —, läuft noch einmal über den Branch-Head: `git -C "$KLON" grep -n -E '<muster>' -- ':!vendor' ':!node_modules' ':!*.lock' ':!*-lock.json' ':!*-lock.yaml'`; `git grep` sieht nur versionierte Dateien, die installierten Pakete also nicht. Jeder Treffer muss auf einer geänderten Zeile liegen — im Diff eine `+`-Zeile, keine Kontextzeile. Ein Treffer auf einer unveränderten Zeile ist ein Kandidat, den du mit `-C2` liest, bevor du ihn einordnest: eine Verwendung, die kein Bericht nennt — oder keine, weil Kommentar, Dokumentation, eigenes Symbol gleichen Namens. Jede Einordnung steht mit Grund im Bericht; ein stillschweigend verworfener Treffer ist eine vergessene Stelle mit gutem Gewissen.

Die Schranke ist passiert, wenn jede gemeldete Stelle abgedeckt ist und kein Treffer auf einer unveränderten Zeile eine Verwendung ist. Eine vergessene Stelle holst du nach, auf dem Weg, den „Einmal verifizieren" für Rot beschreibt — mixed Reset, Einarbeiten, je Paket neu committen —, und gehst erneut durch die Schranke; das ist ein Anlauf, und es gibt höchstens drei. Eine Verwendung, die kein Bericht nennt, ist dagegen keine Stelle, die du anpasst, sondern der Beleg, dass der Prüfbericht unvollständig war und sein Urteil so nicht trägt: Der PR ist „nicht freigegeben: Anpassung unvollständig — <Datei:Zeile> verwendet <API>, kein Bericht nennt sie", geparkt wie eine rote Anpassung, und der Mensch entscheidet mit dem Bericht in der Hand. Das Ergebnis beider Prüfungen — Stellen abgedeckt, Muster gelaufen, Treffer und Einordnung — kommt in den Bericht, auch wenn alles leer blieb.

**Einmal verifizieren.** Ist die Schranke passiert, laufen die Befehle aus dem Probelauf — Build, Typen, Lint, Tests, so wie der Prüfbericht sie nennt — genau einmal auf dem Branch-Head, in der Subshell im Klon. Grün → der PR ist „angepasst", die Empfehlung lautet „approven", und `git -C "$KLON" log --oneline origin/<head>..<head>` liefert SHAs und Betreffs für den Bericht. Rot → die Ursache liegt in den Anpassungen oder in einer Bruchstelle, die der Bericht nicht nannte; beides ist deine Baustelle, solange die Korrektur an den genannten Stellen bleibt und kein Paket installiert, keine Testerwartung entschärft und kein Manifest angefasst wird — eine Testdatei, die der Prüfbericht als Verwendungsstelle nennt, ist eine Anpassungsstelle wie jede andere; eine, die er nicht nennt, ist tabu. Die Korrektur läuft über `git -C "$KLON" reset origin/<head>` — mixed: HEAD und Index zurück, Arbeitskopie bleibt; nicht `--soft`, das ließe alle Pakete gestaged, und der erste Neu-Commit nähme sie alle mit —, Einarbeiten, erneut je Paket stagen und committen, erneut durch die Schranke, erneut verifizieren. Höchstens drei Anläufe, Schranke und Verifikation zusammen. Liegt die Ursache dagegen an einer Stelle, die kein Bericht nennt, oder verlangt sie ein Paket, Manifest oder Lock, gibt es keinen Anlauf — das ist „Anpassung außerhalb des Auftrags" von oben. In beiden Fällen endet der PR gleich: Du parkst, was du geändert hast, und verwirfst nichts. Parken heißt `git -C "$KLON" switch -c <head>-anpassung-rot`; liegt noch Ungestagtes aus der Anpassung herum, sicherst du es dort mit `git -C "$KLON" add -- <pfade>` und einem Commit `WIP: Anpassungsversuch <paket>` — der einzige Commit, dessen Botschaft du selbst schreibst; dann `git -C "$KLON" switch <head>` und `git -C "$KLON" reset --hard origin/<head>`. Der PR ist „nicht freigegeben: Anpassung rot" beziehungsweise „… außerhalb des Auftrags", mit der Fehlermeldung oder der verlangten Änderung in einer Zeile; Branch und Pfad stehen im Bericht, damit der Mensch den Diff lesen kann, bevor er die Anpassung selbst macht. Eine Anpassung, die nur grün wird, indem man sie zurücknimmt, ist keine — dann war das Urteil „mit Anpassungen" falsch, und das ist ein Befund für den Bericht, kein Commit.

**Zurück und weiter.** Nach jedem PR `git -C "$KLON" switch <default>`, dann der nächste. Der Rückweg je PR lautet `git -C "$KLON" branch -f <head> origin/<head>` — er steht im Bericht unter „Nächste Schritte" und wird von dir nie ausgeführt: Er würde die Commits, die du gerade gemacht hast, vom Branch nehmen. Gepusht wird hier noch nicht — der Push kommt erst nach der Freigabe in Abschnitt 7, und ohne Ja gar nicht.

## 6. Freigabe

Wenn der letzte PR geprüft und, wo nötig, angepasst ist, gibst du zuerst den Schlussbericht aus (Abschnitt 8) — vollständig, mit „Freigabe: ausstehend" und den Spalten Push, Approve und reviewDecision auf „—". Dann folgt die Freigabetabelle und genau eine Frage, und damit endet dein Zug:

```
| PR | Paket(e) | Sprung | Urteil | Belastbarkeit | Commit | Aktion |
|----|----------|--------|--------|---------------|--------|--------|
| #<n> | <paket a→b> | patch \| minor \| major | <Urteil> | <Belastbarkeit> | <sha> (<paket>), … \| — | approve \| push + approve \| push (eigener PR, Approval durch anderes Konto) \| keine: <Grund> |
Freigeben? „ja" für alle markierten, „nur #12 #15" für eine Auswahl, „nein" für keine.
```

Markiert sind die PRs mit Aktion `approve`, `push + approve` oder `push`. Gibt es keinen, entfällt die Frage, und der Bericht ist das Ende des Laufs. Die Frage stellst du als Text, nicht über ein Frage-Werkzeug: In einem nicht-interaktiven Lauf kommt dann schlicht keine Antwort, und genau das soll „nichts passiert" heißen — kein Push, keine Approval, der Klon bleibt mit seinen Commits stehen, und der Bericht hat schon alles gesagt, was der Mensch wissen muss. Die Vorgenehmigung der Werkzeuge gilt nur für den Zug, der den Skill aufgerufen hat; dass `git push` und `gh pr review` nach dem Ja noch einmal die normale Freigabe durchlaufen, ist für genau diese zwei Befehle erwünscht.

**Die Antwort** liest du wörtlich: `ja` → alle markierten PRs; `nein` → keiner, der Lauf endet mit einem Satz; `nur #12 #15` (Nummern mit oder ohne `#`, durch Leerzeichen oder Kommas getrennt) → diese, sofern sie markiert sind — eine genannte Nummer ohne Aktion nennst du und lässt sie weg. Alles andere — „ja, aber ohne #15", „erst mal #12", eine Frage — beantwortest du mit einer Rückfrage in einem Satz und wartest erneut; du rätst nicht, was gemeint war, denn jede Fehldeutung pusht in ein fremdes Repository.

## 7. Push und Approve

Je freigegebenem PR, in der Reihenfolge der Tabelle, drei Schritte — und erst alle drei für einen PR, dann der nächste:

**Push**, nur wenn Anpassungs-Commits auf `<head>` liegen: `git -C "$KLON" push origin <head>` — ohne `--force`, ohne `--force-with-lease`, niemals. Weist der Remote den Push ab, hat Renovate den Branch zwischen Klon und jetzt neu aufgesetzt (Referenz, Abschnitt E): `git -C "$KLON" fetch origin <head>`, dann auf dem Branch `git -C "$KLON" switch <head> && git -C "$KLON" rebase origin/<head>` — das trägt genau deine Commits aus `origin/<head>..<head>` auf den neuen Stand —, dann erneut pushen. Höchstens drei Anläufe; ein Konflikt im Rebase ist ein Abbruch mit `git -C "$KLON" rebase --abort`. Danach gilt der PR als „nicht gepusht: <Grund>", bekommt **keine** Approval — eine Approval auf einen PR, dessen Anpassung nicht auf dem Branch liegt, wäre falsch — und der nächste PR kommt dran. Ein PR mit Aktion `approve` ohne Commits überspringt diesen Schritt.

**Approve** — entfällt ohne Versuch, wenn der PR vom eigenen Konto stammt (Abschnitt 3): Die Spalte sagt dann „nicht erteilt: eigener PR", und der Push allein ist das Ergebnis. Sonst kommt der Approval-Text per `mktemp` in eine Datei außerhalb des Klons — in der Sprache der Commit-Botschaften des Projekts (`git -C "$KLON" log --format=%s -20` zeigt sie) und in genau dieser Form:

```
<paket> <ausgang> → <ziel>: <Urteil>, <Anpassungen in einem Halbsatz oder „no adaptations needed">; trial run
<grün oder rot mit den gelaufenen Befehlen und Testzahlen>.

<details><summary>Details</summary>
<p>

<der vollständige Inhalt von <tmp>/reports/pr-<n>-<paket>.md>

</p>
</details>
```

Je Paar eine Kopfzeile und ein eigener `<details>`-Block mit dem vollen Prüfbericht — bei mehreren Paaren mit `<summary>Details: <paket></summary>` —, denn der Bericht liegt sonst nur auf deinem Rechner, und wer den PR liest, soll die Belege sehen, ohne dich zu fragen; die Leerzeilen um den Inhalt braucht GitHub, damit es das Markdown darin rendert. Bei Anpassungs-Commits steht vor der Schlusszeile der Satz, dass diese Commits (SHAs) auf dem Branch liegen und Renovates Rebase-Checkbox sie verwerfen würde. Die Schlusszeile `by tool-check-renovate-pr-in-repo` ist in jeder Sprache gleich und macht sichtbar, dass der Skill die Review geschrieben hat; einen Kontonamen enthält der Text nicht, denn GitHub führt die Review ohnehin unter dem angemeldeten Konto. Beide Skill-Namen enthalten keine Links auf ihre Quelle. Kein Trailer, keine Floskeln. Dann `gh pr review <n> --repo <owner>/<repo> --approve --body-file <datei>`. Schlägt das fehl — „Can not approve your own pull request", fehlendes Recht, Netz —, ist der PR „nicht erteilt: <Ausgabe von gh in einer Zeile>", und du versuchst es nicht erneut: Eine zweite Approval ändert nichts, was die erste nicht konnte.

**Nachlesen:** `gh pr view <n> --repo <owner>/<repo> --json reviewDecision,reviews`. Die eigene Review muss in `reviews` mit `author.login` gleich `<login>` und `state` `APPROVED` stehen — sonst gilt „nicht erteilt", egal was `gh pr review` gemeldet hat. `reviewDecision` liest du dreiteilig: `APPROVED` → zählt; leer → keine Review-Pflicht konfiguriert, die Approval steht; `REVIEW_REQUIRED` trotz eigener Approval → zählt nicht, weil eine Branchregel („Require approval of the most recent reviewable push" nach deinem eigenen Push, oder „Dismiss stale approvals") sie nicht gelten lässt — ein zweiter Mensch muss approven, und du approvst nicht noch einmal; `CHANGES_REQUESTED` → jemand anderes hat Änderungen verlangt, das steht so im Bericht.

**Aufräumen und Nachtrag.** Sind alle freigegebenen PRs durch, gilt für den Klon dieselbe Regel wie in Abschnitt 8: Er geht nur, wenn kein PR mehr ungepushte Commits trägt — auch keiner, der nicht freigegeben wurde — und kein geparkter Branch existiert; sonst bleibt er mit Pfad. Dann der Nachtrag zum Bericht: die Tabelle aus Abschnitt 8 mit gefüllten Spalten Push, Approve und reviewDecision, die Zeile „Freigabe" mit der Antwort, die Klon-Zeile und die nächsten Schritte — der Merge ist Sache des Menschen, und für jeden PR, der nicht freigegeben oder nicht gepusht wurde, stehen die Befehle noch einmal da.

## 8. Bericht und Aufräumen

Vor dem Bericht räumst du den Klon ab — `rm -rf "$KLON"` —, aber nur, wenn nichts Ungepushtes darin liegt: kein Anpassungs-Commit, kein geparkter Branch. Vor der Freigabe ist das nur der Fall, wenn kein PR angepasst wurde; sonst bleibt er stehen, mit Pfad und Rückweg im Bericht, denn die Commits darin sind das Ergebnis dieses Laufs und der Gegenstand der Freigabe. `"$TMP/reports/"` und `"$TMP/prs/"` bleiben in jedem Fall, weil der Bericht dorthin verlinkt; `rm -rf "$TMP"` ist die Aufräumzeile für den Menschen, wenn er fertig ist. Ist das Startverzeichnis ein Git-Repository, zeigt `git status --porcelain` dort denselben Stand wie vor dem Lauf, und das steht als Zeile im Bericht.

```
# Renovate-PRs: <owner/repo> — <n> offen, davon <r> Renovate-PRs: <a> approven, <b> angepasst und approven, <c> nicht freigeben, <d> übersprungen

**Klon:** <pfad> — entfernt | bleibt stehen (Anpassungs-Commits auf <head>, … | geparkter Branch <head>-anpassung-rot) → nach dem Push `rm -rf <pfad>`; Berichte unter <tmp>/reports/ → `rm -rf <tmp>` räumt alles
**Startverzeichnis:** unverändert (`git status --porcelain` leer) | kein Git-Repository
**Freigabe:** ausstehend — Frage folgt | ja (alle) | Auswahl: #… | nein | keine Antwort — nichts gepusht, nichts approved | entfällt — kein PR mit Aktion

| PR | Paket(e) | Sprung | Urteil | Belastbarkeit | CI | Empfehlung | Commit | Push | Approve | reviewDecision |
|----|----------|--------|--------|---------------|----|------------|--------|------|---------|----------------|
| #<n> | <paket a→b> | patch \| minor \| major | <Urteil> | <Belastbarkeit> | grün \| rot \| unbekannt | approven \| angepasst, approven \| nicht freigeben: <Grund> | <sha> (<paket>), … \| — | gepusht @ <sha> \| nicht gepusht: <Grund> \| — | erteilt \| nicht erteilt: eigener PR \| nicht erteilt: <Grund> \| — | APPROVED \| leer: keine Review-Pflicht \| REVIEW_REQUIRED: zweiter Mensch nötig \| CHANGES_REQUESTED \| — |

## PR #<n> — <Titel>
- **Branch:** <head> → <basis>, Autor <login>, gefunden über Autor | Branchpräfix
- **Prüfung:** <Kopfblock je Paar — Urteil, Belastbarkeit, Auflösung, Voraussetzungen>; voller Bericht: <tmp>/reports/pr-<n>-<paket>.md
- **Anpassung:** je Paket <sha> „<Betreff>" — <dateien>; Verifikation des Branch-Heads: <befehl> → grün (Commits einzeln ungeprüft) | rot: <Fehler in einer Zeile> → Versuche geparkt auf <head>-anpassung-rot, Branch gleich origin/<head> | unvollständig: <Datei:Zeile> verwendet <API>, kein Bericht nennt sie → Versuche geparkt auf <head>-anpassung-rot | keine nötig
- **Vollständigkeit:** <n>/<n> gemeldete Stellen im Diff, Re-Grep über <m> Muster nur auf geänderten Zeilen | <k> Treffer auf unveränderten Zeilen: <Datei:Zeile> — <Einordnung>, … | entfällt: keine Anpassung
- **Push:** origin/<head> @ <sha> | nicht gepusht: <Grund> | nichts zu pushen   (nach der Freigabe)
- **Approve:** erteilt — reviewDecision APPROVED | leer: keine Review-Pflicht, Approval steht | REVIEW_REQUIRED: zählt nicht, zweiter Mensch nötig | nicht erteilt: <Grund>   (nach der Freigabe)
- **Nicht geprüft:** <aus dem Prüfbericht> | nichts

## Übersprungen
- #<n> <Titel> — Draft | Konflikt mit <basis> | eigene Approval auf dem aktuellen Head | Lockfile-Maintenance | Docker/Actions/Digest | Pin | Rollback | Ersetzung | kein offener Renovate-PR (aus `prs`)

## Nicht prüfbar
- #<n> <Titel> — Manifest und Body widersprechen sich: <beide Fassungen> | keine Dependency-Änderung erkannt: <Pfade> | Subagent ohne Urteil | Rückfrage des Prüf-Skills: <Satz> | Branch nicht erreichbar

## Nächste Schritte
- <vor der Freigabe: die Freigabefrage folgt; danach: je approvtem PR der Merge — `gh pr merge <n> --repo <owner/repo>` oder der Weg des Projekts —, den machst nicht du>
- <je „nicht freigegeben" oder „nicht gepusht": `git -C <klon> push origin <head>` und `gh pr review <n> --repo <owner/repo> --approve`, falls der Mensch es doch will; Rückweg `git -C <klon> branch -f <head> origin/<head>`>
- <je „Anpassung rot" oder „Anpassung unvollständig": `git -C <klon> diff origin/<head>..<head>-anpassung-rot` zeigt die Versuche; die Anpassung macht der Mensch mit dem Prüfbericht und der nicht gemeldeten Stelle in der Hand>
- <je „nicht freigeben": die Vorbedingung aus dem Prüfbericht; bei Konflikt: Renovates Rebase-Checkbox; bei CI rot: der Lauf, den der Mensch ansehen sollte>
- `rm -rf <tmp>` — Berichte, Bodies und, nach dem Push, der Klon
```

Leere Abschnitte bekommen eine Zeile „keine", damit sichtbar bleibt, dass sie geprüft wurden. Auf GitHub schreibst du nur nach einem Ja und nur zwei Dinge: den Push der eigenen Commits und die Approval.

**Terminierung:** Ein PR, der sich nicht klassifizieren, prüfen, anpassen oder pushen lässt, hält den Lauf nicht an — er bekommt seinen Grund, und der nächste PR kommt dran. Abgebrochen wird nur in Abschnitt 1 und 2, wo ohne Werkzeug, Zugang oder Klon kein einziger PR geprüft werden könnte. Ein Subagent bekommt genau einen zweiten Anlauf, eine unvollständige oder rote Anpassung zusammen höchstens drei, ein abgewiesener Push höchstens drei, eine Approval genau einen; die Freigabefrage wird einmal gestellt, eine unklare Antwort einmal je Nachricht zurückgefragt; ein Ergebnis wird nie erfunden.

## 9. Haltung

Die erste Versuchung ist, den Titel zu glauben. `Update dependency lodash to v4.17.21` ist eine Vorlage, kein Beleg; das Paar steht im Diff, und die Tabelle ist die Gegenprobe. Wo beide auseinanderlaufen, ist das ein Befund, keine Auslegungssache.

Die zweite ist, Bodies und Berichte in den Kontext zu ziehen, weil es bequem ist. Ein Body sind zehn Kilobyte Release Notes, ein Prüfbericht Tausende Tokens — nach dem dritten PR ist der Lauf dann nicht mehr zu retten. Dateien und Grep, Kopfblöcke und Pfade.

Die dritte ist, die Prüfung auf dem PR-Branch laufen zu lassen, weil „da das Update ja schon drin ist". Dort hält der Lock schon das Ziel, und der Prüf-Skill fragt statt zu urteilen. Die Basis ist der Stand, der aktualisiert würde — die Anpassung dagegen gehört auf den PR-Branch, weil sie zum Update gehört.

Die vierte ist, einen Subagent im Sitzungsverzeichnis arbeiten zu lassen. Ohne `cd <klon> &&` in jedem Befehl prüft er das falsche Projekt — und das Startverzeichnis des Menschen ist Lesezone. Der Pfad steht im Vertrag, und der Bericht belegt mit `git status --porcelain`, dass dort nichts passiert ist.

Die fünfte ist, aus „nur statisch" ein „passt schon" zu machen. Ein Probelauf, der nicht lief, hat nichts bewiesen; ein solches Paar ist nicht freizugeben, und der Grund aus „Nicht geprüft" gehört in den Bericht, damit die Umgebung repariert wird statt die Prüfung aufgeweicht.

Die sechste ist, Subagents parallel zu starten, weil es schneller ginge. Sie teilen sich den Klon, und zwei Probeläufe gleichzeitig sind zwei Worktrees, zwei Installs und ein Ergebnis, dem niemand trauen kann.

Die siebte ist, nach jedem Commit zu verifizieren, weil es gründlicher aussieht. Auf einem Gruppen-Branch ist der Lauf nach dem ersten Paket an der Bruchstelle des zweiten rot — ein Ergebnis, das nichts bedeutet und drei Anläufe kostet. Einmal, nach dem letzten Commit.

Die achte ist, eine rote Anpassung grün zu machen, indem man sie zurücknimmt, einen Test entschärft oder ein Paket nachinstalliert. Das Erste macht aus „mit Anpassungen" ein falsches „möglich", das Zweite versteckt den Bruch, das Dritte ist ein anderes Update. Geparkt und berichtet ist ehrlicher als grün.

Die neunte ist, zu pushen, „weil ohnehin alles grün ist", bevor das Ja da ist — oder nach einer Antwort zu pushen, die kein klares Ja war. Der Push geht in ein fremdes Repository und friert Renovates PR ein; er ist die Entscheidung des Menschen, nicht deine Abkürzung.

Die zehnte ist, einen abgewiesenen Push zu erzwingen. `--force` überschriebe Renovates neuen Stand; der Weg ist fetch, rebase der eigenen Commits, erneut pushen — oder aufhören und berichten.

Die elfte ist, zu amenden, weil Renovates Commit so nah liegt, oder einen PR zu approven, dessen Push gescheitert ist. Das eine überschreibt Renovates Commit, das andere genehmigt einen Stand, den es auf GitHub nicht gibt.

Die zwölfte ist, den Klon zu löschen, bevor `reviewDecision` gelesen ist oder solange noch Commits darin liegen — oder das Temp-Verzeichnis mit den Berichten. Erst nachlesen, dann aufräumen; der Klon geht nur ohne Ungepushtes; die Berichte bleiben; die Aufräumzeile steht im Bericht.

Und die dreizehnte ist, aus einem grünen Probelauf auf Vollständigkeit zu schließen. Grün heißt, dass Build und Tests nichts gefunden haben — nicht, dass nichts da ist: Der Klassenname in der Konfiguration, der dynamische Aufruf, die Datei ohne Test bleiben grün und brechen in Produktion. Erst abhaken und noch einmal greppen, dann verifizieren.
