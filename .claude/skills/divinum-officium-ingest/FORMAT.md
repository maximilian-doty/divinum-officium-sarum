# Divinum Officium data format reference

Companion to SKILL.md. All data files are **UTF-8**, organised into sections.

## File anatomy

```
[SectionName]
content line
more content
                       ← blank line allowed inside a section
[NextSection]
...
```

A line `[Name]` starts a section; everything until the next `[…]` is its body.
A leading `#…` line before the first `[…]` is a whole-file `__preamble` (used for
`@File` whole-file inclusions and `(rubrica)` lines that apply file-wide).

## Folders & file names

Office root `web/www/horas/<Language>/`, Mass root `web/www/missa/<Language>/`.
Within each:

| Folder | Holds | File naming |
|--------|-------|-------------|
| `Sancti` | Sanctoral (saints' days) | `MM-DD.txt` (e.g. `07-16.txt`); variants `MM-DDt`, `MM-DDcc`, `MM-DDo` |
| `Tempora` | Temporal/seasonal | season codes: `Adv1-0`, `Nat1`, `Epi1-0`, `Quad3-2`, `Pasc5-0`, `Pent03-0` (`<season><week>-<weekday 0=Sun>`) |
| `Commune` | Commons | `C1`…`C12` with subvariants (`C2-1`, `C4b`, `C5a`, `C9`…) |
| `Psalterium` | Psalter (psalms, ferial antiphons, ordinary psalmody) | by hour/day |
| `Ordinarium` | Fixed text of each hour (only at `horas/Ordinarium/`, not per-language-suffixed) | `Matutinum.txt`, `Laudes.txt`, `Minor.txt`, `Prima.txt`, `Vespera.txt`, `Completorium.txt` |

**Use suffix.** A non-Roman Use stores overrides in suffixed folders:
`Sancti<Suffix>/`, `Tempora<Suffix>/`, `Commune<Suffix>/` — `M` (Monastic),
`OP` (Dominican), `Cist` (Cistercian), `Sar` (Sarum). Same filenames as the
Roman base.

**Calendar:** `web/www/Tabulae/Kalendaria/<ver>.txt`; transfer rules in
`web/www/Tabulae/Transfer/<ver>.txt` and `…/Stransfer/<ver>.txt`.

## Fallback (why you only encode deltas)

`subdirname()` (horascommon.pl) rewrites `Sancti`→`Sancti<Suffix>` per version.
If that file is absent, `checklatinfile()` (SetupString.pl) walks a hierarchy:
`Cist → M → Roman`, and `M | OP | Sar → Roman`. So a missing `SanctiSar/07-16.txt`
falls back to `Sancti/07-16.txt`. **Create a Use file only when it differs.**
Within a file, missing sections are filled from any `vide`/`ex`/`@` reference and
from the language layer below.

## Sections

### Office (a feast/day proper rarely needs all of these)
`[Officium]` (title) · `[Rank]` · `[Rule]` · `[Invit]` (invitatory) ·
`[Hymnus Matutinum|Laudes|Vespera]` and pre-Urban variants `[HymnusM …]`,
`[Hymnus1 …]` · `[Ant Matutinum]` · `[Nocturn 2 Versum]`, `[Nocturn 3 Versum]` ·
`[Lectio1]`…`[Lectio9]` (and `[Lectio94]` = short-form variant) ·
`[Responsory1]`…`[Responsory9]` · `[Ant Laudes]` · `[Capitulum Laudes]` ·
`[Ant Vespera]` · `[Versum 1]`, `[Versum 2]`… · `[Ant 1]`, `[Ant 2]`… (psalm
antiphons) · `[Oratio]` (collect).

### Mass
`[Officium]` · `[Rank]` · `[Rule]` · `[Introitus]` · `[Oratio]` (collect) ·
`[Lectio]` (epistle/OT) · `[Graduale]` · `[Tractus]` / `[Alleluja]` ·
`[Sequentia]` · `[Evangelium]` · `[Offertorium]` · `[Secreta]` · `[Communio]` ·
`[Postcommunio]` · `[Super populum]` (Lenten ferias) · commemorations:
`[Commemoratio Oratio|Secreta|Postcommunio]`; extra lessons `[LectioL1]`,
`[OratioL1]`, `[GradualeL1]`… for ember/multi-lesson Masses.

## The header: [Officium] / [Rank] / [Rule]

```
[Officium]
S. Osmundi Episcopi et Confessoris

[Rank]
;;Duplex;;3;;vide C5-1

[Rule]
vide C5-1;mtv
9 lectiones
```

- **[Rank]** = `<name>;;<RankName>;;<num>;;<commune-ref>`. Leave `<name>` empty
  (the engine fills it from `[Officium]`). `<commune-ref>` like `ex C2a`,
  `vide C5-1`, or omitted.
- **[Rule]** lines: `<commune-ref>;` flags, then options such as `9 lectiones` /
  `12 lectiones` / `3 lectiones`, `Gloria` (Mass), `Antiphonas horas`,
  `Doxology=Heart`, `mtv`, `Symbolum Athanasium`, `Psalmi Dominica`.

### Rank scale (RankName → typical num)
| RankName | num |
|----------|-----|
| Commemoratio / Memoria | 1.1 |
| Simplex | 1 – 1.3 |
| Semiduplex | 2 – 2.2 |
| Duplex | 3 |
| Duplex majus | 4 – 4.1 |
| Duplex II classis | 5 – 5.4 |
| Duplex I classis | 6 |
| (highest feasts, e.g. Easter) | 6 + |

Use the printed grade in the source to choose the name; pick the number from this
scale (note it as an assumption in your report — exact tenths depend on
sub-rank/octave nuances).

## Markup (inside section bodies)

| Syntax | Meaning |
|--------|---------|
| `@File:Section` | Pull a section from another file, e.g. `@Commune/C5:Oratio`, `@Tempora/Pent02-5:Versum 1`. `@:Section` = same file. A bare `@File` in the preamble includes the whole file. `@File:Sec:subst` adds substitutions. |
| `$Name` | Named prayer/ending/conclusion. Common: `$Per Dominum`, `$Per eundem`, `$Qui vivis`, `$Qui tecum`, `$Pater noster`, `$Ave Maria`, `$Fidelium animae`, `$Deus in adjutorium`. Replace printed full endings with these. |
| `!Book ch:vv` | Scripture citation header for a lesson/reading, e.g. `!Isa 1:1-3`, `!Léctio Epístolæ…` may precede it as a black title line. |
| `&function` | Engine script call: `&teDeum`, `&Dominus_vobiscum`, `&Benedicamus_Domino`, `&Divinum_auxilium`, `&psalm(118)`. |
| `*` | Mediation/flex point in psalm verses and antiphons (mark only as the source points). |
| `†` `‡` | Cross marks. `r.` / `v.` line prefixes = red-letter / initial-capital first letter. `R.` `V.` `Ant.` `R.br.` = responsory/versicle/antiphon labels (rendered red). |
| `/:text:/` | Small-red inline rubric; `«…»` inside = black-within-red. |

## Version conditionals — `(rubrica …)`

A line `(rubrica X)` after content makes that content conditional; forms like
`(sed rubrica X omittitur)` / `(deinde rubrica X dicitur)` add/remove with scope.
`X` is either a **named predicate** or a **regex matched against the version
name**:

- Named: `tridentina`, `monastica`, `innovata`, `cisterciensis`, `altovadensis`,
  `paschali`, `feriali`, `summorum pontificum` (defined in SetupString.pl
  `%predicates`). Add a new named predicate there if a Use needs one.
- Regex: `rubrica 1570`, `rubrica 196`, `rubrica ^Trident`, `rubrica Sarum` — the
  text after `rubrica` is `=~ /…/i` against `$version`.

Combine with `aut` (or) / `et` (and) / `nisi` (and-not), e.g.
`(sed rubrica 196 aut rubrica 1955 aut rubrica ^Trident omittitur)`.

To make content **Use-specific**, gate it with that Use's version name, e.g.
`(rubrica Sarum)` for Sarum-only, `(rubrica Praedicatorum)` for Dominican-only.

## Calendar (`Kalendaria/<ver>.txt`)

Month headers `*January*`, then one line per day:

```
DATE=KEY=NAME=RANK[=NAME2=RANK2[=NAME3=RANK3…]]
```

- `DATE` = `MM-DD`.
- `KEY` = the `Sancti/` file stem(s); `~` separates the occurring office from a
  commemoration's file, and variant suffixes appear here, e.g.
  `01-14~01-14cc` (main office `01-14`, commemoration file `01-14cc`).
- `NAME`/`RANK` pairs: the displayed title and its numeric rank (same scale as
  above) for the office and each commemoration.

Examples:
```
01-01=01-01=In Circumcisione Domini=3=
01-14=01-14~01-14cc=S. Hilarii Episcopi Confessoris Ecclesiae Doctoris=3=S. Felicis Presbyteri et Mart=1=
```

## Languages

`Latin/` is the base text layer. A vernacular folder (`English/`, `Magyar/`…)
holds only **translations of the same sections**, layered on top of Latin
(missing sections fall through to Latin, then to the fallback language
`$langfb`). Encode Latin first; add a vernacular file with the matching
`[Section]`s when you have the translation. `Latin-gabc/` carries chant (gabc)
versions of the same sections.

## Registering a new Use (one-time, before its first file)

1. `web/www/Tabulae/data.txt`: add `Version,kalendar,transfer,stransfer,base[,tbase]`.
2. `web/www/horas/horas.dialog` + `web/www/missa/missa.dialog`: add to `[versions]`.
3. `horascommon.pl` `subdirname()`: add `return "${subdir}<Suffix>/" if $version =~ /<Name>/;`.
4. `SetupString.pl` `checklatinfile()`: add `<Suffix>` to the `→ Roman` fallback rewrite.
5. Optional: a named predicate in `SetupString.pl` `%predicates`; legacy-name
   mapping in `RunTimeOptions.pm`.

(For Sarum this is already done — it currently clones Tridentine - 1570 until
real Sarum content is added under the `Sar/` folders and a `Kalendaria/Sarum.txt`.)
