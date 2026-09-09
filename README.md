# Skills

Eigene Skills für agentische Entwicklungswerkzeuge. Ein Skill ist eine Arbeitsanweisung in Markdown, die der Agent bei passender Gelegenheit lädt — entweder automatisch, weil die Anfrage zur Beschreibung passt, oder ausdrücklich per `/name`.

Zwei Familien liegen hier: `news-*` erzeugt recherchierte Briefings, `plan-*` bearbeitet Feature-Pläne über ihren gesamten Lebenszyklus.

## Übersicht

| Verzeichnis | Aufruf | Zweck |
| --- | --- | --- |
| `skills/news-nachrichtenlage` | automatisch oder `/news-nachrichtenlage` | Nachrichten-Briefing aus Live-Recherche, das Gesichertes, Deutung und Unbestätigtes trennt |
| `skills/news-wirtschafts-briefing` | automatisch oder `/news-wirtschafts-briefing` | Wirtschafts- und Finanzmarkt-Briefing: Marktbild, Leitthema, Konjunktur, Termine |
| `skills/plan-review` | `/plan-review <plan>` | Prüft einen Plan gegen die reale Codebase, bevor jemand ihn umsetzt |
| `skills/plan-lint` | `/plan-lint <plan>` | Räumt einen Plan auf: totes Wissen, Inkonsistenzen, Widersprüche |
| `skills/plan-coding` | `/plan-coding <plan> <schritt>` | Setzt genau einen Schritt des Plans um, inklusive Findings-Runden |

Ein Skill heißt so wie sein Verzeichnis. Weil die Skills als Plugin `skills` geladen werden, lautet der volle Name `/skills:plan-review`; die Kurzform `/plan-review` funktioniert, solange kein anderes Plugin einen gleichnamigen Skill mitbringt.

Die News-Skills darf der Agent von selbst ziehen, wenn eine Anfrage zu ihrer Beschreibung passt. Die Plan-Skills tragen `disable-model-invocation: true` — sie laufen nur, wenn du sie ausdrücklich startest, weil sie fremde Dateien anfassen.

## Die Plan-Skills als Kette

Die drei Plan-Skills sind Stationen eines Ablaufs und teilen sich einen Mechanismus: Jeder arbeitet in Runden, jeder schreibt sein Ergebnis in einen eigenen Historien-Abschnitt am Ende der Plandatei, und jeder hört auf, sobald eine vollständige Runde nichts Neues mehr findet.

```
Plan entsteht
     │
     ├─ /plan-review   → trägt der Plan gegen den echten Code?        → ## Review-Historie
     ├─ /plan-lint     → ist der Plan in sich widerspruchsfrei?       → ## Lint-Historie
     └─ /plan-coding   → Schritt für Schritt umsetzen                 → ## Umsetzungs-Historie
```

Die Reihenfolge ist keine Vorschrift. `/plan-lint` lohnt sich besonders, nachdem mehrere Schritte umgesetzt wurden und der Plan Aussagen über einen Zustand enthält, den es nicht mehr gibt. `/plan-review` lohnt sich vor der ersten Zeile Code — und noch einmal, wenn die Umsetzung den Plan spürbar verändert hat.

Die Historien-Abschnitte sind der Übergabepunkt zwischen den Läufen. Ohne sie beginnt jeder Durchgang blind von vorn und macht abgelehnte Befunde neu auf.

**Plan finden:** Alle drei nehmen einen Pfad, einen Dateinamen oder ein Namensfragment. Gesucht wird bevorzugt in `docs/plans/`, `.claude/plans/`, `plans/`, `docs/` und der Projektwurzel. Bei mehreren Treffern bricht der Skill ab und listet auf, statt zu raten.

## Die News-Skills

Beide erzeugen Fließtext im Chat, deutsch, ohne Datei — und beide haben denselben Kern: Nicht die Meldung ist der Wert, sondern ihre Einordnung.

`news-nachrichtenlage` ordnet jede Aussage einer von drei Ebenen zu (gesichert / Deutung / unbestätigt) und arbeitet Quellen von unten nach oben ab: Primärquellen, Agenturen, deutsche Leitmedien über das Spektrum, internationale Presse, unabhängige Medien. Der eigentliche Mehrwert steckt im Abschnitt *Wo die Berichterstattung auseinandergeht* — dort wird benannt, ob eine Differenz auf Fakten, Gewichtung, Deutung oder Auslassung beruht.

`news-wirtschafts-briefing` beginnt mit einem einzigen Überblicks-Abruf, der Kurse und redaktionelle Gewichtung zugleich liefert, vertieft daraus das Leitthema und prüft jede Zahl an der Primärquelle — mit besonderem Augenmerk auf den Bezugszeitraum, weil Suchergebnisse Monate munter mischen.

Beide Skills haben Voreinstellungen (Sprache, Länge, Schwerpunkt) und je eine leere Liste, die auf dich wartet: **Dauerthemen** in `news-nachrichtenlage`, **Watchlist** in `news-wirtschafts-briefing`. Wenn du eine Einstellung dauerhaft anders willst, ändere sie direkt in der `SKILL.md` — genau dafür stehen die Blöcke dort.

## Installation

Das Repo ist zugleich Plugin-Marketplace und Plugin: `.claude-plugin/marketplace.json` beschreibt den Marketplace, `.claude-plugin/plugin.json` das Plugin `skills`, und alles unter `skills/` ist ein Skill. Claude Code holt sich das Plugin direkt aus GitHub — kein Clone, kein `git pull`, kein Symlink pro Skill.

Einmalig pro Rechner:

```bash
claude plugin marketplace add der-audionaut/skills
claude plugin install skills@der-audionaut
```

Danach Claude Code neu starten; `claude plugin list` zeigt das Plugin, `/help` die Skills. Wer die Skills bisher per Symlink eingehängt hatte, entfernt die alten Links vorher, sonst laden sie doppelt:

```bash
rm ~/.claude/skills/{news-nachrichtenlage,news-wirtschafts-briefing,plan-review,plan-lint,plan-coding}
```

**Aktualisieren.** Das Plugin trägt bewusst keine `version`: Claude Code nimmt dann den Commit-SHA als Version, und jeder Push ist ein Update. Manuell holt es `claude plugin update skills@der-audionaut`; automatisch geht es, wenn im `/plugin`-Dialog unter *Marketplaces* das Auto-Update für `der-audionaut` eingeschaltet ist (für fremde Marketplaces ist es standardmäßig aus). Ein neuer Skill ist ein neues Verzeichnis unter `skills/` — kein Eintrag in einer Manifestdatei, kein Symlink; er kommt mit dem nächsten Update auf alle Rechner.

**Entwicklungsrechner.** Wo die Skills bearbeitet werden, ist die Installation aus GitHub im Weg, weil sie eine Kopie des letzten Commits lädt. Dort stattdessen das ganze Repo einmal verlinken:

```bash
ln -sfn "$PWD" ~/.claude/skills/skills
```

Claude Code lädt das Verzeichnis als Plugin `skills@skills-dir`, Änderungen wirken ohne Update sofort, neue Verzeichnisse unter `skills/` ebenso. Nicht beides zugleich: Ist das Plugin aus dem Marketplace installiert, gewinnt es, und der Symlink wird mit einem Hinweis übersprungen.

**Prüfen.** `claude plugin validate .` prüft Manifeste und Skill-Frontmatter; `claude plugin details skills@der-audionaut` (bzw. `skills@skills-dir`) listet die erkannten Skills.

## Aufbau eines Skills

```
skills/<verzeichnis>/
├── SKILL.md              # Frontmatter + Anweisung — wird immer geladen
└── references/           # optional, wird nur bei Bedarf nachgeladen
    └── <thema>.md
```

Im Frontmatter steuern:

- `name` — nur Dokumentation; aufgerufen wird der Skill unter seinem Verzeichnisnamen
- `description` — entscheidet, ob der Agent den Skill von selbst zieht; deshalb enthält sie bewusst viele Formulierungsvarianten der Anfrage
- `allowed-tools` — Werkzeuge, auf die der Skill beschränkt bleibt
- `disable-model-invocation` — `true` verhindert den automatischen Aufruf
- `argument-hint` / `arguments` — benannte Argumente, im Text als `$plan`, `$schritt` verwendbar

Die Dateien unter `references/` sind Auslagerungen der langen Listen — Prüfkriterien, Entwurfsdimensionen, Lint-Kategorien. Sie landen nur im Kontext, wenn der Skill sie über die Variable mit dem Skill-Verzeichnis (`${CLAUDE_SKILL_DIR}`) tatsächlich liest. Das hält die `SKILL.md` lesbar und den Kontext klein.

## Konventionen

- **Deutsch**, durchgängig — Anweisung, Bericht und Ausgabe.
- **Prosa statt Stichwortliste.** Die Skills erklären, *warum* eine Regel gilt; eine Regel ohne Begründung wird in der Umsetzung als Erste gebogen.
- **Jeder iterative Skill braucht ein Abbruchkriterium.** Ohne Terminierung erfindet ein Agent in Runde drei Findings, um beschäftigt zu wirken. Deshalb überall dasselbe Muster: keine neuen Optional-Befunde ab Runde 2, abgelehnte Befunde bleiben abgelehnt, nach drei Runden ohne Fortschritt entscheidet der Mensch.
- **Ein Abschnitt „Vor dem Absenden prüfen" bzw. „Haltung" am Ende.** Dort steht, was typischerweise schiefgeht — die häufigste Fehlerquelle ist nie die fehlende Regel, sondern die bekannte Versuchung.
