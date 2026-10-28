# Intern Task Bot

An AI meeting bot for mentors who run intern programs. It:

1. creates (or accepts) a meeting link and joins the meeting as a bot,
2. explains each intern's tasks by voice (or in meeting chat) and answers their questions,
3. after the meeting, sets deadlines and a review date,
4. emails every intern their tasks, deadlines and review date (with a calendar `.ics`),
5. hosts the review meeting on the review date, checks each task, and schedules the next review if tasks remain open.

A web dashboard manages interns, meetings, tasks, reviews, transcripts and system status.

**Stack:** Python 3.12+, FastAPI, SQLAlchemy + SQLite, Alembic migrations (applied automatically), an asyncio scheduler (20 s tick). No database server needed.

---

## 1. Quick start (works with zero configuration)

The project ships with built-in defaults, so it starts **without any API keys** in *demo mode*:
dashboard works, text is template-based instead of AI-written, no voice, no emails, meeting links are pasted by hand.

**Default dashboard login:** user `admin`, password `change-me`
(demo defaults, only meant for `localhost`. Change them in `.env` before putting this on the internet.)

### Requirements
- Python 3.12 or newer (https://www.python.org/downloads/). On Windows, tick **"Add python.exe to PATH"** during install.
- Internet access (to install packages the first time).

### Windows
1. Right-click `intern-task-bot.zip` > **Extract All...** (or use 7-Zip). Open the extracted `intern-task-bot` folder.
2. Double-click **`start.bat`**. First run creates a virtual environment, installs packages, creates `.env`, and starts the server.
3. Open **http://localhost:8000** and log in (`admin` / `change-me`).

### Mac / Linux
```bash
unzip intern-task-bot.zip
cd intern-task-bot
./start.sh            # if "permission denied": chmod +x start.sh && ./start.sh
```
Then open **http://localhost:8000**.

### Manual start (any OS, if you prefer typing commands)
```bash
cd intern-task-bot
python -m venv .venv
# Windows: .venv\Scripts\activate      Mac/Linux: source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env                    # Windows: copy .env.example .env
uvicorn app.main:app --port 8000
```

### Docker (optional)
```bash
cp .env.example .env
docker compose up -d --build
```
Open http://localhost:8000. Data is stored in `./data/bot.db`.

### Stop the server
Press `Ctrl + C` in the terminal window (or close it).

---

## 2. Using the dashboard (first walkthrough)

1. **Interns** tab: add interns (name + email).
2. **Meetings** tab > **New meeting (Kickoff)**: pick a start time, choose interns, add tasks per intern, paste a Google Meet / Zoom / Teams link (manual mode).
3. At start time the bot joins (needs Recall.ai, see section 4), briefs the interns, answers questions, and leaves when everyone leaves.
4. Afterwards the bot sets deadlines and a review date, creates the review meeting, and emails everyone (needs SMTP).
5. On the review date the bot asks each intern about each task, marks it done / partial / not done, and schedules the next review if needed.
6. **System** tab: shows a setup checklist of what is configured and what is missing.

Without keys you can still explore every screen, create interns/meetings/tasks and see the whole data model working.

---

## 3. Changing configuration (the `.env` file)

All settings live in one file named `.env` in the project folder (created automatically from `.env.example` on first start). It is a plain-text file with `NAME=value` lines.

**How to change a setting**
1. Stop the server (`Ctrl + C`).
2. Open `.env` in any text editor (Notepad is fine).
3. Edit the value after the `=`, with no spaces and no quotes, save.
4. Start the server again.

**If you can't see `.env` on Windows:** it is a hidden-style file name. Open Notepad > File > Open > choose "All files" > select `.env`. On Mac/Linux use `ls -a`.

### Most useful settings
| Setting | Meaning | Default |
|---|---|---|
| `DASHBOARD_USER` / `DASHBOARD_PASSWORD` | Dashboard login | `admin` / `change-me` |
| `WEBHOOK_SECRET`, `SESSION_SECRET` | Random strings that protect webhooks and login cookies. Generate: `python -c "import secrets;print(secrets.token_urlsafe(32))"` | placeholder |
| `TIMEZONE` | Times shown in emails/dashboard, e.g. `Europe/London`, `America/New_York` | `UTC` |
| `MENTOR_EMAIL` | Receives summaries | empty |
| `MEETING_PROVIDER` | `manual` (paste links), `zoom`, `meet`, `teams` | `manual` |
| `RECALL_API_KEY`, `RECALL_REGION` | Bot that joins calls | empty, `us-east-1` |
| `ANTHROPIC_API_KEY` | AI brain (Claude). Or use `GEMINI_API_KEY` (free tier) | empty |
| `TTS_PROVIDER` | `none` (bot types in chat), `elevenlabs`, `openai` | `none` |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_FROM` | Email sending | empty |
| `DEFAULT_DEADLINE_DAYS`, `REVIEW_HOUR`, `FOLLOWUP_DAYS` | Scheduling behaviour | `5`, `11`, `3` |
| `REQUIRE_APPROVAL` | `true` = mentor must approve deadlines before emails go out | `false` |

The full list with comments is in `.env.example`.

> **Security: never commit `.env` or paste real keys into any file that is pushed to Git.** `.env` is already in `.gitignore`. Only `.env.example` (placeholders, no secrets) belongs in the repository.

---

## 4. Turning on the real features (optional accounts)

Each item is independent. Add the key to `.env`, restart, and check **Dashboard > System**.

| Feature | What you need | `.env` keys |
|---|---|---|
| Bot joins live meetings | Recall.ai account + API key | `RECALL_API_KEY`, `RECALL_REGION` (must match your Recall region) |
| Recall must reach your computer | A public https URL (tunnel, see below) | `PUBLIC_BASE_URL` |
| AI answers/briefings/deadline extraction | Anthropic API key (console.anthropic.com, separate from a claude.ai subscription) **or** free Gemini key (aistudio.google.com) | `ANTHROPIC_API_KEY` or `GEMINI_API_KEY` |
| Bot speaks aloud | ElevenLabs or OpenAI key | `TTS_PROVIDER=elevenlabs` + `ELEVENLABS_API_KEY`, or `TTS_PROVIDER=openai` + `OPENAI_API_KEY` |
| Emails with calendar invites | Any SMTP account. Gmail: enable 2-step verification, create an App Password | `SMTP_HOST=smtp.gmail.com`, `SMTP_PORT=587`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_FROM`, `SMTP_SECURITY=starttls` |
| Auto-create Google Meet links | Google Cloud OAuth client | `MEETING_PROVIDER=meet`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, then Dashboard > System > "Connect Google Meet account" |
| Auto-create Zoom meetings | Zoom Server-to-Server OAuth app | `MEETING_PROVIDER=zoom`, `ZOOM_ACCOUNT_ID`, `ZOOM_CLIENT_ID`, `ZOOM_CLIENT_SECRET`, `ZOOM_HOST_USER` |
| Auto-create Teams meetings | Microsoft 365 admin app registration | `MEETING_PROVIDER=teams`, `TEAMS_*` keys |

If you skip a provider, the bot degrades gracefully: no AI key -> template/keyword logic; no TTS -> writes in meeting chat; no SMTP -> emails skipped (logged); `manual` -> you paste links.

### Making the bot reachable (Recall needs a public URL)
**Easiest (laptop, free):** install `cloudflared` (https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/). Windows users: `start.bat` downloads it automatically. `run.py` starts a free tunnel and sets `PUBLIC_BASE_URL` for that run automatically. The URL changes every run and the computer must stay on.

**Stable (VPS):** run with Docker on a server, put Caddy/Nginx in front for HTTPS, set `PUBLIC_BASE_URL=https://your.domain`.
In the Recall dashboard, add a status webhook `<PUBLIC_BASE_URL>/webhooks/recall-status/` and copy its signing secret (`whsec_...`) into `RECALL_WEBHOOK_SECRET`.

### Zoom caveat
Recall's Zoom bot is not the host, so Zoom asks the host for recording consent unless a join token is supplied. Google Meet / Teams have no such step, so `MEETING_PROVIDER=manual` with a Meet link is the lowest-friction path.

---

## 5. Health check

```bash
python -m app.preflight --email you@example.com
```
Tests every configured dependency once and prints PASS / FAIL / SKIP. With no keys, unconfigured items show SKIP; "Config sanity" will FAIL until you change the default password/secret and set a public URL; that is expected in demo mode. (`start.bat` / `start.sh` run this automatically about 20 seconds after launch.)

## 6. Tests

```bash
pip install -r requirements.txt
pytest
```
All outbound services (Zoom, Recall, SMTP, AI) are stubbed, so tests need no network or keys.

## 7. Project layout

```
app/main.py          REST API, webhooks, login, dashboard routes
app/workflow.py      all business logic (kickoff, deadlines, reviews, reminders, scheduler)
app/db.py            SQLAlchemy models, migrations bootstrap
app/config.py        settings and built-in defaults (reads .env)
app/services/        zoom, meet, teams, recall, llm, tts, mailer, http helpers
app/static/          dashboard (single HTML page)
migrations/          Alembic migrations
tests/               pytest suite
run.py               one-command launcher (tunnel + server + preflight)
start.bat / start.sh launchers for Windows / Mac+Linux
```

## 8. Troubleshooting

| Problem | Fix |
|---|---|
| `python` not found | Install Python 3.12+ and re-open the terminal; on Windows tick "Add to PATH" |
| Port 8000 busy | Set environment variable `PORT=8001` (the launcher) or run `uvicorn app.main:app --port 8001` |
| Can't log in | Defaults are `admin` / `change-me`, or whatever you set in `.env`. Login is locked for 15 min after 5 wrong tries per IP; restart the server to clear |
| Bot never joins a meeting | Needs `RECALL_API_KEY` and a public `PUBLIC_BASE_URL` (tunnel running) |
| Bot is held in the lobby | Admit "Task Bot" manually (Meet/Teams lobby settings) |
| No emails | `SMTP_*` not set, or wrong app password; check Dashboard > System and the log |
| Start over with a clean database | Stop the server, delete the `data` folder, start again |

## 9. Demo / video walkthrough (no accounts needed)

Honest 5-minute demo using only the defaults:
1. Start the app (section 1) and log in with `admin` / `change-me`.
2. **Interns**: add a test intern (use a fake address such as `intern@example.com`).
3. **Meetings > New meeting (Kickoff)**: add a task for that intern, paste any meeting link (for example `https://meet.google.com/abc-defg-hij`), save, then open the meeting. Show the tasks, the transcript area, and the Reschedule / Resend-invite buttons.
4. Show the **Tasks** and **System** tabs (the System checklist shows exactly which integrations are connected).
5. Stop the server (`Ctrl + C`) and run `python -m pytest tests -v`. The tests simulate the whole flow (kickoff, briefing, deadlines, review meeting, review answers, next review, results email) with stubbed services. Expect all tests to pass.

Say plainly that joining live meetings, voice and AI answers need your own Recall.ai, AI and meeting accounts (section 4), which are not included. Do not claim a live-call demo you haven't run.

## 10. Before you push to a public Git repo (privacy checklist)

- [ ] No `.env` file in the folder you commit (`git status` must not list it; `.gitignore` already blocks it).
- [ ] No `data/` folder or `*.db` files (they can hold interns' emails and transcripts).
- [ ] Only placeholder values in `.env.example` (it contains none of your real keys, emails or passwords).
- [ ] Your git identity is not personal: `git config user.name` / `git config user.email`. GitHub offers a private "no-reply" email in Settings > Emails.
- [ ] Search for leftovers before committing:
  - Mac/Linux: `grep -rniE "your-name|your-email|password=.+|api_key=.+" . --exclude-dir=.venv --exclude-dir=.git`
  - Windows (PowerShell): `Select-String -Path .\* -Pattern "your-name","your-email" -Recurse`
- [ ] If a real key was ever committed, deleting the file is not enough: revoke the key at the provider and create a new one.

## 11. Limits

Offline-tested only. Live audio in real calls, and each external provider, need a real-account test run: create one short kickoff with yourself as the only intern (about 2 minutes) before real use. Tell attendees the meeting is recorded/transcribed.
