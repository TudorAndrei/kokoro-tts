---
name: kokoro-tts
description: Use the kokoro-tts CLI to convert text, EPUB, or PDF files to speech via the Kokoro ONNX model. Trigger when the user asks to generate audio/speech from text, narrate a file, create an audiobook, stream TTS to speakers, list voices or languages, blend voices, or convert .txt/.epub/.pdf into .wav/.mp3. Do not use for Python API work against `kokoro_onnx` directly — this skill covers the CLI wrapper only.
---

# kokoro-tts

Command-line text-to-speech using the Kokoro ONNX model. Produces `.wav` or `.mp3` files, streams audio to the default output device, and handles chaptered input (EPUB/PDF) with resumable chunked output.

Install: `uv tool install kokoro-tts` (or `pipx install kokoro-tts`). Verify with `kokoro-tts --version`.

## First-time setup — model files

The CLI loads its model and voices file from the XDG data directory (`$XDG_DATA_HOME/kokoro-tts`, or `~/.local/share/kokoro-tts` when unset). If either file is missing, every invocation exits with an error pointing at `kokoro-tts download`.

Before any conversion, ensure the models are present:

```bash
kokoro-tts download
```

Idempotent: skips files that already exist. Only two files are needed, totaling ~350 MB:

- `kokoro-v1.0.onnx`
- `voices-v1.0.bin`

If the user has the files in a custom location, pass `--model <path>` and `--voices <path>` on each invocation instead of running `download`.

## Invocation

```
kokoro-tts <input> [output] [options]
```

- `<input>`: `.txt`, `.epub`, `.pdf`, or `-` / `/dev/stdin` (Linux/macOS) / `CONIN$` (Windows) for stdin.
- `[output]`: `.wav` or `.mp3`. Optional when `--stream` or `--split-output` is used. The extension must match `--format`.

### Key options

| Flag                   | Meaning                                                              |
|------------------------|----------------------------------------------------------------------|
| `--stream`             | Play audio to speakers instead of writing a file                     |
| `--speed <float>`      | Playback speed (default `1.0`)                                       |
| `--lang <code>`        | Language code (default `en-us`)                                      |
| `--voice <name>`       | Voice id, or two-voice blend like `af_sarah:60,am_adam:40`           |
| `--split-output <dir>` | Write per-chunk files per chapter (resumable)                        |
| `--format wav\|mp3`    | Audio format (default `wav`)                                         |
| `--merge-chunks`       | Merge the chunk files under `--split-output` into one per chapter    |
| `--help-voices`        | Print voices the installed model supports                            |
| `--help-languages`     | Print languages the installed model supports                         |
| `--debug`              | Verbose progress, including auto-chunk-resize notices                |
| `--model <path>`       | Override model path (default: XDG data dir)                          |
| `--voices <path>`      | Override voices path (default: XDG data dir)                         |

Subcommands: `download` (shown above), `--merge-chunks` (see "Chaptered input" below).

## Languages

`en-us`, `en-gb`, `fr-fr`, `it`, `ja`, `cmn`. For the authoritative list from the installed model run `kokoro-tts --help-languages`.

## Voices

Voice ids encode language + gender in their prefix:

- `af_*` / `am_*` — en-us female / male (e.g. `af_sarah`, `am_adam`)
- `bf_*` / `bm_*` — en-gb female / male (e.g. `bf_alice`, `bm_george`)
- `ff_*` — fr-fr (e.g. `ff_siwis`)
- `if_*` / `im_*` — Italian
- `jf_*` / `jm_*` — Japanese
- `zf_*` / `zm_*` — Mandarin (`cmn`)

**Pair voices with their language** — a `jf_*` voice with `--lang en-us` will fail. Run `kokoro-tts --help-voices` for the full list the installed model supports.

### Voice blending

Pass two voices, optionally with weights that sum to 100:

```bash
kokoro-tts in.txt out.wav --voice "af_sarah:60,am_adam:40"
kokoro-tts in.txt out.wav --voice "af_sarah,am_adam"        # 50/50
```

Exactly two voices. Weights are normalized if they don't sum to 100.

## Chaptered input (EPUB / PDF)

EPUB chapters come from the book's TOC; PDF chapters come from the TOC first, falling back to markdown heading extraction. For both, `--split-output <dir>` is the recommended flow:

1. Produces `<dir>/chapter_001/chunk_001.wav`, `chunk_002.wav`, … plus an `info.txt` per chapter.
2. Interrupting (Ctrl+C) and re-running the same command resumes — existing chunks are skipped.
3. After all chunks are generated, merge them into per-chapter files:

```bash
kokoro-tts book.epub --split-output ./chunks/ --format mp3 --voice af_sarah
kokoro-tts --merge-chunks --split-output ./chunks/ --format mp3
```

`--merge-chunks` reads chapter titles from each `info.txt` and writes `<title>.mp3` in the root of `--split-output`.

PDF TOC extraction prompts for Enter before processing — pass input via a pipe or `yes ''` if non-interactive.

## Common recipes

Text → WAV with a specific voice and faster pace:
```bash
kokoro-tts book.txt book.wav --voice af_sarah --speed 1.1
```

Stream to speakers, no file:
```bash
kokoro-tts notes.txt --stream --voice am_adam
```

From stdin:
```bash
echo "Hello world" | kokoro-tts - greeting.wav --voice af_sarah
```

Japanese audiobook, chunked:
```bash
kokoro-tts book.epub --split-output ./ja_chunks/ --format mp3 --lang ja --voice jf_alpha
kokoro-tts --merge-chunks --split-output ./ja_chunks/ --format mp3
```

Blend for a neutral narrator voice:
```bash
kokoro-tts report.pdf report.wav --voice "af_sarah:50,am_adam:50"
```

## Failure modes to handle

- **"Required model files are missing"** → run `kokoro-tts download`. Don't ask the user to `wget` manually; the old README instructions are out of date.
- **"Unsupported voice" / "Unsupported language"** → run `kokoro-tts --help-voices` or `--help-languages` and pick from the printed list.
- **Output extension mismatch** → the output filename must end in `.{format}`. Either fix the filename or pass a matching `--format`.
- **"index 510 is out of bounds"** (phoneme overflow) → the CLI auto-splits and retries; no user action needed. Use `--debug` to see the split sizes.
- **Interactive voice picker appears** when no `--voice` is passed and stdin is a TTY. In scripts, always pass `--voice` explicitly (or pipe from stdin, which auto-selects `af_sarah`).

## Non-goals

- Not a server — `--stream` plays locally on the machine running the command and blocks until done.
- Not for realtime low-latency applications — chunk processing takes seconds per chunk.
- Does not translate — input text must already be in the target language that matches `--lang` and the chosen voice.
