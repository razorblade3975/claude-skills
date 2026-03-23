# Bell Notification Setup

Play a sound when Claude Code finishes a task and waits for user input.

## Prerequisites

- **PulseAudio** configured (default on WSL2 with audio passthrough)
- `paplay` installed (`sudo apt-get install pulseaudio-utils`)
- `python3` (for generating the sound file)

## Setup

### 1. Generate the bell sound

```bash
python3 -c "
import wave, struct, math

sample_rate = 44100
duration = 0.8
samples = int(sample_rate * duration)

# C major chime: C5 + E5 + G5 + C6 with gentle attack and decay
harmonics = [
    (523.25, 0.5, 4.0),   # C5 fundamental
    (1046.5, 0.25, 6.0),  # C6 octave
    (659.25, 0.15, 5.0),  # E5 major third
    (783.99, 0.10, 7.0),  # G5 fifth
]

with wave.open('$HOME/.claude/bell.wav', 'w') as f:
    f.setnchannels(1)
    f.setsampwidth(2)
    f.setframerate(sample_rate)
    for i in range(samples):
        t = i / sample_rate
        attack = min(1.0, t / 0.02)
        value = 0
        for freq, amp, decay in harmonics:
            value += amp * math.exp(-t * decay) * math.sin(2 * math.pi * freq * t)
        value = int(32767 * 0.35 * attack * value)
        value = max(-32767, min(32767, value))
        f.writeframes(struct.pack('<h', value))
"
```

Or skip this step and place your own WAV file at `~/.claude/bell.wav`.

### 2. Test the sound

```bash
paplay ~/.claude/bell.wav
```

If `paplay` is not available, `ffplay` works too:

```bash
ffplay -nodisp -autoexit -loglevel quiet ~/.claude/bell.wav
```

### 3. Add the Stop hook

Add this to `~/.claude/settings.json` (merge with existing settings):

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "paplay /home/YOUR_USER/.claude/bell.wav &",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

Replace `/home/YOUR_USER/` with your actual home directory (use `echo $HOME`).

### 4. Verify

Open `/hooks` in Claude Code to reload config, or restart the session. The bell plays after every Claude response.

## Customization

- **Change the sound:** Replace `~/.claude/bell.wav` with any WAV file. No settings change needed.
- **Switch player:** Replace `paplay` with `ffplay -nodisp -autoexit -loglevel quiet` or `aplay` (if ALSA is available).
- **Disable:** Remove the `Stop` hook from settings, or delete the entry via `/hooks`.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| No sound on WSL2 | Ensure PulseAudio/PipeWire is bridged to Windows host |
| `aplay` fails with "Unknown PCM default" | No ALSA device — use `paplay` or `ffplay` instead |
| Hook doesn't fire | Open `/hooks` to reload, or restart Claude Code |
