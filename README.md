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
| Output file | `Faculty-Data/output/fst_faculty.csv` |

Breakdown by position: Lecturer 81, Assistant Professor 67, Associate Professor 26,
Professor 17, Senior Assistant Professor 3, Senior Lecturer 1, Senior Associate Professor 1.

Breakdown by room block: `DN` 98, `DS` 11, `DNGA` 9, `D` 5, `DNO` 3, `DE` 3.

---

## The interesting part: the data is not in the page you open

The URL above returns a page whose faculty cards are built **by JavaScript**. The served HTML
contains the menu, the footer and an empty Alpine.js template — **0 room numbers**. So the
project first has to *find* the real data source, and it does that with regex too:

```text
1. GET the faculty page          ->  HTML shell only (proved with Regex 1 and 2)
2. Regex 3 in that HTML          ->  /Content/employee-profile/script.js?v=23
3. GET the script, Regex 4       ->  /Files/Uploads/public-employee-profiles/employeeProfiles.json
4. GET the feed as raw text      ->  446 records in one long line
5. Regex 5-6                     ->  split into records, pull each field out
6. Regex 7                       ->  keep only Science & Technology
7. Regex 8-13                    ->  clean name / e-mail / room no, then write CSV
```

Nothing is hard-coded except the page URL you were given; the script path and the feed path
are read out of the site at run time.

---

## Regex inventory

| # | Pattern | Job |
|---|---|---|
| 1 | `[\w.+-]+@[\w-]+\.[\w-]+(?:\.[\w-]+)*` | Loose address-shaped scan, used to show the HTML has no faculty data |
| 2 | `room\s*(?:no\|number\|#)?\.?` | Counts the word "room" in the served HTML (result: 0) |
| 3 | `src="(?P<src>[^"]*employee-profile/script\.js[^"]*)"` | Finds the JavaScript file behind the page |
| 4 | `fetch\( *"(?P<url>/Files/Uploads/[^"]+\.json[^"]*)"` | Finds the data-feed URL inside that script |
| 5 | `(?=\{"IsShowProfileDetails")` | Zero-width lookahead split — cuts the feed into one chunk per employee |
| 6 | `"<Key>":"(?P<value>[^"]*)"` | Field extractor, reused for Name / Email / RoomNo / Faculty / Position / … |
| 7 | `SCIENCE\s*(?:&amp;\|&\|AND)\s*TECHNOLOGY` | Faculty filter that survives `&`, `AND`, `&amp;` and any case |
| 8 | `^(?P<title>[A-Za-z]+\.)\s+(?P<rest>.*)$` | Strips leading honorifics (`Prof.`, `Dr.`, `Mst.`) |
| 9 | `^[A-Za-z0-9._%+-]+@[A-Za-z0-9-]+(?:\.[A-Za-z0-9-]+)*\.[A-Za-z]{2,}$` | Anchored e-mail validator used by the sanity check |
| 10 | `(?P<block>[A-Za-z]{1,6})?[\s\-]*(?P<room>\d{1,6})(?:[\s\-]*(?P<wing>[A-Za-z]))?` | Splits a room code into block letters, number and wing letter |
| 11 | `\( *[^)]* *\)` | Captures bracketed notes such as `(OR)` or `(on Study Leave)` |
| 12 | `\b([a-z])` | Title-cases every word of an ALL-CAPS name |
| 13 | `(?<= )\b(Of\|And\|The\|For\|In\|On)\b` | Keeps connector words lowercase in labels |

Two ideas worth being able to explain in a viva: the **lookahead split** in Regex 5 (it marks
the cut position without eating the `{`, so every piece still starts with a complete object),
and **greedy vs lazy / character-class negation** in Regex 6 (`[^"]*` stops exactly at the
closing quote, which is what makes a one-line parser possible).

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

---

## Repository layout

```text
NLP/
├── README.md                              this file
├── .gitignore
└── Faculty-Data/
    ├── AIUB_Faculty.ipynb                 the project pipeline, 10 code cells, all executed
    ├── AIUB_Faculty_Regex_Scraper.ipynb   same pipeline with the full step-by-step write-up
    └── output/
        └── fst_faculty.csv                196 rows: name, email, room_no + extras
```

Both notebooks do exactly the same job; the second one keeps the explanation cells and the
numbered regex comments, so it is the one to read first.

CSV columns: `name`, `email`, `room_no` (exactly what the site stores), `room_parsed`
(normalised), plus `building`, `position`, `department` as free context.

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

The notebook writes `output/fst_faculty.csv`. It also saves `output/faculty_feed_raw.txt`,
a snapshot of the raw feed for inspection — that file is **gitignored**, because it mirrors
data for all 446 employees while this project only needs the 196 Science & Technology rows.

---

## Data and fair use

The names, e-mail addresses and room numbers are published by AIUB on its own public faculty
list, and this copy exists only as coursework for an NLP regex exercise — it is not redistributed
or used for anything beyond the assignment. The scraper downloads three public URLs (page,
script, feed) once per run and caches nothing but the two files in `output/`. If you fork this,
keep it to a couple of runs rather than hammering the site.

---

## Limitations (honest version)

* The parser assumes the feed keeps the `{"IsShowProfileDetails"` record prefix and today's key
  names. If AIUB changes the feed, Regex 5 and 6 need re-checking. This brittleness is the
  price of regex-only parsing versus a real JSON/HTML parser.
* A value containing an escaped quote (`\"`) would break `[^"]*` — none exist in the feed today.
* 11 selected records have no room number on the site; they are kept as blanks, not dropped.
* Codes such as `C` and `VUES` contain no digits, so they stay in `room_no` but cannot be
  normalised (`room_parsed` empty) — 183 of 196 parse.
* The data is collected live at run time, so re-running later may show a slightly different
  count if the university updates the list.

## Extension ideas

* All five faculties: `re.findall(r'href="(/faculty-list/[a-z0-9\-]+)"', html)` gives every menu
  link, then loop the same pipeline — the feed already contains all of them in one file.
* Extra fields (`Designation`, `AcademicInterests`, `ResearchInterests`) need no new code:
  `field(record, "ResearchInterests")` is enough, and those comma-separated interest strings
  are a natural next step for tokenising, frequency lists or a tiny search index.
* Compare `room_parsed` against `BuildingNo` to detect inconsistent room numbering between blocks.

## Note on the data

Names, e-mail addresses and room numbers here are published by AIUB on its own public faculty
directory. The project collects that public page for coursework, stores it locally, and does not
log in, bypass any restriction, or hammer the server — three HTTP requests per run.
