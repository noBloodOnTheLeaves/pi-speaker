---
summary: "Bluetooth speaker and OpenClaw audio on Raspberry Pi"
read_when:
  - Using a Bluetooth speaker with OpenClaw on a Pi
  - Playing TTS, alarms, or announcements on a Pi speaker
title: "Raspberry Pi audio (Bluetooth speaker)"
---

# Raspberry Pi audio output (Bluetooth speaker)

Use a Bluetooth speaker as the Pi's default audio output so OpenClaw can play TTS, alarms, and announcements through it. Do the steps in order and confirm each phase works before moving on.

## Bluetooth speaker setup

Goal: Pair the speaker and set it as the system default output.

### Prerequisites

- **OS**: Raspberry Pi OS (64-bit), Desktop or Lite. On Lite, use SSH and CLI only.
- **Bluetooth**: Pi 5 has built-in Bluetooth. If you disabled it for performance (see [Raspberry Pi](/platforms/raspberry-pi) "Reduce Memory Usage"), re-enable it:

```bash
sudo systemctl enable bluetooth && sudo systemctl start bluetooth
```

### Audio stack: PulseAudio or PipeWire

Raspberry Pi OS (Bookworm and later) use **PipeWire**. Check which you have:

- `wpctl status` — if it works, you have PipeWire.

Install Bluetooth audio support:

- **PipeWire**: `sudo apt install -y pipewire pipewire-pulse wireplumber` and ensure the session is running. Bluetooth is usually handled by WirePlumber.

### Pair and connect the speaker

1. Put the speaker in **pairing mode** (see its manual).
2. On the Pi, open `bluetoothctl`:
   - `power on`
   - `scan on` (wait until you see your speaker's name or MAC)
   - `pair <MAC>`
   - `trust <MAC>`
   - `connect <MAC>`
   - `exit`

On Desktop you can use the Bluetooth tray to pair and connect instead.

### Set Bluetooth as default output

**PulseAudio:**

```bash
pactl list short sinks
# Note the sink name for your Bluetooth device (e.g. bluez_sink.XX_XX_XX_XX_XX_XX.a2dp_sink)
pactl set-default-sink <sink-name>
```

To keep it after reboot, add the same `set-default-sink` command to a user startup script or `~/.config/pulse/default.pa`.

**PipeWire:**

```bash
wpctl status
# Find the Bluetooth sink under "Sinks"
wpctl set-default <sink-id>
```

### Verify playback

Play a test sound to the default output:

- **PulseAudio**: `paplay /usr/share/sounds/alsa/Front_Center.wav`
- **PipeWire**: `pw-play /usr/share/sounds/alsa/Front_Center.wav` or `paplay` if pipewire-pulse is installed

If that file is missing: `sudo apt install -y alsa-utils` then `speaker-test -t wav -c 1 -l 1`.

Confirm sound comes from the **Bluetooth speaker**. If not, check: `bluetoothctl` shows `Connected: yes`, default sink is the Bluetooth sink, and no other app has exclusive access.

---

## OpenClaw playing through the Pi speaker

Play TTS (or any audio file) on the Pi's default output so you hear it on the Bluetooth speaker.

### How it works

- OpenClaw's **TTS** tool and `/tts audio` produce an audio file on the **gateway host** (your Pi when the gateway runs there). See [Text-to-speech](/tts).
- The **bash** tool can run shell commands on the gateway. Use it to run `paplay <path>` or `pw-play <path>` so playback goes to the default sink (your Bluetooth speaker).
- There is no built-in "play on local speaker" action; use the bash tool (or the **pi-speaker** skill) to play the file.

### Gateway user and default sink

If OpenClaw runs as a **systemd service** under a different user (e.g. `openclaw`), that user's PipeWire session must have the Bluetooth sink as default. Either:

- Run the gateway as the **same** user that did Phase 1, or
- Configure a default sink for the service user and ensure they have an audio session at startup (e.g. `loginctl enable-linger` for that user and set default sink in their session).

---

## Alarms and scheduled announcements

### On-demand via chat

Ask in Telegram (or another channel): "Give me a short news summary and play it on the Pi speaker." The agent uses the TTS tool and then the bash tool (or pi-speaker skill) to play the file.

### Scheduled (cron + script)

To run an alarm or daily summary at a fixed time:

1. Create a script on the Pi that generates TTS (or uses a fixed message) and plays it with `paplay`/`pw-play` (or the wrapper script). The script can call OpenClaw's TTS via a small Node script using the same config, or use a pre-generated file.
2. Run that script from **cron** at the desired time. The cron job must run in a context that has access to the same default audio sink (same user as Phase 2, or set `PULSE_SERVER` / PipeWire socket so the cron user has an audio session).

Example cron entry (run at 7:00 as the gateway user):

```cron
0 7 * * * /home/user/bin/openclaw-morning-briefing.sh
```

The briefing script would generate TTS (or fetch a summary and generate TTS), then call `paplay`/`pw-play` or `openclaw-speaker-play.sh` with the resulting file.
Generated files most likely are in /tmp/openclaw folder.
Don't forget give execute permission to the script `chmod +x openclaw-speaker-play.sh`

## TODO:
- [ ] set up different voices for TTS
