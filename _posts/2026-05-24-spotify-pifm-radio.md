---
layout: post
title: "Spotify FM: Because Buying a Radio was Apparently Too Easy"
category: raspberry-pi
tags: raspberry-pi spotify linux audio fm radio
---

So you want to play your Spotify playlists over an actual FM radio frequency. Not because it's practical (it's objectively not) but because you looked at a $6 Raspberry Pi Zero 2W and thought, "yeah, that should be a radio tower."

Welcome. You're here now. Let's get through this together.

## The Pitch

The idea is simple enough to sound like a lie: `spotifyd` runs as a headless Spotify Connect receiver, dumps its audio into an ALSA loopback device, and `PiFmAdv` reads that loopback and uses the Pi's GPIO pin to mathematically generate an actual 87.6 MHz RF signal. You tape a piece of copper wire to it. You tune your radio. Music plays.

It works. It genuinely works. And it will absolutely drive you insane before it does.

## The Players

- [**`spotifyd`**](https://github.com/spotifyd/spotifyd): A lean, headless Spotify Connect daemon. No Electron app, no browser, no nonsense. Just audio.
- [**`PiFmAdv`**](https://github.com/miegl/PiFmAdv): The absolute madlad software that turns a GPIO pin into an FM transmitter. Perfectly legal at low power for personal use. Don't ask me about higher power.
- **ALSA Loopback**: A virtual audio device that lets `spotifyd` play into one end and `PiFmAdv` read out the other. The audio pipe that holds this whole thing together.

## Spotifyd: Headless Spotify Without the Self-Harm

Spotifyd is written in Rust. You could compile it from source on a Pi Zero 2W. You could also watch paint dry while your CPU screams. Don't do that. Just head to the [spotifyd releases page](https://github.com/Spotifyd/spotifyd/releases) and grab the pre-built ARM binary. Drop it in place and move on with your life:

```bash
# grab the latest armhf build from the releases page, then:
sudo mv spotifyd /usr/local/bin/spotifyd
sudo chmod +x /usr/local/bin/spotifyd
```

Save this as `~/.config/spotifyd/spotifyd.conf`:

```toml
[global]
device_name = "Radify FM"
device_type = "speaker"
backend = "alsa"
device = "loopback_playback"
audio_format = "S16"
bitrate = 160
volume_normalisation = true
```

Wire it up as a systemd service at `/etc/systemd/system/spotifyd.service`:

```ini
[Unit]
Description=A spotify playing daemon
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/local/bin/spotifyd --no-daemon --config-path /home/pi/.config/spotifyd/spotifyd.conf
Restart=always
RestartSec=12
User=root

[Install]
WantedBy=default.target
```

At this point, if you open Spotify on your phone, you should see **"Radify FM"** as a Connect device. Play something. Verify audio is flowing through the loopback with `arecord -D loopback_capture -f S16_LE -r 44100 -c 2 /tmp/test.wav`. You're basically a sound engineer now.

## PiFmAdv: Actually Building the Transmitter

This one you do have to compile yourself, but it's C, not Rust. So it takes about 30 seconds instead of 3 hours. Your systemd service will point directly at the build output in your local folder, so just clone it somewhere stable and don't move it:

```bash
cd ~
git clone https://github.com/Miegl/PiFmAdv.git
cd PiFmAdv/src
make clean
make
```

If `make` explodes, you probably need:

```bash
sudo apt install -y libsndfile1-dev gcc make git alsa-utils
```

The compiled binary lives at `/home/pi/PiFmAdv/src/pi_fm_adv` — that's the path your service points at, so don't move it.

## ALSA Loopback: The Invisible Pipe That Keeps It All Together

You want to load the module and make it stick across reboots:

```bash
sudo modprobe snd-aloop
echo "snd-aloop" | sudo tee -a /etc/modules
```

Then set up `/etc/asound.conf` so you have named devices to actually reference:

```text
pcm.loopback_playback {
  type plug
  slave.pcm "hw:Loopback,0,0"
}
pcm.loopback_capture {
  type plug
  slave.pcm "hw:Loopback,1,0"
}
```

`spotifyd` writes to `loopback_playback`. `PiFmAdv` reads from `loopback_capture`. This is the handshake. Don't mess up the device names.

## The Part That Will Ruin Your Evening (The Priority Fix)

Here is where the naive version of this project falls apart. You run `arecord | pi_fm_adv` and it sounds like a robot drowning. The stream stutters. It crashes. You question your choices.

**The problem:** `spotifyd` and `PiFmAdv` are both fighting for CPU time on a tiny single-board computer. When `PiFmAdv` gets preempted mid-cycle, it can't generate the next sample on time, the buffer overruns, and the stream dies.

**The fix:** You hand `PiFmAdv` a priority sledgehammer via `chrt -f 95` (real-time FIFO scheduling, max priority), throw `arecord` at priority 90, and pad the ALSA buffers out so there's actual headroom to absorb any scheduling hiccup.

`/home/pi/pifm-broadcast.sh`:

```bash
#!/bin/bash
chrt -f 90 /usr/bin/arecord -D loopback_capture -f S16_LE -r 44100 -c 2 --period-size=1024 --buffer-size=8192 | chrt -f 95 /home/pi/PiFmAdv/src/pi_fm_adv --audio - --freq 87.6
```

```bash
sudo chmod +x /home/pi/pifm-broadcast.sh
```

Wire it up as a service at `/etc/systemd/system/pifm-broadcast.service`:

```ini
[Unit]
Description=Broadcast ALSA Output via PiFM
Requires=spotifyd.service
After=spotifyd.service

[Service]
ExecStart=/home/pi/pifm-broadcast.sh
Restart=always
RestartSec=15
User=root

[Install]
WantedBy=default.target
```

## WiFi Power Saving: The Silent Killer

You will get this all working, walk away, come back an hour later, and the stream is dead. The service looks fine. Spotifyd looks fine. Nothing in the logs.

WiFi power saving. The Pi idles the WiFi radio when traffic is light, Spotify's keepalive gets dropped, and the stream quietly evaporates.

```bash
sudo iw wlan0 set power_save off
```

Add this to `/etc/rc.local` before `exit 0` to survive reboots:

```text
/sbin/iw wlan0 set power_save off
```

You're welcome. That one took a while.

## The Antenna (The Fun Part)

Cut **32.0 inches** of solid-core copper wire. Attach it to **GPIO 4 (Pin 7)**. Stand it up vertically.

That length is not arbitrary: it's exactly 1/4 wavelength for 87.6 MHz. This minimizes VSWR reflection and maximizes your range. You've effectively built a tuned monopole antenna out of a piece of wire you probably had in a junk drawer.

Tune any FM radio to 87.6 MHz. :musical_note:

## Fire It Up

```bash
sudo systemctl daemon-reload
sudo systemctl enable spotifyd pifm-broadcast
sudo systemctl start spotifyd pifm-broadcast
```

Check status:

```bash
sudo systemctl status spotifyd
sudo systemctl status pifm-broadcast
```

Open Spotify. Select **"Radify FM"** as your output device. Watch your FM radio play it back.

## The Recap

Three things that will save you hours of debug suffering:

1. `chrt -f 95` on `PiFmAdv`. Non-negotiable. Without this, you have a broken stream.
2. Big ALSA buffers (`--period-size=1024 --buffer-size=8192`). Give the pipeline room to breathe.
3. Disable WiFi power saving. Or your stream dies when you stop staring at it.

The hardware cost is somewhere around $10-15 for the Pi Zero 2W and a piece of wire. The time cost is whatever rabbit holes you fell into. Totally worth it.

Happy Broadcasting. :radio:
