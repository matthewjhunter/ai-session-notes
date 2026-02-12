# ai-session-notes

Transcribe multi-speaker audio and video recordings with speaker diarization. Built for tabletop RPG sessions but works for any multi-speaker recording (meetings, interviews, podcasts).

The pipeline uses [WhisperX](https://github.com/m-bain/whisperX) for transcription with word-level alignment, and [pyannote](https://github.com/pyannote/pyannote-audio) for speaker diarization.

## Scripts

### `bin/extract-audio`

Extracts the native audio stream from a video file without transcoding. Uses `ffprobe` to detect the audio codec and maps it to the correct container format.

```bash
extract-audio session.mkv
# -> session.opus (or .m4a, .mp3, etc. depending on source codec)
```

Supported codec mappings: AAC, MP3, Opus, Vorbis, FLAC, PCM, ALAC, AC3, EAC3, WMA. Unknown codecs fall back to using the codec name as extension.

### `bin/transcribe`

Transcribes an audio or video file with speaker diarization. Automatically calls `extract-audio` if given a video file.

```bash
transcribe session.mkv      # video input (extracts audio first)
transcribe session.opus     # audio input (transcribes directly)
# -> session.txt
```

Both scripts are idempotent: if the output file already exists, they print its path and exit.

### `bin/parse-roll20-log`

Parses a saved Roll20 chat log HTML page into timestamped, structured text. Extracts player names, character names, ability/weapon names, and roll results.

```bash
# Parse entire campaign log
parse-roll20-log "Chat Log for My Campaign.html" > full-log.txt

# Filter to a single session by date
parse-roll20-log "Chat Log for My Campaign.html" --session 2026-02-10 > session.txt
```

To save the HTML: open your Roll20 game, click the chat archive button (speech bubble icon in the chat tab), then save the page (Ctrl+S) as a complete HTML file.

No external Python dependencies required — uses only the standard library. Output format:

```
[February 10, 2026 9:06PM] nikki: Irulan: longsword (+6): 13 20 10
[February 10, 2026 9:14PM] Matthew: Bancroft Barleychaser: Light (+3): 7 7
[February 10, 2026 9:55PM] Matthew: Bancroft Barleychaser: Strength (3): 22 8
```

## Dependencies

### System packages

```bash
sudo apt install ffmpeg
```

`ffmpeg` and `ffprobe` are used for audio codec detection and stream extraction.

### Python (WhisperX)

WhisperX runs in an isolated venv. Create it and install:

```bash
python3 -m venv ~/.local/share/whisperx
~/.local/share/whisperx/bin/pip install whisperx
```

The scripts expect `whisperx` at `~/.local/share/whisperx/bin/whisperx`. Adjust the path in `bin/transcribe` if you install it elsewhere.

### Hugging Face token

Speaker diarization requires accepting the pyannote model licenses and providing an access token:

1. Create an account at [huggingface.co](https://huggingface.co)
2. Accept the license for [pyannote/speaker-diarization-3.1](https://huggingface.co/pyannote/speaker-diarization-3.1)
3. Accept the license for [pyannote/segmentation-3.0](https://huggingface.co/pyannote/segmentation-3.0)
4. Create an access token at [huggingface.co/settings/tokens](https://huggingface.co/settings/tokens)
5. Export it in your shell:

```bash
export HF_TOKEN="hf_your_token_here"
```

### PyTorch 2.7+ compatibility patch

PyTorch 2.7 changed `torch.load` to default to `weights_only=True`, which breaks pyannote's model loading. Until upstream fixes this, patch `lightning_fabric`:

```bash
VENV=~/.local/share/whisperx
CLOUD_IO="$VENV/lib/python3.*/site-packages/lightning_fabric/utilities/cloud_io.py"

# Add weights_only=False default for local file loads
sed -i '/^    fs = get_filesystem/i\    if weights_only is None:\n        weights_only = False' $CLOUD_IO
```

This sets `weights_only=False` for local checkpoint files only. The security implications are minimal since pyannote models are downloaded from Hugging Face's authenticated model hub.

## GPU support

### NVIDIA (CUDA)

The fastest path. Install PyTorch with CUDA support and change `--device cpu --compute_type int8` to `--device cuda --compute_type float16` in `bin/transcribe`.

```bash
~/.local/share/whisperx/bin/pip install torch torchaudio --index-url https://download.pytorch.org/whl/cu128
```

### AMD (ROCm)

PyTorch works with ROCm, but WhisperX's transcription engine (`faster-whisper` / `ctranslate2`) is CUDA-only. The transcription step runs on CPU; the diarization and alignment steps can use the GPU.

```bash
~/.local/share/whisperx/bin/pip install torch torchaudio --index-url https://download.pytorch.org/whl/rocm6.2.4
```

Leave `--device cpu` in the transcribe script. The pyannote diarization pipeline will detect and use the ROCm GPU automatically.

### CPU only

Works out of the box. A modern CPU (Zen 5, recent Intel) handles a 2-hour recording in roughly 25-50 minutes with `--compute_type int8`.

## Installation

```bash
git clone https://github.com/matthewjhunter/ai-session-notes.git
cp ai-session-notes/bin/* ~/bin/
chmod +x ~/bin/extract-audio ~/bin/transcribe
```

Ensure `~/bin` is in your `PATH`.

## Performance

Tested on a 114-minute D&D session recording (AMD Ryzen AI MAX+ 395, Zen 5, 16 cores):

| Phase | Time |
|-------|------|
| Audio extraction | < 5 seconds |
| Transcription (CPU, int8, large-v3) | ~23 minutes |
| Diarization | ~5-10 minutes |

NVIDIA GPU users can expect 3-5x faster transcription times.

## License

MIT
