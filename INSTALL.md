# Installing Catalogue Discover

These steps take you from the code to a running site connected to your Koha. To try the app first with sample data and no Koha, see *Quick start* in the [README](README.md).

You need Python 3.11 or later (tested up to 3.14) and a Koha 22.11 or later with its REST API on.

## 0. Get the code

```bash
git clone https://github.com/<owner>/<repo>.git koha-discover
cd koha-discover
python -m venv .venv
.venv/bin/pip install -r requirements.txt        # Windows: .venv\Scripts\python -m pip install -r requirements.txt
```

On Windows, run `$env:PYTHONUTF8=1` in PowerShell first.

## Connect it to your Koha

### 1. Prepare Koha (about 15 minutes, needs a Koha administrator)

Follow **[docs/koha-setup.md](docs/koha-setup.md)**. In short:

1. Create a staff patron for the app (for example `discover-api`) and give it the permissions listed there.
2. Turn on `RESTOAuth2ClientCredentials` in **Administration › System preferences**, then generate an API key on that patron (**More › Manage API keys**).
3. If the API answers every request with 401, add one line to Koha's Apache configuration (see "If the API refuses every login" in the setup guide).

### 2. Fill in `.env`

Copy `.env.example` to `.env`. IIT Gandhinagar can start from `deploy/iitgn.env` instead. The settings that matter:

```ini
DISCOVER_MODE=koha
KOHA_URL=https://koha.yourlibrary.org          # staff interface
KOHA_OPAC_URL=https://catalog.yourlibrary.org  # public OPAC
KOHA_AUTH=oauth
KOHA_CLIENT_ID=...
KOHA_CLIENT_SECRET=...
KOHA_DEFAULT_PICKUP_LIBRARY=MAIN               # branch code for online holds
LIBRARY_NAME=Your Library
SECRET_KEY=...    # python -c "import secrets;print(secrets.token_hex(32))"
ADMIN_TOKEN=...   # same command, a different value
```

`.env` holds your keys. It is listed in `.gitignore` and must never be committed.

For saved-search email alerts, also fill in `PUBLIC_URL` and the `SMTP_*` settings with your institute's mail server (ask IT for an account that may send mail). Without them the app works the same, and readers see new titles on the home page instead of by email.

### Optional: official Dewey names for Browse by subject

Out of the box, classes and divisions have plain-language names, and sections are named from your own most-used subject headings. To use OCLC's official captions instead (for example "541 Physical chemistry"):

1. Download the DDC 23 Summaries PDF: https://www.oclc.org/content/dam/oclc/dewey/ddc23-summaries.pdf
2. Run `python -m app.ddcnames ddc23-summaries.pdf`. It saves `data/ddc-names.csv`.
3. Restart the app. Browse by subject, title pages, search and staff reports now use the official captions, and the home page shows OCLC's copyright notice as OCLC's terms require.

OCLC owns the copyright in the Dewey Decimal Classification. The captions stay in your `data` folder and are not part of the code, so don't commit `data/ddc-names.csv` to a public repository. OCLC allows non-commercial use of the DDC Summaries with that notice; if in doubt, ask OCLC or your WebDewey contact.

### 3. Check the connection

```
python -m app.check
python -m app.check --card <a test card> --password <its password>   # also tests patron sign-in
```

Each step prints `ok` or `FAIL` with what to fix, and how many seconds it took.

### 4. Copy the catalogue

```
python -m app.sync
```

On a slow Koha this takes about an hour for 40,000 titles (IIT Gandhinagar: 56 minutes). It prints progress, and if it is interrupted, running it again carries on from the last finished batch.

| Command | When to use it |
|---|---|
| `python -m app.sync` | Normal refresh (nightly) |
| `python -m app.sync --full` | Start again from the first record |
| `python -m app.sync --availability-only` | Refresh only which copies are on the shelf and their accession dates, used for *New arrivals* (about 20 minutes) |
| `python -m app.sync --reparse` | Re-read records from the saved MARC after a fix to the MARC reading (seconds, no Koha needed) |
| `python -m app.sync --enrich` | Add AI cover descriptions and summaries (needs `ANTHROPIC_API_KEY`) |
| `python -m app.sync --circulation --since 2023-01-01` | First copy of loans for the circulation reports (later syncs top it up) |
| `python -m app.sync --send-alerts` | Only email readers about new titles for their saved searches (every normal sync does this at the end) |

### 5. Start it

```
python -m uvicorn app.main:app --host 0.0.0.0 --port 8080
```

`--host 0.0.0.0` lets other computers on the network open `http://<this computer's IP>:8080`. For anything beyond a trial, put it behind https (see *Run it on the Koha server* below): without it, library passwords cross the network unencrypted.

---

## Meaning search and themes (version 5)

Install the sentence model (about 200 MB of PyTorch now, and the 470 MB model on first use):

```
pip install -r requirements-semantic.txt
python -m app.semantic
```

The first run reads every title. For 42,000 titles on an ordinary PC it takes 15 to 40 minutes, and it ends with `Meaning search ready (model): … titles, … themes`. After that, each nightly sync adds only new or changed titles. Without `requirements-semantic.txt` the app uses latent semantic analysis (scikit-learn), which builds in under a minute.

| Setting in `.env` | Meaning |
|---|---|
| `SEMANTIC` | `auto` (default), `model`, `lsa`, or `off` |
| `EMBED_MODEL` | Any sentence-transformers model name, or a local folder holding one |
| `THEMES` | Number of themes. `0` picks about the square root of half the number of titles |

If the computer cannot reach huggingface.co, save the model on another computer with
`python -c "from sentence_transformers import SentenceTransformer as S; S('sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2').save('mini-model')"`,
copy the `mini-model` folder across, and set `EMBED_MODEL=mini-model`.

## Course readings

On the staff page (`/admin`), open **Course readings**, give the semester a name, and upload its CSV. The app reads these columns, in any order and with any of the usual spellings: Name of Faculty, Course Code, Course Name, Title, Author, Publisher, Year, Print ISBN, Call No., Acc. No., Textbook/Reference Books, Lending Policy (CR, SL or Ref), e-book Link, Biblio no. Each row is one copy. Rows are linked to the catalogue by biblio number, or by ISBN when there isn't one.

A new upload becomes the current semester. Earlier semesters are kept, and any of them can be made current again. E-book links stay hidden unless `COURSE_EBOOK_LINKS=true`. Turn that on only when every link points to a licensed copy.

## Ask the librarian (optional, paid)

Add an Anthropic API key (`ANTHROPIC_API_KEY`) from https://console.anthropic.com and set a monthly spend limit there. Each chat question costs a few US cents. `AI_QUERY_PARSING=false` keeps Claude out of ordinary searches, so you pay only for chat questions. `ASK_PER_HOUR` (default 30) limits questions per computer. Without a key, the chat button is hidden and nothing is sent outside.

---

## Run it on the Koha server

The app runs in one Docker container that listens only on `127.0.0.1:8080`, and Apache publishes it on the OPAC address.

```bash
sudo git clone https://github.com/<owner>/<repo>.git /opt/koha-discover && cd /opt/koha-discover
cp deploy/iitgn.env .env && nano .env          # fill in the key, SECRET_KEY, ADMIN_TOKEN
sudo chmod 600 .env
sudo docker compose up -d --build
sudo docker compose run --rm discover python -m app.check
sudo docker compose run --rm discover python -m app.sync
```

Add the lines in [deploy/apache-discover.conf](deploy/apache-discover.conf) to the OPAC `<VirtualHost *:443>`, then:

```bash
sudo a2enmod proxy proxy_http headers
sudo apachectl configtest && sudo systemctl reload apache2
```

The search is now at `https://<your-opac>/discover/`. Link to it from Koha's `OpacMainUserBlock` or `OpacNav` (**Tools › HTML customizations**).

Nightly refresh at 02:30:

```bash
sudo cp deploy/discover-sync.service deploy/discover-sync.timer /etc/systemd/system/
sudo systemctl daemon-reload && sudo systemctl enable --now discover-sync.timer
```

To update, run `sudo git pull` (your `.env` and data are not in git, so they are kept) and `sudo docker compose up -d --build`. The search index and member reviews live in the `discover-data` Docker volume.

---

## Troubleshooting

| What you see | Cause and fix |
|---|---|
| `app.check` fails with 401 on every step | Apache drops the login header (common with CGI installs). Add the `SetEnvIf Authorization` line from docs/koha-setup.md. If staff use LDAP, use an API key, not a password. |
| `Koha refused the API client credentials` | Wrong `KOHA_CLIENT_ID` / `KOHA_CLIENT_SECRET`, or `RESTOAuth2ClientCredentials` is off. |
| Every step takes about 8 seconds | Koha is running its API as CGI. The app copes, but running Koha under Plack makes it much faster. |
| Sync log: `Koha could not return N record(s)` | Those records are damaged in Koha (the API answers 500). Fix or re-save them in Koha's cataloguing editor. |
| Sync log: `Left out about N copies` | Those items have impossible dates (month or day `00`). See "Items with impossible dates" in docs/koha-setup.md. |
| No authors, publishers or Dewey path on title pages after an update | Run `python -m app.sync --reparse` (seconds). If it says no saved MARC, run `python -m app.sync --full` once. |
| Odd languages or years in the filters | Junk in MARC 008. The app now ignores it. Run `python -m app.sync --reparse` after updating. |
| `UnicodeDecodeError` on Windows | Update to the latest version, or set `$env:PYTHONUTF8=1`. |
| Pillow fails to install on Python 3.14 | `pip install "Pillow>=11.3"`, then install the rest. |

---
