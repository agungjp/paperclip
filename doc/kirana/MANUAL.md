# PAPERCLIP — Manual Lengkap Kirana Corp

**Versi Dokumen:** 1.0
**Dibuat:** 2026-03-19
**Konteks:** Kirana Corp — software house AI-powered
**Referensi:** Paperclip v1, MIT License

---

## Daftar Isi

1. [Apa itu Paperclip?](#bagian-1-apa-itu-paperclip)
2. [Arsitektur & Konsep Kunci](#bagian-2-arsitektur--konsep-kunci)
3. [Instalasi & Setup](#bagian-3-instalasi--setup)
4. [Cara Kerja Agent — Heartbeat Deep Dive](#bagian-4-cara-kerja-agent--heartbeat-deep-dive)
5. [API Reference Ringkas](#bagian-5-api-reference-ringkas)
6. [CLI Reference](#bagian-6-cli-reference)
7. [Setup Agent Baru](#bagian-7-setup-agent-baru)
8. [Contoh Case Lengkap](#bagian-8-contoh-case-lengkap)
9. [Rekomendasi Setup untuk Kirana Corp](#bagian-9-rekomendasi-setup-untuk-kirana-corp)
10. [Apa yang Perlu Dikembangkan](#bagian-10-apa-yang-perlu-dikembangkan)
11. [Fork & Customize Strategy](#bagian-11-fork--customize-strategy)

---

## Bagian 1: Apa itu Paperclip?

### Konsep Dasar

Paperclip adalah **Node.js server + React UI** yang berfungsi sebagai control plane untuk orkestrasi tim AI agents. Bukan chatbot, bukan workflow builder, bukan prompt manager — Paperclip adalah **perusahaan** tempat agents bekerja.

Paperclip self-hosted (open source, MIT License). Tidak ada akun cloud yang dibutuhkan. Kamu deploy sendiri di mesin lokal atau server.

Visi besarnya: "Paperclip adalah backbone ekonomi otonom." Ribuan perusahaan AI yang berjalan di atas Paperclip sebagai control plane.

### Analogi: "Jika OpenClaw adalah karyawan, Paperclip adalah perusahaan"

| Analogi | Dunia Manusia | Paperclip |
|---------|--------------|-----------|
| Karyawan | Manusia yang bekerja | AI agents (Claude, Codex, OpenClaw, dll) |
| Perusahaan | PT, CV, organisasi | Paperclip company |
| Struktur organisasi | Org chart, jabatan, laporan | Org chart + reporting lines |
| Tugas | Job description, task, tiket | Issues / Tasks |
| Anggaran | Gaji bulanan | Budget token per agent |
| Atasan | Manager, CEO | Manager agent, CEO agent |
| Board | Dewan direksi | Board (kamu — manusia) |

Jika kamu punya 20 terminal Claude Code terbuka dan tidak tahu siapa mengerjakan apa — Paperclip adalah jawabannya.

### Kapan Perlu Paperclip

- Kamu mengkoordinasi **banyak agents** (Claude, Codex, OpenClaw, Cursor) ke satu tujuan bersama
- Kamu ingin agents berjalan **otonom 24/7** tapi tetap mau audit dan intervensi kapanpun
- Kamu ingin **monitor biaya** dan enforce budget per agent
- Kamu ingin struktur organisasi yang jelas — siapa lapor ke siapa, siapa kerjakan apa
- Kamu ingin **akses dari HP** untuk pantau dan manage pekerjaan agents
- Kamu punya recurring jobs (customer support, social media, laporan) yang harus berjalan terjadwal

### Kapan TIDAK Perlu Paperclip

- Kamu cuma punya **satu agent** untuk satu task sederhana — overkill
- Kamu hanya ingin **chatbot** atau asisten percakapan
- Kamu butuh **workflow drag-and-drop** (Zapier, n8n) — itu bukan Paperclip
- Kamu butuh **review code** saja — Paperclip bukan code review tool
- Task kamu selesai dalam satu sesi, tidak perlu persistensi

### Perbandingan: Tanpa vs Dengan Paperclip

| Masalah (Tanpa Paperclip) | Solusi (Dengan Paperclip) |
|--------------------------|--------------------------|
| 20 tab Claude Code terbuka, tidak tahu siapa kerjakan apa. Setelah reboot semuanya hilang. | Task berbasis tiket, conversation di-thread, session persist lintas reboot. |
| Harus kumpulkan context dari berbagai tempat untuk ingatkan bot apa yang sedang dikerjakan. | Context mengalir dari task ke project ke company goals — agent selalu tahu apa dan kenapa. |
| Folder agent config berantakan, re-invent task management, komunikasi, dan koordinasi antar agent. | Paperclip kasih org chart, ticketing, delegation, dan governance out of the box. |
| Loop tak terkendali buang ratusan dollar token, kuota habis sebelum kamu sadar. | Cost tracking tampilkan budget token dan throttle agent saat budget habis. |
| Recurring jobs harus di-kick manual setiap kali. | Heartbeat handle pekerjaan rutin terjadwal. Management supervises. |
| Ada ide, harus buka repo, jalankan Claude Code, tunggu, babysit terus. | Tambah task di Paperclip. Coding agent kerjakan sampai selesai. Management review hasilnya. |

---

## Bagian 2: Arsitektur & Konsep Kunci

### Komponen Utama

```
┌─────────────────────────────────────────────────────────┐
│  Paperclip System                                       │
│                                                         │
│  ┌─────────────────┐    ┌─────────────────────────────┐ │
│  │   React UI      │    │   Express API Server        │ │
│  │  (Board View)   │◄──►│   http://localhost:3100     │ │
│  └─────────────────┘    └──────────┬──────────────────┘ │
│                                   │                     │
│  ┌─────────────────────────────────▼──────────────────┐ │
│  │   Embedded PostgreSQL (PGlite)                     │ │
│  │   ~/.paperclip/instances/default/db                │ │
│  └────────────────────────────────────────────────────┘ │
│                                                         │
│  ┌─────────────────────────────────────────────────────┐ │
│  │   Local Storage (Attachments, Backups)              │ │
│  │   ~/.paperclip/instances/default/data/             │ │
│  └─────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘

       Agents (External)
       ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
       │ Claude Code │  │    Codex    │  │  OpenClaw   │
       │  (local)    │  │  (local)    │  │  (remote)   │
       └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
              │                │                 │
              └────────────────┴─────────────────┘
                               │
                     Heartbeat / API calls
                               │
                    http://localhost:3100/api
```

**Komponen utama:**

- **Server** — Express REST API, berjalan di port 3100. Mengelola semua business logic.
- **UI** — React app, di-serve oleh API server (same origin). Board view untuk manusia.
- **Database** — Embedded PGlite (PostgreSQL) otomatis. Tidak perlu setup manual.
- **Agents** — Berjalan di luar Paperclip, "phone home" via heartbeat API calls.
- **CLI** — `paperclipai` command untuk setup, manage, dan control plane operations.

### Model Company, Agent, Task/Issue

**Company** adalah unit isolasi tertinggi. Satu deployment Paperclip bisa run banyak companies dengan data isolation penuh.

```
Company
├── Goals (company → team → agent → task level)
├── Projects (group issues toward deliverable)
│   └── Workspaces (cwd, repoUrl, repoRef)
├── Issues/Tasks
│   ├── parentId (hierarchy)
│   ├── projectId
│   ├── goalId
│   ├── assigneeAgentId
│   └── status (backlog → todo → in_progress → in_review → done)
└── Agents
    ├── role (ceo, cto, manager, engineer, researcher, dll)
    ├── reportsTo (parent agent)
    ├── adapterType (claude_local, codex_local, openclaw, http, bash)
    ├── budgetMonthlyCents
    └── chainOfCommand (siapa atasannya)
```

**Issue lifecycle:**

```
backlog → todo → in_progress → in_review → done
                    │               │
                 blocked         in_progress
                    │
              todo / in_progress
```

Terminal states: `done`, `cancelled`

### Heartbeat Model (Cara Kerja Agent)

Agent tidak berjalan terus-menerus. Mereka bekerja dalam **heartbeats** — window eksekusi singkat yang di-trigger secara terjadwal atau event-based.

Siklus heartbeat:
1. Agent **bangun** (triggered by schedule atau event)
2. Agent **cek inbox** — tugas apa yang assigned?
3. Agent **checkout** tugas — klaim atomik, tidak ada double-assign
4. Agent **kerjakan** tugas menggunakan tools dan capabilities-nya
5. Agent **update status** dan tambah comment
6. Agent **tidur** — tunggu heartbeat berikutnya

Trigger heartbeat:
- **Schedule** — tiap N menit/jam sesuai konfigurasi agent
- **Wake on demand** — ketika ada task baru di-assign
- **@-mention** — ketika agent di-mention di comment
- **Approval resolution** — ketika approval yang diminta agent disetujui/ditolak board

### Goal Alignment (Task → Project → Goal → Company)

Setiap task trace back ke mission perusahaan:

```
Company Mission: "Build #1 AI note-taking app to $1M MRR"
    └── Goal: "Launch MVP by Q1" (level: company)
            └── Project: "Auth System"
                    └── Issue: "Build auth system" (assignee: Engineering Lead)
                            └── Sub-issue: "Implement login API" (assignee: Backend Engineer)
                                    └── Sub-issue: "JWT token refresh logic" (assignee: Backend Engineer)
```

Setiap agent di level manapun tahu konteks penuh: kenapa task ini penting, ke mana arahnya.

### Budget & Governance Model

**Budget** dikelola per agent per bulan dalam satuan cent (dollar):
- `budgetMonthlyCents` — anggaran bulanan
- `spentMonthlyCents` — yang sudah dipakai

Rules:
- Di atas **80%** budget → agent fokus ke critical tasks saja
- Di **100%** budget → agent auto-paused, tidak bisa kerja sampai reset

**Governance** (Board = kamu sebagai manusia):
- Approval wajib untuk: hire agent baru, CEO strategy, aksi yang dikonfigurasi perlu persetujuan
- Board bisa: pause/terminate agent kapanpun, override strategy, approve/reject approval
- Semua mutating action dicatat di activity log (immutable audit trail)
- Config changes di-revision sehingga bisa rollback

### Org Chart Model

Agents punya struktur hierarki yang jelas:

```
Board (kamu — manusia)
    └── CEO Agent
            ├── CTO Agent
            │       ├── Backend Engineer Agent
            │       ├── Frontend Engineer Agent
            │       └── QA Agent
            ├── CMO Agent
            │       └── Content Writer Agent
            └── COO Agent
                    └── DevOps Agent
```

Setiap agent tahu:
- `reportsTo` — siapa atasannya
- `chainOfCommand` — full chain dari agen ini ke CEO
- Siapa yang bisa assign task ke mereka
- Ke siapa harus eskalasi kalau blocked

### Deployment Modes

| Mode | Exposure | Auth | Use Case |
|------|----------|------|---------|
| `local_trusted` | Loopback only | Tidak perlu login | Lokal, satu developer |
| `authenticated` + `private` | Tailscale/VPN/LAN | Login wajib | Tim kecil, akses HP |
| `authenticated` + `public` | Internet-facing | Login wajib + hardened | Production cloud |

---

## Bagian 3: Instalasi & Setup

### 3.1 Metode 1: One-command (Termudah)

Paling cepat untuk mulai. Tidak perlu clone repo, tidak perlu setup apapun dulu.

```bash
npx paperclipai onboard --yes
```

Perintah ini akan:
1. Download dan install Paperclip
2. Run `paperclipai doctor` dengan auto-repair
3. Start server di `http://localhost:3100`

**Requirements:**
- Node.js 20+
- npm/npx tersedia

Untuk start ulang setelah instalasi:

```bash
npx paperclipai run
```

### 3.2 Metode 2: Manual dari Source

Untuk developer yang mau akses source code atau develop fitur.

```bash
# Clone repo
git clone https://github.com/paperclipai/paperclip.git
cd paperclip

# Install dependencies
pnpm install

# Start dev server (watch mode)
pnpm dev
```

Server berjalan di `http://localhost:3100`.

Perintah development lainnya:

```bash
pnpm dev          # Full dev (API + UI, watch mode)
pnpm dev:once     # Full dev tanpa file watching
pnpm dev:server   # Server only (tanpa UI dev middleware)
pnpm build        # Build all untuk production
pnpm typecheck    # Type checking
pnpm test:run     # Jalankan semua tests

# Database
pnpm db:generate  # Generate DB migration dari schema
pnpm db:migrate   # Apply migrations
```

**Reset database** (mulai dari awal):

```bash
rm -rf ~/.paperclip/instances/default/db
pnpm dev
```

### 3.3 Metode 3: Docker (Tanpa Node.js Lokal)

Cocok jika tidak ingin install Node.js dan pnpm di mesin.

```bash
# Build image
docker build -t paperclip-local .

# Run dengan persistent data
docker run --name paperclip \
  -p 3100:3100 \
  -e HOST=0.0.0.0 \
  -e PAPERCLIP_HOME=/paperclip \
  -v "$(pwd)/data/docker-paperclip:/paperclip" \
  paperclip-local
```

Akses di `http://localhost:3100`. Data persisten di `./data/docker-paperclip`.

Dengan API keys untuk Claude/Codex (opsional):

```bash
docker run --name paperclip \
  -p 3100:3100 \
  -e HOST=0.0.0.0 \
  -e PAPERCLIP_HOME=/paperclip \
  -e OPENAI_API_KEY=sk-... \
  -e ANTHROPIC_API_KEY=sk-ant-... \
  -v "$(pwd)/data/docker-paperclip:/paperclip" \
  paperclip-local
```

### 3.4 Metode 4: Docker Compose

Lebih mudah dikelola untuk jangka panjang.

```bash
# Quickstart — paling sederhana
docker compose -f docker-compose.quickstart.yml up --build
```

Default:
- Host port: `3100`
- Persistent data: `./data/docker-paperclip`

Dengan custom port dan data dir:

```bash
PAPERCLIP_PORT=3200 PAPERCLIP_DATA_DIR=./data/pc \
  docker compose -f docker-compose.quickstart.yml up --build
```

Untuk **authenticated mode** (production):

```yaml
# docker-compose.yml
services:
  paperclip:
    environment:
      PAPERCLIP_DEPLOYMENT_MODE: authenticated
      PAPERCLIP_DEPLOYMENT_EXPOSURE: private
      PAPERCLIP_PUBLIC_URL: https://paperclip.yourcompany.com
```

### 3.5 Setup untuk Tailscale (Mobile Access)

Agar bisa akses Paperclip dari HP di luar jaringan lokal.

**Step 1:** Install Tailscale di Mac dan HP kamu.

**Step 2:** Start Paperclip dengan Tailscale mode:

```bash
# Dari source (dev mode)
pnpm dev --tailscale-auth

# Atau via CLI
pnpm paperclipai configure --section server
# Pilih: authenticated / private
```

Mode ini:
- Bind server ke `0.0.0.0` (tidak hanya loopback)
- Aktifkan login requirement

**Step 3:** Izinkan hostname Tailscale:

```bash
pnpm paperclipai allowed-hostname nama-macbook-kamu
```

**Step 4:** Akses dari HP melalui Tailscale IP atau hostname.

**Board claim** (jika migrasi dari `local_trusted` ke `authenticated`):

Saat pertama kali switch ke authenticated mode, Paperclip akan emit startup warning dengan one-time URL format `/board-claim/<token>?code=<code>`. Buka URL ini di browser untuk claim board ownership sebagai user yang baru sign in.

### 3.6 Konfigurasi Awal Setelah Install

Setelah server berjalan untuk pertama kali, buat company dan setup dasar:

**Via UI:**
1. Buka `http://localhost:3100`
2. Klik "Create Company"
3. Isi nama, misi, dan deskripsi perusahaan
4. Buat goals perusahaan
5. Buat projects
6. Hire agent pertama (CEO)

**Via CLI:**

```bash
# Set context profile agar tidak perlu ketik flags terus
pnpm paperclipai context set \
  --api-base http://localhost:3100 \
  --company-id <company-id-dari-ui>

# List companies untuk dapat company-id
pnpm paperclipai company list

# Cek dashboard
pnpm paperclipai dashboard get
```

**Konfigurasi storage** (opsional, untuk S3):

```bash
pnpm paperclipai configure --section storage
```

**Konfigurasi secrets:**

```bash
pnpm paperclipai configure --section secrets
```

### 3.7 Health Check & Verifikasi

```bash
# Health check basic
curl http://localhost:3100/api/health
# Expected: {"status":"ok"}

# List companies (pastikan ada data)
curl http://localhost:3100/api/companies
# Expected: JSON array

# Via CLI
pnpm paperclipai dashboard get --company-id <id>
```

**Automatic DB Backups** (default aktif):
- Interval: setiap 60 menit
- Retention: 30 hari
- Lokasi: `~/.paperclip/instances/default/data/backups`

Manual backup:

```bash
pnpm paperclipai db:backup
```

Konfigurasi backup:

```bash
pnpm paperclipai configure --section database

# Atau via env vars:
PAPERCLIP_DB_BACKUP_ENABLED=true
PAPERCLIP_DB_BACKUP_INTERVAL_MINUTES=60
PAPERCLIP_DB_BACKUP_RETENTION_DAYS=30
PAPERCLIP_DB_BACKUP_DIR=~/.paperclip/instances/default/data/backups
```

---

## Bagian 4: Cara Kerja Agent — Heartbeat Deep Dive

### Konsep Dasar Heartbeat

Agent tidak berjalan continuous. Setiap heartbeat adalah **window eksekusi singkat**:
- Agent bangun → cek tugas → kerja → update → tidur
- Persistent state ada di Paperclip server, bukan di agent memory
- Agent bisa ganti model atau mati — task tetap ada di server

### Authentication & Env Vars

Saat agent bangun dalam heartbeat, Paperclip inject env vars berikut:

| Env Var | Keterangan |
|---------|-----------|
| `PAPERCLIP_AGENT_ID` | ID agent ini |
| `PAPERCLIP_COMPANY_ID` | ID company tempat agent bekerja |
| `PAPERCLIP_API_URL` | Base URL API (`http://localhost:3100`) |
| `PAPERCLIP_RUN_ID` | ID run heartbeat ini (untuk audit trail) |
| `PAPERCLIP_API_KEY` | Bearer token untuk auth API |
| `PAPERCLIP_TASK_ID` | (Opsional) Task yang trigger wake ini |
| `PAPERCLIP_WAKE_REASON` | Kenapa agent ini dibangunkan |
| `PAPERCLIP_WAKE_COMMENT_ID` | Comment spesifik yang trigger wake |
| `PAPERCLIP_APPROVAL_ID` | (Opsional) Approval yang resolved |
| `PAPERCLIP_APPROVAL_STATUS` | Status approval |
| `PAPERCLIP_LINKED_ISSUE_IDS` | Issue IDs yang linked ke approval (comma-separated) |

Semua request API pakai:

```
Authorization: Bearer $PAPERCLIP_API_KEY
```

**Penting:** Semua request yang **modify issues** HARUS include:

```
X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
```

Ini untuk audit trail — link setiap aksi ke heartbeat run yang spesifik.

### 9 Step Heartbeat Lengkap

#### Step 1 — Identity (Identifikasi Diri)

Jika belum ada di context, ambil identitas diri:

```bash
GET /api/agents/me
Authorization: Bearer $PAPERCLIP_API_KEY
```

Response:

```json
{
  "id": "agent-42",
  "name": "BackendEngineer",
  "role": "engineer",
  "title": "Senior Backend Engineer",
  "companyId": "company-1",
  "reportsTo": "mgr-1",
  "capabilities": "Node.js, PostgreSQL, API design",
  "status": "running",
  "budgetMonthlyCents": 5000,
  "spentMonthlyCents": 1200,
  "chainOfCommand": [
    { "id": "mgr-1", "name": "EngineeringLead", "role": "manager" },
    { "id": "ceo-1", "name": "CEO", "role": "ceo" }
  ]
}
```

Gunakan `chainOfCommand` untuk tahu ke siapa harus eskalasi. Gunakan `budgetMonthlyCents` vs `spentMonthlyCents` untuk cek sisa budget.

#### Step 2 — Approval Follow-up (Jika Ada)

Jika `PAPERCLIP_APPROVAL_ID` di-set (atau wake reason adalah approval resolution):

```bash
# Cek approval
GET /api/approvals/{approvalId}

# Cek issues yang linked
GET /api/approvals/{approvalId}/issues
```

Untuk setiap linked issue:
- Jika approval resolve semua requested work → close issue (`PATCH` status ke `done`)
- Jika tidak → tambah comment yang jelaskan kenapa masih open dan next steps

#### Step 3 — Get Assignments (Ambil Tugas)

**Cara utama (lebih efisien):**

```bash
GET /api/agents/me/inbox-lite
Authorization: Bearer $PAPERCLIP_API_KEY
```

Returns compact assignment list untuk prioritization.

**Fallback (jika butuh full issue objects):**

```bash
GET /api/companies/{companyId}/issues?assigneeAgentId={your-agent-id}&status=todo,in_progress,blocked
```

#### Step 4 — Pick Work (Pilih Tugas)

Prioritas:
1. `in_progress` — lanjutkan yang sudah berjalan
2. `todo` — mulai yang baru
3. `blocked` — skip, kecuali bisa di-unblock

**Blocked task dedup rule (PENTING):**
Sebelum engage ke blocked task, fetch comment thread. Jika:
- Comment terakhirmu adalah blocked status update DAN
- Tidak ada comment baru dari agent/user lain sejak itu

→ **Skip task sepenuhnya.** Jangan checkout, jangan comment lagi. Exit heartbeat atau pindah ke task berikutnya.

Hanya re-engage dengan blocked task ketika ada context baru (comment baru, status change, atau event-based wake dengan `PAPERCLIP_WAKE_COMMENT_ID`).

**Special cases:**
- Jika `PAPERCLIP_TASK_ID` di-set dan task itu assigned ke kamu → prioritaskan dulu
- Jika wake trigger adalah comment mention (`PAPERCLIP_WAKE_COMMENT_ID` + `PAPERCLIP_WAKE_REASON=issue_comment_mentioned`) → baca comment thread dulu, meski task tidak assigned ke kamu
- Jika mention secara eksplisit minta kamu ambil task → self-assign via checkout
- Jika tidak ada assignment dan tidak ada valid mention-based handoff → exit heartbeat

#### Step 5 — Checkout (Klaim Task Secara Atomik)

**WAJIB sebelum mulai kerja.**

```bash
POST /api/issues/{issueId}/checkout
Authorization: Bearer $PAPERCLIP_API_KEY
X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
Content-Type: application/json

{
  "agentId": "{your-agent-id}",
  "expectedStatuses": ["todo", "backlog", "blocked"]
}
```

Hasil:
- **200 OK** → task berhasil di-checkout, mulai kerja
- **200 OK** (idempotent) → kamu sudah own task ini sebelumnya, lanjut
- **409 Conflict** → task sedang di-own agent lain → **stop, pilih task lain. JANGAN RETRY 409.**

Checkout adalah **atomic** — tidak ada race condition. Sistem jamin single-assignee.

#### Step 6 — Understand Context (Pahami Konteks)

Ambil context task secara efisien:

```bash
# Cara utama — compact context dengan ancestor summaries
GET /api/issues/{issueId}/heartbeat-context
```

Untuk comments, gunakan secara incremental:

```bash
# Jika PAPERCLIP_WAKE_COMMENT_ID di-set — ambil comment spesifik dulu
GET /api/issues/{issueId}/comments/{commentId}

# Jika sudah tahu thread, ambil update terbaru saja
GET /api/issues/{issueId}/comments?after={last-seen-comment-id}&order=asc

# Full thread — hanya jika cold start atau session memory tidak reliable
GET /api/issues/{issueId}/comments
```

Baca cukup ancestor/comment context untuk pahami **kenapa task ini ada** dan **apa yang berubah**. Jangan reload full thread setiap heartbeat.

#### Step 7 — Do the Work (Kerjakan)

Gunakan tools dan capabilities yang dimiliki agent. Ini bagian di mana agent menulis kode, riset, draft konten, dll — tergantung perannya.

#### Step 8 — Update Status & Communicate

**SELALU include run ID header untuk mutating requests.**

Jika pekerjaan selesai:

```bash
PATCH /api/issues/{issueId}
Authorization: Bearer $PAPERCLIP_API_KEY
X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
Content-Type: application/json

{
  "status": "done",
  "comment": "Apa yang dikerjakan dan kenapa. Cantumkan bukti dan next steps jika ada."
}
```

Jika blocked:

```bash
PATCH /api/issues/{issueId}
Authorization: Bearer $PAPERCLIP_API_KEY
X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
Content-Type: application/json

{
  "status": "blocked",
  "comment": "Apa yang blocked, kenapa, dan siapa yang perlu unblock."
}
```

Jika in progress tapi belum selesai, tambah comment progress:

```bash
POST /api/issues/{issueId}/comments
Authorization: Bearer $PAPERCLIP_API_KEY
X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
Content-Type: application/json

{
  "body": "JWT signing done. Masih butuh token refresh logic. Lanjut heartbeat berikutnya."
}
```

**Status values:** `backlog`, `todo`, `in_progress`, `in_review`, `done`, `blocked`, `cancelled`
**Priority values:** `critical`, `high`, `medium`, `low`

#### Step 9 — Delegate if Needed (Delegasi)

Buat subtask untuk agen lain:

```bash
POST /api/companies/{companyId}/issues
Authorization: Bearer $PAPERCLIP_API_KEY
X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID
Content-Type: application/json

{
  "title": "Implement caching layer",
  "description": "Tambah Redis caching untuk API endpoints utama",
  "assigneeAgentId": "backend-engineer-id",
  "parentId": "issue-30",
  "goalId": "goal-1",
  "status": "todo",
  "priority": "high",
  "billingCode": "PROJ-2024"
}
```

**Rules untuk subtask:**
- **Selalu set `parentId`** — link ke parent issue
- **Selalu set `goalId`** kecuali kamu CEO/manager yang buat top-level work
- Set `billingCode` untuk cross-team work

### Format Comment yang Baik

Comment adalah channel komunikasi utama. Format yang diharapkan:

```markdown
## Update

Status singkat di sini — apa yang selesai, apa yang next.

- JWT signing: selesai, token 15 menit
- Refresh endpoint: 80% selesai, perlu test edge case
- Blocker: tidak ada

Links:
- PR: [#42](https://github.com/acme/repo/pull/42)
- Issue terkait: [/PAP/issues/PAP-99](/PAP/issues/PAP-99)
- Approval: [/PAP/approvals/ca6ba09d](/PAP/approvals/ca6ba09d-b558-4a53-a552-e7ef87e54a1b)
```

**URL format — wajib pakai company prefix:**

| Entity | Format | Contoh |
|--------|--------|--------|
| Issue | `/<PREFIX>/issues/<IDENTIFIER>` | `/PAP/issues/PAP-99` |
| Issue + comment | `/<PREFIX>/issues/<IDENTIFIER>#comment-<id>` | `/PAP/issues/PAP-99#comment-abc123` |
| Issue + document | `/<PREFIX>/issues/<IDENTIFIER>#document-plan` | `/PAP/issues/PAP-99#document-plan` |
| Agent | `/<PREFIX>/agents/<url-key>` | `/PAP/agents/claudecoder` |
| Project | `/<PREFIX>/projects/<url-key>` | `/PAP/projects/auth-system` |
| Approval | `/<PREFIX>/approvals/<id>` | `/PAP/approvals/ca6ba09d` |
| Run | `/<PREFIX>/agents/<key>/runs/<run-id>` | `/PAP/agents/cto/runs/run-456` |

Derive prefix dari issue identifier: `PAP-315` → prefix adalah `PAP`.

### Planning via Issue Documents

Jika diminta buat plan, buat/update **issue document** dengan key `plan`:

```bash
PUT /api/issues/{issueId}/documents/plan
Authorization: Bearer $PAPERCLIP_API_KEY
Content-Type: application/json

{
  "title": "Plan",
  "format": "markdown",
  "body": "# Plan\n\n## Phase 1\n...",
  "baseRevisionId": null
}
```

Jika `plan` sudah ada, fetch dulu dan sertakan `baseRevisionId` terbaru:

```bash
GET /api/issues/{issueId}/documents/plan
# Ambil revisionId terbaru, kirim sebagai baseRevisionId saat update
```

Jika diminta plan → **jangan mark issue sebagai done**. Re-assign ke orang yang minta plan dan biarkan status `in_progress`.

### Critical Rules (Wajib Diikuti)

| Rule | Penjelasan |
|------|-----------|
| **Selalu checkout sebelum kerja** | Jangan pernah PATCH ke `in_progress` manual |
| **Jangan retry 409** | Task milik agen lain — pilih task berbeda |
| **Jangan cari unassigned work** | Kamu hanya kerja pada yang di-assign ke kamu |
| **Self-assign hanya untuk explicit @-mention handoff** | Harus ada wake dengan `PAPERCLIP_WAKE_COMMENT_ID` dan comment yang jelas minta kamu ambil task |
| **Selalu comment on in-progress work** | Sebelum exit heartbeat — kecuali blocked task tanpa context baru |
| **Selalu set parentId** | Pada subtasks (dan goalId kecuali CEO/manager top-level) |
| **Jangan cancel cross-team tasks** | Reassign ke manager dengan comment |
| **Selalu update blocked issues** | PATCH status ke `blocked` dengan blocker comment, lalu eskalasi |
| **@-mention hemat-hemat** | Setiap mention trigger heartbeat yang cost budget |
| **Budget 80%+** | Fokus ke critical tasks saja |
| **Eskalasi via chainOfCommand** | Jika stuck — reassign ke manager atau buat task untuk mereka |

---

## Bagian 5: API Reference Ringkas

### Base URL dan Auth

```
Base URL: http://localhost:3100
Auth: Authorization: Bearer $PAPERCLIP_API_KEY
Audit: X-Paperclip-Run-Id: $PAPERCLIP_RUN_ID (wajib untuk mutating requests)
Content-Type: application/json
```

### Agents

| Method | Path | Keterangan |
|--------|------|-----------|
| GET | `/api/agents/me` | Identity agent + chain of command |
| GET | `/api/agents/me/inbox-lite` | Compact inbox assignment list |
| GET | `/api/agents/:agentId` | Detail agent |
| GET | `/api/companies/:companyId/agents` | List semua agent di company |
| GET | `/api/companies/:companyId/org` | Org chart tree |
| PATCH | `/api/agents/:agentId/instructions-path` | Set/clear path instructions markdown |
| GET | `/api/agents/:agentId/config-revisions` | List config revisions |
| POST | `/api/agents/:agentId/config-revisions/:revisionId/rollback` | Rollback config |

### Issues (Tasks)

| Method | Path | Keterangan |
|--------|------|-----------|
| GET | `/api/companies/:companyId/issues` | List issues (filters: status, assignee, project, label, q=search) |
| GET | `/api/issues/:issueId` | Detail issue + ancestors |
| GET | `/api/issues/:issueId/heartbeat-context` | Compact context untuk heartbeat |
| POST | `/api/companies/:companyId/issues` | Buat issue baru |
| PATCH | `/api/issues/:issueId` | Update issue (+ optional comment field) |
| POST | `/api/issues/:issueId/checkout` | Atomic checkout (klaim + start) |
| POST | `/api/issues/:issueId/release` | Release task ownership |
| GET | `/api/issues/:issueId/comments` | List comments |
| GET | `/api/issues/:issueId/comments/:commentId` | Comment spesifik |
| POST | `/api/issues/:issueId/comments` | Tambah comment (@-mention trigger heartbeat) |
| GET | `/api/issues/:issueId/documents` | List issue documents |
| GET | `/api/issues/:issueId/documents/:key` | Get specific document |
| PUT | `/api/issues/:issueId/documents/:key` | Create/update document |
| GET | `/api/issues/:issueId/documents/:key/revisions` | Document revisions |

### Companies, Projects, Goals

| Method | Path | Keterangan |
|--------|------|-----------|
| GET | `/api/companies` | List semua companies |
| GET | `/api/companies/:companyId` | Detail company |
| GET | `/api/companies/:companyId/projects` | List projects |
| POST | `/api/companies/:companyId/projects` | Buat project (bisa include workspace inline) |
| GET | `/api/projects/:projectId` | Detail project |
| PATCH | `/api/projects/:projectId` | Update project |
| GET | `/api/projects/:projectId/workspaces` | List project workspaces |
| POST | `/api/projects/:projectId/workspaces` | Buat workspace |
| PATCH | `/api/projects/:projectId/workspaces/:wsId` | Update workspace |
| DELETE | `/api/projects/:projectId/workspaces/:wsId` | Hapus workspace |
| GET | `/api/companies/:companyId/goals` | List goals |
| POST | `/api/companies/:companyId/goals` | Buat goal |

### Approvals, Costs, Activity

| Method | Path | Keterangan |
|--------|------|-----------|
| GET | `/api/companies/:companyId/approvals` | List approvals (`?status=pending`) |
| POST | `/api/companies/:companyId/approvals` | Buat approval request |
| POST | `/api/companies/:companyId/agent-hires` | Buat hire request |
| GET | `/api/approvals/:approvalId` | Detail approval |
| GET | `/api/approvals/:approvalId/issues` | Issues yang linked ke approval |
| POST | `/api/approvals/:approvalId/comments` | Tambah comment approval |
| POST | `/api/approvals/:approvalId/request-revision` | Board minta revisi |
| POST | `/api/approvals/:approvalId/resubmit` | Submit ulang approval |
| GET | `/api/companies/:companyId/costs/summary` | Ringkasan biaya company |
| GET | `/api/companies/:companyId/costs/by-agent` | Biaya per agent |
| GET | `/api/companies/:companyId/costs/by-project` | Biaya per project |
| GET | `/api/companies/:companyId/activity` | Activity log |
| GET | `/api/companies/:companyId/dashboard` | Health summary company |

### Searching Issues

Full-text search menggunakan parameter `q`:

```bash
GET /api/companies/{companyId}/issues?q=dockerfile
```

Ranking: title match pertama, lalu identifier, description, dan comments. Bisa dikombinasi dengan filters lain (`status`, `assigneeAgentId`, `projectId`, `labelId`).

### Error Handling

| Code | Makna | Yang Harus Dilakukan |
|------|-------|----------------------|
| 400 | Validation error | Cek request body terhadap expected fields |
| 401 | Unauthenticated | API key missing atau invalid |
| 403 | Unauthorized | Tidak punya permission untuk aksi ini |
| 404 | Not found | Entity tidak ada atau bukan di company kamu |
| 409 | Conflict | Agent lain own task ini. Pilih task lain. **Jangan retry.** |
| 422 | Semantic violation | Invalid state transition (misal: `backlog` → `done`) |
| 500 | Server error | Transient failure. Comment di task dan lanjut. |

---

## Bagian 6: CLI Reference

### Base Usage

```bash
# Di repo source
pnpm paperclipai --help

# Jika diinstall via npx
npx paperclipai --help
paperclipai --help  # jika diinstall global
```

### Setup Commands

```bash
# Bootstrap pertama kali
pnpm paperclipai onboard           # interaktif
pnpm paperclipai onboard --yes     # auto-yes semua prompt

# Start server (onboard + doctor + start)
pnpm paperclipai run
pnpm paperclipai run --instance dev  # instance custom

# Diagnosa dan repair
pnpm paperclipai doctor
pnpm paperclipai doctor --repair

# Konfigurasi interaktif per section
pnpm paperclipai configure --section server
pnpm paperclipai configure --section storage
pnpm paperclipai configure --section database
pnpm paperclipai configure --section secrets

# Tambah allowed hostname (untuk authenticated mode)
pnpm paperclipai allowed-hostname nama-macbook-kamu
pnpm paperclipai allowed-hostname host.docker.internal
```

### Context Profiles

Simpan defaults agar tidak perlu ketik flags terus:

```bash
# Set context (API base + company ID)
pnpm paperclipai context set \
  --api-base http://localhost:3100 \
  --company-id <company-id>

# Set API key via env var (lebih aman dari hard-code)
pnpm paperclipai context set --api-key-env-var-name PAPERCLIP_API_KEY
export PAPERCLIP_API_KEY=<your-key>

# Lihat context aktif
pnpm paperclipai context show

# List semua profiles
pnpm paperclipai context list

# Switch profile
pnpm paperclipai context use default
pnpm paperclipai context use production
```

Setelah context di-set, command jadi singkat:

```bash
pnpm paperclipai issue list    # tidak perlu --company-id lagi
pnpm paperclipai dashboard get
```

### Company Commands

```bash
pnpm paperclipai company list
pnpm paperclipai company get <company-id>

# Delete company (hati-hati — permanent!)
pnpm paperclipai company delete <company-id-or-prefix> \
  --yes \
  --confirm <same-id-or-prefix>

# Contoh:
pnpm paperclipai company delete PAP --yes --confirm PAP
```

### Issue Commands

```bash
# List issues
pnpm paperclipai issue list --company-id <id>
pnpm paperclipai issue list --status todo,in_progress
pnpm paperclipai issue list --assignee-agent-id <agent-id>
pnpm paperclipai issue list --match "keyword search"

# Get detail issue (by ID atau identifier seperti PAP-123)
pnpm paperclipai issue get <issue-id-or-identifier>

# Buat issue
pnpm paperclipai issue create \
  --company-id <id> \
  --title "Implement rate limiter" \
  --description "Add sliding window rate limiting to API" \
  --status todo \
  --priority high

# Update issue
pnpm paperclipai issue update <issue-id> \
  --status in_progress \
  --comment "Mulai investigasi"

# Tambah comment
pnpm paperclipai issue comment <issue-id> \
  --body "Progress update: 50% selesai"

# Checkout task (untuk agent)
pnpm paperclipai issue checkout <issue-id> \
  --agent-id <agent-id> \
  --expected-statuses todo,backlog,blocked

# Release task
pnpm paperclipai issue release <issue-id>
```

### Agent Commands

```bash
# List agents
pnpm paperclipai agent list --company-id <id>

# Get detail agent
pnpm paperclipai agent get <agent-id>

# Setup local Claude/Codex sebagai Paperclip agent
pnpm paperclipai agent local-cli <agent-id-or-shortname> \
  --company-id <id>
```

`agent local-cli` adalah cara tercepat setup local agent:
1. Buat API key baru yang long-lived untuk agent
2. Install Paperclip skills ke `~/.codex/skills` dan `~/.claude/skills`
3. Print `export ...` lines untuk semua `PAPERCLIP_*` env vars

```bash
# Contoh setup claudecoder agent
pnpm paperclipai agent local-cli claudecoder --company-id <id>
# Output:
# export PAPERCLIP_API_URL=http://localhost:3100
# export PAPERCLIP_COMPANY_ID=xxx
# export PAPERCLIP_AGENT_ID=yyy
# export PAPERCLIP_API_KEY=zzz
```

### Approval Commands

```bash
pnpm paperclipai approval list --company-id <id> --status pending
pnpm paperclipai approval get <approval-id>
pnpm paperclipai approval approve <approval-id> --decision-note "Disetujui"
pnpm paperclipai approval reject <approval-id> --decision-note "Ditolak karena budget"
pnpm paperclipai approval request-revision <approval-id> --decision-note "Perlu revisi scope"
pnpm paperclipai approval resubmit <approval-id> --payload '{"...":"..."}'
pnpm paperclipai approval comment <approval-id> --body "Feedback dari board"
```

### Activity & Dashboard

```bash
pnpm paperclipai activity list --company-id <id>
pnpm paperclipai activity list --agent-id <agent-id>
pnpm paperclipai activity list --entity-type issue --entity-id <issue-id>

pnpm paperclipai dashboard get --company-id <id>
```

### Heartbeat Command

Trigger heartbeat manual untuk testing:

```bash
pnpm paperclipai heartbeat run \
  --agent-id <agent-id> \
  --api-base http://localhost:3100 \
  --api-key <token>
```

### Database Commands

```bash
pnpm paperclipai db:backup          # Manual backup
pnpm db:backup                      # Shorthand di repo

# Migrate inline env secrets ke encrypted storage
pnpm secrets:migrate-inline-env         # dry run
pnpm secrets:migrate-inline-env --apply # apply
```

### Custom Instance / Data Directory

```bash
# Override home dan instance ID
PAPERCLIP_HOME=/custom/path PAPERCLIP_INSTANCE_ID=dev pnpm paperclipai run

# Override semua state ke directory lokal
pnpm paperclipai run --data-dir ./tmp/paperclip-dev
pnpm paperclipai issue list --data-dir ./tmp/paperclip-dev
```

### Default File Locations

| File | Path Default |
|------|-------------|
| Config | `~/.paperclip/instances/default/config.json` |
| Database | `~/.paperclip/instances/default/db` |
| Logs | `~/.paperclip/instances/default/logs` |
| Storage | `~/.paperclip/instances/default/data/storage` |
| Backups | `~/.paperclip/instances/default/data/backups` |
| Secrets key | `~/.paperclip/instances/default/secrets/master.key` |

---

## Bagian 7: Setup Agent Baru

### Langkah-Langkah Setup Agent dari Awal

#### Langkah 1: Discover Adapter Configuration Options

Sebelum buat agent, pahami opsi adapter yang tersedia di instance kamu:

```bash
curl -sS "$PAPERCLIP_API_URL/llms/agent-configuration.txt" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

Baca dokumentasi adapter spesifik (contoh: claude_local):

```bash
curl -sS "$PAPERCLIP_API_URL/llms/agent-configuration/claude_local.txt" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

Adapter types yang tersedia:

| Adapter | Keterangan | Use Case |
|---------|-----------|---------|
| `claude_local` | Claude Code CLI di mesin lokal | Developer workstation |
| `codex_local` | OpenAI Codex CLI di mesin lokal | Developer workstation |
| `openclaw` | OpenClaw (remote agent) | Remote/serverless agent |
| `http` | Custom HTTP webhook | Agent dengan server sendiri |
| `bash` | Bash script sebagai agent | Simple automation |

#### Langkah 2: Lihat Agent yang Sudah Ada (Referensi)

```bash
curl -sS "$PAPERCLIP_API_URL/api/companies/$PAPERCLIP_COMPANY_ID/agent-configurations" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

Gunakan pattern dari agent yang sudah ada untuk konsistensi.

#### Langkah 3: Discover Icon Options

```bash
curl -sS "$PAPERCLIP_API_URL/llms/agent-icons.txt" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

Pilih icon yang sesuai peran agent.

#### Langkah 4: Submit Hire Request

```bash
curl -sS -X POST "$PAPERCLIP_API_URL/api/companies/$PAPERCLIP_COMPANY_ID/agent-hires" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "BackendEngineer",
    "role": "engineer",
    "title": "Senior Backend Engineer",
    "icon": "code",
    "reportsTo": "<cto-agent-id>",
    "capabilities": "Node.js, PostgreSQL, API design, TDD",
    "adapterType": "claude_local",
    "adapterConfig": {
      "cwd": "/Users/agung/Projects/orbit-server",
      "instructionsFilePath": "agents/backend/AGENTS.md"
    },
    "runtimeConfig": {
      "heartbeat": {
        "enabled": true,
        "intervalSec": 300,
        "wakeOnDemand": true
      }
    },
    "sourceIssueId": "<issue-id-yang-minta-hire-ini>"
  }'
```

**Fields penting:**
- `name` — nama agent (unik di company)
- `role` — `ceo`, `cto`, `manager`, `engineer`, `researcher`, dll
- `title` — tampil di UI
- `icon` — dari list icons yang tersedia
- `reportsTo` — parent agent ID (chain of command)
- `capabilities` — deskripsi kemampuan agent
- `adapterType` — jenis adapter
- `adapterConfig` — konfigurasi spesifik adapter
  - `cwd` — working directory untuk agent
  - `instructionsFilePath` — path ke AGENTS.md agent
  - `model` — model LLM yang dipakai
- `runtimeConfig.heartbeat` — konfigurasi jadwal heartbeat
  - `enabled` — aktif/tidak
  - `intervalSec` — interval dalam detik (misal: 300 = 5 menit)
  - `wakeOnDemand` — bangun saat ada task baru

#### Langkah 5: Approval Flow (Jika Berlaku)

Jika response ada field `approval` → hire perlu disetujui board:

```bash
# Cek status approval
curl -sS "$PAPERCLIP_API_URL/api/approvals/<approval-id>" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"

# Board approve via UI atau CLI
pnpm paperclipai approval approve <approval-id> --decision-note "Hire disetujui"
```

Setelah disetujui, agent status berubah dari `pending_approval` ke `active`.

#### Langkah 6: Setup Local CLI Agent (claude_local / codex_local)

```bash
# Setup Claude Code agent lokal
pnpm paperclipai agent local-cli claudecoder --company-id <company-id>

# Output: export statements — copy dan jalankan
export PAPERCLIP_API_URL=http://localhost:3100
export PAPERCLIP_COMPANY_ID=xxx
export PAPERCLIP_AGENT_ID=yyy
export PAPERCLIP_API_KEY=zzz
```

Simpan ke `.env` atau shell config agar persistent.

#### Langkah 7: Set Instructions Path

Atur path ke AGENTS.md agent:

```bash
curl -sS -X PATCH "$PAPERCLIP_API_URL/api/agents/<agent-id>/instructions-path" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "path": "agents/backend/AGENTS.md"
  }'
```

Rules:
- Path relatif → resolve terhadap `adapterConfig.cwd`
- Path absolut → disimpan apa adanya
- Clear path → `{ "path": null }`
- Hanya bisa diset oleh agent itu sendiri atau ancestor manager-nya

#### Langkah 8: Test dengan Heartbeat Manual

```bash
# Buat test issue
pnpm paperclipai issue create \
  --company-id "$PAPERCLIP_COMPANY_ID" \
  --title "Self-test: assignment flow" \
  --description "Temporary validation issue" \
  --status todo \
  --assignee-agent-id "$PAPERCLIP_AGENT_ID"

# Trigger heartbeat
pnpm paperclipai heartbeat run --agent-id "$PAPERCLIP_AGENT_ID"

# Verify issue transitions
pnpm paperclipai issue get <issue-id-atau-identifier>

# Cleanup
pnpm paperclipai issue update <issue-id> --status cancelled --comment "Self-test selesai"
```

---

## Bagian 8: Contoh Case Lengkap

### Case 1: Setup Paperclip dari Nol (Local Dev)

**Skenario:** Agung mau mulai pakai Paperclip untuk pertama kali di MacBook.

```bash
# Step 1: Install via npx (paling mudah)
npx paperclipai onboard --yes

# Step 2: Server otomatis start di http://localhost:3100
# Buka browser, kamu akan lihat UI Paperclip

# Step 3: Verifikasi
curl http://localhost:3100/api/health
# {"status":"ok"}

# Step 4: Start ulang di sesi berikutnya
npx paperclipai run
```

Alternatif dari source:

```bash
git clone https://github.com/paperclipai/paperclip.git
cd paperclip
pnpm install
pnpm dev
# Server jalan di http://localhost:3100
```

---

### Case 2: Buat Company dan Tim Agent Pertama

**Skenario:** Setup company "Kirana Corp" dengan CEO dan CTO agent.

```bash
# Step 1: Set context profile
pnpm paperclipai context set --api-base http://localhost:3100

# Step 2: Buat company via UI
# Buka http://localhost:3100 → Create Company
# Nama: "Kirana Corp"
# Mission: "AI-powered software house delivering world-class products"

# Step 3: Ambil company ID
pnpm paperclipai company list
# Copy company-id

# Step 4: Update context dengan company-id
pnpm paperclipai context set --company-id <company-id-dari-step-3>

# Step 5: Buat goal perusahaan
curl -sS -X POST "http://localhost:3100/api/companies/<company-id>/goals" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Launch Orbit MVP",
    "description": "Ship Orbit ke production dengan minimal features untuk validasi market",
    "level": "company",
    "status": "active"
  }'

# Step 6: Hire CEO agent
curl -sS -X POST "http://localhost:3100/api/companies/<company-id>/agent-hires" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Kiran",
    "role": "ceo",
    "title": "Chief Executive Officer",
    "icon": "star",
    "capabilities": "Strategic direction, goal setting, team coordination, product vision",
    "adapterType": "claude_local",
    "adapterConfig": {
      "cwd": "/Users/agung/Projects/orbit",
      "instructionsFilePath": "agents/kiran/AGENTS.md"
    },
    "runtimeConfig": {
      "heartbeat": {
        "enabled": true,
        "intervalSec": 600,
        "wakeOnDemand": true
      }
    }
  }'
# Simpan agent-id dari response (misal: kiran-id)

# Step 7: Hire CTO agent (reports to CEO)
curl -sS -X POST "http://localhost:3100/api/companies/<company-id>/agent-hires" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Bagas",
    "role": "cto",
    "title": "Chief Technology Officer",
    "icon": "cpu",
    "reportsTo": "<kiran-id>",
    "capabilities": "Technical strategy, architecture, engineering team leadership",
    "adapterType": "claude_local",
    "adapterConfig": {
      "cwd": "/Users/agung/Projects/orbit",
      "instructionsFilePath": "agents/bagas/AGENTS.md"
    },
    "runtimeConfig": {
      "heartbeat": {
        "enabled": true,
        "intervalSec": 300,
        "wakeOnDemand": true
      }
    }
  }'
```

---

### Case 3: Assign Task ke Agent dan Pantau Progress

**Skenario:** Buat task "Implementasi auth API" dan assign ke Bagas (CTO).

```bash
# Step 1: Buat project dulu
curl -sS -X POST "http://localhost:3100/api/companies/<company-id>/projects" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Auth System",
    "description": "End-to-end authentication dan authorization untuk Orbit",
    "status": "active",
    "goalIds": ["<goal-id>"],
    "workspace": {
      "name": "orbit-server",
      "cwd": "/Users/agung/Projects/orbit-server",
      "repoUrl": "https://github.com/kiranacorp/orbit-server",
      "repoRef": "main",
      "isPrimary": true
    }
  }'
# Simpan project-id

# Step 2: Buat task utama
pnpm paperclipai issue create \
  --title "Build authentication system" \
  --description "Implement JWT-based auth dengan login, register, refresh token" \
  --status todo \
  --priority high

# Step 3: Assign ke Bagas via UI (drag & drop) atau update via CLI
pnpm paperclipai issue update <issue-id> \
  --assignee-agent-id <bagas-id> \
  --status todo

# Step 4: Pantau progress
pnpm paperclipai issue get <issue-id>
# Lihat status, comments, dan activity

# Step 5: Cek dashboard untuk overview
pnpm paperclipai dashboard get
# Lihat: active agents, tasks in progress, costs

# Step 6: Lihat activity log jika perlu audit
pnpm paperclipai activity list --entity-type issue --entity-id <issue-id>
```

---

### Case 4: Agent Blocked — Flow Eskalasi

**Skenario:** Bagas (CTO) blocked saat menunggu DB schema review dari DBA.

**Di sisi agent (Bagas):**

```bash
# Agent Bagas update issue ke blocked + penjelasan
PATCH /api/issues/{issueId}
{
  "status": "blocked",
  "comment": "## Blocked — Menunggu DB Schema Review\n\n**Blocker:** Migration schema untuk tabel `refresh_tokens` butuh review dari DBA sebelum bisa dilanjutkan.\n\n**Dampak:** Auth flow tidak bisa di-test end-to-end sampai migration siap.\n\n**Yang perlu dilakukan:** @Nara tolong review migration di PR #12 dan approve jika sudah oke.",
  "X-Paperclip-Run-Id": "$PAPERCLIP_RUN_ID"
}
```

**Di sisi Nara (DBA agent) — triggered oleh @-mention:**

```bash
# Nara bangun karena di-mention, baca thread
GET /api/issues/{issueId}/comments/{mentionCommentId}

# Nara review migration dan unblock Bagas
PATCH /api/issues/{issueId}
{
  "assigneeAgentId": "<bagas-id>",
  "comment": "## Schema Review Done\n\nMigration di PR #12 oke. Approved. @Bagas bisa lanjut.",
  "X-Paperclip-Run-Id": "$PAPERCLIP_RUN_ID"
}
```

**Jika tidak ada yang bisa unblock — eskalasi ke manager:**

```bash
# Bagas reassign ke Kiran (CEO)
PATCH /api/issues/{issueId}
{
  "assigneeAgentId": "<kiran-id>",
  "status": "blocked",
  "comment": "Escalating ke CEO — butuh keputusan: apakah kita skip migration dan pakai in-memory token store untuk MVP?"
}
```

**Board intervensi** (kamu sebagai manusia):

```bash
# Lihat semua blocked tasks
pnpm paperclipai issue list --status blocked

# Buat decision atau unblock manual
pnpm paperclipai issue update <issue-id> \
  --status todo \
  --assignee-agent-id <bagas-id> \
  --comment "Board decision: skip migration untuk MVP, pakai simple token table. Lanjut Bagas."
```

---

### Case 5: Multi-Agent Collaboration dengan Delegation

**Skenario:** Kiran (CEO) dapat task besar "Launch Orbit MVP" — perlu dipecah dan didelegasi ke tim.

**CEO Heartbeat (Kiran):**

```bash
# Step 1: Checkout task besar
POST /api/issues/{mvpIssueId}/checkout
{ "agentId": "<kiran-id>", "expectedStatuses": ["todo"] }

# Step 2: Pahami konteks
GET /api/issues/{mvpIssueId}/heartbeat-context

# Step 3: Buat plan document
PUT /api/issues/{mvpIssueId}/documents/plan
{
  "title": "MVP Launch Plan",
  "format": "markdown",
  "body": "# MVP Launch Plan\n\n## Phase 1: Core\n- Auth system\n- Basic dashboard\n\n## Phase 2: Polish\n- Error handling\n- Performance",
  "baseRevisionId": null
}

# Step 4: Delegasi ke CTO — buat subtask
POST /api/companies/{companyId}/issues
{
  "title": "Build core auth system",
  "description": "JWT auth, login, register, refresh token",
  "assigneeAgentId": "<bagas-id>",
  "parentId": "{mvpIssueId}",
  "goalId": "{goalId}",
  "projectId": "{authProjectId}",
  "status": "todo",
  "priority": "critical"
}

# Step 5: Delegasi task lain ke agent lain
POST /api/companies/{companyId}/issues
{
  "title": "Build dashboard UI",
  "description": "React dashboard dengan basic metrics",
  "assigneeAgentId": "<frontend-id>",
  "parentId": "{mvpIssueId}",
  "goalId": "{goalId}",
  "status": "todo",
  "priority": "high"
}

# Step 6: Update status parent task
PATCH /api/issues/{mvpIssueId}
{
  "status": "in_progress",
  "comment": "## Delegasi Selesai\n\nMVP dipecah jadi 2 workstream:\n- [Core auth](/PAP/issues/PAP-42) → Bagas\n- [Dashboard UI](/PAP/issues/PAP-43) → Frontend\n\nPlan: [/PAP/issues/PAP-10#document-plan](/PAP/issues/PAP-10#document-plan)"
}
```

---

### Case 6: Setup Tailscale untuk Akses dari HP

**Tujuan:** Pantau dan manage Paperclip dari iPhone saat tidak di depan Mac.

```bash
# Step 1: Install Tailscale di Mac dan iPhone
# Mac: brew install tailscale
# iPhone: App Store → Tailscale

# Step 2: Login ke Tailscale di kedua device (pakai akun yang sama)
tailscale up

# Step 3: Start Paperclip dalam authenticated/private mode
pnpm dev --tailscale-auth

# Step 4: Izinkan Tailscale hostname Mac
pnpm paperclipai allowed-hostname nama-macbook-kamu
# Nama hostname bisa dilihat di: tailscale status

# Step 5: Dari iPhone, akses via Tailscale IP atau hostname
# http://100.x.x.x:3100  (Tailscale IP)
# atau
# http://nama-macbook-kamu:3100  (Tailscale hostname — jika DNS aktif)

# Step 6: Login dengan credentials yang sudah dibuat
# UI Paperclip akan muncul di mobile browser
```

Konfigurasi deployment mode (jika belum):

```bash
pnpm paperclipai configure --section server
# Pilih: authenticated
# Pilih: private
```

---

### Case 7: Deploy ke Production (Authenticated Mode)

**Skenario:** Deploy Paperclip di VPS atau cloud server untuk tim.

```bash
# Di server — clone dan install
git clone https://github.com/paperclipai/paperclip.git
cd paperclip
pnpm install

# Konfigurasi untuk production
pnpm paperclipai onboard
# Pilih: authenticated
# Pilih: public
# Set public URL: https://paperclip.yourcompany.com

# Atau set via env vars di .env
PAPERCLIP_DEPLOYMENT_MODE=authenticated
PAPERCLIP_DEPLOYMENT_EXPOSURE=public
PAPERCLIP_PUBLIC_URL=https://paperclip.yourcompany.com
DATABASE_URL=postgresql://user:pass@localhost:5432/paperclip  # optional — gunakan external Postgres
PAPERCLIP_SECRETS_STRICT_MODE=true  # recommended untuk production

# Run
pnpm paperclipai run
```

Dengan Docker Compose di production:

```yaml
# docker-compose.yml
version: '3.8'
services:
  paperclip:
    build: .
    ports:
      - "3100:3100"
    environment:
      PAPERCLIP_DEPLOYMENT_MODE: authenticated
      PAPERCLIP_DEPLOYMENT_EXPOSURE: public
      PAPERCLIP_PUBLIC_URL: https://paperclip.yourcompany.com
      PAPERCLIP_HOME: /paperclip
      HOST: 0.0.0.0
    volumes:
      - paperclip-data:/paperclip
    restart: unless-stopped

volumes:
  paperclip-data:
```

```bash
docker compose up -d
```

Setelah deploy, jalankan board claim (migrasi dari local_trusted):

```bash
# Startup akan print claim URL
# Format: https://paperclip.yourcompany.com/board-claim/<token>?code=<code>
# Buka di browser dan sign in untuk claim board ownership
```

---

### Case 8: Budget Management dan Cost Control

**Pantau biaya:**

```bash
# Overview biaya company
pnpm paperclipai dashboard get

# Biaya per agent
curl "http://localhost:3100/api/companies/<id>/costs/by-agent" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"

# Biaya per project
curl "http://localhost:3100/api/companies/<id>/costs/by-project" \
  -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

**Update budget agent:**

Via UI: buka agent settings → ubah monthly budget.

Via API:

```bash
PATCH /api/agents/{agentId}
{
  "budgetMonthlyCents": 10000
}
```

**Pause agent yang overbudget:**

Jika agent auto-paused (100% budget), kamu bisa:
1. Naikkan budget untuk bulan ini
2. Tunggu reset di awal bulan
3. Pause manual di UI jika memang harus stop

**Billing code untuk tracking per project:**

Saat buat subtask, set `billingCode`:

```bash
POST /api/companies/{companyId}/issues
{
  "title": "Implement feature X",
  "billingCode": "ORBIT-2024",
  ...
}
```

---

## Bagian 9: Rekomendasi Setup untuk Kirana Corp

### Setup yang Direkomendasikan Step by Step

**Fase 1: Local Development Setup (Sekarang)**

```bash
# 1. Start Paperclip
cd ~/Sandbox/paperclip
pnpm dev

# 2. Set context profile
pnpm paperclipai context set \
  --api-base http://localhost:3100 \
  --company-id <kirana-corp-id>

# 3. Buat company "Kirana Corp" via UI
# 4. Buat goals perusahaan
# 5. Buat projects (Orbit, Forge)
```

**Fase 2: Agent Registry**

Setup agents yang sesuai struktur Kirana Corp di ORGANIZATION.md:

```
Kiran (CEO) → claude_local
    └── Bagas (CTO) → claude_local
            ├── Arka (Engineering Lead) → claude_local, cwd=orbit-server
            ├── Amelia (Developer) → claude_local, cwd=orbit-server
            ├── Quinn (QA Lead) → claude_local, cwd=orbit
            └── Raka (DevOps) → claude_local, cwd=orbit
```

**Fase 3: Project & Workspace Setup**

```bash
# Project Orbit Server
POST /api/companies/<id>/projects
{
  "name": "Orbit Server",
  "description": "Laravel 12 backend untuk Orbit",
  "status": "active",
  "workspace": {
    "cwd": "/Users/agung/Projects/orbit/orbit-server",
    "repoUrl": "https://github.com/kiranacorp/orbit-server"
  }
}

# Project Orbit Mobile
POST /api/companies/<id>/projects
{
  "name": "Orbit Mobile",
  "description": "Flutter 3.41.2 app untuk Orbit",
  "status": "active",
  "workspace": {
    "cwd": "/Users/agung/Projects/orbit/orbit-mobile",
    "repoUrl": "https://github.com/kiranacorp/orbit-mobile"
  }
}
```

### Struktur Org yang Ideal untuk Software House

```
Board (Agung) — governance, approval, override
    └── Kiran (CEO) — strategic direction, goal setting
            └── Bagas (CTO) — technical strategy, breakdown
                    ├── Arka (Engineering Lead) — architecture, ADR
                    │       ├── Amelia (Developer) — implementation, TDD
                    │       ├── Dira (Backend) — Laravel, Filament
                    │       ├── Sena (Mobile) — Flutter, Riverpod
                    │       └── Nara (Database) — PostgreSQL, RLS
                    ├── Quinn (Quality Lead) — QA gate, E2E
                    │       ├── Vega (Security) — OWASP audit
                    │       ├── Rex (Reality Checker) — adversarial verification
                    │       └── Elara (Evidence) — test output, coverage
                    └── Raka (DevOps Lead) — CI/CD, deploy
                            └── Hiro (Incident) — hotfix, RCA
```

**Budget guidelines (dalam satuan token — bukan dollar):**

| Role | Budget (token/bulan) | Reasoning |
|------|---------------------|-----------|
| CEO (Kiran) | Medium | Strategic decisions, tidak banyak kerja teknis |
| CTO (Bagas) | Medium | Breakdown + coordination |
| Engineering Lead (Arka) | Medium | Architecture + review |
| Developer (Amelia, Dira, Sena) | High | Heavy implementation work |
| QA (Quinn, Vega) | Medium | Review + testing |
| DevOps (Raka) | Low-Medium | Deploy + monitoring |
| Reality Checker (Rex) | Low | Verification only |

### Workflow Patterns yang Terbukti Efektif

**Pattern 1: Top-down decomposition**

```
Agung buat task high-level → Kiran (CEO) strategi →
Bagas (CTO) breakdown → Arka (Engineering Lead) spec →
Amelia (Developer) implement → Quinn (QA) verify →
Raka (DevOps) deploy
```

**Pattern 2: Parallel workstreams**

Untuk project besar — CTO buat beberapa subtask paralel dengan assignee berbeda. Semua berjalan paralel, manager review hasilnya.

**Pattern 3: @-mention untuk review handoff**

```
Amelia selesai code → comment "@Quinn tolong review PR #42" →
Quinn bangun, review, approve → Amelia deploy
```

**Pattern 4: Blocked escalation chain**

```
Agent blocked → update status blocked + comment →
@mention direct manager → manager evaluate →
unblock atau escalate ke atasnya lagi
```

### Anti-Patterns yang Harus Dihindari

| Anti-Pattern | Kenapa Buruk | Solusi yang Benar |
|-------------|-------------|-------------------|
| Buat task tanpa `parentId` | Hierarchy berantakan, task jadi floating | Selalu link ke parent task |
| @-mention agents untuk update biasa | Setiap mention = heartbeat = budget terpakai | Cukup update status di task |
| Satu agent kerjakan semua | Tidak ada parallelism, bottleneck | Delegate ke tim yang tepat |
| Task status dibiarkan `in_progress` lama tanpa update | Tidak ada visibility ke manager | Comment progress setiap heartbeat |
| Buat subtask tanpa `goalId` | Task tidak ter-align ke goal company | Selalu set goalId dari parent |
| Retry 409 checkout | Waste heartbeat, double work | Pick task berbeda |
| Cancel task dari cross-team | Hanya manager assigning team yang bisa cancel | Reassign ke manager dengan comment |
| Agent self-assign tanpa mention handoff | Overkill, tidak ada governance | Tunggu assignment proper |

---

## Bagian 10: Apa yang Perlu Dikembangkan

### 10.1 Gap yang Ada di Paperclip Saat Ini

Berdasarkan analisis kebutuhan Kirana Corp sebagai software house AI-powered:

**Gap 1: Native Slack/Linear Integration**

Saat ini Paperclip tidak punya native webhook untuk Slack atau Linear. Kamu harus build custom integration jika mau notifikasi task ke Slack atau sync issues ke Linear.

**Gap 2: Bring Your Own Ticket System**

Tercatat di roadmap resmi tapi belum ada — tidak bisa sync issues Paperclip ke Jira/Linear/GitHub Issues sebagai canonical source of truth.

**Gap 3: Agent "Health Dashboard" per Agent**

Dashboard aggregate ada, tapi tidak ada per-agent health view yang detail — misalnya "agent ini stuck di task yang sama selama 3 hari" atau "agent ini jarang update".

**Gap 4: Budget dalam Token (Bukan Dollar)**

Saat ini budget dalam `budgetMonthlyCents` (dollar cents). Untuk Claude Code subscription (bukan pay-per-token), metric ini kurang relevan. Kirana Corp pakai model subscription, jadi unit tracking yang lebih berguna adalah waktu aktif atau jumlah heartbeat.

**Gap 5: Worktree-aware Agent Assignment**

Saat ini agent punya satu `cwd`. Untuk multi-repo project (orbit-server + orbit-mobile), agent harus punya multiple workspace contexts yang bisa di-switch per task.

**Gap 6: Template / Clone Company**

Tidak ada cara cepat untuk buat company baru dengan struktur org yang sama. Harus setup ulang dari nol setiap kali.

### 10.2 Fitur yang Masuk Roadmap Resmi

Dari README resmi Paperclip:

| Fitur | Status | Keterangan |
|-------|--------|-----------|
| ClipMart | Coming soon | Download dan run entire companies dengan satu klik |
| Easier OpenClaw onboarding | Planned | Saat ini masih cukup manual |
| Cloud agents (Cursor/e2b) | Planned | Agents yang run di cloud, tidak perlu lokal |
| Easier agent configurations | Planned | UI yang lebih user-friendly untuk config agent |
| Better harness engineering | Planned | Framework untuk test agent behavior |
| Plugin system | Done (green) | Sudah tersedia — bisa tambah knowledgebase, custom tracing, queues |
| Better docs | Planned | Dokumentasi lebih lengkap dan navigable |

### 10.3 Modifikasi Spesifik untuk Kebutuhan Kirana Corp

**Modifikasi 1: Slack Webhook Plugin**

Buat plugin Paperclip yang push notification ke Slack ketika:
- Task status berubah ke `blocked`
- Approval baru menunggu review
- Agent budget di atas 80%
- Task selesai di project tertentu

Kaitkan ke `#kirana-updates` dan `#pegagan-control` channel.

**Modifikasi 2: Linear Sync Plugin**

Buat plugin yang mirror issues Paperclip ke Linear:
- Buat issue baru di Linear ketika agent buat task di Paperclip
- Update status di Linear ketika status Paperclip berubah
- Attach Linear issue ID ke Paperclip task sebagai metadata

**Modifikasi 3: NEXUS Handoff Template**

Extend issue comments dengan NEXUS handoff template validator — ketika agent mark task `done` dan ada `assigneeAgentId` yang berubah, sistem validate bahwa comment mengandung NEXUS format.

**Modifikasi 4: Budget Unit — Heartbeat Count**

Override budget tracking untuk pakai "heartbeat count" daripada dollar cents. Lebih relevan untuk subscription-based model.

**Modifikasi 5: Multi-repo Workspace Context**

Extend agent model untuk support multiple workspace contexts — agent bisa switch `cwd` berdasarkan task yang sedang dikerjakan (sesuai `project.primaryWorkspace`).

### 10.4 Plugin Opportunities (Plugin System Sudah Ada)

Paperclip punya plugin system. Dari `skills/paperclip-create-plugin/SKILL.md`, beberapa plugin yang bisa dibangun:

**Plugin: Knowledgebase Sync**

Auto-sync AGENTS.md dari repo ke Paperclip sebagai agent knowledge context. Setiap kali ada push ke repo, agent knowledge ter-update.

**Plugin: Custom Tracing**

Export heartbeat runs dan task transitions ke observability platform (Datadog, Grafana) untuk monitoring agent performance.

**Plugin: Queue Integration**

Integrate Paperclip tasks dengan message queue (Bull, Temporal) untuk guaranteed delivery dan retry logic yang lebih sophisticated.

**Plugin: Audit Export**

Export activity log ke compliance-friendly format (CSV, structured JSON) untuk kebutuhan audit trail yang lebih formal.

---

## Bagian 11: Fork & Customize Strategy

### Mengapa Fork?

Kirana Corp punya kebutuhan spesifik yang tidak akan ada di upstream cepat:
- Slack/Linear integration
- NEXUS handoff validation
- Custom budget metrics
- Indonesian language support di UI

Fork memungkinkan kustomisasi sambil tetap bisa pull upstream updates.

### Setup Fork yang Benar

```bash
# Step 1: Fork di GitHub via UI
# https://github.com/paperclipai/paperclip → Fork → kiranacorp/paperclip

# Step 2: Clone fork kamu
git clone https://github.com/kiranacorp/paperclip.git
cd paperclip

# Step 3: Tambah upstream remote
git remote add upstream https://github.com/paperclipai/paperclip.git

# Step 4: Verifikasi remotes
git remote -v
# origin    https://github.com/kiranacorp/paperclip.git (fetch)
# origin    https://github.com/kiranacorp/paperclip.git (push)
# upstream  https://github.com/paperclipai/paperclip.git (fetch)
# upstream  https://github.com/paperclipai/paperclip.git (push)
```

### Branch Strategy untuk Kustomisasi

```
master          ← upstream-compatible (selalu bisa pull dari upstream)
    └── kirana/main    ← branch utama Kirana Corp
            ├── kirana/slack-integration    ← fitur Slack
            ├── kirana/linear-sync         ← fitur Linear sync
            └── kirana/nexus-validation    ← NEXUS handoff validator
```

**Workflow:**

```bash
# Buat branch kirana/main dari master
git checkout master
git checkout -b kirana/main

# Buat feature branch dari kirana/main
git checkout kirana/main
git checkout -b kirana/slack-integration

# Kerja di feature branch
# ... buat perubahan ...
git add .
git commit -m "feat: add Slack webhook plugin for task notifications"

# Merge ke kirana/main
git checkout kirana/main
git merge kirana/slack-integration

# Deploy dari kirana/main
git push origin kirana/main
```

### Pull Upstream Updates

```bash
# Fetch upstream changes
git fetch upstream

# Cek ada update apa
git log master..upstream/master --oneline

# Rebase master ke upstream/master
git checkout master
git rebase upstream/master

# Push ke origin (fork)
git push origin master

# Update kirana/main dengan perubahan upstream terbaru
git checkout kirana/main
git rebase master
# Resolve konflik jika ada
git push origin kirana/main --force-with-lease
```

**Jadwal sync upstream:** Rekomendasikan sync setidaknya seminggu sekali, atau segera setelah ada security patch dari upstream.

### Apa yang Aman di-Customize vs yang Harus Ikut Upstream

**AMAN di-customize (kustomisasi Kirana Corp):**

| Area | File/Path | Jenis Kustomisasi |
|------|-----------|------------------|
| Plugins | `server/plugins/` | Tambah plugin baru |
| Skills | `skills/` | Tambah/modify skills |
| UI themes | `ui/src/` | Branding, warna |
| Documentation | `doc/` | Update atau tambah docs |
| Agent prompts | `agents/*/AGENTS.md` | Persona dan instructions |

**HATI-HATI (sync dengan upstream):**

| Area | Alasan |
|------|--------|
| Database schema | Upstream migration harus konsisten — selalu review sebelum merge |
| API contracts | Breaking changes akan affect agents |
| Auth system | Security-critical, ikuti upstream |
| Core heartbeat logic | Fundamental behavior, jangan ubah sembarangan |

**JANGAN customize (biar upstream yang handle):**

| Area | Alasan |
|------|--------|
| `packages/db/` schema tanpa migration | Akan corrupt database |
| Auth session handling | Security risk |
| Cost tracking core logic | Bisa lead ke inaccurate billing |

### Cara Berkontribusi Balik ke Upstream

Jika kamu fix bug atau build fitur yang berguna untuk semua orang — kontribusi ke upstream:

```bash
# Step 1: Buat branch dari upstream master (bukan kirana/main)
git fetch upstream
git checkout -b fix/agent-blocked-dedup upstream/master

# Step 2: Buat perubahan TANPA kiran-specific code
# Step 3: Commit dengan pesan yang jelas
git commit -m "fix: prevent duplicate blocked comments when no new context

When an agent has already posted a blocked status comment and no new
comments from other agents/users have been added, skip re-processing
the blocked task to avoid duplicate notifications."

# Step 4: Push ke fork kamu
git push origin fix/agent-blocked-dedup

# Step 5: Buat PR ke upstream dari GitHub UI
# kiranacorp/paperclip:fix/agent-blocked-dedup → paperclipai/paperclip:master
```

**Guidelines kontribusi yang meningkatkan peluang diterima:**
- Fix bugs yang reproducible dengan test case
- Fitur yang generic, bukan Kirana-specific
- Ikuti code style existing
- Tidak ubah `pnpm-lock.yaml` (CI yang kelola)
- Tulis tests untuk perubahan yang di-submit

### Cherry-pick Selective dari Upstream

Jika upstream punya perubahan tertentu yang ingin kamu ambil tanpa full sync:

```bash
# Lihat commit hash dari upstream
git log upstream/master --oneline -20

# Cherry-pick commit spesifik ke kirana/main
git checkout kirana/main
git cherry-pick <commit-hash>

# Jika ada konflik
git status
# Edit file konflik
git add .
git cherry-pick --continue
```

### Contoh Kasus: Mengelola Upstream Security Patch

```bash
# Upstream release security patch di commit abc123

# Step 1: Fetch upstream
git fetch upstream

# Step 2: Check patch
git show upstream/master:abc123

# Step 3: Apply ke master kirana
git checkout master
git cherry-pick abc123

# Step 4: Test
pnpm typecheck
pnpm test:run
pnpm build

# Step 5: Push ke origin
git push origin master

# Step 6: Propagate ke kirana/main
git checkout kirana/main
git rebase master
git push origin kirana/main --force-with-lease

# Step 7: Jika ada environment production
# Deploy dari kirana/main
```

---

## Appendix: Quick Reference

### Env Vars Penting

| Env Var | Keterangan | Default |
|---------|-----------|---------|
| `PAPERCLIP_HOME` | Home directory untuk instance | `~/.paperclip` |
| `PAPERCLIP_INSTANCE_ID` | Instance ID | `default` |
| `PAPERCLIP_DEPLOYMENT_MODE` | `local_trusted` atau `authenticated` | `local_trusted` |
| `PAPERCLIP_DEPLOYMENT_EXPOSURE` | `private` atau `public` | - |
| `PAPERCLIP_PUBLIC_URL` | URL publik untuk authenticated mode | - |
| `DATABASE_URL` | External PostgreSQL URL (jika tidak pakai embedded) | - |
| `PAPERCLIP_DB_BACKUP_ENABLED` | Aktifkan auto backup | `true` |
| `PAPERCLIP_DB_BACKUP_INTERVAL_MINUTES` | Interval backup | `60` |
| `PAPERCLIP_DB_BACKUP_RETENTION_DAYS` | Retensi backup | `30` |
| `PAPERCLIP_SECRETS_STRICT_MODE` | Strict mode untuk secrets | `false` |
| `PAPERCLIP_ENABLE_COMPANY_DELETION` | Toggle company deletion | `true` (local_trusted) |

### Checklist Onboarding Agent Baru

```
[ ] Discover adapter config options via /llms/agent-configuration.txt
[ ] Review existing agent configs untuk referensi
[ ] Pilih icon yang sesuai via /llms/agent-icons.txt
[ ] Submit hire request dengan lengkap (name, role, title, icon, reportsTo, capabilities, adapterConfig, runtimeConfig)
[ ] Tunggu approval jika perlu
[ ] Setup local CLI: paperclipai agent local-cli <shortname>
[ ] Export env vars ke shell
[ ] Set instructions path via PATCH /api/agents/:id/instructions-path
[ ] Test dengan heartbeat manual
[ ] Assign test task dan verify flow
[ ] Cleanup test task
```

### Checklist Heartbeat Agent

```
[ ] GET /api/agents/me (jika belum ada di context)
[ ] Cek approval follow-up jika PAPERCLIP_APPROVAL_ID di-set
[ ] GET /api/agents/me/inbox-lite
[ ] Pick work: in_progress dulu, lalu todo, skip blocked (kecuali ada context baru)
[ ] POST /api/issues/{id}/checkout dengan X-Paperclip-Run-Id
[ ] GET /api/issues/{id}/heartbeat-context
[ ] Kerjakan task
[ ] PATCH /api/issues/{id} dengan status update + comment + X-Paperclip-Run-Id
[ ] POST subtask jika perlu delegation
[ ] Exit
```

---

*Dokumen ini dibuat untuk Kirana Corp — last updated 2026-03-19. Untuk perubahan pada Paperclip upstream, cek https://github.com/paperclipai/paperclip/releases*
