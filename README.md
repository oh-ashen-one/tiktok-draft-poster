# tiktok-draft-poster

OpenClaw skill: automatically post TikTok carousel drafts from a real Android phone via ADB — finds the draft, adds audio, posts. No TikTok API needed.

## The Problem

You use a scheduling tool (Postiz, Buffer, etc.) to push carousel posts to TikTok drafts. But TikTok's `UPLOAD` mode just saves to drafts — you still have to manually open the phone, find the draft, add audio, and hit Post. Every. Single. Day.

## The Fix

This skill automates the entire post-draft flow:
1. Finds the draft on the phone (via TikTok's in-app inbox or the stale draft popup)
2. Opens the audio picker
3. Searches for and selects the right audio
4. Taps Next → Post

Your scheduling tool pushes the draft. OpenClaw handles everything after that.

## Setup

See `SKILL.md` for full configuration — device IP, audio search terms, cron setup.

## Requirements

- Android phone with TikTok installed and logged in
- ADB WiFi enabled on the phone (Settings → Developer Options → Wireless Debugging)
- `adb` installed on your machine: `brew install android-platform-tools`
- `uiautomator2` Python library: `pip3 install uiautomator2`
- OpenClaw (to run as a scheduled cron)

## Cost

- Model: Haiku (cheapest tier)
- ~$0.001–0.003 per run
- Runs 3× daily = ~$0.01/day

## Schedule

Runs on a cron you define — typically 10 minutes after each scheduled post time (to give the scheduling tool time to push the draft first).

## License

MIT
