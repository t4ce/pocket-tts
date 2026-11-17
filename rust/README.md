# pocket-tts, the standalone binary

A port of the `pocket-tts` CLI (`generate`, `serve`, `export-voice`) that runs without Python,
built on the [xn-ptts](https://github.com/gradium-ai/xn-ptts) implementation of the model,
through [a fork](https://github.com/gabrieldemarmiesse/xn-ptts/tree/pocket-tts-small-models)
that adds what pocket-tts models need. The release workflow (`.github/workflows/rust-release.yml`) attaches its builds to every GitHub
release.

```bash
cargo build --release
./target/release/pocket-tts generate --text "Hello world."
```

Building needs CMake and a C++ compiler, for the SentencePiece tokenizer that most community
models use. x86 builds target x86-64-v3 (AVX2), see `.cargo/config.toml`. For a CPU without AVX2,
build with `RUSTFLAGS="-C target-cpu=x86-64-v2"`, as the `-v2` archives of the releases are:
`--quantize` is slower there (xn picks its int8 kernels at compile time), the default f32 mode is
not.

## Differences with the Python CLI

### Optional pronunciation dictionary (local Rust extension)

Add `pronunciation_overrides` to a custom model YAML selected with `--config`:

```yaml
pronunciation_overrides:
  TRUEOS: "True O S"
  CPU: "C P U"
```

Entries match whole words case-insensitively (`CPU` and `cpu` both match), before
tokenization and sentence chunking. Other words and punctuation are preserved;
`CPU2` and `my_cpu` do not match `CPU`. Keys contain letters, digits or underscores;
empty values and duplicate case-insensitive keys are rejected. Replacements are
applied once. The dictionary defaults to empty and affects only synthesized text,
in both CLI generation and server requests. Letter names follow the selected
language model; do not apply an English spelling dictionary to every language.

This is a local Rust extension, not SSML or a phoneme API. The Python CLI does not
accept this additional YAML field. Existing language configs are unchanged.

### Backend differences

- Runs on the CPU only (`--device cpu`), and `--quantize` uses xn's int8 weights.

- `serve --reload` is accepted and ignored. The server answers `/`, `/health` and `/tts` as the
  Python one does, without FastAPI's `/docs`.
- The noise is drawn from a different generator, so the audio differs sample by sample from the
  Python package's, as two runs with different seeds would. The seed is fixed, so a command
  always gives the same audio; `POCKET_TTS_SEED` sets another one.
- The tensor operations use 4 threads by default; set `RAYON_NUM_THREADS` to change it.
