---
name: plan-lint
description: Räumt einen Feature-Plan auf — entfernt totes Wissen, vereinheitlicht Inkonsistenzen und löst Widersprüche zwischen Architektur, Datenfluss, API, Schema und Caching auf, iterativ bis ein Durchgang keine neuen Findings mehr bringt. Nimmt die Bezeichnung des Plans als Argument.
argument-hint: "[plan-datei]"
arguments: plan
disable-model-invocation: true
allowed-tools: Read Grep Glob Edit
---

**Ausgabesprache: die Sprache des Plans.** Bericht, Rückfragen, Abbruchmeldungen und Lint-Historie schreibst du in der Sprache, in der der Plan verfasst ist — bis der Plan gelesen ist, auf Deutsch. Dass diese Anweisung deutsch ist, ändert daran nichts.

Räume den Plan `$plan` auf. Lies ihn wie ein erfahrener Systemarchitekt, der ein fremdes Entwurfsdokument übernimmt: Trägt der Entwurf durchgängig dieselbe Vorstellung vom System, oder stehen in Kapitel 2 und Kapitel 6 zwei verschiedene Systeme?

Du prüfst hier nicht, ob der Plan gut ist — das macht `/plan-review`. Du prüfst, ob er in sich stimmt, und stellst her, was du belegbar herstellen kannst.

## 1. Plan auflösen

**Plan.** `$plan` kann ein Pfad, ein Dateiname oder ein Namensfragment ohne Endung sein.

1. Existiert `$plan` als Pfad → nimm ihn.
2. Sonst per Glob `**/*$plan*.md`, bevorzugt in `docs/plans/`, `.claude/plans/`, `plans/`, `docs/`, Projektwurzel.
3. Mehrere Treffer → auflisten und abbrechen, statt zu raten.

**Sprache.** Die Ausgabesprache ist die Sprache des Plans. Kopfblock und Überschriften entscheiden — nicht Zitate, Bezeichner oder Codeblöcke, die in einem deutschen Plan oft englisch sind. Die Sprache ändert den Text, nicht die Struktur: Das Berichtsformat bleibt, nur seine Beschriftungen werden übersetzt.

**Historie.** Wo diese Anweisung `## Review-Historie`, `## Umsetzungs-Historie` oder `## Lint-Historie` sagt, ist jeweils der Abschnitt dieser Bedeutung gemeint, gleich in welcher Sprache seine Überschrift steht — ein Plan kann aus `/plan-create` in einer anderen Sprache stammen. Einen vorhandenen erkennst du auch in Übersetzung — und entfernst ihn ebenso wenig —, einen neuen legst du in der Plansprache an. Die Finding-Kennungen T, I und W bleiben unübersetzt, damit Historie und Mensch sie eindeutig referenzieren können.

Den Plan selbst lintest du auf Inhalt, nicht auf Sprache. Ein Plan, der Deutsch und Englisch mischt, ist erst dann ein I-Finding, wenn dasselbe Ding dadurch zwei Namen trägt.

Lies den Plan vollständig, bevor du irgendetwas änderst. Inkonsistenzen erkennt man nur im Ganzen — der Abschnitt, der falsch aussieht, ist oft der richtige.

## 2. Findings sammeln

Drei Kategorien, die Erkennungsmerkmale stehen in `${CLAUDE_SKILL_DIR}/references/lint-kategorien.md`:

- **T — Totes Wissen.** Aussagen über einen Zustand, den es nicht mehr gibt: erledigte TODOs, Platzhalter, Dubletten, Beschreibungen bereits umgesetzter Schritte im Futur, Verweise auf Dateien oder Felder, die nicht existieren, Abschnitte, auf die kein Schritt zugreift.
- **I — Inkonsistenzen.** Dasselbe Ding, verschieden benannt oder verschieden typisiert. Feld heißt im Schema-Abschnitt anders als in der API. Schrittnummern, die nach einer Einfügung nicht mehr aufgehen.
- **W — Widersprüche.** Zwei Aussagen, die nicht gleichzeitig gelten können. Synchron im Request und gleichzeitig im Job. Feld nullable und Pflichtfeld. Cache-Gültigkeit an zwei Stellen mit zwei Werten.

Jedes Finding bekommt eine stabile ID (T1, I1, W1, …), die Fundstelle als Abschnitt oder Zeile und ein Zitat des betroffenen Satzes. Ein Widerspruch braucht **beide** Fundstellen.

Belege deine Einstufung. Ob eine Aussage tot ist, entscheidet nicht ihr Alter, sondern die Codebase: Existiert die Datei, das Feld, die Funktion? Nutze Grep und Read, statt zu vermuten. Was du nicht belegen kannst, ist kein Finding, sondern eine Frage.

## 3. Findings behandeln

- **T → entfernen.** Ersatzlos. Wenn beim Entfernen ein Verweis irgendwo anders ins Leere zeigt, gehört das Nachziehen zur selben Änderung.
- **I → vereinheitlichen.** Auf die Variante, die im Code, in `CONTEXT.md` oder in einem ADR belegt ist. Gibt es keinen Beleg, nimm die Variante, die im Plan überwiegt, und sag im Bericht, dass es eine Konvention und kein Beleg ist.
- **W → nur auflösen, wenn eine Seite belegbar falsch ist.** Der Beleg kann im Code liegen, in einer späteren Entscheidung der `## Review-Historie` oder in einer bereits erfolgten Umsetzung. Steht Aussage gegen Aussage ohne Beleg, ist das eine Entscheidung und keine Aufräumarbeit: leg beide Seiten vor, frag nach, ändere nichts.

**Was du niemals entfernst:**

- Begründungen und verworfene Alternativen. Sie sehen überflüssig aus und sind der Grund, warum der Plan aussieht, wie er aussieht.
- `## Review-Historie`, `## Umsetzungs-Historie`, `## Lint-Historie`.
- Offene Fragen an den Menschen, auch unbeantwortete. Besonders die.
- Anforderungen, nur weil kein Schritt sie abdeckt. Das ist eine Lücke, kein totes Wissen — melde sie, lösche sie nicht.

Im Zweifel gilt: nicht entfernen, sondern als Finding vorlegen. Ein zu voller Plan ist ein kleineres Problem als ein Plan, dem die entscheidende Zeile fehlt.

## 4. Iteration

Eine Runde ist: sammeln → behandeln → den geänderten Plan erneut vollständig lesen.

Das erneute Lesen ist der Kern, denn Aufräumen erzeugt neue Findings. Entfernte Passagen hinterlassen Verweise ins Leere. Vereinheitlichte Begriffe machen zwei Absätze plötzlich zu Dubletten. Ein aufgelöster Widerspruch macht sichtbar, dass ein dritter Abschnitt derselben falschen Seite folgte.

**Terminierung:** Die Schleife endet, wenn ein vollständiger Durchgang kein neues T-, I- oder W-Finding mehr ergibt. Sag das dann klar und hör auf zu suchen.

Damit sie auch terminiert:

- Ab Runde 2 keine reinen Stilbefunde mehr — Formulierung, Reihenfolge, Überschriftenebenen sind kein Lint, solange sie nicht mehrdeutig sind.
- Ein Finding, das der Mensch abgelehnt hat, wird nicht neu aufgemacht.
- Nach drei Runden ohne Fortschritt hörst du auf. Was dann noch offen ist, ist in aller Regel ein ungeklärter Widerspruch, und der gehört dem Menschen.

## 5. Bericht

```
# Plan-Lint: <Plan-Datei> — Runde <N>

**Ergebnis:** <x entfernt, y vereinheitlicht, z Widersprüche offen>
**Geprüft gegen:** <Dateien/Dokumente, die du für die Belege gelesen hast>

## Entfernt (T)
### T1 — <Titel>
- **Fundstelle:** <Abschnitt>
- **Zitat:** "<der entfernte Satz>"
- **Beleg:** <warum tot, z.B. Datei:Zeile existiert nicht / Schritt 2 abgeschlossen>

## Vereinheitlicht (I)
### I1 — <Titel>
- **Vorher:** "<Variante A>" (<Abschnitt>) / "<Variante B>" (<Abschnitt>)
- **Nachher:** <gewählte Variante>
- **Beleg:** <Datei:Zeile / ADR / Konvention im Plan>

## Widersprüche (W)
### W1 — <Titel>  [aufgelöst | offen — Entscheidung nötig]
- **Aussage A:** "<Zitat>" (<Abschnitt>)
- **Aussage B:** "<Zitat>" (<Abschnitt>)
- **Beleg / Frage:** <auflösender Beleg, oder die Frage an dich>

## Unverändert gelassen
- <was nach Finding aussah, aber keins ist, mit einem Satz warum>
```

Zitiere jede Entfernung. Der Bericht ist die einzige Stelle, an der rekonstruierbar bleibt, was verschwunden ist — und die Grundlage dafür, dass du eine Löschung zurückholen kannst, ohne den Diff zu lesen.

## 6. Historie

Nach der letzten Runde eine kompakte Zeile in `## Lint-Historie` am Ende des Plans, angelegt falls nicht vorhanden:

```
## Lint-Historie
### <Datum> — 2 Runden — 4 entfernt, 3 vereinheitlicht, 1 Widerspruch offen (W1: Cache-Gültigkeit)
```

In einem fremdsprachigen Plan trägt der neue Abschnitt die übersetzte Überschrift — `## Lint History` — und seine Zeilen folgen der Plansprache; die Vorlage hier ist deutsch, weil diese Anweisung es ist, nicht weil der Abschnitt es sein müsste.

Kein Protokoll der einzelnen Zeilen — dafür gibt es die Versionsverwaltung. In der Historie steht nur, was offen blieb, damit der nächste Durchgang dort ansetzt.

## 7. Haltung

Die Versuchung eines Aufräum-Durchgangs ist, Kürze mit Qualität zu verwechseln. Ein Plan ist nicht besser, weil er kürzer ist — er ist besser, wenn nichts mehr drinsteht, das in die Irre führt. Kontext, Begründung und Randbedingung sind kein Ballast.

Die zweite Versuchung ist, einen Widerspruch aufzulösen, indem man sich für eine Seite entscheidet. Das ist keine Aufräumarbeit, sondern eine Architekturentscheidung, die dir nicht zusteht, solange kein Beleg sie trägt.

Und bevor du absendest: Steht die Antwort in der Sprache des Plans? Die deutsche Anweisung zieht sonst ins Deutsche, auch wenn der Plan englisch ist.
