# Video Cross-Poster

Self-hosted Python app that uploads **one vertical video** and publishes it to multiple **TikTok**, **Instagram Reels**, and **YouTube Shorts** accounts using **official APIs only** (no browser automation, no scraping).

Built as a local demo you can show TikTok / Meta / Google when applying for audit / App Review / API verification. Until those approvals land, posts are forced **private** where the platforms require it.

## What it does

1. Connect multiple accounts per platform via OAuth (tokens encrypted at rest with Fernet).
2. Upload one video through a local web UI (`http://127.0.0.1:8000`).
3. Select accounts with checkboxes; caption mode **same for all** or **per account**.
4. YouTube title (defaults to first caption line), tags, privacy, made-for-kids.
5. TikTok: privacy from `creator_info`, comments / duet / stitch (respecting creator flags).
6. Post now or schedule for later.
7. Live status dashboard (uploading / processing / published / failed) + retry per account.
8. History with links when platforms return them.
9. `ffprobe` validation with **per-platform** warnings (does not block other platforms).

## Prerequisites

- **Python 3.11+**
- **ffmpeg** (provides `ffprobe`) on your PATH  
  - Windows: [ffmpeg.org](https://ffmpeg.org/download.html) or `winget install ffmpeg`
- Developer apps on each platform (steps below). You can run the UI without keys; OAuth stays disabled until `.env` is filled.

## Public site (GitHub Pages)

Static pages for the YouTube API audit form live in `docs/`:

- Home
- Privacy policy (YouTube API disclosures)
- Terms

Create a **public** GitHub repo, push this project, then: **Settings → Pages → Deploy from a branch → `main` / `/docs`**.

After it builds, the URLs are:

- `https://YOURUSER.github.io/video-crossposter/`
- `https://YOURUSER.github.io/video-crossposter/privacy.html`
- `https://YOURUSER.github.io/video-crossposter/terms.html`

Paste those into the audit form as Primary Access URL and Privacy Policy URL.

## Quick start

```powershell
cd C:\Users\tudor\video-crossposter
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
# Paste that value into TOKEN_KEY in .env
python -m app
```

Open **http://127.0.0.1:8000**.

---

## Developer account setup (step by step)

### 1. TikTok — Content Posting API + Login Kit

1. Go to [TikTok for Developers](https://developers.tiktok.com/) and create an app.
2. Add products: **Login Kit** and **Content Posting API**.
3. Enable **Direct Post** on Content Posting API.
4. Request scopes: `user.info.basic`, `video.publish` (optional: `video.upload` for draft/inbox mode).
5. Set redirect URI to: `http://127.0.0.1:8000/oauth/tiktok/callback`
6. Copy **Client Key** / **Client Secret** into `.env` as `TIKTOK_CLIENT_KEY` / `TIKTOK_CLIENT_SECRET`.

**Unaudited apps:** TikTok restricts posts to **private (`SELF_ONLY`)**. This app detects that and locks the privacy UI accordingly.

**How to apply for audit:** After you can successfully Direct Post as `SELF_ONLY`, submit a Content Posting API audit in the TikTok developer portal (provide screen recordings of the consent + export UX — this app’s post flow is designed for that). When approved, set `TIKTOK_AUDITED=true` in `.env`.

**API flow used:**  
`POST /v2/post/publish/creator_info/query/` → `POST /v2/post/publish/video/init/` (`FILE_UPLOAD`) → chunked `PUT` to `upload_url` → poll `/v2/post/publish/status/fetch/`.

---

### 2. Instagram — Graph API Reels (Facebook Login)

This project uses **Facebook Login for Business** so it can use **resumable uploads** (`rupload.facebook.com`) and avoid needing a public video URL.

**Tradeoff vs Instagram Login:**

| | Facebook Login (this app) | Instagram Login |
|---|---|---|
| Host | `graph.facebook.com` + `rupload` | `graph.instagram.com` |
| Local file upload | Yes (resumable) | No — needs public `video_url` |
| Scopes | `instagram_basic`, `instagram_content_publish`, Page perms | `instagram_business_basic`, `instagram_business_content_publish` |
| Account type | IG Professional linked to a Facebook Page | IG Professional |

**Setup:**

1. [Meta for Developers](https://developers.facebook.com/) → create an app (type: Business).
2. Add **Facebook Login** and **Instagram** / Graph products as required by current Meta UI.
3. Valid OAuth redirect: `http://localhost:8000/oauth/instagram/callback` (Meta allows plain `http` only for `localhost`, and only while the app is in development mode)
4. Put App ID / Secret in `META_APP_ID` / `META_APP_SECRET`.
5. Convert each IG account to **Professional** (Business or Creator) and link it to a **Facebook Page**.
6. Add yourself as an app role / tester while in development.

**Publishing limit:** 100 API-published posts per IG account per rolling 24 hours (`/{ig-user-id}/content_publishing_limit`). The UI shows remaining quota.

**App Review:** Until Advanced Access / content publish review is approved, publishing may be limited to roles/testers. Set `INSTAGRAM_REVIEWED=true` after approval.

**API flow used:**  
Create container `media_type=REELS` + `upload_type=resumable` → upload binary to rupload → poll `status_code=FINISHED` → `media_publish`.

---

### 3. YouTube — Data API v3 Shorts

1. [Google Cloud Console](https://console.cloud.google.com/) → create a project.
2. Enable **YouTube Data API v3**.
3. Configure OAuth consent screen (External is fine for personal use; add test users).
4. Create OAuth client ID → **Web application**.
5. Authorized redirect URI: `http://127.0.0.1:8000/oauth/youtube/callback`
6. Copy Client ID / Secret to `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET`.

**Quota (current):** `videos.insert` uses a dedicated **Video Uploads** bucket — default **100 uploads/day** per project (1 unit each). This is separate from the general 10,000-unit pool. The UI warns before you burn the bucket.

**Unverified projects** (created after 28 Jul 2020): uploads are forced **private**. Apply for the [YouTube API Services audit](https://developers.google.com/youtube/v3/guides/quota_and_compliance_audits) to unlock public/unlisted. Then set `YOUTUBE_VERIFIED=true`.

**Shorts:** There is no separate Shorts endpoint. Vertical (or square) videos ≤ **3 minutes** are classified as Shorts. Optional `#Shorts` append toggle is included but not required for classification.

**API flow used:** OAuth `youtube.upload` → resumable `videos.insert` via `google-api-python-client`.

---

## How to connect accounts

1. Fill `.env`, restart `python -m app`.
2. Open **Accounts** → click **+ TikTok / Instagram / YouTube**.
3. Complete the provider’s OAuth consent.
4. Instagram may connect **multiple** IG accounts if several Pages are linked.
5. Disconnect anytime; encrypted tokens are deleted from SQLite.

## How to post

1. **New post** → choose a video → select accounts.
2. Caption: same for all, or per-account (YouTube title editable).
3. Per-account privacy / interaction options load from live APIs where available.
4. **Post now** or set a schedule.
5. Watch **Status**; use **Retry** on failures (other accounts are unaffected).

## Auto subtitles (for DaVinci Resolve)

`make_subs.py` transcribes an audio or video file locally with Whisper (`large-v3-turbo`, runs on the GPU) and writes a time-synced `.srt` next to it.

```powershell
pip install -r requirements-subtitles.txt
python make_subs.py voiceover.wav              # -> voiceover.srt
python make_subs.py clip.mp4 --short           # 1 line, few words per subtitle (TikTok/Reels style)
python make_subs.py clip.mp4 -l en --prompt "DaVinci Resolve, Tudor"   # force language, fix spellings
```

Or drag files onto `make_subs.bat`. The first run downloads the model (~1.6 GB).

In Resolve: drag the `.srt` into the Media Pool, then onto the timeline so it starts at the same frame as the audio clip (or **File → Import → Subtitle**). If the audio doesn't start at the very beginning of the timeline, either drop the subtitle where the audio starts or use `--offset <seconds>`.

## Project layout

```
video-crossposter/
├── app/
│   ├── main.py              # FastAPI app
│   ├── config.py            # .env settings
│   ├── crypto.py            # Fernet token encryption
│   ├── db.py / models.py    # SQLite schema
│   ├── worker.py            # Parallel upload queue
│   ├── video_validate.py    # ffprobe checks
│   ├── platforms/           # TikTok / Instagram / YouTube clients
│   ├── routers/             # OAuth, API, HTML pages
│   ├── templates/           # Jinja2 UI
│   └── static/              # CSS / JS
├── .env.example
├── requirements.txt
└── README.md
```

## Known limitations (demo / audit)

| Platform | Before approval | After approval |
|---|---|---|
| TikTok | `SELF_ONLY` only | Public privacy levels from `creator_info` (`TIKTOK_AUDITED=true`) |
| Instagram | Dev mode / role users; App Review needed for broad publish | `INSTAGRAM_REVIEWED=true` |
| YouTube | Forced private uploads | Public/unlisted (`YOUTUBE_VERIFIED=true`) |
| YouTube quota | ~100 uploads/day/project | Request increase via Google |
| Instagram quota | 100 publishes / 24h / account | Same hard limit |
| Scheduling | Local scheduler polls every ~20s; YouTube uses `publishAt` (private until then) | Works the same |

Logs: `data/logs/crossposter.log`  
DB: `data/crossposter.db`  
Uploads: `data/uploads/`

## Showing this to platforms for API access

1. Keep `DEMO_MODE=true` and audit flags `false`.
2. Screen-record: connect → upload → private publish → status → history.
3. Point reviewers at this README + your developer app IDs.
4. After each approval, flip the matching `*_AUDITED` / `*_REVIEWED` / `*_VERIFIED` flag and restart.

## License

Personal / internal use. You are responsible for complying with each platform’s Terms of Service and developer policies.
