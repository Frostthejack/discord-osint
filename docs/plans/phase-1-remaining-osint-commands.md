# Phase 1: Remaining OSINT Commands — Implementation Plan

> **For Hermes:** Use subagent-driven-development skill to implement this plan task-by-task.

**Goal:** Add 6 remaining OSINT slash commands to the Hermes Agent Discord gateway (`discord.py`).

**Architecture:** Each command follows the existing pattern in `gateway/platforms/discord.py`: `@tree.command` decorator, `interaction.response.defer()`, subprocess.run with timeout, error handling, output truncation to 1900 chars. All commands are independent and can be implemented in parallel.

**Tech Stack:** Python 3.12, discord.py, subprocess, PyJWT, qrcode/pyzbar

**Depends on:** Phase 0 (project infrastructure) — ✅ Complete

**Review Gate:** After all 6 commands are implemented, a reviewer task will verify all commands are registered, the gateway restarts cleanly, and each command responds correctly in Discord.

---

## Objectives
- [x] Project infrastructure (Phase 0) — repo, board, cron jobs, workspaces
- [ ] Implement `/nuclei` — vulnerability scanner
- [ ] Implement `/ghunt` — Google account OSINT
- [ ] Implement `/shodan` — device search
- [ ] Implement `/url_scan` — VirusTotal URL scanning
- [ ] Implement `/jwt_decode` — JWT token decoding
- [ ] Implement `/qr_decode` — QR code decoding from image URLs
- [ ] All commands registered and verified in Discord

## Tasks Overview

| # | Task | Assignee | Est. | Notes |
|---|------|----------|------|-------|
| 1.1 | Add `/nuclei` slash command | backend-eng | 5 min | Go binary already installed at `~/go/bin/nuclei` |
| 1.2 | Add `/ghunt` slash command | backend-eng | 5 min | `ghunt` CLI already installed |
| 1.3 | Add `/shodan` slash command | backend-eng | 5 min | `shodan` CLI already installed, needs API key |
| 1.4 | Add `/url_scan` slash command | backend-eng | 5 min | Needs VirusTotal API key |
| 1.5 | Add `/jwt_decode` slash command | backend-eng | 5 min | PyJWT already installed, no external API |
| 1.6 | Add `/qr_decode` slash command | backend-eng | 5 min | Needs pyzbar + Pillow install |
| 1.7 | Phase 1 Review: All 6 commands | reviewer | 10 min | Verify all commands work in Discord |

## Phase Review Criteria
1. [ ] All 6 commands appear in Discord slash picker
2. [ ] Gateway restarts without errors after changes
3. [ ] Each command returns results for valid input
4. [ ] Each command handles errors gracefully (invalid input, timeout, missing tool)
5. [ ] Output is truncated to 1900 chars max
6. [ ] No regressions in existing commands
7. [ ] `project-state.md` updated with completion status

---

## Implementation Details

### Task 1.1: `/nuclei` — Vulnerability Scanner

**File:** `src/discord.py` (symlink → `/home/frostthejack/.hermes/hermes-agent/gateway/platforms/discord.py`)

**Pattern:** Follow `slash_dnsx` (Go binary via subprocess)

```python
@tree.command(name="nuclei", description="Scan a target for vulnerabilities (nuclei)")
@app_commands.describe(
    target="URL or IP to scan",
    severity="Filter by severity: low, medium, high, critical (default: all)",
    templates="Specific template tags, e.g. 'cve,misconfig' (default: all)",
)
async def slash_nuclei(
    interaction: discord.Interaction,
    target: str,
    severity: str = "",
    templates: str = "",
):
    await interaction.response.defer(ephemeral=False)
    import subprocess
    cmd = ["/home/frostthejack/go/bin/nuclei", "-u", target, "-silent"]
    if severity:
        cmd.extend(["-severity", severity])
    if templates:
        cmd.extend(["-tags", templates])
    try:
        proc = subprocess.run(cmd, capture_output=True, text=True, timeout=180)
        output = (proc.stdout + proc.stderr).strip()
        if not output:
            output = f"No findings for `{target}`."
    except subprocess.TimeoutExpired:
        output = f"⏱️ nuclei timed out after 180s for `{target}`."
    except FileNotFoundError:
        output = "❌ nuclei is not installed. Run: `go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest`"
    except Exception as e:
        output = f"❌ Error running nuclei: {e}"
    if len(output) > 1900:
        output = output[:1900] + "\n…(truncated)"
    await interaction.edit_original_response(
        content=f"🔍 **nuclei scan for `{target}`:**\n```\n{output}\n```"
    )
```

**VERIFICATION:**
(1) `~/go/bin/nuclei --version` — confirm installed
(2) Add function + `@tree.command` to discord.py
(3) `python -c "import ast; ast.parse(open('src/discord.py').read())"` — parses cleanly
(4) Gateway restarts: `hermes gateway restart`
(5) Discord: `/nuclei scanme.nmap.org` — returns scan results
(6) Discord: `/nuclei invalid-target-xyz` — graceful error

---

### Task 1.2: `/ghunt` — Google Account OSINT

**File:** `src/discord.py`

**Pattern:** Follow `slash_holehe` (email-based CLI tool via subprocess)

```python
@tree.command(name="ghunt", description="Google account OSINT (ghunt)")
@app_commands.describe(
    email="Google email address to investigate",
)
async def slash_ghunt(
    interaction: discord.Interaction,
    email: str,
):
    await interaction.response.defer(ephemeral=False)
    import subprocess
    cmd = ["ghunt", "email", email]
    try:
        proc = subprocess.run(cmd, capture_output=True, text=True, timeout=120)
        output = (proc.stdout + proc.stderr).strip()
        if not output:
            output = f"No results found for `{email}`."
    except subprocess.TimeoutExpired:
        output = f"⏱️ ghunt timed out after 120s for `{email}`."
    except FileNotFoundError:
        output = "❌ ghunt is not installed. Run: `pip install ghunt`"
    except Exception as e:
        output = f"❌ Error running ghunt: {e}"
    if len(output) > 1900:
        output = output[:1900] + "\n…(truncated)"
    await interaction.edit_original_response(
        content=f"🔍 **ghunt results for `{email}`:**\n```\n{output}\n```"
    )
```

**VERIFICATION:**
(1) `ghunt --version` — confirm installed
(2) Add function + `@tree.command` to discord.py
(3) Parse check passes
(4) Gateway restarts cleanly
(5) Discord: `/ghunt test@gmail.com` — returns results or "no results"
(6) Discord: `/ghunt invalid-email-xyz` — graceful error

---

### Task 1.3: `/shodan` — Device Search

**File:** `src/discord.py`

**Pattern:** Follow `slash_dnsx` (CLI tool via subprocess). Needs `SHODAN_API_KEY` env var.

```python
@tree.command(name="shodan", description="Search Shodan for an IP or domain")
@app_commands.describe(
    target="IP address or domain to look up",
)
async def slash_shodan(
    interaction: discord.Interaction,
    target: str,
):
    await interaction.response.defer(ephemeral=False)
    import subprocess, os
    cmd = ["shodan", "host", target]
    try:
        proc = subprocess.run(cmd, capture_output=True, text=True, timeout=60)
        output = (proc.stdout + proc.stderr).strip()
        if not output:
            output = f"No Shodan data found for `{target}`."
    except subprocess.TimeoutExpired:
        output = f"⏱️ shodan timed out after 60s for `{target}`."
    except FileNotFoundError:
        output = "❌ shodan CLI is not installed. Run: `pip install shodan`"
    except Exception as e:
        if "Invalid API key" in str(e) or "SHODAN_API_KEY" in str(e):
            output = "❌ Shodan API key not configured. Set `SHODAN_API_KEY` in `~/.hermes/.env`."
        else:
            output = f"❌ Error running shodan: {e}"
    if len(output) > 1900:
        output = output[:1900] + "\n…(truncated)"
    await interaction.edit_original_response(
        content=f"🔍 **shodan lookup for `{target}`:**\n```\n{output}\n```"
    )
```

**VERIFICATION:**
(1) `shodan --version` — confirm installed
(2) `SHODAN_API_KEY` set in `~/.hermes/.env`
(3) Add function + `@tree.command` to discord.py
(4) Parse check passes
(5) Gateway restarts cleanly
(6) Discord: `/shodan 8.8.8.8` — returns Shodan data
(7) Discord: `/shodan invalid-target` — graceful error

---

### Task 1.4: `/url_scan` — VirusTotal URL Scanner

**File:** `src/discord.py`

**Pattern:** Uses VirusTotal API via `curl` (no CLI tool needed). Needs `VT_API_KEY` env var.

```python
@tree.command(name="url_scan", description="Scan a URL with VirusTotal")
@app_commands.describe(
    url="URL to scan",
)
async def slash_url_scan(
    interaction: discord.Interaction,
    url: str,
):
    await interaction.response.defer(ephemeral=False)
    import subprocess, os, json as _json
    vt_api_key = os.environ.get("VT_API_KEY", "")
    if not vt_api_key:
        # Try to read from .env file
        try:
            with open(os.path.expanduser("~/.hermes/.env")) as f:
                for line in f:
                    if line.strip().startswith("VT_API_KEY="):
                        vt_api_key = line.strip().split("=", 1)[1].strip().strip('"').strip("'")
                        break
        except Exception:
            pass
    if not vt_api_key:
        await interaction.edit_original_response(
            content="❌ VirusTotal API key not configured. Set `VT_API_KEY` in `~/.hermes/.env`."
        )
        return
    # Submit URL for scanning
    try:
        scan_cmd = [
            "curl", "-s", "-X", "POST",
            "https://www.virustotal.com/api/v3/urls",
            "-H", "x-apikey: " + vt_api_key,
            "-d", "url=" + url,
        ]
        proc = subprocess.run(scan_cmd, capture_output=True, text=True, timeout=30)
        data = _json.loads(proc.stdout)
        scan_id = data.get("data", {}).get("id", "")
        if not scan_id:
            output = f"❌ Failed to submit URL to VirusTotal. Response: {proc.stdout[:500]}"
        else:
            # Get analysis results
            report_cmd = [
                "curl", "-s",
                f"https://www.virustotal.com/api/v3/analyses/{scan_id}",
                "-H", "x-apikey: " + vt_api_key,
            ]
            proc2 = subprocess.run(report_cmd, capture_output=True, text=True, timeout=30)
            report = _json.loads(proc2.stdout)
            stats = report.get("data", {}).get("attributes", {}).get("stats", {})
            malicious = stats.get("malicious", 0)
            suspicious = stats.get("suspicious", 0)
            harmless = stats.get("harmless", 0)
            undetected = stats.get("undetected", 0)
            total = malicious + suspicious + harmless + undetected
            output = (
                f"**VirusTotal Scan for `{url}`**\n"
                f"🔴 Malicious: {malicious}\n"
                f"🟡 Suspicious: {suspicious}\n"
                f"🟢 Harmless: {harmless}\n"
                f"⚪ Undetected: {undetected}\n"
                f"📊 Total engines: {total}"
            )
    except subprocess.TimeoutExpired:
        output = f"⏱️ VirusTotal API timed out for `{url}`."
    except Exception as e:
        output = f"❌ Error scanning URL: {e}"
    if len(output) > 1900:
        output = output[:1900] + "\n…(truncated)"
    await interaction.edit_original_response(content=output)
```

**VERIFICATION:**
(1) `VT_API_KEY` set in `~/.hermes/.env`
(2) Add function + `@tree.command` to discord.py
(3) Parse check passes
(4) Gateway restarts cleanly
(5) Discord: `/url_scan https://example.com` — returns scan results
(6) Discord: `/url_scan not-a-url` — graceful error

---

### Task 1.5: `/jwt_decode` — JWT Token Decoder

**File:** `src/discord.py`

**Pattern:** Pure Python using PyJWT (already installed). No subprocess needed.

```python
@tree.command(name="jwt_decode", description="Decode a JWT token and show header + payload")
@app_commands.describe(
    token="JWT token string to decode",
)
async def slash_jwt_decode(
    interaction: discord.Interaction,
    token: str,
):
    await interaction.response.defer(ephemeral=False)
    import jwt, json as _json
    try:
        # Decode without verification (we just want to inspect)
        header = jwt.get_unverified_header(token)
        payload = jwt.decode(token, options={"verify_signature": False})
        output = (
            f"**JWT Decoded**\n\n"
            f"**Header:**\n```json\n{_json.dumps(header, indent=2)}\n```\n"
            f"**Payload:**\n```json\n{_json.dumps(payload, indent=2)}\n```"
        )
    except jwt.InvalidTokenError as e:
        output = f"❌ Invalid JWT token: {e}"
    except Exception as e:
        output = f"❌ Error decoding JWT: {e}"
    if len(output) > 1900:
        output = output[:1900] + "\n…(truncated)"
    await interaction.edit_original_response(content=output)
```

**VERIFICATION:**
(1) `pip show PyJWT` — confirm installed
(2) Add function + `@tree.command` to discord.py
(3) Parse check passes
(4) Gateway restarts cleanly
(5) Discord: `/jwt_decode eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c` — returns decoded header + payload
(6) Discord: `/jwt_decode not-a-jwt` — graceful error

---

### Task 1.6: `/qr_decode` — QR Code Decoder

**File:** `src/discord.py`

**Pattern:** Download image from URL, decode with pyzbar + Pillow.

```python
@tree.command(name="qr_decode", description="Decode a QR code from an image URL")
@app_commands.describe(
    image_url="URL of an image containing a QR code",
)
async def slash_qr_decode(
    interaction: discord.Interaction,
    image_url: str,
):
    await interaction.response.defer(ephemeral=False)
    import subprocess, tempfile, os
    try:
        # Download the image
        tmp = tempfile.NamedTemporaryFile(suffix=".png", delete=False)
        tmp.close()
        dl_cmd = ["curl", "-s", "-L", "-o", tmp.name, image_url]
        dl_proc = subprocess.run(dl_cmd, capture_output=True, text=True, timeout=30)
        if dl_proc.returncode != 0:
            output = f"❌ Failed to download image from `{image_url}`."
        else:
            # Decode QR code using Python
            py_cmd = [
                "python3", "-c",
                f"""
import sys
try:
    from pyzbar.pyzbar import decode
    from PIL import Image
    img = Image.open('{tmp.name}')
    results = decode(img)
    if results:
        for r in results:
            print(f"Type: {{r.type}}\\nData: {{r.data.decode('utf-8')}}")
    else:
        print("No QR codes found in the image.")
except ImportError as e:
    print(f"Missing dependency: {{e}}. Run: pip install pyzbar Pillow")
except Exception as e:
    print(f"Error: {{e}}")
"""
            ]
            proc = subprocess.run(py_cmd, capture_output=True, text=True, timeout=30)
            output = (proc.stdout + proc.stderr).strip()
            if not output:
                output = "No QR codes found in the image."
        os.unlink(tmp.name)
    except subprocess.TimeoutExpired:
        output = f"⏱️ QR decode timed out for `{image_url}`."
    except Exception as e:
        output = f"❌ Error decoding QR: {e}"
    if len(output) > 1900:
        output = output[:1900] + "\n…(truncated)"
    await interaction.edit_original_response(
        content=f"🔍 **QR decode result:**\n```\n{output}\n```"
    )
```

**VERIFICATION:**
(1) `pip install pyzbar Pillow` — install dependencies
(2) Add function + `@tree.command` to discord.py
(3) Parse check passes
(4) Gateway restarts cleanly
(5) Discord: `/qr_decode https://example.com/qr.png` — returns decoded data
(6) Discord: `/qr_decode https://example.com/not-a-qr.png` — "No QR codes found"

---

### Task 1.7: Phase 1 Review

**Assignee:** reviewer

**Body:**
Review all 6 commands from Phase 1: Remaining OSINT Commands.

**CHECK:**
- [ ] All 6 commands appear in Discord slash picker
- [ ] Gateway restarts without errors (`hermes gateway restart` → check logs)
- [ ] `/nuclei scanme.nmap.org` returns scan results
- [ ] `/ghunt test@gmail.com` returns results or graceful "no results"
- [ ] `/shodan 8.8.8.8` returns Shodan data (requires SHODAN_API_KEY)
- [ ] `/url_scan https://example.com` returns VirusTotal results (requires VT_API_KEY)
- [ ] `/jwt_decode <valid-jwt>` returns decoded header + payload
- [ ] `/qr_decode <qr-image-url>` returns decoded data
- [ ] All commands handle invalid input gracefully
- [ ] No regressions in existing commands (spot-check `/nmap`, `/sherlock`, `/dnsx`)
- [ ] `project-state.md` updated

**VERIFICATION:**
(1) `hermes gateway restart` — no errors in logs
(2) `grep "slash command" ~/.hermes/logs/gateway.log | tail -10` — all 6 new commands registered
(3) Test each command in Discord
(4) Update project-state.md: move all 6 from "Pending" to "Done"
