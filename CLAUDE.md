# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Divinum Officium generates the texts of the traditional Roman liturgy — the Divine
Office (Breviary) and the Mass — for any date and across multiple historical
rubric versions (e.g. Trident 1570, Divino Afflatu, Rubrics 1960) and many
vernacular languages. It is a Perl CGI application; the bulk of the project is
actually plain-text liturgical *data*, not code. This fork (`-sarum`) tracks the
Sarum Use alongside the upstream content.

## Architecture

The application is split into a thin computation layer (Perl) and a very large
content layer (text data files). Understanding the data layer is usually more
important than the code.

### Code layer (`web/cgi-bin/`)
- `horas/` — the Office. `officium.pl` is the main entry point (a CGI script run
  per-hour). `Cofficium.pl`/`Pofficium.pl` are the comparison and print variants
  (`C`/`P` prefixes). `horas.pl`, `horascommon.pl`, `specials.pl`,
  `kalendar.pl`, `monastic.pl` etc. are included via the `do`/`require` style.
- `missa/` — the Mass. `missa.pl` is the entry point; `propers.pl`, `ordo.pl`,
  `Cmissa.pl`/`Emissa.pl` are variants.
- `DivinumOfficium/` — the reusable Perl modules (`use strict`, exported subs):
  - `Date.pm` — liturgical calendar date math (`date_to_days`, `prevnext`, …).
  - `Directorium.pm` — resolves which feast/office "wins" on a given date.
  - `SetupString.pl` — the data-file parser: loads a `.txt` file into named
    sections and resolves cross-references (the heart of the data engine).
  - `LanguageTextTools.pm` — vernacular lookup, rubric/prayer/translate helpers,
    Alleluia handling.
  - `Scripting.pm` — registry for in-data `&function` calls (see below).
  - `RunTimeOptions.pm` — validates `version`/`hora`/`language` CGI params.
  - `Main.pm`, `FileIO.pm` — vernacular list and file reading.

The CGI scripts predate the modules and are mostly procedural, sharing state via
package globals (`our $version`, `our $winner`, `our $lang1`, …). They do **not**
use `strict`. New, factorable logic should go in a `DivinumOfficium/*.pm` module
that *does* use `strict`/`warnings`.

### Data layer (`web/www/horas/` and `web/www/missa/`)
One subdirectory per language (`Latin`, `English`, `Magyar`, …) plus structural
dirs like `Ordinarium`, `Psalterium`, `Tempora` (temporal/seasonal),
`Sancti` (sanctoral/saints' days), `Commune` (commons). Suffixed variants
(`SanctiM`, `TemporaCist`, `SanctiOP`) hold Monastic/Cistercian/Dominican proper
content.

Data files are UTF-8 (Windows-1252 is supported but deprecated). Each file is a
series of sections introduced by a name in square brackets:

```
[Officium]
Dominica I Adventus

[Lectio1]
Incipit liber Isaíæ Prophétæ
!Isa 1:1-3
...
```

Inline markup used inside sections (parsed in `SetupString.pl`):
- `@File:Section` / `@:Section` — include another section (cross-reference).
- `$PrayerName` — pull in a named prayer/conclusion.
- `!Ref` — a scripture citation.
- `&function(args)` — call a Perl script function registered via the
  `ScriptFunc`/`ScriptShortFunc` attributes in `Scripting.pm`.
- `(rubrica ...)` — conditional rubric markers gating content by version.

When editing liturgical text, you are almost always editing these `.txt` data
files, not Perl. Match the existing section structure and markup exactly.

## Common commands

This is Perl with no package manager checked in; dependencies are listed in
`Build.pl` (CGI, Encode, DateTime, List::MoreUtils, Test::Cmd, Test::Carp, …).

Install dependencies:
```bash
cpanm --installdeps .
```

Run the app locally (Apache + Perl in Docker, code bind-mounted for live edits):
```bash
docker-compose up        # serves http://localhost:8080
```

Run a CGI script directly from the shell (how the regression harness drives it —
params are passed as `key=value` args, output is the HTTP response):
```bash
perl web/cgi-bin/horas/officium.pl \
  "version=Rubrics 1960" command=prayMatutinum date=01-01-2026
```

### Unit tests (`t/`) — correctness of the implementation
Run from the repository root (working directory matters):
```bash
prove -r -I web/cgi-bin/ t
```
Run a single test file:
```bash
prove -I web/cgi-bin/ t/DivinumOfficium/Main.t
```

### Regression tests (`regress/`) — diff generated offices between two commits
Compares rendered output of two git refs over a date range, for given versions.
Not pass/fail — it surfaces the textual diff so you can review the effect of a
change. CI runs `regress/scripts/travis.sh`; locally:
```bash
regress/scripts/generate-diff.sh <base_ref> <test_ref> <testspec_file> Divino 1960
```
A testspec line is a date range, e.g. `01-01-2026:12-31-2026`.

### Linting — perltidy (enforced in CI on `*.pl`/`*.pm` changes)
Config is `.perltidyrc` (2-space indent, 120 cols, cuddled blocks). The CI job
fails if any tracked Perl file is not tidy. Before committing Perl changes:
```bash
./perltidy.sh              # tidies every .pl/.pm in place
perltidy -st path/to/file.pl   # preview a single file to stdout
```

## Conventions
- Rubric *versions* (e.g. `Rubrics 1960`, `Divino Afflatu`, `Trident 1570`,
  Monastic, Cistercian, Dominican) are a cross-cutting concern: most logic and
  much data branches on the active version. When changing behavior, consider its
  effect across versions and check it with the regression harness.
- The `C` and `P` script prefixes mean Compare and Print variants of the same
  office/Mass — keep them in sync when changing shared rendering.
- `obsolete/` holds retired code; don't build on it. `standalone/` and `epub/`
  generate offline/eBook outputs (see `standalone/tools/epubgen2/README.md`).
- Production image is published to `ghcr.io/divinumofficium/divinum-officium`
  via `.github/workflows/build.yml` on push to `master`.
