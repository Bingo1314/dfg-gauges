# dfg · AI pipeline gauges

**English** | [简体中文](README.md)

[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.9%2B-blue.svg)](pyproject.toml)
[![Dependencies](https://img.shields.io/badge/core%20deps-none-brightgreen.svg)](pyproject.toml)
[![Tests](https://img.shields.io/badge/tests-32%20passing-brightgreen.svg)](tests/test_gauges.py)

**Four gauges for AI content pipelines: prompt adherence, LLM channel liveness,
argument reconciliation, and character consistency. Measure only — never generate.**

AI video pipelines rarely fail loudly. They fail *quietly*: the render finishes,
the progress bar reaches 100%, and the output is wrong in a way nobody catches.

- You built a command with `--force`, but the script never registered that flag.
  It explodes at launch — after you had queued a whole episode.
- Your fallback provider is hardcoded, and that model was decommissioned months ago.
  Every failure now burns one extra round-trip before anyone notices.
- The prompt asked for "a bronze ring". There is no ring. Nothing warns you.
- Five views of one character look fine one by one; measured, the side view
  drifted by 0.15.

This package turns those four things into **a number or a pass/fail you can
re-check**, with zero GPU, zero API keys, and zero network for the first three.

## Install

```bash
pip install -e .                 # the three core gauges: pure stdlib, no deps
pip install -e ".[clip]"         # optional: consistency via CLIP (torch + transformers)
pip install -e ".[face]"         # optional: real-face backend (insightface)
```

Python ≥ 3.9.

## 60-second smoke test (needs no assets)

```bash
pip install -e .
dfg selftest                                                     # 24 checks, zero deps

dfg check --script examples/demo_stage.py --cmd "--force --reroll 2"
dfg check --script examples/demo_stage.py --cmd "--force --nope 3"   # fails on purpose, exit 1

dfg health --ledger examples/demo_ledger.jsonl --pick moonshot,qwen,deepseek,ghost
dfg adherence --prompt "rainy night, wooden bridge, oil-paper umbrella, red coat, silver ring" \
              --storyboard examples/demo_storyboard.json
```

`examples/` holds three toy files — a fake stage script, a call ledger, and a storyboard —
purely so every command above reproduces the output below on your own machine.
`consistency` is the only one that needs your images; it comes last.

## The four commands

### 1. `dfg check` — will this command actually run?

```bash
$ dfg check --script examples/demo_stage.py --cmd "--force --reroll 2"
[PASS] demo_stage.py（2 个 flag 对上）

$ dfg check --script examples/demo_stage.py --cmd "--force --nope 3"
[FAIL] demo_stage.py 不认 ['--nope']
$ echo $?
1
```

> CLI messages are currently in Chinese (the tool grew up in a Chinese-language
> pipeline). `[PASS]`/`[FAIL]`, the exit codes, and everything under `--json`
> are language-neutral, so CI usage is unaffected. See *Known limitations*.

It parses the script's registered `add_argument` calls with `ast` and reconciles
them against the command you built. **The target program is never executed** —
no side effects. Drop it into CI or `pre-commit` and this class of bug disappears.

### 2. `dfg health` — is the provider I'm using still alive?

```bash
$ dfg health --ledger examples/demo_ledger.jsonl --pick moonshot,qwen,deepseek,ghost
活 2 / 死 1 / 未知 0
  ✓ deepseek           alive
  ✗ moonshot           dead
  ✓ qwen               alive

兜底候选（跳过死通道）：['qwen', 'deepseek', 'ghost']
```

Note `ghost` in the fallback list: it has **no samples**, so it is not killed.
A brand-new channel should not be condemned before its first call.

Two sources: the health report (preferred if < 24h old) and the call ledger
(JSONL, last N entries). The rule is deliberately conservative:
**ever succeeded = alive / 3+ consecutive failures = dead / no samples = unknown,
pessimistically treated as alive** — a brand-new channel should not be killed
before its first call. `--assert` exits 1 when a dead channel is still a
candidate, which makes it a CI assertion.

### 3. `dfg adherence` — did the model follow the prompt?

```bash
$ dfg adherence --prompt "rainy night, wooden bridge, oil-paper umbrella, red coat, silver ring" \
                --storyboard examples/demo_storyboard.json
[提示词] 遵从度 3/5（60%） · 未落地：red coat、silver ring
```

That output is real: `examples/demo_storyboard.json` genuinely has no red coat
and no silver ring. Multi-word English phrases stay whole (`red coat` is one
term, not `red` + `coat`) — splitting them would let "red umbrella" satisfy "red".

It splits the prompt into key terms (quoted proper nouns first, then phrases cut
on punctuation and function words) and looks for **verbatim** occurrences in the
final text, then reports the hit rate. An empty prompt means *not applicable* —
it will never invent a failure.

> **Verbatim only, no semantic similarity.** Semantic scoring needs a model and
> cannot be re-checked by a human. "Every element is literally present" is a hard
> floor. A high hit rate does not mean *good*, only *on-topic* — so it warns, it
> does not block.

### 4. `dfg consistency` — is this the same person?

```bash
$ dfg consistency --anchor cast/hero/anchor.png --views cast/hero/views --backend hf
[consistency] backend=hf median=0.872 min=0.755 (4 images) ⚠ drift · below 0.8: full.png
   full.png                 0.755
   side.png                 0.865
   three_quarter.png        0.883
   back.png                 0.905
```

**Scores are only comparable within the same backend and preprocessing.** The
same anime character scores 0.86 on CLIP-L, 0.92 on CLIP-H, and doesn't apply at
all to ArcFace. So every result carries its `backend` and `floor`:

| backend | good for | not for | default strong-consistency line |
|---|---|---|---|
| `hf` (OpenAI CLIP-L) | anime / 3D / realistic full body | fine facial differences | 0.80 |
| `comfy` (CLIP-ViT-H-14 laion2B) | same lineage as your ComfyUI renders | needs local weights | 0.70 |
| `arcface` (InsightFace) | real human faces | anime, back views, full body | 0.45 |

#### Measuring full-body / back views? Use `--floor auto`

Applying a flat 0.70 to full-body views marks **a quarter of your cards** as
drifted. The model isn't broken — it's physics: full-body and back views vary
the most in composition, so global CLIP features get diluted by background and
pose. Measured on 419 real character cards (comfy backend):

| view | median | p5 | false-alarm rate at flat 0.70 | per-view floor |
|---|---|---|---|---|
| three_quarter | 0.872 | 0.760 | 0.5% | 0.70 |
| side | 0.865 | 0.776 | 1.2% | 0.70 |
| full | 0.755 | 0.547 | **24.8%** | 0.48 |
| back | 0.812 | 0.593 | **16.5%** | 0.50 |

```bash
$ dfg consistency --anchor a.png --views v/ --backend comfy --floor auto
[consistency] backend=comfy median=0.833 min=0.683 (4 images) ✅ consistent
    full.png           0.683  (floor 0.48)
    side.png           0.832  (floor 0.7)
    back.png           0.834  (floor 0.5)
    three_quarter.png  0.847  (floor 0.7)
```

With per-view floors, only **16 of 419 cards** actually alarm (3.8%), and those
16 genuinely needed a redraw. Flat 0.70 produced 173 alarms — 157 of them false.

**A gauge that always alarms is not a gauge.** People become immune to it, which
is worse than having none.

You can also supply your own: `--floor '{"full":0.45}'` (or a dict in Python).
Omitting `--floor` keeps byte-for-byte compatibility with the flat default.

<details>
<summary>Why arcface is useless for full-body / back views</summary>

ArcFace is a **face recognition** model: detect a face box, then encode the face.
Back views and distant full-body shots contain no detectable face. It doesn't
score low — it has **no score to give**. This package raises an explicit error
(`no face detected in this image`) instead of returning a plausible-looking 0.2.
</details>

#### Batch runs: VRAM and speed

A gauge is a **side tool**; it has no business interfering with production. So it
does three things that only matter when you measure hundreds of images:

```bash
# Your GPU is busy with ComfyUI / a local model (typical on a 16G card) — don't fight it
$ dfg consistency --anchor a.png --views v/ --backend comfy \
    --clip ~/ComfyUI/models/clip_vision/CLIP-ViT-H-14-laion2B-s32B-b79K.safetensors \
    --device cpu
```

1. **`--device cpu` actually works.** ComfyUI expresses this through a global
   `cpu_state`; the package sets it. Without that, the flag is just a wish.
2. **Weights load once per process.** CLIP-ViT-H-14 is 3.9 GB. Re-`load()`ing on
   every `compare()` means reading 3.9 GB per card — 10× slower in practice.
3. **Per-image reclamation.** Without `inference_mode` and `soft_empty_cache`,
   memory accumulates and CUDA OOMs on roughly the 4th image.

<details>
<summary>Why the CI doesn't test real consistency scores</summary>

It needs 3.9 GB of weights and a working torch environment. This package's CI
tests only the model-free parts (math, liveness, argument reconciliation). The
consistency backend cases print `[SKIP]` and count as *skipped*, not *passed* —
pretending you tested something is worse than not testing it.
</details>

## Use it from Python

```python
from dfg import key_terms, coverage
from dfg.llm_health import statuses, pick_alive
from dfg.preflight import check_cmd
from dfg.consistency import similarity, compare

key_terms('midnight store, the lead is called "Lin Xiaoman"')   # ['Lin Xiaoman', 'midnight store']
coverage('rainy day, oil-paper umbrella', 'under a paper umbrella in the rain')

st = statuses('output/llm_call_log.jsonl')
pick_alive(['moonshot', 'qwen'], st)              # ['qwen']

check_cmd(['python', 'x.py', '--force']).ok       # True

similarity('a.png', 'b.png')                      # 0.8721
compare('anchor.png', ['v1.png', 'v2.png'])['verdict']   # 'consistent'
```

## Three things it deliberately won't do

1. **No auto-repair.** A dead channel doesn't get swapped for you; low adherence
   doesn't trigger a re-run. A regeneration can cost minutes and real money —
   that decision belongs to a human.
2. **No silent degradation.** Missing dependencies raise `BackendMissing` with an
   install hint. It never returns 0 — a fake score is far more dangerous than no
   score.
3. **No pretending to be universal.** Consistency scores carry their backend;
   adherence only does verbatim matching. The measurement basis is printed so
   others can re-check it. Hidden methodology is just "felt about right".

## Where it came from

These four gauges were extracted from DreamForge, a local AI production system
that is not open source. They grew out of real incidents, not design documents:

| gauge | the incident |
|---|---|
| `preflight` | command built with `--force`; script never registered it → `unrecognized arguments` killed a whole episode |
| `llm_health` | fallback hardcoded to a provider whose model was decommissioned → one wasted round-trip per failure |
| `adherence` | demo sample ignored the prompt entirely and emitted canned plot, with no UI indication at all |
| `consistency` | cross-view drift from 0.507 to 0.905 — invisible on any single image, obvious once the shots are cut together |

The point of extracting them isn't "we open-sourced a library". It's that
**nobody else should have to step on these same rakes.**

## Known limitations

- **CLI messages are Chinese.** The tool grew up inside a Chinese-language
  pipeline. `[PASS]`/`[FAIL]`, exit codes, and `--json` are language-neutral;
  an English message table is not written yet.
- `consistency` scores are only comparable within one backend (see above).
- `adherence` is verbatim matching. It cannot tell you a sentence is *good*,
  only that the elements are present.

## Boundaries

- No models, no character assets, no prompt libraries, no business rules.
  Changing your business never touches this package.
- It does not generate, concatenate, render, or publish.
- The `comfy` backend of `consistency` needs a ComfyUI runtime; it is not covered
  by CI.

## License

MIT. See [LICENSE](LICENSE).
