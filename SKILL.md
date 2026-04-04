---
name: tiktok-draft-poster
description: Post TikTok carousel drafts from a real Android phone via ADB. Finds the draft pushed by Postiz/Buffer, adds audio, posts like a human. No TikTok API needed.
---

# TikTok Draft Poster

Automates the final step of TikTok scheduling: your scheduling tool pushes the draft, this skill handles finding it, adding audio, and posting.

**The flow:**
- Scheduling tool (Postiz, Buffer, etc.) sends carousel to TikTok via `content_posting_method: "UPLOAD"`
- This skill connects to your phone via ADB, finds the draft, picks audio, posts it
- No human action needed

---

## ⚠️ HOW TO RUN THIS — READ FIRST

### Run in the MAIN OpenClaw session. Never subagents. Never isolated sessions.

```
sessionTarget: "current"   ✅
sessionTarget: "isolated"  ❌ WILL FAIL
```

**Why subagents fail on this task:**
- ADB is a stateful connection — isolated sessions start cold on every tool call
- The screenshot → analyze → tap loop requires fast back-and-forth that isolated sessions can't sustain
- Isolated sessions hit context/time limits mid-flow and die with no recovery path
- Tested and confirmed: main session completes in ~2 min, isolated sessions timeout every time

**Cron config:**
```json
{
  "sessionTarget": "current",
  "payload": {
    "kind": "agentTurn",
    "message": "Post today's TikTok draft. Read SKILL.md and execute the full flow."
  }
}
```

### The correct posting sequence (end-to-end, proven under 2 min):

```
1. Pre-flight
   adb connect → zen_mode 0 → force-stop app → relaunch → wait 4s

2. Find draft
   Inbox tab → System notifications → tap scheduling tool notification

3. Verify you're in the editor
   UI dump must show "Your Story" + "Next" + an audio name in top bar

4. Check existing audio
   If top bar already shows correct audio → skip to step 7
   If wrong audio → proceed to step 5

5. Open picker + search
   Tap audio name in top bar → picker opens
   Tap Search → type your search term → press enter → wait 3s

6. Select audio
   Tap 1st carousel thumbnail → tap checkmark on that same card
   DO NOT browse multiple options. Pick the first result and stop.

7. ⚠️ HARD GATE — verify audio before posting
   Close picker fully (press back until editor shows)
   READ THE TOP BAR TEXT in the main editor
   If it shows the correct audio → proceed
   If it shows ANYTHING ELSE → DO NOT POST — go back to step 5
   Audio selection from search can revert on back navigation.
   This gate catches that. Never skip it.

8. Post
   Tap Next → wait 3s → tap Post → wait 5s

9. Confirm
   Screenshot → verify "Photos posted!" banner → done
```

**Target time: under 2 minutes.** Do not deviate from these steps.

### What makes the fast run work:
- **No deliberation** — don't compare audio options, pick the first carousel result
- **No extra screenshots** — `dump_hierarchy()` is faster for checking UI state
- **One shot** — if the draft already has correct audio, skip straight to step 7
- **Hard gate at step 7** — catches audio reversion bugs. Never skip it.

---

## Configuration

Fill these in before using the skill:

| Placeholder | What to put here |
|---|---|
| `YOUR_DEVICE_IP` | Your Android phone's local IP (e.g. `192.168.1.100`) |
| `YOUR_TIKTOK_PACKAGE` | TikTok package name — usually `com.zhiliaoapp.musically` (US) or `com.ss.android.ugc.trill` (some regions) |
| `YOUR_AUDIO_SEARCH_TERM` | Short single-word search for the audio you want (e.g. `lofi`, `quran`, `nasheed`) |
| `YOUR_POSTIZ_KEY` | Your Postiz API key (or equivalent scheduling tool key) |

---

## Quick Start

1. Fill in all placeholders above
2. Run the pre-flight check to confirm ADB connects:
   ```bash
   export PATH="/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:$PATH"
   adb connect YOUR_DEVICE_IP:5555
   adb -s YOUR_DEVICE_IP:5555 shell echo "connected"
   ```
3. Add to your OpenClaw cron (fires ~10 min after your scheduled post time):
   ```
   openclaw cron add --name "tiktok-post" --schedule "10 9 * * *" --tz "America/Chicago" \
     --message "Read SKILL.md path and post today's TikTok draft"
   ```
4. Add this skill path to your AGENTS.md or cron payload so the agent can read it

---

## Pre-Flight Checklist — Run EVERY Time

```bash
export PATH="/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:$PATH"
DEVICE="YOUR_DEVICE_IP:5555"

# 1. Connect
adb connect $DEVICE

# 2. Disable Do Not Disturb — CRITICAL (see Gotchas)
adb -s $DEVICE shell settings put global zen_mode 0

# 3. Keep screen awake
adb -s $DEVICE shell settings put global stay_on_while_plugged_in 3

# 4. Verify screen is awake
adb -s $DEVICE shell dumpsys power | grep mWakefulness
# Must say: mWakefulness=Awake
```

---

## Step 1 — Find the Draft

Scheduling tools that use TikTok's `UPLOAD` mode push drafts to TikTok's **in-app notification inbox** — NOT the system notification shade and NOT the Profile → Drafts tab.

### Method 1: TikTok In-App Notification Inbox (Primary)

```bash
# Launch TikTok
adb -s $DEVICE shell monkey -p YOUR_TIKTOK_PACKAGE 1
sleep 3

# Tap Inbox tab (bottom nav, 4th icon)
# Coordinates for 1200x2670 screen — adjust for your device resolution
adb -s $DEVICE shell input tap 840 2560
sleep 2

# Screenshot and analyze
adb -s $DEVICE shell screencap -p /sdcard/inbox.png
adb -s $DEVICE pull /sdcard/inbox.png /tmp/inbox.png
```

Look at the screenshot. Find the **"System notifications"** row and tap it:
```bash
adb -s $DEVICE shell input tap 600 490
sleep 2
```

Screenshot again. Look for **"Your content from Postiz is ready"** (or equivalent). Tap it to open the draft editor.

### Method 2: Stale Draft Popup (Fallback)

If TikTok was previously mid-edit and you relaunch it, a **"Continue editing this post?"** popup appears automatically. This is a reliable fallback.

```bash
# Kill and relaunch TikTok
adb -s $DEVICE shell am force-stop YOUR_TIKTOK_PACKAGE
sleep 1
adb -s $DEVICE shell monkey -p YOUR_TIKTOK_PACKAGE 1
sleep 3

# Screenshot — look for popup
adb -s $DEVICE shell screencap -p /sdcard/launch.png
adb -s $DEVICE pull /sdcard/launch.png /tmp/launch.png

# If popup visible: tap Edit button (right side, near top)
# Approximate coordinates for 1200x2670: x=950, y=200
adb -s $DEVICE shell input tap 950 200
sleep 2
```

---

## Step 2 — Add Audio

Once in the draft editor, use `uiautomator2` for reliable UI interaction. Raw ADB coordinates fail on React Native elements (see Gotchas).

```python
import uiautomator2 as u2
import time

d = u2.connect('YOUR_DEVICE_IP')

# Open audio picker
d(text="Add sound").click()
time.sleep(2)

# Search for audio
d(description="Search").click()
time.sleep(1)
d(className="android.widget.EditText").set_text("YOUR_AUDIO_SEARCH_TERM")
time.sleep(2)

# Select from carousel — tap the 2nd thumbnail (first result)
# 1st thumbnail = current/original audio, 2nd = first search result
# Coordinates calibrated for 1200x2670 Seeker — adjust for your screen
d.click(747, 665)   # tap 2nd carousel thumbnail to preview
time.sleep(1)
d.click(860, 870)   # tap checkmark on carousel card to confirm
time.sleep(1)

# Close audio picker
d.swipe(600, 1500, 600, 2500)
time.sleep(1)
```

**Why carousel, not list items:** TikTok renders audio results in a React Native ScrollView with a transparent overlay that blocks all ADB taps. The carousel thumbnails above the list are native Android — they respond to ADB. Always target the carousel.

---

## Step 3 — Post

```python
# Navigate to post
d(text="Next").click()
time.sleep(2)
d(text="Post").click()
time.sleep(3)

# Screenshot to confirm
import subprocess
subprocess.run(['adb', '-s', 'YOUR_DEVICE_IP:5555', 'shell', 'screencap', '-p', '/sdcard/confirm.png'])
subprocess.run(['adb', '-s', 'YOUR_DEVICE_IP:5555', 'pull', '/sdcard/confirm.png', '/tmp/confirm.png'])
```

---

## Full Script

Save as `post_draft.py` and call with: `python3 post_draft.py YOUR_AUDIO_SEARCH_TERM`

```python
#!/usr/bin/env python3
"""
TikTok Draft Poster
Usage: python3 post_draft.py <audio_search_term>
Example: python3 post_draft.py lofi
"""
import sys, time, subprocess

DEVICE_IP = "YOUR_DEVICE_IP"
DEVICE = f"{DEVICE_IP}:5555"
TIKTOK_PKG = "YOUR_TIKTOK_PACKAGE"
audio_term = sys.argv[1] if len(sys.argv) > 1 else "lofi"

def adb(cmd):
    return subprocess.run(f"adb -s {DEVICE} shell {cmd}", shell=True, capture_output=True, text=True)

def screenshot(name):
    adb(f"screencap -p /sdcard/{name}.png")
    subprocess.run(f"adb -s {DEVICE} pull /sdcard/{name}.png /tmp/{name}.png", shell=True)
    print(f"Screenshot saved: /tmp/{name}.png")

# Pre-flight
print("[1] Pre-flight checks...")
subprocess.run(f"adb connect {DEVICE}", shell=True)
adb("settings put global zen_mode 0")           # disable DND
adb("settings put global stay_on_while_plugged_in 3")
time.sleep(1)

# Open TikTok and find draft
print("[2] Opening TikTok...")
adb(f"force-stop {TIKTOK_PKG}")
time.sleep(1)
adb(f"monkey -p {TIKTOK_PKG} 1")
time.sleep(4)
screenshot("launch")

print("Check /tmp/launch.png — look for 'Continue editing' popup or navigate inbox")
print("If popup visible: running Edit tap...")
adb("input tap 950 200")   # Edit button on stale draft popup
time.sleep(2)
screenshot("editor")

# Add audio via uiautomator2
print(f"[3] Adding audio: {audio_term}")
import uiautomator2 as u2
d = u2.connect(DEVICE_IP)

d(text="Add sound").click(); time.sleep(2)
d(description="Search").click(); time.sleep(1)
d(className="android.widget.EditText").set_text(audio_term); time.sleep(2)
d.click(747, 665); time.sleep(1)   # 2nd carousel thumbnail
d.click(860, 870); time.sleep(1)   # checkmark
d.swipe(600, 1500, 600, 2500); time.sleep(1)

# Post
print("[4] Posting...")
d(text="Next").click(); time.sleep(2)
d(text="Post").click(); time.sleep(3)

screenshot("confirm")
print("Done. Check /tmp/confirm.png to verify post went live.")
```

---

## Cron Setup (OpenClaw)

Add three crons — one per post time. Replace times with your schedule:

```
# 9 AM post — fires at 9:10 AM to give scheduling tool time to push draft
openclaw cron add \
  --name "tiktok-morning" \
  --schedule "10 9 * * *" \
  --tz "America/Chicago" \
  --message "Post today's 9 AM TikTok draft. Run pre-flight, find draft via inbox or popup, search audio 'YOUR_AUDIO_TERM', post. Report result."
```

The cron message tells OpenClaw what to do. It reads this SKILL.md and executes.

---

## Coordinate Reference

Calibrated for **1200×2670 @ 480dpi** (SuiPlay Seeker / similar tall Android). Adjust for your device by taking a screenshot and measuring.

| Element | X | Y | Notes |
|---|---|---|---|
| Inbox tab (bottom nav) | 840 | 2560 | 4th icon in bottom bar |
| System notifications row | 600 | 490 | May shift — screenshot first |
| Edit button (draft popup) | 950 | 200 | Right side of top popup |
| 2nd carousel thumbnail | 747 | 665 | First audio search result |
| Checkmark on carousel | 860 | 870 | Confirms audio selection |

To recalibrate for your device:
```bash
adb -s YOUR_DEVICE:5555 shell screencap -p /sdcard/s.png
adb -s YOUR_DEVICE:5555 pull /sdcard/s.png /tmp/s.png
# Open and measure coordinates
```

---

## Gotchas

### Do Not Disturb Silently Kills Notifications
**#1 failure mode.** DND on = Postiz notifications silently blocked = ADB can't find the draft = cron times out.

```bash
adb -s YOUR_DEVICE:5555 shell settings put global zen_mode 0
adb -s YOUR_DEVICE:5555 shell settings get global zen_mode  # must return 0
```

### Postiz "PUBLISHED" + Missing Release ID ≠ Posted
When Postiz shows `state: PUBLISHED` but `releaseId: "missing"` — Postiz fired the request but TikTok didn't confirm back. **The draft is still on the phone.** Go find it via Method 1 or 2 above. It has NOT been posted.

### React Native UI Blocks Raw ADB Taps
TikTok's audio picker list items are rendered in React Native with a transparent overlay that intercepts all touches. Raw `adb shell input tap` aimed at list items will silently miss. Use the carousel thumbnails above the list — those are native Android.

### "Navigate to Profile → Drafts" Does Not Work
Postiz UPLOAD drafts do not appear in the Profile → Drafts tab. That tab only shows manually saved drafts. Always use the TikTok notification inbox (Method 1) or the stale draft popup (Method 2).

### keyevent 26 Toggles Power — Use 224 Instead
`adb shell input keyevent 26` toggles the screen — if it's ON, it turns it OFF. Use `keyevent 224` (KEYCODE_WAKEUP) which only wakes, never sleeps.

### Always Prefix Audio Searches with the Genre/Category
Single-word searches return unrelated popular results. "nas" returns rapper Nas. "ikhlas" returns random content. Always include the category prefix:
- ✅ `surah nas`, `surah ikhlas`, `surah mulk`, `surah yasin`
- ✅ `lofi beats`, `nasheed instrumental`
- ❌ Never just `nas`, `ikhlas`, `asr` alone

### Audio Selection — Pink Checkmarks ≠ Selected
The audio results list shows pink checkmark buttons on the right of every item. These mean "available to add to favorites" — **not** that the audio is selected for your post.

**How to confirm audio is actually selected:**
- ✅ The **top bar text changes** from the old track name to the new one — this is the only reliable indicator
- ❌ The "Current sound" label at the bottom of the picker lags and may still show the old track
- ❌ Pink checkmarks on list rows = available/favorited, NOT selected
- ✅ Pink/red **highlighted row text** = that row is the active selection

### `d(description="Add sound")` May Not Exist
When the draft already has audio attached (from a previous edit session), the button is labeled as the current track name (e.g. "Heavenly Rapture"), not "Add sound". Find it via `dump_hierarchy()` and look for the track name text, then tap that element to reopen the picker.

### Audio Search — Keep Terms Short, No Spaces
`adb shell input text` URL-encodes spaces (`%20`). Use `uiautomator2`'s `set_text()` instead. Keep search terms short — two words max.

---

## Install uiautomator2

```bash
pip3 install uiautomator2 adbutils --break-system-packages
# First run auto-installs the server on the device
python3 -c "import uiautomator2 as u2; d = u2.connect('YOUR_DEVICE_IP'); print(d.info)"
```
