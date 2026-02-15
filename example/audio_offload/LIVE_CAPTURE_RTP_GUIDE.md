# Live Capture and RTP Migration Guide (AM62D EVM)

This guide explains how to evolve the current `rpmsg_audio_offload_example` from file-based playback (`.wav`) to:

1. **Live analog capture** from board LINE-IN, and playback to LINE-OUT.
2. **RTP (SIP/VoIP) input** over Ethernet, then the same ARM/DSP processing and output path.

---

## 1) Current behavior in example (baseline)

The current thread in `rpmsg_audio_example.c`:
- opens `SAMPLE_AUDIO_FILE` with libsndfile,
- reads PCM frames from the wav,
- writes frames to `lbuf.data_buf`,
- runs ARM or DSP processing,
- plays back using ALSA playback.

So only the **source stage** is file-based; DSP/ARM processing and playback are already reusable.

---

## 2) Migration strategy (recommended)

### Phase A — Replace WAV source with live ALSA capture

Keep the processing loop almost unchanged and replace only the input read path:

- Replace:
  - `sf_open(...)` / `sf_readf_short(...)`
- With:
  - `snd_pcm_open(..., SND_PCM_STREAM_CAPTURE, ...)`
  - `snd_pcm_set_params(...)`
  - `snd_pcm_readi(...)`

Frame flow should become:

`LINE-IN (capture) -> lbuf.data_buf -> process_on_dsp/process_on_arm -> LINE-OUT (playback)`

### Phase B — Add a source abstraction

Create a source-agnostic interface:

```c
int audio_source_init(const AppConfig *cfg);
int audio_source_read(int16_t *dst_interleaved, int frames);
void audio_source_close(void);
```

Supported source modes:
- `wav`
- `capture`
- `rtp`

Then `run_audio_processing_thread()` always calls `audio_source_read()` regardless of input type.

### Phase C — RTP/SIP source

For RTP audio path:

`UDP socket -> RTP depacketize -> (optional SRTP decrypt) -> codec decode -> PCM frame align -> processing`

Add jitter/sequence handling before decode.

---

## 3) Config additions to make

Extend `dsp_offload.cfg` and parser with:

- `AUDIO_SOURCE_MODE=wav|capture|rtp`
- `PCM_CAPTURE_DEVICE=hw:0,0` (or board-specific card/device)
- `PCM_PLAYBACK_DEVICE=default` (already present as `PCM_DEVICE`; can keep/rename consistently)
- `RTP_LOCAL_IP=0.0.0.0`
- `RTP_LOCAL_PORT=5004`
- `CODEC=pcmu|pcma|opus|l16`
- `SRTP_ENABLE=0|1`
- `JITTER_BUFFER_MS=40`

Tip: keep defaults equal to existing behavior (`AUDIO_SOURCE_MODE=wav`) so current flow remains stable.

---

## 4) Board bring-up checklist for live LINE-IN/LINE-OUT

1. Verify ALSA cards/devices on target:
   - `aplay -l`
   - `arecord -l`
2. Verify capture route and level with `alsamixer` (input mux, gain, mute state).
3. Quick raw loopback sanity:
   - capture: `arecord -D <capture_dev> -f S16_LE -r 48000 -c <N> /tmp/test.wav`
   - playback: `aplay -D <playback_dev> /tmp/test.wav`
4. Match application frame format with DSP assumptions:
   - sample rate,
   - channels,
   - interleaved S16_LE.

---

## 5) RTP/SIP end-goal notes

- SIP is signaling; media arrives as RTP/SRTP.
- For compressed payloads (e.g., G.711/Opus), decode to PCM before writing to `lbuf.data_buf`.
- If incoming RTP stream differs from DSP format (e.g., mono/8k), add resample/channel mapping before processing.
- Add packet-loss behavior (PLC, concealment) in the RTP input stage, not in DSP offload stage.

---

## 6) Minimal implementation order

1. Add `AUDIO_SOURCE_MODE` and keep default `wav`.
2. Implement `capture` mode with ALSA `snd_pcm_readi()`.
3. Validate end-to-end on EVM line-in -> line-out in ARM mode.
4. Switch to DSP mode and re-validate.
5. Add RTP mode (without SRTP first).
6. Add SRTP and jitter/robustness features.

This sequence keeps risk low and preserves a known-good path at each step.
