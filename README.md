# NLP — Regex-Only AIUB Faculty Data Collector

A small NLP coursework project: it collects **faculty name, e-mail and room number** for the
*Faculty of Science & Technology* from the AIUB website and writes them to CSV — using
**regular expressions only**, with no BeautifulSoup, no `lxml`, no `json` module and no `pandas`.

Source page: <https://www.aiub.edu/faculty-list/faculty-of-science--technology?faculty=FACULTY+OF+SCIENCE+%26+TECHNOLOGY>

---

## What it collects

Numbers are from the newest run saved in this repo. The site is live, so they move.

| Metric | Value |
|---|---|
| Employee records found in the site feed | 449 |
| Records for Faculty of Science & Technology | **197** |
| Names collected | 197 / 197 |
| E-mails collected and passing the strict pattern | 197 / 197 (0 duplicates) |
| Room numbers present | 186 (11 blank in the source itself) |
| Room codes normalised into block + number + wing | 183 |
| Profile links built and matching the site's own pattern | 197 / 197 |
| Records carrying research-interest text | 186 / 197 |
| Distinct research terms after stop-wording | 166 (162 stems after Step 18) |
| Stems in the inverted index | 641 over 197 documents; 442 used by exactly one person |
| E-mail ids explained by a name rule | 141 of 197 (72%) |
| Records missing a room number **and** interest text | 32 of 449 (11 in this faculty) — the same people, one empty profile block causes both |
| Departments inside this faculty | 4 (Computer Science 141, Mathematics 27, Physics 21, Chemistry 8) |

Breakdown by position: Lecturer 81, Assistant Professor 67, Associate Professor 27,
Professor 17, Senior Assistant Professor 3, Senior Lecturer 1, Senior Associate Professor 1.

Breakdown by room block: `DN` 96, `DS` 11, `DNGA` 9, `D` 5, `DNO` 3, `DE` 3.

---

## The interesting part: the data is not in the page you open

The URL above returns a page whose faculty cards are built **by JavaScript**. The served HTML
contains the menu, the footer and an empty Alpine.js template — **0 room numbers**. So the
project first has to *find* the real data source, and it does that with regex too:

```text
1.  GET the faculty page          ->  HTML shell only (proved with Regex 1 and 2)
2.  Regex 3 in that HTML          ->  /Content/employee-profile/script.js?v=23
3.  GET the script, Regex 4       ->  /Files/Uploads/public-employee-profiles/employeeProfiles.json
4.  GET the feed as raw text      ->  449 records in one long line
5.  Regex 5-6                     ->  split into records, pull each field out
6.  Regex 7                       ->  keep only Science & Technology
7.  Regex 8-30                    ->  clean, validate, query, stem, index, write 6 CSVs
```

Nothing is hard-coded except the page URL you were given; the script path and the feed path
are read out of the site at run time.

---

## Beyond the three required fields

| Feature | Notebook step | What it gives you |
|---|---|---|
| `collect(faculty)` | 10 | The same pipeline for all five faculties, cross-checked against the menu links, plus a raw-vs-parsed disagreement count (0) |
| `faculty_summary.csv` | 10 | Records, room coverage and interest coverage per faculty — aggregates only, no extra names |
| `find(pattern, fields)` | 11 | A regex lookup over name, e-mail, department, room, position and interests; tries `machine learning`, `^DN`, `physics`, `study leave` |
| `profile_url` column | 6, 8 | The `faculty-profile?q=<user-id>` link the site itself builds, validated by Regex 16 |
| research-term counts | 12 | Tokenising, stop-words, mentions vs distinct-faculty counts, `research_terms.csv` |
| word-vs-phrase honesty check | 12 | Shows that the bare word `machine` appears in 21 records but *machine learning* in only 1 — the rest say *Machine to Machine* |
| room and department map | 13 | Normalised building + room keys, 14 keys holding more than one person, department sizes |
| `ask(query)` | 14 | A small query language read by regex: `dept:physics room:^DN`, `interest:"machine to machine"`, quoted values, conditions combined |
| research phrases | 15 | Two- and three-word topics instead of single words — `deep learning` 76, `machine to machine` 19, `machine learning` 1 |
| name-to-e-mail rules | 16 | Every address tested against patterns built from that person's own name words: 140 of 196 (71%) explained |
| change tracking | 8, 17 | The previous table is kept before it is overwritten, then joiners, leavers and moved rooms are reported, saved, and archived with a date so a later run cannot erase the finding |
| stemming | 18 | Ordered `re.sub` suffix rules with guards: `learning`/`learned` -> `learn`, `networks`/`networking` -> `network`, while `business`, `analysis` and `using` survive untouched |
| inverted index + boolean search | 19 | 641 stemmed terms indexed over every field; `neural AND vision`, `"deep learning" AND mathematics`, `blockchain OR nanotechnology`, `"internet of things" NOT smart` |

---

## Regex inventory

| # | Pattern | Job |
|---|---|---|
| 1 | `[\w.+-]+@[\w-]+\.[\w-]+(?:\.[\w-]+)*` | Loose address-shaped scan, used to show the HTML has no faculty data |
| 2 | `room\s*(?:no\|number\|#)?\.?` | Counts the word "room" in the served HTML (result: 0) |
| 3 | `src="(?P<src>[^"]*employee-profile/script\.js[^"]*)"` | Finds the JavaScript file behind the page |
| 4 | `fetch\( *"(?P<url>/Files/Uploads/[^"]+\.json[^"]*)"` | Finds the data-feed URL inside that script |
| 5 | `(?=\{"IsShowProfileDetails")` | Zero-width lookahead split — cuts the feed into one chunk per employee |
| 6 | `"<Key>":"(?P<value>[^"]*)"` | Field extractor, reused for Name / Email / RoomNo / Faculty / Position / interests |
| 7 | `SCIENCE\s*(?:&amp;\|&\|AND)\s*TECHNOLOGY` | Faculty filter that survives `&`, `AND`, `&amp;` and any case |
| 8 | `^(?P<title>[A-Za-z]+\.)\s+(?P<rest>.*)$` | Strips leading honorifics (`Prof.`, `Dr.`, `Mst.`) |
| 9 | `^[A-Za-z0-9._%+-]+@[A-Za-z0-9-]+(?:\.[A-Za-z0-9-]+)*\.[A-Za-z]{2,}$` | Anchored e-mail validator used by the sanity check |
| 10 | `(?P<block>[A-Za-z]{1,6})?[\s\-]*(?P<room>\d{1,6})(?:[\s\-]*(?P<wing>[A-Za-z]))?` | Splits a room code into block letters, number and wing letter |
| 11 | `\( *[^)]* *\)` | Captures bracketed notes such as `(OR)` or `(on Study Leave)` |
| 12 | `\b([a-z])` | Title-cases every word of an ALL-CAPS name |
| 13 | `(?<= )\b(Of\|And\|The\|For\|In\|On)\b` | Keeps connector words lowercase in labels |
| 14 | `(?P<user>[^@]+)@` | The user-id that the site itself puts into `faculty-profile?q=` |
| 15 | `[,;]+` | Splits the interest blob into separate phrases |
| 16 | `^https://www\.aiub\.edu/faculty-list/faculty-profile\?q=[a-z0-9._-]+$` | Validates every generated profile link |
| 17 | `"Faculty":"([^"]*)"` | Re-reads the faculty value straight from the raw record, as a check on Regex 6 |
| 18 | `href="(/faculty-list/[a-z0-9\-]+)"` | The faculty pages linked from the menu |
| 19 | compiled per call inside `find()` | The search pattern of the lookup tool |
| 20 | `[A-Za-z][A-Za-z0-9+#.\-/]{2,}` | Research terms, keeping symbols that matter (`C++`, `.NET`, `e-commerce`) |
| 21 | `[^A-Z0-9]` | Strips punctuation to build a comparable building + room key |
| 22 | `(?:(?P<field>[a-z]+)\s*:\s*)?(?P<value>"[^"]*"\|'[^']*'\|[^\s,]+)` | Reads a `field:value` query, quoted or bare |
| 23 | `^["']\|["']$` | Removes the quotes around a multi-word query value |
| 24 | `[^a-z0-9+#\- ]+` | Normalises interest text into words and single spaces |
| 25 | `\b[a-z][a-z+#\-]{2,} [a-z][a-z+#\-]* [a-z][a-z+#\-]{2,}\b` | Three-word phrases, allowing a short connector in the middle (`internet of things`) |
| 26 | `\b[a-z][a-z+#\-]{2,} [a-z][a-z+#\-]{2,}\b` | Two-word phrases, applied after trigrams are removed |
| 27 | `[A-Za-z][A-Za-z'\-]*` | The words of a person's name, honourifics and credentials dropped |
| 28 | `^{given}\.{family}$` and friends | Rule templates for how AIUB turns a name into an e-mail id |
| 29 | `(\w{2,}[^aeiou])ies$` and the rest of the suffix list | Stemming: `studies` -> `study`, `applied` -> `apply`, never `business` |
| 30 | `"[^"]+"\|\bAND\b\|\bOR\b\|\bNOT\b\|\b[a-z][a-z0-9+#.\-]{1,}\b` | Reads a boolean query: quoted phrase, operator, or single word |

Two ideas worth being able to explain in a viva: the **lookahead split** in Regex 5 (it marks the
cut position without eating the `{`, so every piece still starts with a complete object), and
**character-class negation** in Regex 6 (`[^"]*` stops exactly at the closing quote, which is what
makes a one-line parser possible).

---

## How the raw text is cleaned

| Raw value from the site | Cleaned result |
|---|---|
| `MD. ANWARUL KABIR` | `Md. Anwarul Kabir` (`Md.` kept — it belongs to the given name) |
| `PROF. DR. KH. ABDUL MALEQUE` | `Prof. Dr. Kh. Abdul Maleque` |
| `DR. MD. ABDULLAH - AL - JUBAIR` | `Dr. Md. Abdullah - Al - Jubair` |
| `DNO -216` | block `DNO` + room `216` → `DNO216` |
| `9210K` | room `9210` + wing `K` → `9210K` |
| `1122 (OR)` | room `1122`, note `(OR)` kept separate — nothing is invented |
| `DEPARTMENT OF COMPUTER SCIENCE` | `Department of Computer Science` |
| `" Materials and Processing,Polymer Chemistry"` | `Materials and Processing, Polymer Chemistry` |
| `kabir@aiub.edu` | `https://www.aiub.edu/faculty-list/faculty-profile?q=kabir` |

---

## Repository layout

```text
NLP/
├── README.md                              this file
├── .gitignore
└── Faculty-Data/
    ├── Simple_Version.ipynb               shortest version: `re` + `urllib` only, 7 regexes
    ├── AIUB_Faculty.ipynb                 the project pipeline, 10 code cells, all executed
    ├── AIUB_Faculty_Regex_Scraper.ipynb   full version: 19 steps, 21 code cells, 30 regexes
    └── output/
        ├── fst_faculty.csv                197 detail rows, 9 columns
        ├── faculty_summary.csv            one line per faculty, counts only
        ├── research_terms.csv             single-word research terms
        ├── research_phrases.csv           two- and three-word phrases
        ├── index_terms.csv                every indexed stem, its document count, its surface forms
        ├── changes.csv                    what the newest run detected
        ├── history/changes_<date>.csv     dated archive of every non-empty detection
        └── simple_faculty.csv             197 rows from the simple version, raw values
```

Read `Simple_Version.ipynb` first for the shortest path to a working collector, then
`AIUB_Faculty_Regex_Scraper.ipynb` for what each regex does, the extra features and the
normalisation rules. The simple version keeps raw ALL-CAPS names, unparsed room codes and a
hand-written feed URL, and imports only `re` and `urllib.request` — its CSV is written with
builtin `open()`, quoting a field by regex only when it contains a comma, quote or newline.

`fst_faculty.csv` columns: `name`, `email`, `room_no` (exactly what the site stores),
`room_parsed` (normalised), `building`, `position`, `department`, `interests`, `profile_url`.

---

## How to run

Only the standard library is needed, so any Python 3.8+ works — including the JNBook venv or a
bare system Python. Install nothing.

```bash
# inside JupyterLab / VS Code: open the notebook and Run All  (about 15 s, it re-downloads)
# or from the command line:
cd Faculty-Data
jupyter nbconvert --to notebook --execute --inplace AIUB_Faculty_Regex_Scraper.ipynb
```

The full notebook writes the six CSVs above plus a dated change archive, and also saves `output/faculty_feed_raw.txt`, a
snapshot of the raw feed for inspection. That snapshot is **gitignored**, because it mirrors data
for all 449 employees while this project only needs the 197 Science & Technology rows.

`changes.csv` compares this run with the `fst_faculty.csv` that was already on disk, so on a fresh
clone the first run diffs against the committed version and reports no changes. Delete
`fst_faculty.csv` and run again to see the "first run" branch.

---

## Data and fair use

The names, e-mail addresses and room numbers are published by AIUB on its own public faculty
directory, and this copy exists only as coursework for an NLP regex exercise — it is not
redistributed or used for anything beyond the assignment. Each run makes three requests to public
URLs (page, script, feed), logs in nowhere and bypasses nothing. If you fork this, keep it to a
couple of runs rather than hammering the site, and drop the CSVs if your institution asks you to.

---

## What a second run caught: the roster is not static

Change tracking was added on 12 October. The next run found real edits on the site and saved them
to `output/history/changes_2026-10-19.csv` (that run clocked past midnight, so the filename is the
19th while the work belongs to the 18th):

| Change | Who | Detail |
|---|---|---|
| joined | Dr. Muhammad Abul Hasan | `muhammad.hasan@aiub.edu` |
| moved room | Dipta Justin Gomes | `9408G` -> `RGEO`, Annex 9 -> Annex 6 |
| moved room | Dr. Mohammad Rabiul Islam | `DN0217` -> `9408K`, D Building -> Annex 9 |
| moved room | Tonny Shekha Kar | `DN 0720` -> `9408J`, D Building -> Annex 9 |

That took the faculty from 196 to 197 records and moved three people off the `DN` block, which is
why the room-block and position counts here differ from the 12 October run. Rooms shared by more
than one person went from 14 to 12.

`changes.csv` always describes the run that wrote it, so it is empty on a second run of the same
day. The dated copy under `output/history/` is what preserves the finding.

---

## Limitations (honest version)

* The parser assumes the feed keeps the `{"IsShowProfileDetails"` record prefix and today's key
  names. If AIUB changes the feed, Regex 5 and 6 need re-checking. This brittleness is the price
  of regex-only parsing versus a real JSON/HTML parser.
* A value containing an escaped quote (`\"`) would break `[^"]*` — none exist in the feed today.
* 11 records have no room number on the site, and codes such as `C` and `VUES` carry no digits so
  they cannot be normalised — 183 of 196 parse. Blanks stay blank instead of being guessed. Those
  11 are also exactly the 11 without interest text: their `PersonalOtherInfo` block comes back
  empty, so Step 10 checks the two sets match instead of trusting what looks like a coincidence.
* Single-word counts mislead on their own: `machine` mostly comes from *Machine to Machine*, so
  Step 12 prints the word count and the phrase count side by side, and Step 15 counts real
  phrases. Phrases are matched greedily — `natural language processing` is counted once, as the
  trigram, not again as two bigrams.
* The name-to-e-mail rules in Step 16 explain 140 of 196 addresses (71%). The rest are nicknames
  (`danny`), numbers (`habib583`), initial clusters (`dmri`, `agmzaman`) or legacy ids — printed,
  not hidden, because a regex rule set that claims 100% is usually lying.
* The stemmer in Step 18 is single-pass and shallow. Its rules carry guards because the first
  version produced `appy` from *applied* and `studie` from *studies*: the length floor made the
  matched group too long for short words, so they fell through to the plural rule. `computation`
  still becomes `computat`, which a real Porter stemmer would map properly.
* The boolean search is left-to-right with no parentheses, so `a OR b AND c` means
  `(a OR b) AND c`, and `NOT` subtracts from whatever came before it.
* Step 17 only diffs the Faculty of Science & Technology table, since that is the snapshot kept on
  disk. The other four faculties are counted in `faculty_summary.csv`, not compared row by row.
* Room sharing is judged on `BuildingNo` + code, so two people in one physical room can still look
  different when the site spells the building differently (`D - Building` vs `D BUILDING - ANNEX`).
* Profile links reuse the site's own `?q=<user-id>` convention; verified live for a lower-case and
  a mixed-case user-id, but it depends on that convention continuing.
* Data is collected live at run time, so re-running later may show slightly different counts.

## Extension ideas

* Learn the addresses Step 16 cannot explain: nicknames (`danny` for Danilo), initial clusters
  (`dmri`, `agmzaman`) and legacy ids would each need a small alias table fed back as one more rule.
* Wrap `ask()` as a small command-line lookup, or export the room map as a floor list per building.
* Rank the search results with tf-idf instead of returning a bare set, and swap the hand-written
  suffix rules for a full Porter stemmer to compare the two vocabularies.
* Step 10 already builds rows for all five faculties; widening the detail CSV to the whole
  university is a one-line change, and deliberately not done here.
