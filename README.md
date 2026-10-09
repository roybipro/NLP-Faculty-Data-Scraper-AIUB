# NLP — Regex-Only AIUB Faculty Data Collector

A small NLP coursework project: it collects **faculty name, e-mail and room number** for the
*Faculty of Science & Technology* from the AIUB website and writes them to CSV — using
**regular expressions only**, with no BeautifulSoup, no `lxml`, no `json` module and no `pandas`.

Source page: <https://www.aiub.edu/faculty-list/faculty-of-science--technology?faculty=FACULTY+OF+SCIENCE+%26+TECHNOLOGY>

---

## What it collects

| Metric | Value |
|---|---|
| Employee records found in the site feed | 446 |
| Records for Faculty of Science & Technology | **196** |
| Names collected | 196 / 196 |
| E-mails collected and passing the strict pattern | 196 / 196 (0 duplicates) |
| Room numbers present | 185 (11 blank in the source itself) |
| Room codes normalised into block + number + wing | 183 |
| Profile links built and matching the site's own pattern | 196 / 196 |
| Records carrying research-interest text | 185 / 196 |
| Distinct research terms after stop-wording | 166 |
| Records missing a room number **and** interest text | 38 of 446 (11 in this faculty) — the same people, one empty profile block causes both |
| Departments inside this faculty | 4 (Computer Science 140, Mathematics 27, Physics 21, Chemistry 8) |

Breakdown by position: Lecturer 81, Assistant Professor 67, Associate Professor 26,
Professor 17, Senior Assistant Professor 3, Senior Lecturer 1, Senior Associate Professor 1.

Breakdown by room block: `DN` 98, `DS` 11, `DNGA` 9, `D` 5, `DNO` 3, `DE` 3.

---

## The interesting part: the data is not in the page you open

The URL above returns a page whose faculty cards are built **by JavaScript**. The served HTML
contains the menu, the footer and an empty Alpine.js template — **0 room numbers**. So the
project first has to *find* the real data source, and it does that with regex too:

```text
1.  GET the faculty page          ->  HTML shell only (proved with Regex 1 and 2)
2.  Regex 3 in that HTML          ->  /Content/employee-profile/script.js?v=23
3.  GET the script, Regex 4       ->  /Files/Uploads/public-employee-profiles/employeeProfiles.json
4.  GET the feed as raw text      ->  446 records in one long line
5.  Regex 5-6                     ->  split into records, pull each field out
6.  Regex 7                       ->  keep only Science & Technology
7.  Regex 8-21                    ->  clean, validate, link, analyse, then write 3 CSVs
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
    ├── AIUB_Faculty_Regex_Scraper.ipynb   full version: 13 steps, 15 code cells, 21 regexes
    └── output/
        ├── fst_faculty.csv                196 detail rows, 9 columns
        ├── faculty_summary.csv            one line per faculty, counts only
        ├── research_terms.csv             top 60 research terms with mention counts
        └── simple_faculty.csv             196 rows from the simple version, raw values
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

The full notebook writes the three CSVs above and also saves `output/faculty_feed_raw.txt`, a
snapshot of the raw feed for inspection. That snapshot is **gitignored**, because it mirrors data
for all 446 employees while this project only needs the 196 Science & Technology rows.

---

## Data and fair use

The names, e-mail addresses and room numbers are published by AIUB on its own public faculty
directory, and this copy exists only as coursework for an NLP regex exercise — it is not
redistributed or used for anything beyond the assignment. Each run makes three requests to public
URLs (page, script, feed), logs in nowhere and bypasses nothing. If you fork this, keep it to a
couple of runs rather than hammering the site, and drop the CSVs if your institution asks you to.

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
* Unigram term counts mislead on their own: `machine` mostly comes from *Machine to Machine*, so
  Step 12 prints the word count and the phrase count side by side.
* Room sharing is judged on `BuildingNo` + code, so two people in one physical room can still look
  different when the site spells the building differently (`D - Building` vs `D BUILDING - ANNEX`).
* Profile links reuse the site's own `?q=<user-id>` convention; verified live for a lower-case and
  a mixed-case user-id, but it depends on that convention continuing.
* Data is collected live at run time, so re-running later may show slightly different counts.

## Extension ideas

* Bigram counting — `re.findall(r"\b[a-z]+ [a-z]+\b", interests.lower())` — then drop two-word
  stop-phrase pairs, which fixes the `machine` / `machine learning` confusion properly.
* Wrap `find()` as a small command-line lookup, or export the room map as a floor list per building.
* Snapshot the feed weekly and regex-diff the name list to catch joiners and leavers.
