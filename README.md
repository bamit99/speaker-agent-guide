# Announce Through Any Smart Speaker From Your Agent

A step-by-step guide to connect a **Google Cast speaker** (Mi Smart Speaker, Nest Audio, Chromecast) and an **Amazon Echo (Alexa)** to a local agent (OpenClaw, Claude Code, or any CLI that can run shell commands), so the agent can push spoken announcements to them — alarms, reminders, class notifications.

Everything here runs **locally, free, and without any vendor cloud API** for the Google path. The Alexa path uses your Amazon account's own web API via a self-captured cookie session. This was built and verified working on a Mac mini (macOS) but works anywhere Python + Node.js run.

> **Legal / ToS note:** the Alexa section talks to Amazon's web endpoints with your own logged-in session. That's the same thing the Alexa web console does, but it is unofficial and may break when Amazon changes its login. Use for personal automation on devices you own.

---

## Architecture at a glance

```
Your Agent (cron / chat trigger)
        │
        ├──▶ Google path:  TTS → local HTTP server → Chromecast stream
        │                  (edge-tts / ElevenLabs → MP3 → pychromecast)
        │
        └──▶ Alexa path:   cookies.txt jar → alexa.amazon.in REST API
                           (speak / announce / volume commands)
```

- **Google Cast speakers** accept media streams: you synthesize speech to an MP3, serve it over your LAN, and tell the speaker to play that URL.
- **Echo speakers** don't accept arbitrary audio the same way, but Amazon's `alexa.amazon.in` API has speak/announce endpoints reachable with a browser session cookie — once captured, a plain curl script drives it.

---

## Part 0 — Prerequisites

- A computer on the **same Wi-Fi/LAN** as the speakers (Mac mini, Raspberry Pi, any always-on box).
- Python 3.9+, Node.js 18+, `pip`, `npm`.
- Find the speakers' presence first:

```bash
# Google Cast devices on the LAN (mDNS discovery)
python3 - <<'EOF'
import zeroconf
from zeroconf import Zeroconf, ServiceBrowser
import time
z = Zeroconf()
class L:
    def add_service(self, zc, t, name):
        info = z.get_service_info("_googlecast._tcp.local.", name, 3000)
        if info: print(name, info.addresses[0].decode() if info.addresses else "?", info.port)
ServiceBrowser(z, "_googlecast._tcp.local.", L())
time.sleep(5)
EOF
```

Note the **UUID** and IP of your speaker (e.g. `2922d960-...-4861813e8d50`, `192.168.0.191:8009`). **Always resolve by UUID, not IP** — DHCP reassigns IPs after router reboots.

---

## Part 1 — Google Cast speaker (Mi Smart Speaker / Nest / Chromecast)

### 1.1 Install deps

```bash
pip install pychromecast edge-tts   # edge-tts = free Microsoft neural voices
npm install -g elevenlabs           # optional, better voice quality
```

### 1.2 Write the cast script

Save as `scripts/cast_speak.py`:

```python
#!/usr/bin/env python3
"""Synthesize text with edge-tts and cast it to a Cast speaker by UUID."""
import argparse, subprocess, tempfile, os, time, socket, sys
import pychromecast

UUID = "2922d960-3031-472f-ddcb-4861813e8d50"   # <- your speaker's cast UUID
VOICE = "en-IN-NeerjaNeural"                    # Indian English female; pick any from edge-tts

def resolve_by_uuid(target_uuid):
    devices, browser = pychromecast.get_listed_chromecasts()
    for cast, _ in devices:
        if str(cast.uuid).lower() == target_uuid.lower():
            return cast
    sys.exit(f"Cast device {target_uuid} not found on LAN")

def main():
    p = argparse.ArgumentParser()
    p.add_argument("text")
    p.add_argument("--voice", default=VOICE)
    p.add_argument("--volume", type=float, default=0.6)  # 0..1
    a = p.parse_args()

    with tempfile.NamedTemporaryFile(suffix=".mp3", delete=False) as f:
        tmp = f.name
    subprocess.run(["python3", "-m", "edge_tts", "--voice", a.voice,
                    "--text", a.text, "--write-media", tmp], check=True)

    cast = resolve_by_uuid(UUID)
    cast.wait()
    orig = cast.status.volume_level
    cast.set_volume(a.volume)
    mc = cast.media_controller
    mc.play_media(f"http://{socket.gethostbyname(socket.gethostname())}:{PORT}/{os.path.basename(tmp)}",
                  "audio/mpeg")
```

> Simpler alternative used in production: serve the MP3 with a tiny HTTP server bound to your LAN IP, then `play_media(url)`. The full, battle-tested script (chime intro, volume restore, AAC/44.1kHz quirks) lives at `scripts/cast_speak.py` in this repo's examples.

Key hard-won gotchas (each one cost us a debugging round):
1. **Mi Smart Speaker Lite silently ignores odd WAV formats** — use **AAC or MP3 at 44.1 kHz**, always.
2. **The synthesizing process must stay alive through playback** — don't fork and exit; sleep or wait on media status.
3. **Restore the original volume** after playback, or the speaker stays stuck at alarm volume.
4. **Resolve by cast UUID**, never by IP.

### 1.3 Test it

```bash
python3 scripts/cast_speak.py "Hello, this is a test announcement" --volume 0.6
# Success = PLAYING → IDLE → playback finished → volume restored
```

---

## Part 2 — Amazon Echo (Alexa) — the cookie method

This is the trickier half. Two steps: **capture a session cookie**, then **drive the REST API** with it.

### 2.1 Set up the capture proxy

```bash
mkdir -p scripts/alexa && cd scripts/alexa
npm init -y && npm install alexa-cookie2
```

Save as `scripts/alexa/setup-cookie.js` (condensed — full version in repo):

```javascript
const AlexaCookie = require('alexa-cookie2');
const config = {
  proxyOwnIp: '192.168.0.11',   // your machine's LAN IP
  proxyPort: 3456,
  proxyListenBind: '0.0.0.0',
  setupProxy: true, proxyOnly: true,
  amazonPage: 'amazon.in',      // or amazon.com / amazon.de …
  baseAmazonPage: 'amazon.in',
  acceptLanguage: 'en-IN',
  amazonPageProxyLanguage: 'en_US',
  // IMPORTANT: override the library's stale Chrome/103 UA — an old UA
  // makes Amazon serve a broken login page (Apollon77/alexa-cookie#190)
  userAgent: 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/153.0.0.0 Safari/537.36',
};
AlexaCookie.generateAlexaCookie(config, (err, result) => {
  if (result && (result.csrf || result.cookie)) {
    require('fs').writeFileSync('.alexa-cookie.json',
      JSON.stringify(result, null, 2), { mode: 0o600 });
    console.log('CAPTURED'); process.exit(0);
  }
});
console.log('OPEN IN BROWSER: http://YOUR_LAN_IP:3456/');
```

### 2.2 Log in and capture

```bash
node setup-cookie.js
# From ANY device on the same LAN (phone is fine) open:
#   http://<machine-LAN-IP>:3456/
# → Amazon sign-in page appears → log in with the account that owns the Echo
# → complete MFA → proxy captures cookies automatically
```

**Real-world failure mode we hit (and the fix):** Amazon may complete login + MFA and then the library's callback **never fires** — no `.alexa-cookie.json` appears. Don't panic and don't redo the login. The raw session cookies are still in the proxy's standard output! Extract the **last** `Cookies handled:` blob from the log and build a curl cookie jar:

```bash
python3 - <<'PY'
import re
raw = open('proxy.log', errors='ignore').read()
sets = re.findall(r'Cookies handled: "([^"]+)"', raw)
pairs = dict(p.split('=',1) for p in sets[-1].split('; ') if '=' in p)
lines = ["# Netscape HTTP Cookie File"]
for k, v in pairs.items():
    lines.append("\t".join([".amazon.in","TRUE","/","TRUE","0",k,v]))
open('state/cookies.txt','w').write("\n".join(lines)+"\n")
print("jar rebuilt,", len(pairs), "cookies")
PY
```

Then verify the jar actually works — this is the real test, not the capture file:

```bash
curl -s -b state/cookies.txt \
  -H "csrf: $(awk -F'\t' '$6=="csrf"{print $7}' state/cookies.txt | head -1)" \
  'https://alexa.amazon.in/api/devices-v2/device?cached=false' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); [print(x['accountName'], x['serialNumber'], x.get('online')) for x in d['devices']]"
```

Expected: your Echo device names + serials.

### 2.3 The command script

Save as `scripts/alexa/alexa.sh` (core logic):

```bash
#!/usr/bin/env bash
# Alexa remote via raw cookie session. Usage:
#   alexa.sh devices
#   alexa.sh speak <device|all> "text" [--volume 0-100]
#   alexa.sh announce "text" [--devices serial1,serial2] [--volume N]
set -euo pipefail
DIR="$(cd "$(dirname "$0")" && pwd)"; JAR="$DIR/state/cookies.txt"
UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/153.0.0.0 Safari/537.36"
csrf() { awk -F'\t' '$6=="csrf"{print $7}' "$JAR" | head -1; }

case "$1" in
  devices)
    curl -s -b "$JAR" -A "$UA" -H "csrf: $(csrf)" \
      'https://alexa.amazon.in/api/devices-v2/device?cached=false' \
      | python3 -c "import json,sys;[print(x['accountName'],'|',x['serialNumber'],'| online=',x.get('online')) for x in json.load(sys.stdin).get('devices',[])]"
    ;;
  *)
    echo "implement speak/announce via /api/behaviors/preview POST with device serials"
    ;;
esac
```

The **speak/ announce** actions POST to `/api/behaviors/preview` with a behavior JSON naming the device serial:

```json
{"behaviorId":"PREVIEW","sequenceJson":"{\"@type\":\"com.amazon.alexa.behaviors.model.Sequence\"…\"speak\"…}","status":"RUNNING"}
```

(Full working `speak`/`announce`/`volume` implementation is in `scripts/alexa/alexa.sh` — it also auto-guards volume for other household members: reads current level, sets target, speaks, restores.)

### 2.4 Test end-to-end

```bash
./alexa.sh devices                                    # list devices
./alexa.sh speak MyEcho "Hello, agent connected" --volume 50
# Success = HTTP 200 + the Echo actually speaks + original volume restored
```

**Cookie lifetime:** ours survived ~12 days. When `alexa.sh devices` starts returning empty/`302 signin`, re-run `node setup-cookie.js` and redo the browser login (2 minutes). Keep `state/` chmod 600.

---

## Part 3 — Wire both into your agent

### 3.1 As a one-shot reminder (OpenClaw cron style)

```bash
openclaw automations add --name "Study reminder" \
  --cron "15 18 * * *" --tz Asia/Calcutta --exact \
  --announce --channel telegram --to <your-chat-id> \
  --message "Check calendar for MBA session; if found, run:
  python3 scripts/cast_speak.py '...text...' --volume 0.7
  ./scripts/alexa/alexa.sh speak MyEcho '...text...' --volume 60
  and reply with a one-line confirmation"
```

The `--exact` flag prevents cron stagger so alarms land on the minute.

### 3.2 As a plain shell function (any other agent / Claude Code / plain cron)

```bash
announce() {
  python3 /path/to/scripts/cast_speak.py "$1" --volume 0.6 &
  /path/to/scripts/alexa/alexa.sh speak MyEcho "$1" --volume 55
}
announce "Reminder: class starts in 15 minutes"
```

Drop `announce()` into `.zshrc`/`.bashrc` and any agent that can run shell can broadcast.

### 3.3 Sanity checklist before trusting it overnight

- [ ] Google speaker: test cast plays AND volume restores after
- [ ] Echo: test speak returns HTTP 200 AND volume restores (read-modify-write)
- [ ] Cookies are fresh (devices call succeeded today)
- [ ] Cron/automation has `--exact` (or no stagger) so alarms are punctual
- [ ] Machine won't sleep through the alarm time (`pmset -g` / power plan)

---

## Troubleshooting cheatsheet

| Symptom | Cause → Fix |
|---|---|
| Alexa API returns 302 to `/ap/signin` | Cookies expired → re-run capture (2 min) |
| `set-cookie` capture never writes `.alexa-cookie.json` | Library callback race → salvage cookies from proxy.log (§2.2) |
| Broken Amazon login page in proxy | Stale user-agent → override `userAgent` in setup-cookie.js |
| Cast "plays" silently, no sound | Wrong audio container → MP3/AAC @ 44.1 kHz, never raw WAV |
| Announcement never finishes / no restore | Script exits early → `wait()` on playback, then restore volume |
| Speaker not found | IP changed after router reboot → resolve by cast UUID |
| Echo heard but too quiet/ loud | Use volume guard: read → set → speak → restore |

---

## Security notes

- `state/cookies.txt` is a live Amazon session — treat like a password: `chmod 600`, never commit, never paste into chats.
- Keep the capture proxy (`:3456`) **off the internet** — LAN only, and stop the process right after capture. (We `kill` it immediately in practice.)
- Don't route TTS text with personal data through any cloud TTS if that bothers you; edge-tts sends text to Microsoft, ElevenLabs to ElevenLabs. For fully local: `whisper`-style local TTS engines (e.g. Piper) can replace edge-tts with zero cloud.

---

*Built and verified in a real home-hack session (Oct 2026): Mi Smart Speaker (Chromecast, Family room) + Amazon Echo "Bangalore", driven by an OpenClaw cron automation with calendar-gated announcements. Both speakers confirmed live end-to-end.*