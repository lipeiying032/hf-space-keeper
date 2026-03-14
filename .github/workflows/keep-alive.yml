"""
HuggingFace Space Keep-Alive Script
====================================
Keeps one or more HuggingFace Spaces alive by periodically pinging them
and waking them up via the HF API if they've gone to sleep.

Environment Variables:
  SPACE_IDS  - Comma-separated list of space IDs in "owner/space-name" format
               Example: "myuser/my-app,anotheruser/their-app"
  HF_TOKEN   - HuggingFace API token (with read+write access to your spaces)
"""

import os
import sys
import time
import json
import logging
import urllib.request
import urllib.error
from datetime import datetime, timezone

# ─── Logging Setup ────────────────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
log = logging.getLogger(__name__)

# ─── Constants ────────────────────────────────────────────────────────────────
HF_API_BASE = "https://huggingface.co/api/spaces"
MAX_RETRIES = 3
RETRY_DELAY = 10        # seconds between retries
PING_TIMEOUT = 30       # seconds for HTTP ping timeout
WAKE_TIMEOUT = 60       # seconds to wait after waking a space

# Statuses that indicate a space is running fine
HEALTHY_STATUSES = {"RUNNING", "RUNNING_BUILDING"}

# Statuses that indicate a space needs to be woken
SLEEPING_STATUSES = {"SLEEPING", "PAUSED", "STOPPED", "NO_APP_FILE"}

# ─── Helpers ──────────────────────────────────────────────────────────────────

def hf_request(method: str, url: str, token: str | None = None, data: bytes | None = None) -> dict | None:
    """Make an HTTP request to the HuggingFace API with retry logic."""
    headers = {
        "Content-Type": "application/json",
        "User-Agent": "hf-space-keeper/1.0",
    }
    if token:
        headers["Authorization"] = f"Bearer {token}"

    req = urllib.request.Request(url, headers=headers, method=method, data=data)

    for attempt in range(1, MAX_RETRIES + 1):
        try:
            with urllib.request.urlopen(req, timeout=PING_TIMEOUT) as resp:
                body = resp.read().decode("utf-8")
                return json.loads(body) if body else {}
        except urllib.error.HTTPError as e:
            log.warning(f"HTTP {e.code} on attempt {attempt}/{MAX_RETRIES}: {url}")
            if e.code in (401, 403):
                log.error("Auth error — check your HF_TOKEN.")
                return None
            if attempt < MAX_RETRIES:
                time.sleep(RETRY_DELAY)
        except (urllib.error.URLError, TimeoutError, OSError) as e:
            log.warning(f"Network error on attempt {attempt}/{MAX_RETRIES}: {e}")
            if attempt < MAX_RETRIES:
                time.sleep(RETRY_DELAY)

    return None


def get_space_status(space_id: str, token: str | None) -> str:
    """Return the runtime status string for a space, or 'UNKNOWN' on failure."""
    url = f"{HF_API_BASE}/{space_id}"
    data = hf_request("GET", url, token=token)
    if data is None:
        return "UNKNOWN"
    # Navigate: runtime.stage
    stage = data.get("runtime", {}).get("stage", "UNKNOWN")
    return stage.upper()


def wake_space(space_id: str, token: str) -> bool:
    """Send a restart request to wake a sleeping/paused space."""
    url = f"{HF_API_BASE}/{space_id}/restart?factory=false"
    log.info(f"  ↺  Sending wake-up request to {space_id} ...")
    result = hf_request("POST", url, token=token, data=b"{}")
    return result is not None


def ping_space_url(space_id: str) -> bool:
    """
    Ping the public URL of a space to generate activity.
    URL format: https://{owner}-{space_name}.hf.space
    """
    owner, name = space_id.split("/", 1)
    slug = f"{owner}-{name}".lower().replace("_", "-")
    url = f"https://{slug}.hf.space"

    req = urllib.request.Request(
        url,
        headers={"User-Agent": "hf-space-keeper/1.0"},
        method="GET",
    )
    for attempt in range(1, MAX_RETRIES + 1):
        try:
            with urllib.request.urlopen(req, timeout=PING_TIMEOUT) as resp:
                status = resp.status
                log.info(f"  ✓  Pinged {url} → HTTP {status}")
                return True
        except urllib.error.HTTPError as e:
            # 4xx/5xx from the app itself still counts as "alive"
            log.info(f"  ✓  Pinged {url} → HTTP {e.code} (app responded)")
            return True
        except (urllib.error.URLError, TimeoutError, OSError) as e:
            log.warning(f"  ✗  Ping failed attempt {attempt}/{MAX_RETRIES}: {e}")
            if attempt < MAX_RETRIES:
                time.sleep(RETRY_DELAY)

    return False


# ─── Core Logic ───────────────────────────────────────────────────────────────

def process_space(space_id: str, token: str | None) -> dict:
    """
    Full keep-alive cycle for a single space:
    1. Check current status via API
    2. Wake it if sleeping (requires token)
    3. Ping the public URL
    Returns a result dict with details.
    """
    result = {
        "space_id": space_id,
        "status_before": "UNKNOWN",
        "woken": False,
        "pinged": False,
        "success": False,
    }

    log.info(f"━━━ Processing: {space_id} ━━━")

    # Step 1: Get status
    if token:
        status = get_space_status(space_id, token)
        result["status_before"] = status
        log.info(f"  ℹ  Current status: {status}")
    else:
        log.info("  ℹ  No HF_TOKEN provided — skipping status check")
        status = "UNKNOWN"

    # Step 2: Wake if sleeping
    if status in SLEEPING_STATUSES:
        if token:
            woken = wake_space(space_id, token)
            result["woken"] = woken
            if woken:
                log.info(f"  ✓  Wake-up sent. Waiting {WAKE_TIMEOUT}s for space to boot...")
                time.sleep(WAKE_TIMEOUT)
            else:
                log.warning(f"  ✗  Wake-up request failed for {space_id}")
        else:
            log.warning("  ⚠  Space is sleeping but no HF_TOKEN set — cannot wake via API.")

    # Step 3: Ping the URL
    pinged = ping_space_url(space_id)
    result["pinged"] = pinged
    result["success"] = pinged

    status_icon = "✅" if result["success"] else "❌"
    log.info(f"  {status_icon}  Done — woken={result['woken']}, pinged={result['pinged']}")
    return result


def main():
    log.info("=" * 60)
    log.info("  HuggingFace Space Keep-Alive")
    log.info(f"  Time: {datetime.now(timezone.utc).isoformat()}")
    log.info("=" * 60)

    # ── Read config from environment ──────────────────────────────
    raw_ids = os.environ.get("SPACE_IDS", "").strip()
    token   = os.environ.get("HF_TOKEN", "").strip() or None

    if not raw_ids:
        log.error("SPACE_IDS environment variable is not set or empty.")
        log.error("Set it to a comma-separated list like: owner/space1,owner/space2")
        sys.exit(1)

    space_ids = [s.strip() for s in raw_ids.split(",") if s.strip()]

    # Validate format
    valid_ids = []
    for sid in space_ids:
        if "/" in sid and len(sid.split("/")) == 2:
            valid_ids.append(sid)
        else:
            log.warning(f"Skipping invalid space ID (expected 'owner/name'): {sid!r}")

    if not valid_ids:
        log.error("No valid space IDs found. Aborting.")
        sys.exit(1)

    if not token:
        log.warning("HF_TOKEN not set — sleeping spaces cannot be woken via API.")

    log.info(f"Spaces to keep alive ({len(valid_ids)}): {', '.join(valid_ids)}")
    log.info("")

    # ── Process each space ────────────────────────────────────────
    results = []
    for i, space_id in enumerate(valid_ids):
        if i > 0:
            time.sleep(2)  # small pause between spaces
        result = process_space(space_id, token)
        results.append(result)

    # ── Summary ───────────────────────────────────────────────────
    log.info("")
    log.info("=" * 60)
    log.info("  SUMMARY")
    log.info("=" * 60)

    successes = sum(1 for r in results if r["success"])
    failures  = len(results) - successes

    for r in results:
        icon = "✅" if r["success"] else "❌"
        parts = [f"status={r['status_before']}"]
        if r["woken"]:
            parts.append("woken=yes")
        if not r["pinged"]:
            parts.append("ping=FAILED")
        log.info(f"  {icon}  {r['space_id']}  ({', '.join(parts)})")

    log.info("")
    log.info(f"  Total: {len(results)} | Success: {successes} | Failed: {failures}")
    log.info("=" * 60)

    if failures > 0:
        sys.exit(1)


if __name__ == "__main__":
    main()