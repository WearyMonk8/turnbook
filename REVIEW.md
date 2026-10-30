# Turnbook review: is it a fit for a solo Claude Code developer on Windows 11?

Reviewed commit: `HEAD` of `main` as cloned on 2026-10-30 (version 0.10.4, `pyproject.toml`).
Paths are relative to the repo root. "Hypothesis" marks inference; everything else is tied to a
file, a command output, or git metadata.

## Verdict: **Sandbox trial only**

The core claims hold up in code: saving makes no model calls, storage is local SQLite, and the
shared-hub encryption is real. However, the project is 22 days old with a single author, ships
several releases per day, and its installer pipes remote scripts into a shell and rewrites the
config of every AI client it finds (including adding a hook to `~/.claude/settings.json` and a
Windows Startup entry). Try it in a disposable Windows VM or Windows Sandbox first, not on the
main machine.

## Architecture summary

- **Entry points** (`pyproject.toml` `[project.scripts]`): `turnbook` (MCP server,
  `src/graph_mind/mcp_server.py`), `turnbook-install`, `turnbook-capture`, `turnbook-forget`,
  `turnbook-view`, `turnbook-hub`.
- **Capture, passive:** a background "capture service" (`src/graph_mind/capture_cli.py` →
  `automatic_capture.py`) watches each client's own transcript files. Claude Code's JSONL
  transcripts are read by the class with `CLIENT = "claude-code"`
  (`conversation_memory.py:692`), and Codex, Cursor, Copilot, OpenCode and Cowork have siblings
  (`conversation_memory.py:930-952`).
- **Capture, hooks:** for clients without readable transcripts (Gemini CLI), `hook.py` appends
  each message to `~/.graph-mind/hooks/<agent>/`, where the capture service picks it up
  (`hook.py:1-10`).
- **Capture, explicit:** the model can also call the `brain_remember` MCP tool
  (`tests/test_secret_masking.py:61`).
- **Incremental scan:** `scan()` (`conversation_memory.py:580`) reads new JSONL lines by offset,
  runs `_redact()` on each turn (`:639`), and stores it.
- **Redaction:** `_redact()` (`conversation_memory.py:153`) applies `SECRET_PATTERNS`
  (`:27`) plus a random-token detector (`RANDOM_CANDIDATE`).
- **Storage:** `ConversationMemoryStore` (`conversation_memory.py:220-290`) is SQLite in WAL
  mode under `~/.graph-mind/` (`paths.py`, `GRAPH_MIND_HOME` overrides it). The tables are
  `conversation_turns` plus an FTS5 full-text index `conversation_turns_fts`.
- **Compression, not encryption:** turns of 200 bytes or more are zlib-compressed into
  `content_z` (`packing.py:1-25`).
- **Embeddings:** a local ONNX model is fetched once from Hugging Face (`local_embedder.py:133`).
  The default is EmbeddingGemma, 4-bit (`docs/benchmarks.md:19`). Vectors are cached by
  `vector_cache.py`.
- **Retrieval:** MCP tools in `mcp_server.py` fuse word search (FTS) with meaning search
  (vectors), then apply a learned re-ranker (`research/full500_norerank.py` swaps out
  `mcp_server.rerank`/`fuse`). `context_packet.py` and `packing.py` build a bounded "evidence
  packet".
- **Per-message injection:** a `UserPromptSubmit` hook (`install.py` `_hook_files`) runs
  `graph_mind.hook claude-code UserPromptSubmit`. That calls `inject.py`, which asks the
  running service over `127.0.0.1` with an auth key (`inject.py:60-75`) and prepends memory to
  the prompt, with a 3-second limit (`hook.py` `DEADLINE`).
- **Other stores:** `coding_memory.py` (code folder watching, opt-in per `install.py`
  `capture_service`), `documents.py` (`turnbook docs add`), `semantic_memory.py`,
  `associative_memory.py`, `local_brain.py`.
- **Sharing (optional):** `sharing.py` has two backends: a synced folder of append-only JSONL
  logs, or a Postgres "hub" that Turnbook downloads and runs itself. `project_brains.py` adds
  team hubs.
- **Forgetting:** `forget.py` deletes turns locally and propagates ids (not text) to shared
  peers (`docs/privacy.md:44-46`).

## Claim audit

| Claim | Verdict | Evidence |
|---|---|---|
| Saving makes no LLM or model API calls and spends no tokens | **Verified** | `grep` over `src/graph_mind/*.py` finds no `openai`, `anthropic`, `api_key` or HTTP client in the capture/store path. `scan()` → `_redact()` → SQLite insert (`conversation_memory.py:580-673`). The only model is the local ONNX embedder (`local_embedder.py`), which costs CPU, not tokens. |
| Nothing leaves the local machine | **Partially verified** | No telemetry, analytics or crash reporting found (`grep` for telemetry/analytics/posthog/sentry: none). Capture/store/search use only SQLite and a loopback socket (`inject.py:69-71`). **Outbound calls do exist:** (1) Hugging Face model download on install or first use (`local_embedder.py:133-137`, `install.py:472`). After that, `HF_HUB_OFFLINE=1` is set once the model is on disk (`mcp_server.py:29-37`). (2) `turnbook-view` downloads a JS library from unpkg (`memory_view.py:232-248`). (3) Sharing downloads Postgres binaries from GitHub or Maven (`sharing.py:254-260`, SHA-256-checked at `:262`). (4) `project_brains.py:728` runs `gh pr view`. No update check found. |
| Secrets are masked before storage | **Verified, with limits** | `SECRET_PATTERNS` (`conversation_memory.py:27-60`) covers: OpenAI `sk-`/`sk-proj-`, `key_…`, `Authorization: Bearer`, quoted and unquoted `password/secret/token/api_key/access_key/private_key/client_secret = …`, JWTs, DigitalOcean, GitLab `glpat-`, npm, Hugging Face `hf_`, GitHub `ghp_`/`github_pat_`, AWS `AKIA`/`ASIA`, Google `AIza`, Slack `xox*`, PEM private keys, passwords in DB URLs, Korean password phrases, and a generic "looks random" token detector. Applied at `conversation_memory.py:639` and to code at `coding_memory.py:345,472`. Tests: `tests/test_secret_masking.py` (7 tests), all passing. **Limits:** masking can be switched off by `"mask_secrets": false` (`:146-150`). It is regex-based, so Anthropic `sk-ant-` keys rely on the `sk-` pattern (hypothesis: matched, since `sk-ant-…` fits `sk-[A-Za-z0-9_-]{16,}`). Your client's own transcript files (`~/.claude/projects`) are untouched and stay unmasked (`docs/privacy.md:45`). |
| Storage is one local SQLite file | **Partially verified** | It is SQLite under `~/.graph-mind/` (`paths.py`), but not one file. There are several stores: conversations, `development.sqlite` (`paths.py`), per-project coding state (`paths.py` `default_coding_state_dir`), WAL sidecar files (`use_wal`), `hooks/`, `outbox.jsonl` (`sharing.py:1061`), `config.json`. **Not encrypted at rest:** text is plain, or zlib-compressed (`packing.py`), and the FTS index holds plain words. |
| Folder-sharing encrypts data before it leaves the PC | **Partially verified / contradicted for the synced-folder backend** | **Postgres hub backend** (what `turnbook folder share` sets up, `sharing.py:423`, `docs/privacy.md:49-55`): each line is sealed with **Fernet** (AES-128-CBC with HMAC-SHA256) under a key from **HKDF-SHA256** with a fixed salt `b"graph-mind sharing"` (`sharing.py:954-980`). The input is the share password, `secrets.token_urlsafe(18)` (~144 bits, `sharing.py:449`), which travels inside the connection code. Transport is TLS with a pinned self-signed ECDSA P-256 cert (`sharing.py:513-558`). **Synced-folder backend** (OneDrive/Dropbox path, `sharing.py:1-15`): `_folder_append` (`sharing.py:730-746`) writes lines **without calling `_seal`**. Those logs reach the sync provider in plain text. |

## Security findings (most severe first)

1. **High: remote code piped into a shell, twice over.** `scripts/install.ps1:9`
   `Invoke-RestMethod https://astral.sh/uv/install.ps1 | Invoke-Expression`, and the README's own
   entry point `irm https://raw.githubusercontent.com/goyohan0611-png/turnbook/main/scripts/install.ps1 | iex`
   (`scripts/install.ps1:3`). `scripts/install.sh:12`: `curl -LsSf https://astral.sh/uv/install.sh | sh`.
   `scripts/Turnbook-Setup.cmd:9-11` runs the same with `-ExecutionPolicy Bypass`. The script
   tracks the unpinned `main` branch, and `uv tool install` takes the latest PyPI `turnbook`
   unless `GRAPH_MIND_VERSION` is set (`install.ps1:15`).
2. **High: broad, automatic changes to every AI client** (`src/graph_mind/install.py`).
   - Claude Code: `claude mcp add -s user graph-mind-memory …` (`:91-93`), or a direct edit of
     `~/.claude.json` (`:98`).
   - A `UserPromptSubmit` hook in `~/.claude/settings.json` (`_hook_files`) that injects memory
     into **every** prompt.
   - Also edited: Claude Desktop config, `~/.codex/config.toml` and `~/.codex/hooks.json`,
     OpenCode config, plus an `opencode/plugins/graph-mind.js` plugin and
     `opencode/graph-mind.md` (written), Cursor, Windsurf, Gemini CLI (3 hooks), Copilot CLI
     (MCP + hook file), LM Studio, VS Code `mcp.json`, and Cline.
   - Everything is done without a confirmation prompt.
3. **Medium: persistence via a Windows Startup shortcut.** `install.py:376-379,418-426` creates
   `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\Graph-MIND Automatic Capture.lnk`
   running `pythonw.exe -m graph_mind.capture_cli run` at every login. There is no scheduled task
   or service.
4. **Medium: memory is plaintext at rest** (`packing.py`, `conversation_memory.py:240`). Anyone
   or anything with access to your user profile can read every captured conversation, minus the
   masked secrets.
5. **Medium: the synced-folder sharing mode is unencrypted** (`sharing.py:730-746`), contrary to
   the README line "every line encrypted before it leaves a PC" (`README.md:120-123`). That line
   is true only for the Postgres hub.
6. **Low–medium: sharing opens a firewall port.** `sharing.py:588-596` adds an inbound rule
   through an elevated `netsh` (UAC prompt), limited to LocalSubnet and Tailscale.
7. **Low: prompt injection through recalled memory** (hypothesis). Recalled text, including
   text from shared or team hubs, is injected into the prompt on every message (`inject.py`,
   `hook.py`). Something said in an earlier session or written by a teammate can steer the model.
8. **Low: the binary download is pinned.** Postgres binaries are checked against a hardcoded
   SHA-256 (`sharing.py:254-262`). That's good practice, but only relevant if sharing is used.

**Installers, full inventory**

- **URLs:**
  - `install.ps1`: `https://astral.sh/uv/install.ps1`, then PyPI (`turnbook`, plus Python 3.12
    via uv).
  - `install.sh`: `https://astral.sh/uv/install.sh`, plus PyPI.
  - Then `turnbook-install` (`install.py`) downloads the Hugging Face model, and the
    Visual C++ link is only printed (`install.py:458`).
  - Sharing only: `github.com/theseus-rs/postgresql-binaries`, `repo1.maven.org` (`sharing.py:254-258`).
- **Files written:**
  - Everything listed in finding 2, plus `~/.local/bin/uv*`, the uv tool environment, and the
    Startup `.lnk`.
  - `~/.graph-mind/automatic-capture` config (`install.py` `capture_service`), and
    `~/.graph-mind/config.json` (`--name`).
  - The Hugging Face cache.
  - Each JSON is written as `*.graph-mind.tmp` and then replaced (`install.py` `_json_entry`).
- **Removed:** an old `graph-mind-memory` uv tool (`install.ps1:17`), and old Hugging Face
  model revisions (`install.py` `remove_old_models`).
- **Uninstall:** `turnbook-uninstall --purge` exists (`pyproject.toml`). CI runs it at
  `.github/workflows/*.yml` (`turnbook-uninstall --purge --yes`). Whether it fully reverts the
  Claude Code hook was not tested here.

## Test results

Command, as documented in `.github/workflows/tests.yml`:
`python -m unittest discover -s tests -p "test_*.py"` (Python 3.13.17, Linux container).

- **Run 1 (as root): 172 run, 159 passed, 0 failed, 13 errors.** All 13 errors were
  `initdb: error: cannot be run as root`. That's an **ENVIRONMENT** issue: Postgres refuses to
  run as root.
- **Run 2 (those 5 modules, as an unprivileged user): 42 run, 40 passed, 2 errors.** The errors
  were `test_forget.ForgetTests.test_forgotten_from_a_postgres_hub` and
  `test_hub_retry.HubRetryTests.test_a_start_that_failed_after_initdb_still_gets_the_memory_database`,
  both `HTTP Error 429: Too Many Requests` while downloading Postgres binaries. That's
  **ENVIRONMENT** (rate limiting), and I did not retry.
- **Net: 170 of 172 passed, 0 code failures, 2 unresolved because of the environment.** The
  tests were not run on Windows. Note that the test suite itself downloads binaries from the
  internet.

## Benchmark audit

- **Source:** `docs/benchmarks.md:13`, the run `research/runs/graph-v9.14-full500-mpnet8-gemma/`.
- **Dataset:** LongMemEval_S, "cleaned" variant (`research/longmemeval_retrieval_executor.py:31`,
  `longmemeval_s_cleaned.json`). The dataset file is not in the repo (`external/` is absent), so
  the exact version is **unverified**.
- **Answer model:** `gpt-5-mini-2025-08-07`, in 499 records of `answers.jsonl`.
- **Judge:** `gpt-4o-2024-08-06` (`official-judge-report.json`).
- **Score:** `official-judge.jsonl` has 500 rows, and **459** have `"label": true`, which
  matches 91.8%.
- **Caveats** (from `docs/benchmarks.md:22-24`):
  - The re-ranker was trained partly on LongMemEval, so the score is not fully held out. The
    held-out-ish subset scored 91.1% (346/380), which I did not recount.
  - `official-judge-report.json`'s `note` field still describes an older 60-question pilot,
    which is a stale artifact.
- **Reproducible from this repo?** Partly. The retrieval side is free and offline:
  `bench/evidence.py`, which the author says takes about an hour of CPU. Answer accuracy needs
  paid OpenAI API calls (`research/product_answer_eval.py`) and the external dataset. I did not
  rerun it, so the score is **unverified, not false**. The packets (`spans.jsonl`) and verdicts
  are committed, so the scoring can be audited.

## Maturity

Source: `git log` after `git fetch --unshallow`.

- **Age:** first commit 2026-10-08, latest 2026-10-30, which is **22 days**.
- **Commits:** 393. **Contributors:** 1 (`goyohan0611-png`, all 393 commits).
- **Releases:** 81 git tags. `CHANGELOG.md` has 77 entries, including **5 releases on
  2026-10-30 alone** (0.10.0–0.10.4).
- **Package rename:** from `graph-mind` to `turnbook` at 0.9.44 (`pyproject.toml`
  compatibility aliases).
- **Bus factor: 1.** All commits come from one person, and the pace is very high (up to 56
  commits a day). Hypothesis: the code is largely AI-assisted. That's not a defect, but it means
  review depth is unknown.
- **Stars, issues, external users:** **unverified**. The GitHub API was not available to this
  session (HTTP 403 from `gh api`). The README badges are claims, not evidence.

## Risks, effort and cost of adopting it

- **What it would replace:** Claude Code's own memory is curated text that you or Claude edit
  (CLAUDE.md, memory files). Turnbook is different. It keeps everything and searches it, and it
  injects results into every prompt. It complements curated memory rather than replacing it, and
  it overlaps directly with your own persistent-memory server. Hypothesis: running both would
  double the injected context and confuse which one is the source of truth.
- **Per-prompt cost:** an evidence packet of about 3.1k tokens (`docs/benchmarks.md:13`),
  injected on prompts it judges relevant. That's real context-window spend on every session,
  even though writing is free.
- **Runtime cost:**
  - A background process at login.
  - A local embedding model, about 7–22 seconds to load cold per `hook.py` comments.
  - Disk for every conversation, kept forever until you run `turnbook-forget`.
- **Effort:**
  - One command to install, but you'll need to audit and diff about a dozen client configs.
  - The upgrade churn is high (several releases a day), so pin a version.
- **Architectural fit:** good on local-first (SQLite, local ONNX, no API key). The design is
  readable and documents its failures (`research/REPORT.md`). The weak points are the
  install-everywhere behavior and the single maintainer.
- **Mitigations if trialed:**
  - Install with `pip`/`uv` from a pinned version instead of the piped scripts.
  - Run `turnbook-install` only inside a Windows Sandbox or VM first.
  - Diff `~/.claude.json` and `~/.claude/settings.json` before and after.
  - Don't enable synced-folder sharing.

## Open questions only a local run can answer

1. On Windows 11, does `turnbook-uninstall --purge` remove the Claude Code hook, the MCP entry
   and the Startup shortcut cleanly?
2. How much latency does the `UserPromptSubmit` hook add to each Claude Code prompt in practice,
   with a cold and a warm service?
3. Does the injected memory improve or degrade answers in your real projects, compared with your
   own memory server?
4. How fast does `~/.graph-mind/` grow with your actual Claude Code usage?
5. Does the masking catch the secret formats that actually show up in your sessions?
6. Does Windows Defender or SmartScreen object to the uv download, `pythonw` at login, or the
   unsigned ONNX runtime DLLs?
