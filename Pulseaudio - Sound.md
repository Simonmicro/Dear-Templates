---
title: Pulseaudio - Sound
summary: Improve linux sound quality
type: blog
banner: "/img/dear-templates/default.jpg"
---

[See here](https://medium.com/@gamunu/enable-high-quality-audio-on-linux-6f16f3fe7e1f)

```
default-sample-format = float32le
default-sample-rate = 48000
alternate-sample-rate = 44100
default-sample-channels = 2
default-channel-map = front-left,front-right
;default-fragments = 2
;default-fragment-size-msec = 125
resample-method = soxr-vhq
enable-lfe-remixing = no
high-priority = yes
nice-level = -11
realtime-scheduling = yes
realtime-priority = 9
rlimit-rtprio = 9
daemonize = no
```