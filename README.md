# Catalogue Discover

A natural-language search front end for a [Koha](https://koha-community.org/) library catalogue. Readers search the way they would ask a librarian, and the app runs beside Koha's own OPAC without replacing it.

Built for IIT Gandhinagar Library (Koha 25.05.05, about 42,000 titles), and works with any Koha 22.11 or later.

## What readers get

**New in version 5: intelligent search**

- **"By" and "on" mean different things.** *Books by Gandhi* searches authors only. When several authors share the name it lists them: *Which one did you mean? M. K. Gandhi (120) · Rajmohan Gandhi (8)*. *Books on Gandhi* (also *about*, *life of*, *biography of*) searches what books are about (subject headings, title, summary), not who wrote them. A bare *gandhi* offers both readings.
- **Meaning search.** Each title gets a meaning fingerprint from its title, subjects and notes, made on the library's own computer by a small open model (sentence-transformers, 50+ languages including Hindi). Searches find titles that are close in meaning even with no shared words, blended with the word matches. Titles found this way say *Close in meaning to your search*. Without the model the app uses latent semantic analysis (scikit-learn).
- **Explore themes.** The collection is grouped by meaning into about 150 themes across Dewey classes (topic modelling: k-means on the fingerprints, named by class-based TF-IDF of the subject headings). They appear on the home page and on each title page.
- **Course readings.** Students can type *readings for CE 404*, *transportation engineering course* or *Prof. Sushobhan Sen's readings* and get the course list with Textbook/Reference, Course Reserve (not for issue) and Short Loan (3-day issue) copies, and live shelf status. A *Course readings* page has faculty, course and code menus and Export to CSV. Staff upload each semester's CSV on the admin page.
- **Ask the librarian** (needs an Anthropic API key). A chat where readers describe what they need in their own words. The app searches the catalogue, and Claude picks 3 to 8 of the titles found and says why each fits. It cannot recommend a book the library doesn't have. Only the question and catalogue records are sent, never who is asking.

- **Search in plain words:** "physical chemistry books after 2010", "hindi novels", "books by feynman", "machine learning available now". The page shows how it understood the search, and each part can be removed with one click.
- **Spelling correction**, Google style: "quantm mechancs" shows results for "quantum mechanics".
- **Search by cover:** describe it ("black cover with a gold bird"), take a photo of the cover or barcode, or type the ISBN.
- **Home page** with *New arrivals* (newest copies by accession date), *Picked for you* (for a signed-in reader), and *Browse by subject*.
- **Clickable authors, publishers and subjects:** every name and subject heading on a title page opens all the library's titles for it. The left-hand filters list them too.
- **Subject map from Dewey numbers** (semantic search): each book's class number (082, or the call number) places it in a tree of 10 classes, 100 divisions and their sections. A search also brings in books shelved in the most closely related sections, even when the words don't appear in their record, and says which sections it used.
- **Picked for you:** suggestions from the signed-in reader's own borrowing (their most-read Dewey sections, authors and subjects), with the reason shown. Only that reader sees them.
- **Real cover images** by ISBN (Amazon, Open Library, or covers uploaded to Koha), with a drawn cover when none exists.
- **On each title:**
  - a plain-language explanation built from the catalogue notes (5XX fields)
  - call number, location, and live availability
  - ratings and review links by ISBN (Goodreads, Amazon, Google Books, Open Library)
  - reviews written by library members
  - suggestions: same series, same author, "readers who borrowed this also borrowed", similar summary, same genre
- **Library account:** sign in with a library card to see loans (and renew them), holds (place and cancel), fines, and reading history.
- **Share:** every search and every title has its own link (*Copy link*), so a reader or a teacher can send it to others.
- **Export:** download search results as Excel or CSV, or as RIS (Zotero, Mendeley, EndNote) or BibTeX (LaTeX). Each title page shows an APA citation with a copy button.
- **Reading lists:** anyone can make a list (*Add to list* on a title page) and share it by link. Only the browser that made the list can change it.
- **Saved searches:** a signed-in reader can save a search. The home page shows how many new titles match since they last looked, and the library can email them when new titles arrive.

## What staff get

- **Reports** at `/admin`, for displays and collection development. Each one covers the last 7, 30 or 90 days or a year, and can be saved under a name, downloaded as Excel or CSV, printed or saved as PDF, and shared with colleagues as a link (they need the staff token to open it).

  | Report | What it tells you |
  |---|---|
  | Top searches | What readers look for most: display themes |
  | Searches with no results | Demand the collection does not meet: purchase ideas, or records that need better subjects |
  | Demand by subject | For each Dewey division: titles, copies, on loan now, share on loan, title views and searches |
  | Display ideas | Titles on the shelf now from the subject areas in highest demand |
  | New arrivals by subject | Where recent purchases went, by accession date |
  | Collection age by subject | Median year and share older than 20 years, oldest areas first: weeding and renewal |

  **Circulation (version 4), for any two dates:**

  | Report | What it tells you |
  |---|---|
  | Circulation summary | Checkouts, renewals, returns, different titles and borrowers, by month; share of the collection borrowed |
  | Top 20 most-used titles | Checkouts plus renewals, copies and checkouts per copy |
  | 20 least-used titles | Titles held before the period that were borrowed least, longest unused first |
  | Well-used and under-used subjects | Use factor by Dewey division: share of checkouts ÷ share of collection (above 1 well used, below 0.5 under-used) |
  | Use by main class | The same by Dewey main class |
  | Most active borrowers | Top 20 borrowers by checkouts, with category and department (names, staff only) |
  | Borrowing by patron category / by department | Registered, active, % active, checkouts per active borrower |
  | Fines by year | Charged, collected, waived and outstanding, from a public Koha SQL report |

  **Version 5:** *Themes readers search for* places each logged search in its nearest theme, next to the theme's titles and loans. Many searches per 10 titles with few loans is a buying signal.

  Set up: `python -m app.sync --circulation --since 2023-01-01` once, and the fines report in docs/koha-setup.md.

  The search and view logs behind these reports are anonymous. They keep the words searched, the number of results, the titles opened and the time, and never who searched.
- Member review approval and a *Sync now* button on the same page.
- A nightly copy of the catalogue from Koha, with optional AI cover descriptions and summaries.
- `python -m app.check`, which tests the connection to Koha step by step and says what to fix.

---

## How it works

```
Browser ──> Catalogue Discover (Python, FastAPI) ──> Koha REST API (/api/v1)
                    │
                    └── local search index (SQLite): a copy of the catalogue, refreshed nightly
```

Searching runs on the app's own copy of the catalogue, so it is fast even when Koha is slow. Live data (availability on a title page, a reader's loans, holds and fines, renewals and holds) is read from Koha when it's needed.

| Part | File | What it does |
|---|---|---|
| Koha connection | `app/koha.py` | REST API with an API key (OAuth client credentials) or a username and password. Patron PINs go straight to Koha's password check and are never stored. |
| Catalogue copy | `app/sync.py` | Reads records 250 at a time and copies 1,000 at a time. Skips records Koha can't return, and carries on after an interruption. |
| MARC reading | `app/marc.py` | Titles, authors (1XX/7XX), ISBN (020), language (008/041), audience and fiction (008), series (490/8XX), notes (5XX), subjects (6XX). Ignores junk codes such as `|||` and impossible years. |
| Subject map | `app/dewey.py`, `app/ddcnames.py` | Builds the Dewey tree. Uses OCLC's official DDC Summaries captions when the library has loaded them (see above); otherwise plain-language class and division names, and sections named from this library's own most-used subject heading. Each section learns a word profile from its books' titles and subjects, which powers the related-subject search. |
| Search | `app/search.py`, `app/catalog.py` | Understands filters and topics, corrects spelling against the catalogue's own words, and ranks title, author, subject, notes and cover matches. |
| Claude (optional) | `app/ai.py` | Understands searches, writes plain-language summaries, describes covers, and reads photos. Every call falls back to the built-in rules. |
| Reviews | `app/reviews.py` | Google Books and Open Library ratings by ISBN, cached for a week. Goodreads closed its API in 2020 and Amazon needs an Associates account, so both are links. |
| Reports | `app/reports.py` | The staff search and display reports, built from the catalogue copy and the anonymous logs. |
| Exports | `app/exports.py` | CSV, Excel (openpyxl), RIS, BibTeX and APA citations. |
| Circulation reports | `app/circulation.py` | Use between two dates, use factor, borrowers, fines. |
| Alerts | `app/alerts.py` | Reruns saved searches after a sync and emails new titles through the library's mail server. |
| Meaning search and themes | `app/semantic.py` | Sentence embeddings (sentence-transformers) or latent semantic analysis (scikit-learn), blended with word matches by reciprocal rank fusion; k-means themes named by class-based TF-IDF. |
| Course readings | `app/courses.py` | Reads the semester CSV, links rows to the catalogue by biblio number (or ISBN), and recognises course codes, course names and faculty in a search. |
| Front end | `app/static/` | Plain HTML and JavaScript, no build step. Works on phones. |

---

## Quick start: try it with sample data

You need Python 3.11 or later.

**Windows (PowerShell)**

```powershell
cd koha-discover
python -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
copy .env.example .env
.venv\Scripts\python -m uvicorn app.main:app --port 8080
```

**Linux / macOS**

```bash
cd koha-discover
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
cp .env.example .env
.venv/bin/uvicorn app.main:app --port 8080
```

Open http://localhost:8080 and sign in with card **LIB1024**, PIN **1234**.

---

## Install

**[INSTALL.md](INSTALL.md)** covers connecting to your Koha, copying the catalogue, turning on meaning search, course readings and the optional Claude chat, running it on the Koha server behind Apache, and troubleshooting.

---

## Privacy

- Patron PINs go straight to Koha for checking and are never stored. A sign-in lasts 8 hours in a signed cookie.
- Patrons can only renew loans and cancel holds on their own account.
- "Readers who borrowed this also borrowed" uses anonymised borrowing history. See the report notes in docs/koha-setup.md.
- The search and view logs for staff reports never record who searched. Saved searches are tied to the reader's card number and are deleted when the reader deletes them.
- Reading lists are not tied to anyone. Anyone with a list's link can see it.
- Meaning search runs on the library's own computer. No catalogue data or searches leave it for that.
- With Claude enabled, search text, catalogue notes and cover images are sent to Anthropic's API. Patron data is not. Without `ANTHROPIC_API_KEY` nothing is sent.
- Course readings hold faculty names and course codes from the library's public list, nothing about students.

## Development

```
pip install -r requirements-dev.txt
pytest -q
```

The tests cover demo mode and a mock Koha server (`tests/mock_koha.py`) that mirrors Koha's REST responses, including page-size caps, damaged records and invalid item dates.

## Project layout

```
app/            the application (Python) and its web pages (app/static)
deploy/         Apache, systemd and IIT Gandhinagar settings
docs/           Koha setup guide
tests/          automated tests and a mock Koha
Dockerfile, docker-compose.yml, requirements.txt, requirements-semantic.txt, .env.example
```

## Not in this repository

- `.env`: your Koha API key, admin token and other secrets. Copy `.env.example` and fill it in.
- `data/`: the local catalogue copy, logs, loans, reading lists and the meaning index. Each installation makes its own.
- `data/ddc-names.csv`: OCLC's Dewey captions (copyright OCLC). Each library makes it from the PDF with `python -m app.ddcnames`.

## Versions

| Version | What it added |
|---|---|
| 5.0 | "By" vs "on" searches, meaning search, themes, course readings, Ask the librarian chat |
| 4.x | Circulation reports between two dates, borrowers, fines by year |
| 3.0 | Staff reports, exports, citations, reading lists, saved searches with alerts, official Dewey captions |
| 2.0 | Clickable authors and subjects, Dewey subject map, Picked for you, New arrivals |
| 1.0 | Natural-language search, spelling correction, patron account, reviews, recommendations |
