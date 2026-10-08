---
name: divinum-officium-ingest
description: Convert OCR'd text (and page scans) of a missal or breviary into Divinum Officium data files. Use when ingesting/transcribing liturgical sources — propers of the Mass or Office, calendars, commons — into the project's section-based .txt format, for any Use or rubric version (Roman, Sarum, Dominican/OP, Monastic, Cistercian, etc.). Triggers: "ingest this breviary/missal scan", "encode this feast into DO format", "add these propers", "transcribe this Mass/Office", "import the Dominican breviary".
---

# Divinum Officium data ingestion

Convert source liturgical books (OCR text plus, ideally, the page scans) into the
project's data files. The output is **section-based `.txt` files** plus a
**calendar line**, formatted exactly as the engine expects. See
[FORMAT.md](FORMAT.md) for the complete section vocabulary, rank scale, markup,
and calendar syntax — read it before encoding.

## Core principles

1. **Never fabricate liturgical text.** Encode only what is in the source. Where
   OCR is illegible or the source is silent, insert `{?? ...}` and list every
   such gap in your final report for the user to resolve. Do not "complete"
   prayers, lessons, or antiphons from memory — a wrong word silently corrupts
   the data.
2. **Use the scan, not just the OCR.** In missals/breviaries, **red = rubric**
   (instructions, section markers) and **black = spoken text**. OCR loses color.
   Open the scan images to (a) tell rubric from text, (b) find section
   boundaries, and (c) verify Latin accents (á é í ó ú ǽ œ́), which OCR mangles.
   Flag accents you are unsure of.
3. **Only encode deltas.** The engine falls back from a Use's folder to the base
   Roman files (see FORMAT.md "Fallback"). Create a file only for material that
   actually *differs* from the fallback. Don't duplicate identical Roman text.
4. **Reuse via references, don't copy.** Prefer `@File:Section`, `vide`/`ex C…`
   commune references, and `$Prayer` endings over pasting shared text (FORMAT.md
   "Markup").
5. **Verify by rendering.** After writing files, render the day and read the
   output (command in step 7). Encoding that parses is not the same as encoding
   that's correct.

## Inputs to confirm before starting

- **Which Use / rubric version?** (e.g. Sarum, Ordo Praedicatorum, Monastic.)
  This sets the folder suffix and the version name. If the Use isn't registered
  yet, see FORMAT.md "Registering a new Use".
- **Office or Mass?** → `web/www/horas/` vs `web/www/missa/`.
- **Language?** Latin is the base layer; vernaculars layer on top (FORMAT.md
  "Languages").
- **What does the source cover?** A single feast, a season, a common, a calendar?

## Workflow

1. **Survey the source.** From OCR + scan, identify the feast(s)/day(s), the
   hour(s) or Mass parts present, and the Use. Note the feast's rank/grade as
   printed (e.g. "Duplex", "ix lect", "Festum").
2. **Pick target files.** Map to folder + filename (FORMAT.md "Folders & file
   names"): Sancti → `MM-DD.txt`; Tempora → season code; Commune → `C…`. Apply
   the Use suffix (`Sancti<Suffix>/…`). Decide per file whether it differs from
   the fallback; skip those that don't.
3. **Segment the text into sections.** Walk the source top to bottom, using red
   rubrics and the known order of an Office hour / a Mass to assign each block to
   a `[Section]` (FORMAT.md "Sections").
4. **Apply markup.** Replace standard prayer endings with `$` refs (`$Per
   Dominum`, `$Qui vivis`, `$Per eundem`, `$Pater noster`…); turn scripture
   incipits into `!Book ch:vv` headers; point antiphons/psalm verses with `*` at
   the mediation only if the source does; convert "from the Common" instructions
   into `@Commune/C…:Section` or `vide`/`ex` in `[Rule]`/`[Rank]`. Gate
   version-specific bits with `(rubrica …)` (FORMAT.md "Version conditionals").
5. **Write the header.** Every proper file starts with `[Officium]` (the title),
   `[Rank]` (`;;<RankName>;;<num>;;<commune-ref>` — pick the number from
   FORMAT.md "Rank scale"), and `[Rule]` (lectiones count, `vide C…`, `Gloria`,
   etc.).
6. **Add the calendar line.** Append/insert into
   `web/www/Tabulae/Kalendaria/<ver>.txt` under the right `*Month*`, using the
   `DATE=KEY=NAME=RANK=…` syntax (FORMAT.md "Calendar").
7. **Render to verify.** Office:
   `perl web/cgi-bin/horas/officium.pl "version=<Version>" command=pray<Hour> date=MM-DD-YYYY`
   Mass: `perl web/cgi-bin/missa/missa.pl "version=<Version>" command=praySanctaMissa date=MM-DD-YYYY`
   Read the HTML; confirm the new text appears in the right place with no
   `cannot be opened` errors.
8. **Report.** List: files created/changed, the calendar line, every `{??}` gap
   and uncertain accent, each assumption (rank number, commune chosen, endings
   substituted), and the render command you used.

## Encoding @ scale (multiple feasts/seasons)

Do a small **pilot of one feast end-to-end** first, get the user to confirm the
encoding conventions, then batch. Keep a running list of recurring decisions
(how a given rank maps to a number, which commune a class of feast uses) so the
batch stays consistent.
